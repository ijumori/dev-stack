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
- 24時間稼働: タスクスケジューラ `WSL-Ubuntu-Keepalive`（`HOME-SERVER\ijumo` で起動時に `wsl sleep infinity` を実行し、WSL VMを常駐させる）

## 接続方法

```
ssh home-server   # Mac側 ~/.ssh/config の Host home-server ブロック（Port 2222, User server）
```

## 既知の注意点

- `.wslconfig` は `C:\Users\ijumo\.wslconfig` に配置（ユーザーごとの設定のため）。`networkingMode=mirrored` によりWindowsとWSLがネットワークインターフェースを共有する。
- mirrored networking のため、WindowsとUbuntu内で同じポートを使うサービスは衝突する。
  実例: Windows側にネイティブの `postgres.exe` が既にポート5432で稼働しており、
  dev-stackのpostgresコンテナと衝突したため、コンテナ側を `5433:5432` に変更した。
- `wsl -u server -e bash -c "sudo ..."` は非対話実行だとパスワード待ちでハングする。
  root操作は `wsl -u root -e ...` を直接使うこと。
- `ijumo` はAdministrators所属だが、通常のPowerShellセッションは非昇格。
  ファイアウォールルールやタスクスケジューラ登録など管理者権限が必要な操作は
  `Start-Process -Verb RunAs` でUAC昇格が必要（画面上での「はい」クリックが必要）。

## dev-stack 構成

`docker-compose.yml` の重複（`/mnt/d/Server/docker/dev-stack/` と本リポジトリ）を解消し、
本リポジトリ（ext4上、`~/projects/dev-stack/`）を唯一の正とした。
旧ディレクトリは `/mnt/d/Server/backup/deprecated/dev-stack-20260809/` に退避済み（削除はしていない）。

pnpm は `node/Dockerfile` で `corepack` により永続化。コンテナ再作成後も残ることを確認済み。
