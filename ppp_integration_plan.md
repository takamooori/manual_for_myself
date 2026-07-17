# UM982 PPP活用計画 — フェーズ別引き継ぎ資料

orange2026A の GNSS パイプラインに PPP 品質情報を統合する計画。
各フェーズは独立したチャットで進める前提でまとめている。

作成日: 2026-07-15

---

## 全体像（全フェーズ共通の前提知識）

### 現状パイプライン（5ノード構成）

```
UM982 (115200bps, /dev/sensors/GNSS_UM982)
  └→ get_gnss_data_ttyUSB.py     … 1Hzタイマーで open→GPHDT→GGA探索→close
      └→ gnss_odom_publisher_ttyUSB.py … 緯度経度→XY変換、/odom/UM982 配信
          └→ odom_combination.py       … wheel+IMU odom と合成、/odom/combine
              └→ ekf_myself_gps.py     … EKF（try_navigation側）
```

### PPPに関する重要な理解

- **PPP補正は受信機内部で適用される**。`CONFIG PPP ENABLE AUTO` 設定済みなので、
  収束すれば GGA の緯度経度は既にPPP補正済みの値になる
- **PPPNAVA の出力ON/OFFは測位品質に影響しない**。品質情報（σ・収束状態）を
  見えるようにするだけの「観測窓」
- UM982 は **B2b-PPP / Galileo E6-HAS / QZSS L6E (MADOCA)** に対応。
  日本国内ではみちびきのMADOCAが利用可能。補正は衛星放送で直接届く
  （ネット・基地局不要）。SIGNALGROUP 3 6 設定済みでL6/E6も受信対象
- **GGAをPPPに置き換える必要はない**。位置はGGAのまま、品質情報を追加する

### 既知バグ（関連するもの）

- Bug①: EKFサイクルをまたいだGPS再利用（GpsXYがリセットされない）
- Bug②: covariance[0] に衛星数を格納（x位置分散フィールドの誤用）
- Bug③: determination_of_R() のハードコードXY矩形（修論の研究対象）

---

## フェーズ① PPPNAVA出力確認 【コード変更なし】

### ゴール
`#PPPNAVA` が実際に出力されることを確認し、フィールド構造を確定する。

### 手順

```bash
# 出力有効化
sudo stty -F /dev/sensors/GNSS_UM982 115200 raw -echo
echo -e "PPPNAVA 1\r"  | sudo tee /dev/sensors/GNSS_UM982 > /dev/null
sleep 0.5
echo -e "SAVECONFIG\r" | sudo tee /dev/sensors/GNSS_UM982 > /dev/null

# ダンプ確認
sudo timeout 10 cat /dev/sensors/GNSS_UM982 > /tmp/ppp_check.log
grep "PPPNAVA" /tmp/ppp_check.log
```

### 確認ポイント
- `#PPPNAVA` 行が出るか（LOGLISTにも登録されているか）
- フィールド数・区切り・各フィールドの意味（生データからClaude と一緒に解読）
- **屋外で** PPP解の状態を確認（屋内では収束しない）
  - GGA fixtype の変化も同時に観察（PPP収束時に値が変わるか）
- 収束までの時間（PPP一般に10〜20分程度かかる。MADOCAの実力を実測）

### 完了条件
PPPNAVAの生データサンプルを取得し、パースに必要なフィールド定義が確定していること。

---

## フェーズ② パース追加＋新トピック配信 【今回のメイン】

### ゴール
`get_gnss_data_ttyUSB.py` にPPPNAVAの読み取りを追加し、PPP品質情報を
新トピックで配信する。位置系（GGA→/odom/UM982）は一切変更しない。

### 方針（確定済みの設計判断）
- **新ノードは作らない**。既存ノードの読み取りループに追加する
  （別ノードで同じシリアルポートを開くと行の奪い合いが起き、
  GGA欠損が再発するため）
- 1Hz・open/close方式は現状維持（最小変更）
- GGA探索と同様、`find(b"PPPNAVA")` ベースで探索
  （PPPNAVAが未出力・未収束でも既存機能に影響しないようにする）

### 未確定の設計判断（このフェーズの最初に決める）
- 出力トピックの型:
  - 案1: `sensor_msgs/NavSatFix`（標準。position_covarianceにσ格納。推奨）
  - 案2: 独自msg `my_msgs/PppNav`（収束状態など全フィールド保持可能）
  - フェーズ①のダンプ結果を見てから決定するのが安全

### 対象ファイル
- `orange2025/orange_gnss/orange_gnss/get_gnss_data_ttyUSB.py`（読み取り追加）
- `orange2025/orange_gnss/orange_gnss/gnss_odom_publisher_ttyUSB.py`（配信追加）
- 独自msgの場合: `navigation_control/my_msgs/`（msg定義追加）

### 注意点
- タイムアウトループ内でGGAとPPPNAVAを同時に探す構造にする
  （GGA用2秒ループを2回まわすと1Hz周期を圧迫する）
- PPPNAVA未受信でもGGA処理は正常継続すること（フォールバック維持）
- fixtype=0時の安全設計（GPS odom非配信→EKFデッドレコニング継続）を壊さない

### 完了条件
`ros2 topic echo` でPPP品質トピックが確認でき、既存の /odom/UM982 が
従来どおり配信されていること。

---

## フェーズ③ EKF統合（σベースの連続R制御） 【修論と直結】

### ゴール
PPP品質（σ）をEKFの観測ノイズ共分散Rに反映し、
ハードコード矩形（Bug③）に依存しない連続的なR制御を実現する。

### 修論との関係
- 現EKF: fixtype≠0の二値判定 + gps_rr_flag（ハードコード矩形）でR切替
- 修論の本命: **LiDAR空遮蔽率ベースの連続R制御**
- このフェーズで作る **σベースR制御は比較手法**としてそのまま評価実験に使える
  （遮蔽率 vs 受信機自己申告σ vs 従来の矩形、の3手法比較が可能になる）

### 対象ファイル
- `try_navigation/try_navigation/ekf_myself_gps.py`
  - `determination_of_R()` の改修
  - PPP品質トピックのsubscribe追加

### 前提・注意
- Bug②（covariance[0]=衛星数）にEKF側が依存している。
  フィールドの用途を変える場合は影響範囲の確認必須
- Bug①（GpsXYリセット）はこのフェーズか20Hz化の前までに修正したい
- GSTメッセージ（$GPGST、位置誤差統計）も同様のσ情報源として利用候補。
  PPPNAVAと比較検討する価値あり

### 完了条件
σに応じてRが連続的に変化することをrosbag再生等で確認。
遮蔽率ベースR制御との比較実験の土台ができていること。

---

## フェーズ④ 20Hz化 【独立した将来課題】

### ゴール
GNSSデータを20Hzで取得・配信できるようにする。

### 方針
- **新ノードは作らない**。`get_gnss_data_ttyUSB.py` の内部構造を書き換える:
  - 現状: 1Hzタイマーごとに open→読む→close
  - 変更後: ポート開きっぱなし + 専用読み取りスレッドが常時読む
- GGA探索は時間タイムアウト（2秒）→ **行数上限ループ**に変更
  （1出力フレーム分の行数を上限に探索。周波数非依存で堅牢）
- ボーレート 115200 → **460800** に引き上げ
  - 概算: NMEA 1行≈80byte(800bit) × 6メッセージ × 20Hz ≈ 96kbps で
    115200bpsはほぼ飽和するため必須
  - 受信機側: `CONFIG COM2 460800`（要SAVECONFIG）
  - ROS側: baud パラメータ変更
- 受信機側レート変更: `GNGGA 0.05` / `GPHDT 0.05` 等（0.05=20Hz）

### 前提
- **Bug①（GpsXYリセット）修正が必須**。20HzだとEKFサイクルをまたいだ
  GPS再利用の影響が拡大するため
- udevルール登録（/dev/sensors/GNSS_UM982 固定）も事前に済ませる

### 完了条件
/odom/UM982 が20Hzで安定配信され、長時間走行でCPU・取りこぼしに問題がないこと。

---

## フェーズ間の依存関係

```
① 出力確認 ──→ ② パース+配信 ──→ ③ EKF統合（修論比較手法）
                                   
Bug①修正 ─────────────────────→ ④ 20Hz化（③とは独立に着手可）
```

- ①→②→③ は直列
- ④ は ②完了後ならいつでも着手可（ただしBug①修正が前提）
- ③と④は互いに独立
