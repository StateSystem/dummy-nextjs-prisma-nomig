# dummy-nextjs-prisma-nomig

- 対象: EV165。
- 起動: `npm ci` の後、`DATABASE_URL` を設定し、下記の初期化を行って `npm run build && npm start`。
- デフォルト PORT: 3000。環境変数 `PORT` で変更可。待受: `0.0.0.0`。
- ブランチ: `main` のみ、デフォルトも `main`。ルートに Dockerfile あり。
- Node.js 22 以上。`GET /health` は HTTP 200 と `{"ok":true,"app":"dummy-nextjs-prisma-nomig"}` を返す。

## Docker

```sh
docker build -t dummy-nextjs-prisma-nomig .
```

## データベース

PostgreSQL。Prisma 6 系を使用し、接続先は `schema.prisma` の `DATABASE_URL` から読む。
接続 URL の `schema` にプラットフォーム指定の schema を設定し、`public` や特定テナント名をコードに固定しない。
`.env.example` はプレースホルダーのみ。実際の接続情報は環境変数か Git 管理外の `.env` で設定する。
`Item(id, name)` をトップ画面で動的に読み込む。DB 障害を空リストに置き換えない。
`/health` はアプリの生存確認で、DB 接続の確認ではない。DB 接続はトップ画面で確認する。
seed ファイル・設定・自動 INSERT はない。ビルド・起動では migrate / db push を実行しない。
デプロイ時の DB 初期化・更新は DEParture が担当する。

ローカル専用の空 DB を用意し、接続先を確認して以下を実行する。

```sh
npx prisma db push
npm run build
npm start
```

Docker の起動には実際の接続 URL を設定してから `docker run --rm -p 3000:3000 -e DATABASE_URL dummy-nextjs-prisma-nomig` を使用する。
DB はコンテナから接続可能で、初期化済みであること。Docker build に接続情報は不要。

SQL コンソールで対象 schema を選択し、次のように手動で投入するとトップ画面に表示される。
schema が検索パスにない場合は、実際の schema 名でテーブルを修飾する。

```sql
SELECT * FROM "Item";
INSERT INTO "Item" ("name") VALUES ('manual verification');
```

## EV165

`prisma/migrations` は存在しない。初回の db push はプラットフォームの経路を確認する。
初回デプロイ後に schema.prisma の Item に `description String?` を追加して commit / push し、再デプロイする。
SQL / introspect でカラムが未追加であること、migrations 推奨案内が出ることを確認する。
変更後にローカルから対象 DB に db push を実行しない。Docker や起動スクリプトにも追加しない。
画面のクエリが新 schema と不一致で失敗する場合も、DB の未変更確認と区別して記録する。

## 依存関係の既知事項

検証時の npm audit は Prisma 6.19.3 の間接依存 deepmerge-ts に関する high を 3 件報告（同一問題の依存チェーン）。GHSA-ggr8-5vv4-36mx。自動の強制ダウングレードは適用していない。依存更新時は migration と接続 schema の動作を再確認する。
