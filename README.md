<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/philtzjp/.github/main/images/philtz-dark.png">
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/philtzjp/.github/main/images/philtz-light.png">
  <img src="https://raw.githubusercontent.com/philtzjp/.github/main/images/philtz-outline.png" width="96" alt="Philtz">
</picture>

# minecraft-crossplay

Run a Minecraft server where Java and Bedrock players share one world, on a single VPS with Docker Compose.<br>
<sub>Java 版と Bedrock 版のプレイヤーが同じワールドで遊べる Minecraft サーバーを、VPS 1台で動かすための Docker Compose 構成です。</sub>

<p align="center"><a href="#en">Read more in English</a> · <a href="#ja">日本語で読む</a></p>

<a id="en"></a>

## English

<p align="center">
  <a href="https://github.com/philtzjp/minecraft-crossplay"><img src="https://img.shields.io/github/stars/philtzjp/minecraft-crossplay?style=social" alt="Star minecraft-crossplay on GitHub"></a><br>
  <sub>If minecraft-crossplay helps you, a star keeps us going.</sub>
</p>

> [!IMPORTANT]
> You need a VPS with at least 2GB of memory, Docker, and two open ports: TCP 25565 for Java Edition and UDP 19132 for Bedrock Edition. This runs Paper, so Paper and Spigot plugins work but Forge and Fabric mods do not. Pin the Paper version in `.env`: following the latest release will eventually move the server to a version Geyser has not caught up with, and Bedrock players are locked out until it does.

Java Edition and Bedrock Edition normally cannot meet. The official server software picks one side, and Realms does not bridge them either. This setup puts [Geyser](https://geysermc.org/) and [Floodgate](https://wiki.geysermc.org/floodgate/) on top of Paper, so a Bedrock player on a phone or a console joins the same world without buying a Java account.

### Quick start

1. On a VPS with Docker installed, open the ports. Some providers have a second firewall in their control panel, such as XServer VPS or Oracle Cloud; opening only the OS side is not enough.

```sh
sudo ufw allow 22/tcp && sudo ufw allow 25565/tcp && sudo ufw allow 19132/udp && sudo ufw enable
```

2. Clone the repository and set your passwords in `.env`.

```sh
git clone https://github.com/philtzjp/minecraft-crossplay.git
cd minecraft-crossplay
cp .env.example .env   # set RCON_PASSWORD and RWA_PASSWORD
```

3. Start it. The first run downloads the server and the plugins and generates the world, which takes a few minutes.

```sh
docker compose up -d
docker compose logs -f mc
```

When the log prints `Done`, the server is up. Connect with the VPS address: port 25565 from Java Edition, and port 19132 from Bedrock Edition.

> [!TIP]
> A Bedrock player's name is prefixed with `.` in game. That prefix is how you know Floodgate is doing its job.

Then try:

```sh
docker compose exec mc rcon-cli list             # both editions appear in one list
docker compose exec mc rcon-cli "geyser version" # which Java and Bedrock versions are supported
docker compose exec backups backup now           # take a backup right now
```

### Technology

<details>
<summary>Bedrock players join without a Java account, and without a Java client</summary>
<br>

Geyser translates the Bedrock protocol into the Java protocol, and Floodgate authenticates Bedrock players against Xbox Live instead of a Java account. Both are fetched when the container starts, so no plugin files are committed here.

Geyser listens on UDP 19132, the port Bedrock clients already expect, so there is no config file to edit. Java Edition keeps TCP 25565.

| Who | Where they connect | What they need |
| --- | --- | --- |
| Java Edition | TCP 25565 | A Minecraft Java account |
| Bedrock Edition (phone, console, Windows) | UDP 19132 | An Xbox Live account |

</details>

<details>
<summary>The world never lives in git, so updating the repository cannot destroy it</summary>
<br>

`data/` and `backups/` are ignored by git, and the deploy workflow excludes them from both the transfer and the deletion pass. Pulling a new version replaces the compose definition and nothing else.

Backups run every four hours by default and are kept for seven days. `docker compose --profile restore run --rm restore-backup` restores one.

</details>

<details>
<summary>One file holds every setting</summary>
<br>

Everything tunable lives in `.env`, including ports, so two servers can run side by side on one host. Apply a change with `docker compose up -d`.

The admin UI binds to localhost only. Reach it over SSH, then open `http://localhost:4326`.

```sh
ssh -L 4326:127.0.0.1:4326 -L 4327:127.0.0.1:4327 <user>@<vps>
```

</details>

### Specification

<details>
<summary>Services</summary>
<br>

| Service | Role |
| --- | --- |
| mc | The Paper server ([itzg/minecraft-server](https://github.com/itzg/docker-minecraft-server)) with Geyser and Floodgate |
| backups | Scheduled backups, every 4 hours, kept 7 days |
| restore-backup | Restores a backup, run by hand with the `restore` profile |
| rcon-web | Browser console, bound to localhost |

</details>

<details>
<summary>Settings</summary>
<br>

| Name | Default | Note |
| --- | --- | --- |
| `MC_VERSION` | — | Required. Pin it to a version Geyser supports |
| `MC_MEMORY` | `4G` | Leave 1–2GB of the VPS for the system |
| `RCON_PASSWORD` | — | Required |
| `RWA_PASSWORD` | — | Required, for the admin UI |
| `MOTD` | `A Minecraft Server` | Shown in the server list |
| `DIFFICULTY` | `normal` | |
| `VIEW_DISTANCE` | `10` | Lower it first when the server feels heavy |
| `MAX_PLAYERS` | `20` | |
| `JAVA_PORT` | `25565` | TCP |
| `BEDROCK_PORT` | `19132` | UDP |
| `RCON_WEB_PORT` | `4326` | Bound to 127.0.0.1 |
| `BACKUP_INTERVAL` | `4h` | |
| `PRUNE_BACKUPS_DAYS` | `7` | |

</details>

<details>
<summary>Using a domain name</summary>
<br>

Point an A record at the VPS address. On Cloudflare, turn the proxy off (DNS only); the proxy carries HTTP and HTTPS, not the Minecraft protocol.

To let Java players omit the port, add an SRV record `_minecraft._tcp.<subdomain>` with priority 0, weight 5, port 25565. Bedrock clients ignore SRV records, so the port stays part of the address there.

</details>

<details>
<summary>Deploying from GitHub Actions</summary>
<br>

[deploy](.github/workflows/deploy.yml) sends main to the VPS over rsync and recreates the containers. `data/` and `backups/` are excluded, so the world survives.

| Secret | Value |
| --- | --- |
| `SSH_HOST` | Host name or address of the VPS |
| `SSH_USER` | User to log in as |
| `SSH_KEY` | Private key whose public half is in `~/.ssh/authorized_keys` |
| `SSH_KNOWN_HOSTS` | Output of `ssh-keyscan -p 22 <host>` |
| `RCON_PASSWORD` | Same as `.env` |
| `RWA_PASSWORD` | Same as `.env` |

Repository variables fill the rest of `.env`; anything left unset falls back to the compose default. `MC_VERSION` is the exception and must be set, otherwise the run stops and says so. `SSH_PORT` defaults to `22` and `REMOTE_DIR` to `/opt/minecraft-crossplay`.

</details>

<details>
<summary>Raising the server version</summary>
<br>

Check both before you change `MC_VERSION`:

1. Geyser supports the version. `docker compose exec mc rcon-cli "geyser version"` prints the supported range.
2. Paper has a stable build for it. On [the Paper downloads page](https://papermc.io/downloads/paper), avoid versions that only offer beta or alpha builds.

Then edit `.env`, run `docker compose up -d`, and match your client to the new version.

</details>

<details>
<summary>When something does not work</summary>
<br>

| Symptom | Where to look |
| --- | --- |
| Only Bedrock players cannot join | UDP 19132. Usually both the OS firewall and the provider's packet filter need it |
| The client reports a version mismatch | `MC_VERSION` versus the client. `docker compose exec mc rcon-cli version` prints the server's |
| The server feels heavy | Lower `VIEW_DISTANCE`; 10 to 8 is noticeable. Raise `MC_MEMORY` if the VPS has room |
| It does not start | `docker compose logs mc`. The first run spends minutes generating the world |

</details>

<a id="ja"></a>

## 日本語

<p align="center">
  <a href="https://github.com/philtzjp/minecraft-crossplay"><img src="https://img.shields.io/github/stars/philtzjp/minecraft-crossplay?style=social" alt="Star minecraft-crossplay on GitHub"></a><br>
  <sub>minecraft-crossplay が役に立ったら、スターを付けてもらえると励みになります。</sub>
</p>

> [!IMPORTANT]
> メモリ 2GB 以上の VPS と Docker、そして TCP 25565（Java 版）と UDP 19132（Bedrock 版）の2つのポートが要ります。Paper で動かすため、Paper と Spigot のプラグインは使えますが、Forge や Fabric の MOD は入りません。Paper のバージョンは `.env` で固定してください。最新に追従させると、Geyser がまだ対応していない版へ上がったときに、対応するまで Bedrock 版だけ入れなくなります。

Java 版と Bedrock 版は、本来は一緒に遊べません。公式のサーバーはどちらか一方しか選べず、Realms も版をまたげません。この構成は Paper に [Geyser](https://geysermc.org/) と [Floodgate](https://wiki.geysermc.org/floodgate/) を載せて、その壁をなくします。Bedrock 版のプレイヤーは、スマートフォンやコンシューマー機から、Java 版のアカウントを買わずに同じワールドへ入れます。

### クイックスタート

1. Docker を入れた VPS で、ポートを開けます。XServer VPS や Oracle Cloud のように、管理画面側にもファイアウォールがある事業者では、OS 側だけでは通りません。

```sh
sudo ufw allow 22/tcp && sudo ufw allow 25565/tcp && sudo ufw allow 19132/udp && sudo ufw enable
```

2. リポジトリを取得し、`.env` にパスワードを設定します。

```sh
git clone https://github.com/philtzjp/minecraft-crossplay.git
cd minecraft-crossplay
cp .env.example .env   # RCON_PASSWORD と RWA_PASSWORD を自分の値にする
```

3. 起動します。初回はサーバー本体とプラグインの取得、ワールドの生成で数分かかります。

```sh
docker compose up -d
docker compose logs -f mc
```

ログに `Done` と出れば起動しています。接続先は VPS のアドレスで、Java 版はポート 25565、Bedrock 版はポート 19132 です。

> [!TIP]
> Bedrock 版から入ったプレイヤーの名前は、ゲーム内で `.` から始まります。この印が、Floodgate が効いている証拠です。

次を試してみてください。

```sh
docker compose exec mc rcon-cli list             # 両方の版のプレイヤーが同じ一覧に出る
docker compose exec mc rcon-cli "geyser version" # 対応する Java と Bedrock の版が分かる
docker compose exec backups backup now           # 今すぐバックアップを取る
```

### テクノロジー

<details>
<summary>Bedrock 版のプレイヤーは、Java 版のアカウントもクライアントも要らない</summary>
<br>

Geyser が Bedrock の通信を Java の通信に翻訳し、Floodgate が Java 版のアカウントの代わりに Xbox Live で認証します。どちらもコンテナの起動時に取得するので、プラグインのファイルはこのリポジトリに含めていません。

Geyser は Bedrock 版のクライアントが既定で見にいく UDP 19132 で待ち受けるため、設定ファイルを書き換える必要がありません。Java 版は TCP 25565 のままです。

| 誰が | どこへつなぐか | 必要なもの |
| --- | --- | --- |
| Java 版 | TCP 25565 | Minecraft Java 版のアカウント |
| Bedrock 版（スマートフォン、コンシューマー機、Windows） | UDP 19132 | Xbox Live のアカウント |

</details>

<details>
<summary>ワールドは git に入らないので、更新で壊れない</summary>
<br>

`data/` と `backups/` は git では追跡せず、配置の workflow でも転送と削除の両方から外しています。新しい版を取り込んでも、入れ替わるのは compose の定義だけです。

バックアップは既定で4時間ごとに取り、7日分を残します。戻すときは `docker compose --profile restore run --rm restore-backup` を実行します。

</details>

<details>
<summary>設定は1つのファイルに集まっている</summary>
<br>

変えられる値はポートも含めてすべて `.env` にあるので、1台のホストで2つのサーバーを並べて動かせます。変更は `docker compose up -d` で反映します。

管理画面は localhost にだけ開いています。SSH でポートを転送してから `http://localhost:4326` を開いてください。

```sh
ssh -L 4326:127.0.0.1:4326 -L 4327:127.0.0.1:4327 <ユーザー>@<VPS のアドレス>
```

</details>

### 仕様

<details>
<summary>サービス</summary>
<br>

| サービス | 役割 |
| --- | --- |
| mc | Paper サーバー本体（[itzg/minecraft-server](https://github.com/itzg/docker-minecraft-server)）。Geyser と Floodgate を載せる |
| backups | 定期バックアップ。4時間ごと、7日保持 |
| restore-backup | バックアップからの復元。`restore` プロファイルで手動実行する |
| rcon-web | ブラウザから使う管理画面。localhost にだけ開く |

</details>

<details>
<summary>設定</summary>
<br>

| 名前 | 既定値 | 補足 |
| --- | --- | --- |
| `MC_VERSION` | — | 必須。Geyser が対応する版に固定する |
| `MC_MEMORY` | `4G` | VPS の搭載メモリから 1〜2GB 残す |
| `RCON_PASSWORD` | — | 必須 |
| `RWA_PASSWORD` | — | 必須。管理画面のパスワード |
| `MOTD` | `A Minecraft Server` | サーバー一覧に出る説明 |
| `DIFFICULTY` | `normal` | |
| `VIEW_DISTANCE` | `10` | 重いときに最初に下げる値 |
| `MAX_PLAYERS` | `20` | |
| `JAVA_PORT` | `25565` | TCP |
| `BEDROCK_PORT` | `19132` | UDP |
| `RCON_WEB_PORT` | `4326` | 127.0.0.1 にだけ開く |
| `BACKUP_INTERVAL` | `4h` | |
| `PRUNE_BACKUPS_DAYS` | `7` | |

</details>

<details>
<summary>ドメインを使う</summary>
<br>

A レコードを VPS のアドレスに向けます。Cloudflare ではプロキシをオフ（DNS only）にしてください。プロキシが運ぶのは HTTP と HTTPS で、Minecraft の通信は通りません。

Java 版でポート番号の入力を省きたいときは、SRV レコード `_minecraft._tcp.<サブドメイン>` を優先度 0、重み 5、ポート 25565 で足します。Bedrock 版は SRV を見ないので、ポートの入力は必要なままです。

</details>

<details>
<summary>GitHub Actions から配置する</summary>
<br>

[deploy](.github/workflows/deploy.yml) が、main を VPS へ rsync で送り、コンテナを入れ替えます。`data/` と `backups/` は除いているので、ワールドは残ります。

| secret | 中身 |
| --- | --- |
| `SSH_HOST` | VPS のホスト名かアドレス |
| `SSH_USER` | ssh で入るユーザー名 |
| `SSH_KEY` | 秘密鍵。対応する公開鍵を VPS の `~/.ssh/authorized_keys` に置く |
| `SSH_KNOWN_HOSTS` | `ssh-keyscan -p 22 <ホスト名>` の出力 |
| `RCON_PASSWORD` | `.env` と同じ値 |
| `RWA_PASSWORD` | `.env` と同じ値 |

`.env` の残りはリポジトリの variables から埋まり、設定しなかった項目は compose の既定値になります。`MC_VERSION` だけは必須で、未設定なら理由を出して止まります。`SSH_PORT` の既定は `22`、`REMOTE_DIR` の既定は `/opt/minecraft-crossplay` です。

</details>

<details>
<summary>サーバーのバージョンを上げる</summary>
<br>

`MC_VERSION` を書き換える前に、次の2つを確かめます。

1. Geyser が上げたい版に対応していること。`docker compose exec mc rcon-cli "geyser version"` が対応範囲を表示します。
2. Paper にその版の安定版ビルドがあること。[PaperMC のダウンロード](https://papermc.io/downloads/paper)で、beta や alpha しかない版は避けます。

どちらも満たしたら `.env` を書き換えて `docker compose up -d` し、クライアント側の版も合わせます。

</details>

<details>
<summary>うまくいかないとき</summary>
<br>

| 症状 | 見るところ |
| --- | --- |
| Bedrock 版だけ入れない | UDP 19132。OS 側と事業者のパケットフィルターの両方が要ることが多い |
| 版が違うと出る | `MC_VERSION` とクライアントの版。サーバーの版は `docker compose exec mc rcon-cli version` |
| 重い | `VIEW_DISTANCE` を下げる。10 から 8 でも効く。メモリに余裕があれば `MC_MEMORY` を増やす |
| 起動しない | `docker compose logs mc`。初回はワールドの生成に数分かかる |

</details>
