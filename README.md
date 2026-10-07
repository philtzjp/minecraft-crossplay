# minecraft-crossplay

Java 版と Bedrock 版のプレイヤーが同じワールドで遊べる Minecraft サーバーを、VPS 1台で動かすための Docker Compose 構成。

公式のサーバーはどちらか一方しか選べず、Realms も版をまたげない。このリポジトリは Paper に [Geyser](https://geysermc.org/) と [Floodgate](https://wiki.geysermc.org/floodgate/) を載せて、その壁をなくす。Bedrock 版のプレイヤーは Java 版のアカウントを買わなくてよく、スマートフォンや Switch からそのまま入れる。

[![check](https://github.com/philtzjp/minecraft-crossplay/actions/workflows/check.yml/badge.svg)](https://github.com/philtzjp/minecraft-crossplay/actions/workflows/check.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> [!NOTE]
> Paper のバージョンは `.env` で固定する。最新に追従させると、Geyser がまだ対応していない版へ上がったときに Bedrock 版だけ入れなくなる。[上げ方](#サーバーのバージョンを上げる)を参照。

## 始める

メモリ 2GB 以上の VPS に Docker を入れて、次を実行する。

```sh
git clone https://github.com/philtzjp/minecraft-crossplay.git
cd minecraft-crossplay
cp .env.example .env   # RCON_PASSWORD と RWA_PASSWORD を自分の値にする
docker compose up -d
```

初回はサーバー本体とプラグインの取得、ワールドの生成で数分かかる。`docker compose logs -f mc` に `Done` と出れば起動している。

ポートを開けるのを忘れずに。TCP 25565 と UDP 19132 の両方が要る。

```sh
sudo ufw allow 22/tcp && sudo ufw allow 25565/tcp && sudo ufw allow 19132/udp && sudo ufw enable
```

> [!IMPORTANT]
> VPS によっては、OS のファイアウォールとは別に管理画面側のパケットフィルターがある。XServer VPS や Oracle Cloud がこれにあたる。片方だけ開けても通らない。

## つなぐ

| 版 | 入力する内容 |
| --- | --- |
| Java 版 | `<VPS の IP かドメイン>`（ポート 25565） |
| Bedrock 版 | サーバー `<VPS の IP かドメイン>`、ポート `19132` |

入れたら試すこと。

- Bedrock 版から入ったプレイヤーの名前が `.` で始まっていれば、Floodgate が効いている
- `docker compose exec mc rcon-cli list` で、両方の版のプレイヤーが同じ一覧に出る
- `docker compose exec mc rcon-cli "geyser version"` で、対応する Java と Bedrock の版が分かる

ドメインを使うなら、A レコードを VPS の IP に向ける。Cloudflare なら**プロキシをオフ（DNS only）にする。** プロキシは HTTP と HTTPS 向けで、Minecraft の通信は通らない。

Java 版でポート番号の入力を省きたいなら、SRV レコード `_minecraft._tcp.<サブドメイン>` を優先度 0、重み 5、ポート 25565 で足す。Bedrock 版は SRV を見ないので、ポートの入力は必要なまま。

## 向いていること、向いていないこと

| | |
| --- | --- |
| 向いている | 友人と遊ぶ数人から数十人のサーバー。Java 版と Bedrock 版が混ざる集まり。VPS を自分で借りて、設定を git で管理したい場合 |
| 向いていない | Forge や Fabric の MOD を入れたい場合（Paper のプラグインのみ）。数百人規模。自宅の回線で公開したい場合（ポート開放できる環境が要る） |

Bedrock 版から見ると、一部の操作感は Java 版と異なる。看板の編集やインベントリの細かい挙動など、Geyser が埋めきれない差は残る。[Geyser の既知の問題](https://github.com/GeyserMC/Geyser/issues)を参照。

## 構成

| サービス | 役割 |
| --- | --- |
| mc | Paper サーバー本体（[itzg/minecraft-server](https://github.com/itzg/docker-minecraft-server)）。Geyser と Floodgate は起動時に取得する |
| backups | 定期バックアップ。既定は4時間ごと、7日保持 |
| restore-backup | バックアップからの復元。`restore` プロファイルで手動実行する |
| rcon-web | ブラウザからコマンドを送る管理画面。localhost にだけ開く |

| ファイル | 内容 |
| --- | --- |
| [docker-compose.yml](docker-compose.yml) | サービスの定義 |
| [.env.example](.env.example) | 設定の一覧。`.env` にコピーして使う |
| [.github/workflows/deploy.yml](.github/workflows/deploy.yml) | main の変更を VPS へ配置する |
| `data/` | ワールドとサーバーの設定。git では追跡しない |
| `backups/` | バックアップの保存先。git では追跡しない |

## 操作

```sh
docker compose logs -f mc                                  # ログを見る
docker compose exec mc rcon-cli                            # サーバーのコンソールに入る
docker compose stop -t 120                                 # ワールドを保存して止める
docker compose exec backups backup now                     # 今すぐバックアップを取る
docker compose --profile restore run --rm restore-backup   # バックアップから戻す
```

設定は `.env` に集めてある。変えたら `docker compose up -d` で反映する。

### 管理画面

`rcon-web` は localhost にだけ開いている。手元のブラウザから見るときは SSH でポートを転送して、`http://localhost:4326` を開く。

```sh
ssh -L 4326:127.0.0.1:4326 -L 4327:127.0.0.1:4327 <ユーザー>@<VPS の IP>
```

## サーバーのバージョンを上げる

上げる前に次の2つを確かめる。

1. Geyser が上げたい版に対応していること。`docker compose exec mc rcon-cli "geyser version"` が対応範囲を表示する
2. Paper にその版の安定版ビルドがあること。[PaperMC のダウンロード](https://papermc.io/downloads/paper)で、beta や alpha しかない版は避ける

どちらも満たしたら `.env` の `MC_VERSION` を書き換えて `docker compose up -d` する。上げたあとはクライアント側の版も合わせる。

## GitHub Workflow から配置する

[deploy](.github/workflows/deploy.yml) は、main への変更を VPS へ rsync で転送し、コンテナを入れ替える。手動でも実行できる。`data/` と `backups/` は転送からも削除からも外しているので、ワールドは消えない。

必要な secrets。

| 名前 | 中身 |
| --- | --- |
| `SSH_HOST` | VPS のホスト名か IP |
| `SSH_USER` | ssh で入るユーザー名 |
| `SSH_KEY` | 秘密鍵。対応する公開鍵を VPS の `~/.ssh/authorized_keys` に置く |
| `SSH_KNOWN_HOSTS` | `ssh-keyscan -p 22 <VPS のホスト名>` の出力 |
| `RCON_PASSWORD` | RCON のパスワード |
| `RWA_PASSWORD` | 管理画面のパスワード |

variables は設定したものだけ `.env` に書き込まれ、残りは compose の既定値になる。`MC_VERSION` だけは必須で、未設定なら理由を出して止まる。他に `MC_MEMORY`、`MOTD`、`DIFFICULTY`、`VIEW_DISTANCE`、`MAX_PLAYERS`、`RWA_USERNAME`、`SSH_PORT`、`REMOTE_DIR` を設定できる。

## 困ったとき

| 症状 | 見るところ |
| --- | --- |
| Bedrock 版だけ入れない | UDP 19132 が開いているか。OS 側と VPS の管理画面側の両方が必要なことが多い |
| 版が違うと出る | `.env` の `MC_VERSION` とクライアントの版。サーバーの版は `docker compose exec mc rcon-cli version` |
| 重い | `VIEW_DISTANCE` を下げる。10 から 8 にするだけでも効く。メモリに余裕があれば `MC_MEMORY` を増やす |
| 起動しない | `docker compose logs mc`。初回はワールド生成で数分かかる |

## エージェント向け

このリポジトリで作業するエージェントは [AGENTS.md](AGENTS.md) から始める。

## コントリビューション

使い方の質問は [Discussions](https://github.com/philtzjp/minecraft-crossplay/discussions)、不具合と提案は [Issues](https://github.com/philtzjp/minecraft-crossplay/issues) へ。

PR は、先に Issue で方針を決めてから出してほしい。個人で運用している構成なので、合わなかった変更を戻す手間を避けたい。誤字の修正のような小さいものは、そのまま PR で構わない。

脆弱性を見つけたときは、公開の Issue ではなく GitHub の [Report a vulnerability](https://github.com/philtzjp/minecraft-crossplay/security/advisories/new) から知らせてほしい。

## ライセンス

MIT。[LICENSE](LICENSE) を参照。

Minecraft は Mojang Studios の商標で、このリポジトリは Mojang Studios とも Microsoft とも関係がない。
