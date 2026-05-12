# Docker Ubuntu コンテナ 日本語入力セットアップ

## 環境
- ベースOS: Ubuntu (Jammy 22.04)
- GUI環境: X11/VNC (`$DISPLAY=:5`)
- 入力メソッド: fcitx5 + Mozc

---

## 0. まとめコマンド（初回セットアップ全体）

```bash
# 1. インストール
sudo apt update && sudo apt install -y \
  fcitx5 fcitx5-mozc fcitx5-config-qt dbus-x11 im-config

# 2. 環境変数
cat >> ~/.bashrc << 'EOF'
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export DefaultIMModule=fcitx
export DISPLAY=:5
EOF
source ~/.bashrc

# 3. 起動
dbus-launch fcitx5 -d &

# 4. 永続化（ログ非表示）
echo 'dbus-launch fcitx5 -d > /dev/null 2>&1 &' >> ~/.bashrc
```

その後 `fcitx5-configtool` でMozcを追加して完了。

---

## 1. GUI環境の確認

コンテナ内で以下を実行し、DISPLAYが設定されているか確認する。

```bash
echo $DISPLAY
# 出力例: 5  または :5
```

値が返ってくればX11/VNC環境が動いている。

---

## 2. パッケージインストール

### ❌ ミスの例（よくあるエラー）

```bash
# sudoをapt updateにしかつけていないと apt install でエラーになる
sudo apt update && apt install -y fcitx5 ...
# → E: Could not open lock file /var/lib/dpkg/lock-frontend (Permission denied)
```

### ✅ 正しいコマンド

```bash
sudo apt update && sudo apt install -y \
  fcitx5 \
  fcitx5-mozc \
  fcitx5-config-qt \
  dbus-x11 \
  im-config
```

---

## 3. 環境変数の設定

```bash
cat >> ~/.bashrc << 'EOF'
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export DefaultIMModule=fcitx
export DISPLAY=:5
EOF

source ~/.bashrc
```

### 確認方法

```bash
echo $GTK_IM_MODULE   # → fcitx
echo $QT_IM_MODULE    # → fcitx
echo $XMODIFIERS      # → @im=fcitx
```

---

## 4. fcitx5 の起動

```bash
dbus-launch fcitx5 -d
```

起動するとターミナルにログが流れてブロックされる。  
これは正常。**別のターミナルを開いて**次の手順へ進む。

### 起動確認方法

```bash
pgrep -a fcitx5
# 出力例:
# 19753 fcitx5 -d
# 19861 /usr/bin/fcitx5
```

---

## 5. Mozc の追加（GUI設定）

別ターミナルで設定ツールを開く。

```bash
fcitx5-configtool
```

### 手順

1. 右側の `Available Input Method` 欄に注目
2. **`Only Show Current Language` のチェックを外す**（これをしないとMozcが表示されない）
3. 検索欄に `mozc` と入力
4. **Mozc** を選択し、`←` ボタンで左側（Current）に追加
5. `Apply` → `OK`

---

## 6. 動作確認

### テキストエディタで確認

```bash
gedit &
```

`Ctrl + Space` で日本語↔英語の切替を確認。

### ブラウザ（Firefox）で確認

環境変数が引き継がれるよう、**ターミナルからFirefoxを起動する**。

```bash
pkill firefox   # 既存のFirefoxを閉じる
sleep 1
firefox &       # ターミナルから再起動
```

`Ctrl + Space` で日本語入力が切り替わればOK。

> ⚠️ デスクトップアイコンやランチャーからブラウザを起動すると  
> 環境変数が引き継がれず日本語入力できないことがある。  
> 必ずターミナルから起動すること。

---

## 7. 永続化（再起動後も自動起動）

```bash
echo 'dbus-launch fcitx5 -d > /dev/null 2>&1 &' >> ~/.bashrc
```

次回ターミナル起動時に自動でfcitx5が立ち上がる。  
`> /dev/null 2>&1` でログを捨てているのでターミナルが汚れない。

### ⚠️ すでに `dbus-launch fcitx5 -d &` を追加済みの場合

ログが垂れ流しになるので以下で修正する。

```bash
sed -i 's|dbus-launch fcitx5 -d &|dbus-launch fcitx5 -d > /dev/null 2>\&1 \&|' ~/.bashrc
source ~/.bashrc
```

確認：

```bash
grep fcitx5 ~/.bashrc
# → dbus-launch fcitx5 -d > /dev/null 2>&1 &
```

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|------|------|------|
| `Permission denied` (apt install) | sudoが抜けている | `sudo apt install` にする |
| fcitx5起動後にターミナルが固まる | ログ表示でブロック | 別ターミナルを開く |
| Mozcが候補に出ない | "Only Show Current Language" ON | チェックを外してから検索 |
| geditでは動くがブラウザで動かない | 環境変数未引き継ぎ | ターミナルからブラウザを起動 |
| `XDG_RUNTIME_DIR not set` エラー | Wayland未接続（警告） | 無視してOK（X11で動作する） |
| fcitx5を再起動したい | 設定が反映されない | `pkill fcitx5 && dbus-launch fcitx5 -d &` |

---

## Dockerfile で設定する場合

コンテナを作り直すたびに手動セットアップが不要になる。  
既存のDockerfileに以下を追記する。

```dockerfile
# 日本語入力（fcitx5 + Mozc）
RUN apt-get update && apt-get install -y \
    fcitx5 \
    fcitx5-mozc \
    fcitx5-config-qt \
    dbus-x11 \
    im-config \
  && rm -rf /var/lib/apt/lists/*

# 環境変数
ENV GTK_IM_MODULE=fcitx \
    QT_IM_MODULE=fcitx \
    XMODIFIERS=@im=fcitx \
    DefaultIMModule=fcitx \
    DISPLAY=:5

# fcitx5 自動起動を bashrc に追加
RUN echo 'dbus-launch fcitx5 -d &' >> /root/.bashrc
```

### ⚠️ 注意点

- `fcitx5-configtool` でのMozc追加はGUI操作のためDockerfileでは自動化できない。
- 初回コンテナ起動後に一度だけ手動で `fcitx5-configtool` を実行してMozcを追加する。
- Mozc設定は `~/.config/fcitx5/` に保存されるので、volumeマウントしておくと次回以降も引き継げる。

```yaml
# docker-compose.yml の場合
volumes:
  - ./fcitx5-config:/root/.config/fcitx5
```