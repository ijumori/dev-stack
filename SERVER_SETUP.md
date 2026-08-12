# HOME-SERVER セットアップ記録（2026-08-09）

## 背景・根本原因

Mac から `ssh home-server` で WSL Ubuntu に入れない問題が発生した。
原因は、WSL のディストリビューション登録が Windows ユーザーごとに独立していること
（`HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Lxss` は per-user）。
Ubuntu-24.04 と Docker Desktop は `ijumo` アカウントに登録されており、
SSH ログイン先だった `server`（Windowsアカウント）には何も登録されていなかった。

当初は `server` アカウント側に WSL を再インポートする計画だったが、
デスクトップログインが必要になる手前で方針転換し、
**`server` Windowsアカウントは使わず、既存の `ijumo` に紐づく Ubuntu-24.04 をそのまま使う**構成に確定した。
Ubuntu内のLinuxユーザー名がたまたま `server` であるため、`ssh server@...` という見た目は変わらない。

## 確定構成

```
Mac ──SSH:2222 (Tailscale)──→ Ubuntu sshd（ijumoアカウント配下のUbuntu-24.04） ──→ Docker Engine (systemd)
```

- マシン: Windows 11 25H2 (Build 26200)（引継書の「Windows 10 Home」表記は誤り）
- WSLディストリビューション: `Ubuntu-24.04`（`ijumo` Windowsアカウント配下、Linux内ユーザー名は `server`）
- SSH: Ubuntu内 sshd がポート2222で直接待ち受け（鍵認証のみ）。Windows OpenSSH (:22) は予備経路として残置。
- Docker: Docker Desktop は撤去済み。Ubuntu内に Docker Engine (docker-ce) を導入し、systemdで自動起動。
- 24時間稼働: タスクスケジューラ `WSL-Ubuntu-Keepalive`（`HOME-SERVER\ijumo`、`LogonType: S4U`で起動時に`wsl sleep infinity`を実行しWSL VMを常駐させる。詳細は下記「起動時自動化のバグ」参照）

## 接続方法

```
ssh home-server   # Mac側 ~/.ssh/config の Host home-server ブロック（Port 2222, User server, StrictHostKeyChecking no は付けない）
```

## 完了条件チェックリスト（原計画§5 対応状況・2026-08-09時点）

| # | 項目 | 状態 |
|---|---|---|
| 1 | Macから1コマンドでUbuntuに入れる | 済 |
| 2 | パスワード不要（鍵認証） | 済 |
| 3 | scpが動く | 済 |
| 4 | DockerがUbuntuネイティブ | 済 |
| 5 | Docker自動起動 | 済 |
| 6 | sshd自動起動 | 済 |
| 7 | Windows再起動後・未ログインで接続可 | 修正済みだが**次回の自然な再起動で再確認要**（下記参照） |
| 8 | DrvFsでchmod可能（`options="metadata,..."`） | 済 |
| 9 | pnpmがコンテナ再作成後も残る | 済 |
| 10 | GitHub接続維持（`ssh -T git@github.com`） | 済 |
| 11 | Mac configにStrictHostKeyChecking無し | 済 |
| 12 | バックアップ保全（`D:\Server\backup\ubuntu-24.04-20260809.tar`） | 済 |

## mirrored networking の外部着信トラブルと解決（重要）

STEP6（ファイアウォール・疎通確認）で、Macからポート2222への接続が長時間タイムアウトする問題が発生した。
原因は1つではなく、複数の独立した問題が重なっていた。

### 原因1: Tailscaleの所有権が`server`アカウントにロックされていた

`tailscale status` が `ijumo` から実行すると
`Tailscale already in use by HOME-SERVER\server` エラーで失敗していた。
`server`アカウントのTailscale GUIセッション（トレイアイコン、別セッションID）が
接続の所有権を握ったまま止まっており、結果としてTailscaleアダプタが
APIPAアドレス（169.254.x.x）のままでtailnetに接続できていなかった。
`ijumo`・`server`のアカウント分離が、WSLだけでなくTailscaleにも影響していた形。

**対処**: `Tailscale`のWindowsサービス（LocalSystem実行）を再起動すると所有権ロックが解消され、
`ijumo`側のトレイアイコンが正しく接続を引き継いだ。

```powershell
Restart-Service -Name "Tailscale" -Force   # 管理者権限必要
```

### 原因2: mirrored networkingの外部着信に必要な3つの設定

mirrored networkingモード（`.wslconfig`の`networkingMode=mirrored`）では、
WindowsとWSLがネットワークインターフェース（IPアドレス）を共有するが、
**外部から着信したTCP接続をWSL内のリスニングソケットまで転送するには、
以下3点すべての設定が必要**だった。

1. **Windows Defender Firewall** のインバウンドルール（ポート単位で許可）
   ```powershell
   New-NetFirewallRule -DisplayName "WSL Ubuntu SSH 2222" -Direction Inbound -Protocol TCP -LocalPort 2222 -Action Allow -Profile Any
   ```
2. **Hyper-Vファイアウォール**（VM単位の既定動作。既定は`NotConfigured`＝実質ブロック）
   ```powershell
   Get-NetFirewallHyperVVMCreator   # WSLのVMCreatorIdを取得（例: {40E0AC32-46A5-438A-A0B2-2B479E8F2E90}）
   Get-NetFirewallHyperVVMSetting -All | Where-Object { $_.VMCreatorId -eq "{...}" } |
       Set-NetFirewallHyperVVMSetting -DefaultInboundAction Allow
   ```
3. **`.wslconfig`の`firewall=true`**（mirrored networkingの着信をWindows Firewallと統合するための明示設定）
   ```ini
   [wsl2]
   networkingMode=mirrored
   firewall=true
   ```

### 原因3（本命）: `wsl --shutdown`だけでは反映されず、Windows自体の再起動が必要だった

上記3点をすべて設定しても、`wsl --shutdown`でWSLだけ再起動した状態では、
Windows自身が`localhost:2222`にすら到達できない状態が続いた
（Hyper-VのvSwitch/VFPレイヤーの設定変更が反映されないため、と推測）。

**Windowsを完全に再起動したところ、外部（Mac）からの接続が成功した。**
ただし再起動後もWindows自身が自分の外部IP（LAN IP・Tailscale IP）宛てに
ループバック接続することはできない（`localhost`は成功するが外部IPは失敗する）。
これはmirrored networkingの既知の制約（hairpin非対応）であり、実害はない
（外部の実クライアントからは正常に到達する）。

**教訓**: mirrored networking関連の設定変更を検証する際、
**Windows自身からのテスト（`Test-NetConnection`、`netstat`）は信頼できない。**
必ず外部の実クライアント（Mac等）から接続テストすること。

## 起動時自動化のバグ（STEP8）

Windows再起動後の検証で、タスクスケジューラ`WSL-Ubuntu-Keepalive`が
起動時に自動実行されていないことが判明した。原因は登録時に`LogonType`を
明示しなかったため既定の`Interactive`になっており、**`ijumo`が対話ログオンするまで
実行が保留される**設定になっていたこと。これでは無人稼働（P-1）の目的を果たせない。

**対処**: タスクを`LogonType: S4U`で再登録した（パスワード不要かつログオン状態に依存しない）。

```powershell
$principal = New-ScheduledTaskPrincipal -UserId "HOME-SERVER\ijumo" -LogonType S4U -RunLevel Limited
# Unregister-ScheduledTask → Register-ScheduledTask -Principal $principal で再登録
```

修正後、正しい設定になっていることは確認済みだが、**実際に「再起動→ログインせずに自動起動」まで
一気通貫で確認できたのは次回Windowsが自然に再起動したタイミングになる**（今回は動作確認のため手動起動した）。

## その他の既知の注意点

- mirrored networking のため、WindowsとUbuntu内で同じポートを使うサービスは衝突する。
  実例: Windows側にネイティブの `postgres.exe` が既にポート5432で稼働しており、
  dev-stackのpostgresコンテナと衝突したため、コンテナ側を `5433:5432` に変更した。
- `wsl -u server -e bash -c "sudo ..."` は非対話実行だとパスワード待ちでハングする。
  root操作は `wsl -u root -e ...` を直接使うこと。
- `ijumo` はAdministrators所属だが、通常のPowerShellセッションは非昇格。
  ファイアウォールルールやタスクスケジューラ登録など管理者権限が必要な操作は
  `Start-Process -Verb RunAs` でUAC昇格が必要（画面上での「はい」クリックが必要）。
  多行スクリプトは `-Command` へのインライン文字列渡しだと引用符のネストで失敗しやすいため、
  一時`.ps1`ファイルに書いて`-File`で渡す方が確実。

## dev-stack 構成

`docker-compose.yml` の重複（`/mnt/d/Server/docker/dev-stack/` と本リポジトリ）を解消し、
本リポジトリ（ext4上、`~/projects/dev-stack/`）を唯一の正とした。
旧ディレクトリは `/mnt/d/Server/backup/deprecated/dev-stack-20260809/` に退避済み（削除はしていない）。

pnpm は `node/Dockerfile` で `corepack` により永続化。コンテナ再作成後も残ることを確認済み。
