# dev-stack

HOME-SERVER (WSL Ubuntu-24.04) 上で動かす個人開発用スタック。PostgreSQL / Redis / Node.js を Docker Compose でまとめて起動する。

## 起動

```bash
docker compose up -d
```

| サービス | ポート | 備考 |
|---|---|---|
| postgres | 5433 → 5432 | Windows側ネイティブpostgresと衝突するため5433に変更済み |
| redis | 6379 | |
| node | 3000, 5173 | pnpmはDockerfileで永続化（corepack） |

## セットアップの経緯・サーバー構成

このマシン（HOME-SERVER）自体のSSH/Docker構成、mirrored networkingの注意点、
既知のポート衝突などは [SERVER_SETUP.md](./SERVER_SETUP.md) を参照。
