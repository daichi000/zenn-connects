---
title: "Cloudflare Pages + Next.js + Hono + D1 + Drizzleで始めるフルスタック構成"
emoji: "🏔️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "nextjs", "d1", "hono", "drizzle"]
published: false
---

# はじめに
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

```schema.sql
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
```app/api/[[...route]]/route.ts
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

# Drizzle導入