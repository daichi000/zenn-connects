---
title: "Cloudflare Pages + Next.js + Hono + D1 + Drizzleで始めるフルスタック構成"
emoji: "🏔️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "nextjs", "d1", "hono", "drizzle"]
published: true
---

# はじめに
:::message
この記事は [Cloudfalre Advent Calendar 2023](https://qiita.com/advent-calendar/2023/cloudflare) の12日目の記事となります。
:::

個人開発でWebサービスを作るにあたり、CloudflareやHonoなど普段の業務では縁のない技術を試してみた。
CRUD操作ができるところまでをまとめてみる。

## Cloudflare Pagesとは
Cloudflare社が提供するJAMstackプラットフォーム。
Freeプランで月500回までビルド可能、bandwidthが無制限と無料枠が充実しているのが特徴。

(Next.jsのホスティングといえばVercelだけど商用利用で$20/月は個人だと痛手。。)
https://www.cloudflare.com/ja-jp/developer-platform/pages/

## Honoとは
軽量かつ高速なWebフレームワーク。
Cloudflare Pages/Workersと親和性が高いExpress的なイメージ。
https://hono.dev/

# 技術構成
- CLI: wrangler2
- Hosting: Cloudflare Pages + Functions(サーバー処理)
- FW: Next.js(App Router), Hono
- 言語: TypeScript
- DB: Cloudflare D1
- ORM: Drizzle

# 事前準備
- [Cloudflareのサイト](https://www.cloudflare.com/ja-jp/)にてアカウントを作成
- プロジェクト作成などするために、CLIツール `wragler` をインストール&ログインする
```
$ npm install -g wrangler
$ wrangler --version
 ⛅️ wrangler 3.18.0
$ wrangler login
```

# Next.jsのセットアップ
C3(create-cloudflare CLI)を使うことで新規プロジェクトの作成からデプロイまで一気通貫で行うことができてしまう。
@cloudflare/next-on-pages
https://developers.cloudflare.com/pages/framework-guides/deploy-a-nextjs-site/

```
$ npm create cloudflare@latest my-next-app -- --framework=next
│ 
╰ Do you want to deploy your application?
  Yes / No
```
デプロイするを選択するとわずか1分ほどでホスティング完了！
https://github.com/bajji-corporation/capturex-web/assets/35623457/0a5e0eec-8dcd-4f93-b1a8-5d609fb605b6

ダッシュボードでもプロジェクトが作成されていることが確認できる。
https://github.com/bajji-corporation/capturex-web/assets/35623457/dc1e6249-5ad4-4e25-97cc-f04d6b4c4c1a

# Honoのセットアップ
https://hono.dev/getting-started/vercel
今回はApp Routerにて簡単なAPIを作成してみる。

```
$ npm install hono
```
```ts:app/api/[[...route]]/route.ts
import { Hono } from 'hono'
import { handle } from 'hono/vercel'

export const runtime = 'edge';

const app = new Hono().basePath('/api')

app.get('/hello', (c) => {
  return c.json({
    message: 'Hello Next.js!',
  })
})

export const GET = handle(app)
```
ローカルでAPIを叩いてmessageが返ってきたらセットアップ完了！
```
$ npm run pages:build
$ npm run pages:dev
```
https://github.com/bajji-corporation/capturex-web/assets/35623457/f4a6b17f-bb8e-4c21-9053-e05ab96adf61

# D1のセットアップ
https://developers.cloudflare.com/d1/get-started/

## DB作成
`wrangler`コマンドでDBを作成する。
```
$ npx wrangler d1 create my-next-app-db

✅ Successfully created DB 'my-next-app-db' in region APAC
Created your database using D1's new storage backend. The new storage backend is not yet recommended for production
workloads, but backs up your data via point-in-time restore.

[[d1_databases]]
binding = "DB" # i.e. available in your Worker on env.DB
database_name = "my-next-app-db"
database_id = "database-id-xxxxxxxxx"
```
[[d1_databases]]以下はローカルからD1への接続に必要な情報。
ルートディレクトリに `wrangler.toml` を作成して貼り付けておく。

なおこの時点でダッシュボードからDBが作成されていることが確認できる。
https://github.com/bajji-corporation/capturex-web/assets/35623457/9ba57267-3e85-4d3b-88af-186b0e2b21e2

## SQL実行
試しに公式にあるschemaファイルをルートディレクトリにおいて実行してみる。

```sql:schema.sql
DROP TABLE IF EXISTS Customers;
CREATE TABLE IF NOT EXISTS Customers (CustomerId INTEGER PRIMARY KEY, CompanyName TEXT, ContactName TEXT);
INSERT INTO Customers (CustomerID, CompanyName, ContactName) VALUES (1, 'Alfreds Futterkiste', 'Maria Anders'), (4, 'Around the Horn', 'Thomas Hardy'), (11, 'Bs Beverages', 'Victoria Ashworth'), (13, 'Bs Beverages', 'Random Name');
```

ローカルで実行。
なお、`--local`を外すと本番で実行される。
```
$ npx wrangler d1 execute my-next-app-db --local --file=./schema.sql
```

データが格納されていることを確認する。
```
$ npx wrangler d1 execute my-next-app-db --local --command='SELECT * FROM Customers'

┌────────────┬─────────────────────┬───────────────────┐
│ CustomerId │ CompanyName         │ ContactName       │
├────────────┼─────────────────────┼───────────────────┤
│ 1          │ Alfreds Futterkiste │ Maria Anders      │
├────────────┼─────────────────────┼───────────────────┤
│ 4          │ Around the Horn     │ Thomas Hardy      │
├────────────┼─────────────────────┼───────────────────┤
│ 11         │ Bs Beverages        │ Victoria Ashworth │
├────────────┼─────────────────────┼───────────────────┤
│ 13         │ Bs Beverages        │ Random Name       │
└────────────┴─────────────────────┴───────────────────┘
```

本番にも反映
```
$ npx wrangler d1 execute my-next-app-db --file=./schema.sql
$ npx wrangler d1 execute my-next-app-db --command='SELECT * FROM Customers'
```

## Binding D1
デプロイしたNext.js(Pages Function)とD1を紐づける。
Cloudflare Pagesの場合、ローカルでは`wrangler.toml`から紐付けされるが本番では読み込まれないため、ダッシュボードから設定する必要がある。

該当のプロジェクトから、設定、Functionsを開く。
https://github.com/bajji-corporation/capturex-web/assets/35623457/58151d29-e965-4a55-95a2-0a7792d0c0d9

D1データベース バインディングの編集にて、該当のD1を選択する。
https://github.com/bajji-corporation/capturex-web/assets/35623457/e69f489a-fad0-4e15-9df6-a8f17d320f34

これで本番のPages Functionとの紐付けができたのでD1のセットアップ完了！

# APIからD1を呼び出す
`route.ts`に処理を書いていく。
https://developers.cloudflare.com/d1/examples/d1-and-hono/#

Honoのドキュメントを参考に以下のように書いてみる
```ts:app/api/[[...route]]/route.ts
import { D1Database } from '@cloudflare/workers-types';
import { Hono } from 'hono'
import { handle } from 'hono/vercel'

export const runtime = 'edge';

// This ensures c.env.DB is correctly typed
type Bindings = {
  DB: D1Database
}

const app = new Hono<{ Bindings: Bindings}>().basePath('/api')

// Accessing D1 is via the c.env.YOUR_BINDING property
app.get("/query/customers", async (c) => {
  let { results } = await c.env.DB.prepare("SELECT * FROM customers").all()
  return c.json(results)
})

export const GET = handle(app)
```
ローカルの場合は起動コマンドに`--d1=BINDING_NAME`のフラグが必要みたいなので`--d1=DB`を追加して起動してみる。

`/api/query/customers`を叩いてみるが500エラー
```
$ ✘ [ERROR] TypeError: Cannot read properties of undefined (reading 'prepare')
```

prepareに問題がありそう？

いろいろドキュメントを漁ると、next-on-pagesからはprocess経由でBindingsにアクセスができるらしい。
https://github.com/cloudflare/next-on-pages/issues/1#issuecomment-1403845325

env周りを以下のように修正
```diff
- await c.env.DB.prepare("SELECT * FROM customers").all()
+ await process.env.DB.prepare("SELECT * FROM customers").all()
```

tsエラーがでるのでこちらも追加しておく
```
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      DB: D1Database;
    }
  }
}
```

再度実行。
が、またもエラー...
```
$ [wrangler:inf] GET /api/query/customers 500 Internal Server Error (28ms)
✘ [ERROR] Error: D1_ERROR: no such table: customers
```

customersテーブルがない？
先ほどテーブルの作成は確認できているので、ローカルからD1への接続がうまくいっていないぽい。
こちらも調査したところ[公式ドキュメント](https://developers.cloudflare.com/d1/learning/local-development/#develop-locally-with-pages)に答えがあった。
```toml:wrangler.toml
# If you are only using Pages + D1, you only need the below in your wrangler.toml to interact with D1 locally.
[[d1_databases]]
binding = "DB" # Should match preview_database_id
database_name = "YOUR_DATABASE_NAME"
database_id = "the-id-of-your-D1-database-goes-here" # wrangler d1 info YOUR_DATABASE_NAME
preview_database_id = "DB" # Required for Pages local development <- 追記
```
ローカルでD1に接続する場合は、`prevew_database_id`を追記する必要があるらしい。
:::message
なお、設定が追加された後のDBはデータが空状態になってしまうので、再度SQLを実行してデータが格納されていることを確認する。
:::

上記のとおり修正し、再度実行。

https://github.com/bajji-corporation/capturex-web/assets/35623457/46235a42-c090-4466-8765-088de0a27b36

ようやく成功！

# Drizzle導入
## Drizzleとは
https://github.com/drizzle-team/drizzle-orm

D1をサポートしているエッジ対応のORM。
他にも[kysery](https://github.com/aidenwallis/kysely-d1)などいくつか候補があったが、一番評判が良さそうなDrizzleを使ってみる。

## Drizzleセットアップ
というわけでインストール。
```
$ npm install drizzle-orm
$ npm install -D drizzle-kit
```
続いてschemaの定義を作成
```ts:schema.ts
import { sqliteTable, text, integer } from "drizzle-orm/sqlite-core";

export const users = sqliteTable("users", {
  userId: integer("userId", { mode: "number" })
    .primaryKey({ autoIncrement: true })
    .notNull(),
  userName: text("userName").notNull(),
});
```
drizzleの設定ファイルも追加
```ts:drizzle.config.ts
import type { Config } from "drizzle-kit";

export default {
  schema: "./schema.ts",
  out: "./drizzle/migrations",
  driver: "d1",
  dbCredentials: {
    wranglerConfigPath: "wrangler.toml",
    dbName: "my-next-app-db",
  },
} satisfies Config;
```
:::message
以前までローカルで動かすにはbetter-sqliteを入れる必要があったが不要になったらしい
:::

generateを実行すると、es5がサポート外というエラーになるのでtsconfigを修正する
```
$ npx drizzle-kit generate:sqlite
drizzle-kit: v0.20.6
drizzle-orm: v0.29.1

No config path provided, using default 'drizzle.config.ts'
ERROR: Transforming const to the configured target environment ("es5") is not supported yet
```
```diff json:tsconfig.json
- "target": "es5",
+ "target": "es2017",
```
成功するとmigrationファイルが作成される(ファイル名は自動で命名)
```
$ [✓] Your SQL migration file ➜ drizzle/migrations/0000_far_smasher.sql 🚀
```
`wrangler.toml`にmigrations_dirを追加
```diff toml:wrangler.toml
+ migrations_dir = "drizzle/migrations"
```
migration実行
```
$ npx wrangler d1 migrations apply my-next-app-db --local

Migrations to be applied:
┌──────────────────────┐
│ name                 │
├──────────────────────┤
│ 0000_far_smasher.sql │
└──────────────────────┘
✔ About to apply 1 migration(s)
Your database may not be available to serve requests during the migration, continue? … yes
🌀 Mapping SQL input into an array of statements
🌀 Executing on local database my-next-app-db (DB) from .wrangler/state/v3/d1:
┌──────────────────────┬────────┐
│ name                 │ status │
├──────────────────────┼────────┤
│ 0000_far_smasher.sql │ ✅       │
└──────────────────────┴────────┘
```
## CRUD処理
Honoで処理を書いてローカルD1に繋いで動かしてみる。
```ts:route.ts
/**
 * get users
 */
app.get("/users", async (c) => {
  const db = drizzle(process.env.DB);
  const result = await db.select().from(users).all();
  return c.json(result);
});

/**
 * create users
 */
app.post("/users", async (c) => {
  const params = await c.req.json<typeof users.$inferSelect>();
  const db = drizzle(process.env.DB);
  const result = await db
    .insert(users)
    .values({
      userName: params.userName,
    })
    .execute();
  return c.json(result);
});

export const GET = handle(app);
export const POST = handle(app);
```
作成したPOSTとGETを実行してみる
https://github.com/bajji-corporation/capturex-web/assets/35623457/1fa2c643-362c-4083-b194-3a2c94aa27bd

https://github.com/bajji-corporation/capturex-web/assets/35623457/1a218bdc-0e5f-45c5-8f7d-25a5544bcec7
# まとめ
さくっとCloudflareでフルスタック構成を試せた。
次はSSRなどパフォーマンスの観点からも調査していきたい。