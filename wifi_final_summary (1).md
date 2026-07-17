# Wi-Fiトラブル 最終まとめ（2026/07/15〜16）

## 結論

**原因：未特定。**

底面カバーを開けて確認した際、一時的に「アンテナ線2本（Main/Aux）が外れている」と見立てたが、
これは誤りだった（コネクタは上から挿す構造で、抜けていたわけではない）。
内蔵Wi-Fiカード（Realtek 8852BE）が機能していないこと自体は確定しているが、
根本原因はハードウェア故障か別の要因か、引き続き不明。
**当面はUSB子機で運用**する方針。

---

## 環境

| 項目 | 内容 |
|---|---|
| PC | ASUS TUF Gaming A16 (FA617NS) |
| OS | Windows 11 **25H2**（7/15にアップグレード・Insider離脱済み） |
| Wi-Fiカード | AzureWave **AW-XB547NF** = Realtek 8852BE（M.2） |
| 発生 | 2026/07/13(月) 夕方から突然 |

---

## 症状

| 症状 | 備考 |
|---|---|
| SSIDは約20個見える | 受信自体は機能している |
| どのSSIDにも接続できない（オープンNW含む） | 送信/アソシエーションが失敗 |
| スマホのホットスポットを10cmに置いてもダメ | 電波強度の問題ではなさそう |
| Bluetoothは正常 | 同じカードのBT機能は生きている |
| 有線LANは正常 | 別チップ |
| 月曜夕方から突然 | きっかけ不明 |
| ドライバー・OS交換で直らない | ソフト要因ではなさそう |

### 主要ログ
```
イベントID 8002 / WLAN-AutoConfig
FailureReason : The driver disconnected while associating.
ReasonCode    : 229378
RSSI          : 255（測定不能値）
```

---

## 試して効果がなかったこと（＝すべて無関係と判明）

- Wi-Fi無効→有効 / 再起動 / ネットワークのリセット
- ドライバー4種（元の6001.15.155.1、ASUS最新V6001.15.160.100、ASUS旧V6001.15.155.1、25H2上で再インストール）
- `netsh int ip reset` / `winsock reset` / `advfirewall reset` / `flushdns` + 再起動
- **Insider Preview → 安定版25H2 へのインプレースアップグレード**
- MAC Randomization 無効化
- 詳細設定タブの確認（すべて正常値）

### 切り分けの決定打
手持ちの **BUFFALO WLI-UC-GN**（古いUSB子機）を挿したら学校のWi-Fiに接続成功
→ Windows側・ネットワーク側は完全にシロ、問題は内蔵カードに限定と確定。

### 底面カバーを開けての確認（結果：原因特定には至らず）
アンテナケーブル2本が外れているように見えたが、これは誤認だった（構造上、抜けていたわけではない）。
内蔵カード自体は機能していないが、根本原因は未特定のまま。

---

## 副産物・注意点

- **25H2アップグレードで「個人用ファイルのみ引き継ぐ」を選択** → アプリと設定は消えた（ファイルは残っている）
  - ※「個人用ファイルとアプリを引き継ぐ」はInsider→安定版のためグレーアウトで選べなかった
- **このPCは底面カバーを外すと光センサー（ALS）保護で電源が入らなくなる**
  - カバーを戻す → ACアダプター接続 → 電源オン、の順が必須
  - それでも入らない場合はハードリセット（AC抜き→電源ボタン40秒長押し→AC接続）
- **底面ネジが3本紛失・ねじ穴も緩んでいる**（要補充：ノートPC用ネジセット、M2.5×5〜8mm目安）
- 旧環境は `C:\Windows.old` に約10日間残っている（7/25頃まで）

---

## 内蔵Wi-Fiを直したくなったら（将来の選択肢）

原因が未特定のため、まず切り分けからやり直す必要がある。

| 案 | 内容 | 費用 |
|---|---|---|
| A | M.2 Wi-Fiカードごと交換（Intel AX210等）で切り分け | 3,000〜4,000円 |
| B | ASUS修理窓口に相談・診断依頼 | ー |
| C | 当面USB子機で運用し様子見 | 済み |

---

# 次チャット用｜① USB子機の選定

```
【USB Wi-Fi子機の選定相談】
■ 状況：ASUS TUF A16 (FA617NS) / Windows 11 25H2 の内蔵Wi-Fi (Realtek 8852BE) が
　アンテナ線の脱落で使えない。USB子機で代替したい
■ 候補：TP-Link Archer T2U Nano（AC600・11ac・デュアルバンド・3年保証・4.1★）
■ 却下した候補：BUFFALO WI-U2-150M/N（11n・150Mbps・2.4GHz専用）
　→ 手持ちのBUFFALO WLI-UC-GN（同世代・2.4GHz専用）が研究室Wi-Fiに繋がらなかったため
■ 確認したいこと：
　1. 研究室Wi-Fi（SSID: takedalab）の周波数帯と暗号化方式は？
　　 → WLI-UC-GNで繋がらなかった理由が「5GHzのみ」なら T2U Nano で解決
　　 　 「WPA3必須」なら T2U Nano でも解決しない
　2. 大学のWi-Fi（hosei-wifi / eduroam）はWPA2-Enterprise(802.1X)対応が必要
　3. Wi-Fi 6対応機（TP-Link Archer TX20U Nano 等、2,000〜3,000円台）の方が安全か
■ 調べ方：他の端末（スマホ等）で takedalab に接続 → 接続情報から
　周波数帯・セキュリティの種類を確認できる
```

---

# 次チャット用｜② 消えたアプリの復元

```
【25H2アップグレード後のアプリ復元】
■ 状況：Windows 11 25H2 へ「個人用ファイルのみ引き継ぐ」でアップグレードしたため、
　アプリと設定が全消去された。個人ファイルは残っている
■ デスクトップにあるバックアップ：
　- gitconfig_backup
　- vscode_settings.json
　- vscode_extensions.txt（15個）
　- installed_apps_32.txt / installed_apps_64.txt / installed_apps_store.txt（アプリ一覧）
■ 復元済み：Chrome
■ 課題：Office（Microsoft 365 Apps for enterprise）が、
　個人用も職場用もどちらも学校アカウントになってしまい Plusプランが使えない
■ 環境メモ：Docker/WSL/VirtualBox/Node.js/ローカルPythonは実質未使用
　（作業は基本リモートサーバーのコンテナ）
```

## 復元手順（推奨順）

### Step 1: Chrome ✅ 済み
Googleアカウントでサインイン → ブックマーク・パスワード・拡張機能が自動同期

### Step 2: Git
1. https://git-scm.com/ から再インストール
2. ```powershell
   Copy-Item "$env:USERPROFILE\Desktop\gitconfig_backup" -Destination "$env:USERPROFILE\.gitconfig"
   git config --global --list
   ```

### Step 3: VS Code
1. https://code.visualstudio.com/ から再インストール → 一度起動して終了
2. ```powershell
   Copy-Item "$env:USERPROFILE\Desktop\vscode_settings.json" -Destination "$env:APPDATA\Code\User\settings.json" -Force
   Get-Content "$env:USERPROFILE\Desktop\vscode_extensions.txt" | ForEach-Object { code --install-extension $_ }
   ```

**復元される拡張機能（15個）**
```
aliyaman.python-colorful-print / anthropic.claude-code /
ms-azuretools.vscode-containers / ms-ceintl.vscode-language-pack-ja /
ms-python.debugpy / ms-python.python / ms-python.vscode-pylance /
ms-python.vscode-python-envs / ms-toolsai.jupyter /
ms-toolsai.jupyter-keymap / ms-toolsai.jupyter-renderers /
ms-toolsai.vscode-jupyter-cell-tags / ms-toolsai.vscode-jupyter-slideshow /
ms-vscode-remote.remote-containers / pdconsec.vscode-print
```

### Step 4: Office / OneDrive ← 要相談
Microsoftアカウントでサインイン。ただし上記の「学校アカウント競合」問題あり。

**次チャットで見せるもの：**
- Word/Excel → ファイル → アカウント の画面
- 設定 → アカウント → 職場または学校にアクセスする の画面
- 「Plusプラン」が具体的に何を指すか（Microsoft 365 Personal？大学配布のA3/A5？）

### Step 5: MATLAB R2024a
学校ライセンス。MathWorksアカウントから再ダウンロード・インストール・認証。

### Step 6: その他（必要なものだけ）
| アプリ | 入手先 |
|---|---|
| 7-Zip | 公式サイト |
| PowerToys | Microsoft Store |
| LINE / Slack | Microsoft Store |
| miniconda | 公式サイト（VSCodeがmyenvironment環境を参照していた） |
| OpenVPN Connect | 公式サイト（プロファイルは再取得） |
| Bambu Studio | 公式サイト（作成データは残っている） |
| Claude Desktop | 公式サイト |
| Armoury Crate等ASUS系 | ASUS公式 / Store |

### 不要（未使用と確認済み）
Docker Desktop / WSL / VirtualBox / Node.js / iTunes / 秀丸 / XYZmaker / Riot Vanguard

---

## その他の残タスク

- [ ] Windows Update の一時停止を解除
- [ ] 「利用可能になったらすぐに最新の更新プログラムを入手する」を**オフ**（元々オンだった）
- [ ] 底面ネジの補充
- [ ] `C:\Windows.old` から必要なものがないか確認（7/25頃までに）
