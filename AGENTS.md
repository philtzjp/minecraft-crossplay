# AGENTS.md

このリポジトリで作業するエージェント向けの覚え書き。人間向けの説明は [README.md](README.md) にある。

## このリポジトリの中身

Minecraft サーバー（Paper + Geyser + Floodgate）を VPS 1台で動かす Docker Compose 構成。アプリケーションのコードはなく、compose の定義と設定と手順書だけで成り立っている。ビルドもテストもない。

## 守ること

1. `data/` と `backups/` に触れない。ワールドデータそのもので、git では追跡していない。消す操作（`docker compose down -v`、`rm -rf data`）を実行しない
2. `.env` の中身を出力しない。RCON のパスワードが入っている
3. ワールドの中身を変えるときは、ファイルを直接編集せず RCON かゲーム内のコマンドを使う
4. `MC_VERSION` を上げるときは、先に Geyser の対応範囲と Paper の安定版ビルドの有無を確かめる。README の「サーバーのバージョンを上げる」に手順がある

## 確認のしかた

```sh
docker compose --env-file .env.example config --quiet   # compose の設定を検証する
```

実際に動かして確かめるなら、`.env` でポートを既定から変え、終わったら `docker compose down` と生成された `data/` の削除まで行う。

## 変更したら

| 変えたもの | 一緒に直すもの |
| --- | --- |
| compose の環境変数 | `.env.example` と README の該当する表 |
| ポート | README の「つなぐ」とファイアウォールの手順 |
| サービスの追加や削除 | README の構成の表 |
| workflow の secrets や variables | README の「GitHub Workflow から配置する」 |

## コミット

`type(scope): 説明` の形式で、説明は日本語の短い文にする。type は feat、fix、docs、chore、ci のいずれか。scope は変更したトップレベルのディレクトリ名で、リポジトリ全体に関わるなら `repo`。`Co-Authored-By` は付けない。
