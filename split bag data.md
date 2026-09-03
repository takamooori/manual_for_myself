# rosbag2 大容量bagの分割手順（ROS2 Humble / sqlite3）

大きすぎて rqt_bag で扱いづらい bag を一定サイズごとに分割し、
それぞれ独立した bag として開けるようにする手順。

bag の置き場は `~/ros2_ws/src/bag/` を前提とする。

> **注意**
> 変換は元データを残したままコピーを作るため、**元と同じ容量が追加で消費される**。
> 20GB の bag なら合計 40GB になる。

---

## 全体の流れ

```
元bag ──[convert]──> 一時分割bag ──[個別ディレクトリ化 + reindex]──> 各part
```

`convert` の出力は「1つの bag 内に複数の .db3」という形式なので、
rqt_bag で個別に開くには分割後に **各ファイルを別ディレクトリへ移して reindex** する。

---

## 共通の変数設定

以降のコマンドはすべてこの3行を先に実行しておく。

```bash
cd ~/ros2_ws/src/bag

SRC="対象のbagディレクトリ名"     # ← ここだけ書き換える
OUT="${SRC}_parts"
TMP="${SRC}_tmpsplit"
```

---

## STEP 0. metadata.yaml の確認（無い場合のみ）

`convert` はディレクトリ単位で入力を受けるため `metadata.yaml` が必須。

```bash
ls "$SRC/metadata.yaml"
```

見つからない場合は生成する。

```bash
ros2 bag reindex "$SRC" -s sqlite3
```

---

## STEP 1. convert.yaml の作成

`EOF` をクォートしないことで `${TMP}` が展開され、bag 名に応じた YAML が自動生成される。

```bash
cat > "convert_${SRC}.yaml" << EOF
output_bags:
- uri: ./${TMP}
  storage_id: sqlite3
  all: true
  max_bagfile_size: 4294967296
EOF

cat "convert_${SRC}.yaml"     # 中身確認
```

**分割条件の切り替え**

```yaml
max_bagfile_size: 4294967296     # サイズ分割（バイト）4GiB
max_bagfile_duration: 300        # 時間分割（秒）
```

両方書いた場合は先に到達したほうで分割される。

---

## STEP 2. 分割の実行

```bash
rm -rf "$TMP" "$OUT"
ros2 bag convert -i "$SRC" -o "convert_${SRC}.yaml"
```

- 同一シリアライズ形式なら生コピーのため、サイズ相応の I/O 時間で済む
- 長尺 bag は `tmux` か `nohup` 推奨（中断すると出力が壊れる）
- 出力先が既に存在すると失敗するので、冒頭で `rm -rf` している

---

## STEP 3. 個別ディレクトリ化と reindex

分割された `.db3` を1つずつ独立した bag ディレクトリに仕立てる。

```bash
mkdir -p "$OUT"

for f in "$TMP"/${TMP}_*.db3; do
  n=$(basename "$f"); n="${n##*_}"; n="${n%.db3}"
  d="$OUT/${SRC}_${n}"
  mkdir -p "$d"
  mv "$f" "$d/${SRC}_${n}.db3"
  ros2 bag reindex "$d" -s sqlite3
done

rm -rf "$TMP"
ls "$OUT"
```

`reindex` が db3 内の `topics` テーブルからトピック定義を復元し、
各ディレクトリに `metadata.yaml` を生成する。これで単体の bag として成立する。

---

## 生成される構造

```
<SRC>/                     ← 元データ（変更されない）
<SRC>_parts/               ← 親ディレクトリ
├── <SRC>_0/
│   ├── <SRC>_0.db3
│   └── metadata.yaml
├── <SRC>_1/
├── <SRC>_2/
└── <SRC>_3/
convert_<SRC>.yaml         ← 自動生成されたYAML
```

---

## 確認と再生

```bash
ros2 bag info "$OUT/${SRC}_0"

source ~/ros2_ws/install/setup.bash
ros2 run rqt_bag rqt_bag "$OUT/${SRC}_0"
```

カスタムメッセージ（`livox_ros_driver2/msg/CustomMsg` など）を含む場合、
型解決のためワークスペースを source した端末で起動すること。

---

## 一括スクリプト

STEP 0〜3 をまとめたもの。`SRC` だけ書き換えて実行する。

```bash
#!/bin/bash
set -e

cd ~/ros2_ws/src/bag

SRC="対象のbagディレクトリ名"
OUT="${SRC}_parts"
TMP="${SRC}_tmpsplit"

[ -f "$SRC/metadata.yaml" ] || ros2 bag reindex "$SRC" -s sqlite3

cat > "convert_${SRC}.yaml" << EOF
output_bags:
- uri: ./${TMP}
  storage_id: sqlite3
  all: true
  max_bagfile_size: 4294967296
EOF

rm -rf "$TMP" "$OUT"
ros2 bag convert -i "$SRC" -o "convert_${SRC}.yaml"

mkdir -p "$OUT"
for f in "$TMP"/${TMP}_*.db3; do
  n=$(basename "$f"); n="${n##*_}"; n="${n%.db3}"
  d="$OUT/${SRC}_${n}"
  mkdir -p "$d"
  mv "$f" "$d/${SRC}_${n}.db3"
  ros2 bag reindex "$d" -s sqlite3
done
rm -rf "$TMP"

ls "$OUT"
```

---

## 軽量化スクリプト（分割の代わり／併用）

容量の大半を占めるのは点群トピック（`/livox/lidar`）。
IMU・GNSS・odom の波形確認が目的なら、トピックを絞るほうが効果が大きく、分割自体が不要になることが多い。

```bash
cd ~/ros2_ws/src/bag
SRC="対象のbagディレクトリ名"

cat > "convert_${SRC}_light.yaml" << EOF
output_bags:
- uri: ./${SRC}_light
  storage_id: sqlite3
  topics: [/odom, /odom/UM982, /fix, /imu, /imu/spresense, /livox/imu, /movingbase/quat]
EOF

ros2 bag convert -i "$SRC" -o "convert_${SRC}_light.yaml"
```

出力は元の数%程度に収まる見込み。分割せずそのまま rqt_bag で開ける。
`topics:` は対象 bag の `ros2 bag info` を見て調整する。

---

## 全part の一括確認スクリプト

```bash
cd ~/ros2_ws/src/bag
SRC="対象のbagディレクトリ名"

for d in "${SRC}_parts"/*/; do
  echo "--- $d"
  ros2 bag info "$d" | grep -E "Bag size|Duration|Messages"
done
```

---

## つまずきやすい点

| 症状 | 原因と対処 |
|---|---|
| `bad file: convert.yaml` | YAML構文エラーではなく **ファイルが見つからない**。実行ディレクトリからの相対パスになっているか確認 |
| convert が失敗する | 出力先ディレクトリが既に存在している。先に `rm -rf` する |
| 出力が4GiBを少し超える | 書き込み後にサイズ判定する仕様のため。正常 |
| 途中で中断した | 出力が壊れる。長尺 bag は `tmux` / `nohup` で実行する |
| rqt_bag で型エラー | ワークスペースを source していない |

### 用語メモ：`uri`

出力先の **bagディレクトリのパス（＝名前）**。中身には影響しない。
ROS2 の bag は単一ファイルではなくディレクトリ単位で管理され、
ディレクトリ名がそのまま分割ファイルの接頭辞になる。

```
split_out/
├── metadata.yaml
├── split_out_0.db3       ← uri の名前が接頭辞になる
└── split_out_1.db3
```
