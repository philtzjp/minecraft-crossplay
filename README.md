# minecraft-crossplay

Java Edition と Bedrock Edition のどちらからも入れる Minecraft サーバーを、VPS 1台で動かすための Docker Compose 構成。

Paper に [Geyser](https://geysermc.org/) と [Floodgate](https://wiki.geysermc.org/floodgate/) を載せているので、Bedrock 版のプレイヤーは Java 版のアカウントを持っていなくても参加できる。スマートフォンや Switch からでも入れる。

| サービス | 役割 |
|---|---|
| mc | Paper サーバー本体（[itzg/minecraft-server](https://github.com/itzg/docker-minecraft-server)） |
| backups | 定期バックアップ。既定は4時間ごと、7日保持 |
| restore-backup | バックアップからの復元。手動で実行する |
| rcon-web | ブラウザからコマンドを送る管理画面。localhost にだけ開く |

## 必要なもの

- メモリ 2GB 以上の VPS（4GB あると余裕がある）
- Docker と Docker Compose
- 開放できるポート: TCP 25565（Java 版）、UDP 19132（Bedrock 版）

## VPS の準備

Ubuntu 24.04 を例にする。

```bash
sudo apt update && sudo apt install -y docker.io docker-compose-v2 git
sudo usermod -aG docker "$USER"   # 入れたら再ログインする
```

ファイアウォールを開ける。

```bash
sudo ufw allow 22/tcp        # SSH。先に許可しないと締め出される
sudo ufw allow 25565/tcp     # Java 版
sudo ufw allow 19132/udp     # Bedrock 版
sudo ufw enable
```

**VPS によっては、これとは別に管理画面側のファイアウォールがある。** XServer VPS のパケットフィルターや、Oracle Cloud のセキュリティリストなどがそれにあたる。OS 側だけ開けても通らないので、両方を開ける。

## 起動

```bash
git clone https://github.com/philtzjp/minecraft-crossplay.git
cd minecraft-crossplay
cp .env.example .env
```

`.env` の `RCON_PASSWORD` と `RWA_PASSWORD` を自分の値にする。

```bash
docker compose up -d
docker compose logs -f mc     # 起動の様子を見る。Ctrl-C で抜けても止まらない
```

初回はサーバー本体とプラグインの取得、ワールドの生成があるので数分かかる。

## 接続先

| 版 | 入力する内容 |
|---|---|
| Java 版 | `<VPS の IP かドメイン>`（ポート 25565） |
| Bedrock 版 | サーバー `<VPS の IP かドメイン>`、ポート `19132` |

ドメインを使うなら、A レコードを VPS の IP に向ける。Cloudflare を使っている場合は、**プロキシをオフ（DNS only）にする。** プロキシは HTTP と HTTPS 向けで、Minecraft の通信は通らない。

Java 版でポート番号の入力を省きたいなら、SRV レコードを足す。

| 種類 | 名前 | 値 |
|---|---|---|
| SRV | `_minecraft._tcp.<サブドメイン>` | 優先度 0、重み 5、ポート 25565、ターゲットはサーバーのホスト名 |

Bedrock 版は SRV を見ないので、ポートの入力は必要なまま。

## 操作

```bash
docker compose logs -f mc                        # ログを見る
docker compose exec mc rcon-cli                  # サーバーのコンソールに入る
docker compose exec mc rcon-cli list             # コマンドを1つ送る
docker compose stop -t 120                       # ワールドを保存して止める
docker compose up -d                             # 起動する
docker compose exec backups backup now           # 今すぐバックアップを取る
docker compose --profile restore run --rm restore-backup   # バックアップから戻す
```

### 管理画面

`rcon-web` は localhost にだけ開いている。手元のブラウザから見るときは、SSH でポートを転送する。

```bash
ssh -L 4326:127.0.0.1:4326 -L 4327:127.0.0.1:4327 <ユーザー>@<VPS の IP>
```

転送したまま `http://localhost:4326` を開く。

## 設定

設定は `.env` に集めてある。項目は [.env.example](.env.example) を参照。変更したら `docker compose up -d` で反映する。

ワールドデータとサーバーの設定は `data/` に、バックアップは `backups/` にできる。どちらも git では追跡しない。

## サーバーのバージョンを上げる

`MC_VERSION` は固定してある。最新に追従させると、Geyser がまだ対応していない版へ上がったときに Bedrock 版だけ接続できなくなるため。

上げるときは、先に次の2つを確かめる。

1. Geyser が上げたい版に対応していること。`docker compose exec mc rcon-cli "geyser version"` が対応範囲を表示する。
2. Paper にその版の安定版ビルドがあること。[PaperMC のダウンロード](https://papermc.io/downloads/paper)で、beta や alpha しかない版は避ける。

どちらも満たしたら `.env` の `MC_VERSION` を書き換えて `docker compose up -d` する。上げたあとはクライアント側の版も合わせる。

## GitHub Workflow から配置する

[deploy](.github/workflows/deploy.yml) は、main への変更を VPS へ rsync で転送し、コンテナを入れ替える。手動で実行することもできる。`data/` と `backups/` は転送からも削除からも外しているので、ワールドは消えない。

必要な secrets。

| 名前 | 中身 |
|---|---|
| `SSH_HOST` | VPS のホスト名か IP |
| `SSH_USER` | ssh で入るユーザー名 |
| `SSH_KEY` | 秘密鍵。対応する公開鍵を VPS の `~/.ssh/authorized_keys` に置く |
| `SSH_KNOWN_HOSTS` | `ssh-keyscan -p 22 <VPS のホスト名>` の出力 |
| `RCON_PASSWORD` | RCON のパスワード |
| `RWA_PASSWORD` | 管理画面のパスワード |

variables は設定したものだけ `.env` に書き込まれ、残りは compose の既定値になる。

| 名前 | 既定値 |
|---|---|
| `MC_VERSION` | compose の指定が必須のため、ここでの設定を推奨 |
| `MC_MEMORY` | `4G` |
| `MOTD` | `A Minecraft Server` |
| `DIFFICULTY` | `normal` |
| `VIEW_DISTANCE` | `10` |
| `MAX_PLAYERS` | `20` |
| `RWA_USERNAME` | `admin` |
| `SSH_PORT` | `22` |
| `REMOTE_DIR` | `/opt/minecraft-crossplay` |

## 困ったとき

**Bedrock 版だけ入れない。** UDP 19132 が開いているか確かめる。OS 側と、VPS の管理画面側の両方が必要になることが多い。

**Java 版で「サーバーのバージョンが違う」と出る。** `MC_VERSION` とクライアントの版を合わせる。サーバーの版は `docker compose exec mc rcon-cli version` で確認できる。

**重い。** `VIEW_DISTANCE` を下げる。10 を 8 にするだけでも負荷が目に見えて下がる。メモリに余裕があれば `MC_MEMORY` を増やす。

## ライセンス

MIT。[LICENSE](LICENSE) を参照。

Minecraft は Mojang Studios の商標で、このリポジトリは Mojang Studios とも Microsoft とも関係がない。
