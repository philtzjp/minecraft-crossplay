<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/philtzjp/.github/main/images/philtz-dark.png">
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/philtzjp/.github/main/images/philtz-light.png">
  <img src="https://raw.githubusercontent.com/philtzjp/.github/main/images/philtz-outline.png" width="96" alt="Philtz">
</picture>

# Contributing

How to propose a change to minecraft-crossplay.<br>
<sub>minecraft-crossplay に変更を提案する方法です。</sub>

<p align="center"><a href="#en">Read more in English</a> · <a href="#ja">日本語で読む</a></p>

<a id="en"></a>

## English

> [!IMPORTANT]
> This is a setup one person runs in production, so a change that does not fit has to be reverted on a live server. Open an [issue](https://github.com/philtzjp/minecraft-crossplay/issues) and agree on the approach before you write a pull request. Typos and broken links are the exception; send those straight as a pull request. Questions about using it belong in [Discussions](https://github.com/philtzjp/minecraft-crossplay/discussions), not in issues.

Found a security problem? Do not open a public issue. Use [Report a vulnerability](https://github.com/philtzjp/minecraft-crossplay/security/advisories/new).

### How to work on it

1. Open an issue describing the problem and the approach, and wait for a reply.
2. Branch as `<type>/<issue-number>-<short-summary>`, where type is one of feat, fix, docs, chore, ci.
3. Make the change. Keep `data/` and `backups/` out of it; those are world data, not source.
4. Validate the compose file: `docker compose --env-file .env.example config --quiet`.
5. Open a pull request that references the issue.

<details>
<summary>Commit messages</summary>
<br>

Write `type(scope): description`, with the description as one short sentence in Japanese. Type is one of feat, fix, docs, chore, ci. Scope is the top-level directory you changed, or `repo` when the change spans the repository. Do not add `Co-Authored-By`.

| | |
| --- | --- |
| OK | `fix(repo): Bedrock 版のポートを既定に戻す` |
| NG | `fix: ポート` |

The description should say what changed and end with an action, so the log reads as a list of changes rather than a list of nouns.

</details>

<details>
<summary>Testing a change for real</summary>
<br>

Copy `.env.example` to `.env`, change the ports so they do not collide with anything running, and start it.

```sh
docker compose up -d
docker compose logs -f mc
```

When you are done, `docker compose down` and delete the generated `data/` and `backups/`. Do not commit them.

</details>

<details>
<summary>Repository layout</summary>
<br>

| Path | Contents |
| --- | --- |
| `docker-compose.yml` | Every service |
| `.env.example` | All settings, with defaults |
| `.github/workflows/` | Compose validation, and deployment to a VPS |
| `AGENTS.md` | Conventions for coding agents |

</details>

<a id="ja"></a>

## 日本語

> [!IMPORTANT]
> 個人が実運用している構成なので、合わなかった変更は動いているサーバー側で戻すことになります。pull request を書く前に [Issue](https://github.com/philtzjp/minecraft-crossplay/issues) で方針を決めてください。誤字やリンク切れの修正は例外で、そのまま pull request で構いません。使い方の質問は Issue ではなく [Discussions](https://github.com/philtzjp/minecraft-crossplay/discussions) へお願いします。

脆弱性を見つけたときは、公開の Issue ではなく [Report a vulnerability](https://github.com/philtzjp/minecraft-crossplay/security/advisories/new) から知らせてください。

### 進め方

1. 問題と方針を書いた Issue を立て、返事を待ちます。
2. `<type>/<issue 番号>-<短い要約>` の形でブランチを切ります。type は feat、fix、docs、chore、ci のいずれかです。
3. 変更します。`data/` と `backups/` は触りません。ワールドのデータであって、ソースではありません。
4. compose の設定を検証します。`docker compose --env-file .env.example config --quiet`
5. Issue を参照した pull request を出します。

<details>
<summary>コミットメッセージ</summary>
<br>

`type(scope): 説明` の形式で、説明は日本語の短い一文にします。type は feat、fix、docs、chore、ci のいずれかです。scope は変更したトップレベルのディレクトリ名、リポジトリ全体にまたがるなら `repo` にします。`Co-Authored-By` は付けません。

| | |
| --- | --- |
| OK | `fix(repo): Bedrock 版のポートを既定に戻す` |
| NG | `fix: ポート` |

説明は動作で終えます。履歴が名詞の一覧ではなく、変更の一覧として読めるようにするためです。

</details>

<details>
<summary>実際に動かして確かめる</summary>
<br>

`.env.example` を `.env` にコピーし、動いているものと衝突しないようポートを変えてから起動します。

```sh
docker compose up -d
docker compose logs -f mc
```

終わったら `docker compose down` を実行し、生成された `data/` と `backups/` を削除します。コミットしないでください。

</details>

<details>
<summary>リポジトリの構成</summary>
<br>

| パス | 中身 |
| --- | --- |
| `docker-compose.yml` | すべてのサービス |
| `.env.example` | 設定の一覧と既定値 |
| `.github/workflows/` | compose の検証と、VPS への配置 |
| `AGENTS.md` | コーディングエージェント向けの規約 |

</details>
