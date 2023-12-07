---
title: "Cloudflare Pages + Next.js + Hono + D1 + drizzleで始めるフルスタック構成"
emoji: "🏔️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "typescript", "nextjs", "d1", "hono", "drizzle"]
published: false
---

# はじめに
個人開発でWebサービスを作るにあたり、CloudflareやHonoなど普段の業務では縁のない技術を試してみた。
いったんCRUD操作ができるところまでをまとめてみる。

## Cloudflare Pagesとは
Cloudflare社が提供するJAMstackプラットフォーム。
Freeプランで月500回までビルド可能、bandwidthが無制限と無料枠が充実しているのが特徴。
* Next.jsのホスティングといえばVercelだけど商用利用で$20/月は個人だと痛手。。
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

## 事前準備
# [Cloudflareのサイト](https://www.cloudflare.com/ja-jp/)にてアカウントを作成
# プロジェクト作成などするために、CLIツール `wragler` をインストールする
```
$ npm install -g wrangler
$ wrangler --version
 ⛅️ wrangler 3.18.0
```

## Next.jsのセットアップ
なんと、C3(create-cloudflare CLI)を使うことで新規プロジェクトの作成からデプロイまで一気通貫で行うことができてしまう。
@cloudflare/next-on-pages
https://developers.cloudflare.com/pages/framework-guides/deploy-a-nextjs-site/

```
$ npm create cloudflare@latest my-next-app -- --framework=next

│ 
╰ Do you want to deploy your application?
  Yes / No
```
デプロイするを選択するとわずか1分ほどでホスティング完了。
https://github.com/bajji-corporation/capturex-web/assets/35623457/0a5e0eec-8dcd-4f93-b1a8-5d609fb605b6

ダッシュボードでもプロジェクトが確認できる
https://github.com/bajji-corporation/capturex-web/assets/35623457/dc1e6249-5ad4-4e25-97cc-f04d6b4c4c1a

## Honoのセットアップ
