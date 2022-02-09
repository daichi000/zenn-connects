---
title: "Next.js + react-query + react-testing-libraryでモックサーバーからデータを取得するテストを書く"
emoji: "🍙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "react", "nextjs", "test"]
published: true
---
# 概要
本記事ではSSG + prefetch + Client Side Fetching(react-query)におけるテストを紹介します。
SEOを意識したい、かつリアルタイムで情報を更新したいようなサービスだと上記のようなレンダリング方法が採用されるかと思います。

# 実行環境
・Next 11.1.2
・react-query 3.34.12
・@testing-library/react 12.1.2
・jest 27.4.7

# テストしたいページ
SSG + prefetch + CSFのページを用意します。
ビルド時に呼ばれるgetStaticPropsでAPIを叩き、レスポンスをpropsに渡しています。

その後クライアントでreact-queryによるfetchが走りますが、ここではinitialDataとして先程のpropsで渡されたデータを入れているのでそちらがpostsDataとしてレンダリングされます。

`refetchOnMount: true` のオプションを追加しているので、レンダリング後再度react-queryのfetchが走り、postsDataが上書きされるという流れになります。

```tsx:posts-page.tsx
interface STATICPROPS {
  posts: POST[]
}

const PostsPage: React.FC<STATICPROPS> = ({ posts }) => {
  const { data: postsData, isError, isLoading } = useQuery<POST[], Error>('PostsList', getPostsList, {
    initialData: posts,
    refetchOnMount: true,
  });

  return (
    <ul>{postsData && postsData.map((post) => <Post key={post.id} {...post} />)}</ul>
  )
}
export default PostsPage

export const getStaticProps: GetStaticProps = async () => {
  const posts = await getAllPostsData()
  return {
    props: { posts },
  }
}
```
```ts:fetch.ts
export const getAllPostsData = async () => {
  const res = await fetch(
    new URL('https://myapi.com/posts/?offset=0&limit=50')
  )
  const posts = await res.json()
  return posts
}
```
```ts:postsList.ts
export const getPostsList = async () => {
  const res = await api
    .get("/posts", {
      params: {
        offset: 0,
        limit: 50,
      },
    })
    .catch((err) => {
      return err;
    });

  return res.data;
};
```
`fetch.ts` と `postsList.ts` は同様のAPIを叩く処理になりますが、axiosをサーバーサイドで実行しようとするとエラーになるためfetch関数で分けて定義する必要があります。
# テストコード
## mockサーバーを立てる
テストでfetch処理を書くにはmockサーバーを立てて擬似的なレスポンスを用意する必要があります。
:::message
URLをパラメータ付きで書くとwarningが出るので下記のようにパラメータの値で分岐させます。
:::

```ts:PostsPage.test.tsx
const handlers = [
  rest.get(
    'https://myapi.com/posts/',
    (req, res, ctx) => {
      const query = req.url.searchParams
      const offset = query.get("offset")
      const limit = query.get("limit")
      if (limit === '50') {
        return res(
          ctx.status(200),
          ctx.json([
            {
              id: 1,
              title: 'title1',
              description: 'description1',
              userId: 1,
              createdAt: new Date(),
              updatedAt: new Date(),
            }
          ])
        )
      }
    }
  )
]

const server = setupServer(...handlers);
beforeAll(() => {
  server.listen();
});
afterEach(() => {
  server.resetHandlers();
  cleanup();
});
afterAll(() => {
  server.close();
});
```
## SSG + prefetchのテスト
初回レンダリングで先程定義したjsonデータが返っているかどうか確認します。

```ts:PostsPage.test.tsx
describe('Posts Page Static', () => {
  it('Pre-fetched by getStaticProps', async () => {
    const { page } = await getPage({
      route: '/',
    });
    render(page);
    expect(await screen.findByText('title1')).toBeInTheDocument()
  })
})
```
mockで定義したtitleの値が画面に描画されていることが確認できます。
なお `getPage` は `next-page-tester` のライブラリの関数です。

## CSFのテスト
今回はprefetchで取得したデータがpropsとして渡されinitialDataとして表示されますので、そちらのデータをstaticPropsとして定義します。
その後CSFが走りデータが上書きされることを確認したいので、先程のmockサーバーで定義したものとは別の値を返すようにします。

```ts:PostsPage.test.tsx
describe('Home Page / useQuery', () => {
  const queryClient = new QueryClient()
  let staticProps: Posts[] = [
    {
      id: 2,
      title: 'title2',
      description: 'description2',
      userId: 2,
      createdAt: new Date(),
      updatedAt: new Date(),
    }
  ];
...
```
下記のテストでstaticPropsのデータが表示後にCSFでmockサーバーで定義した擬似データが返っていることが確認できます。
テストページのコンポーネントを `QueryClientProvider` でラップすることを忘れないようにします。

```tsx:PostsPage.test.tsx
it('CSF', async () => {
  render(
    <QueryClientProvider client={queryClient}>
      <PostsPage posts={staticProps} />
    </QueryClientProvider>
  )

  expect(await screen.findByText('title2')).toBeInTheDocument(); //<- staticProps(SSG)
  expect(await screen.findByText('title1')).toBeInTheDocument(); //<- CSF
});
```

# おわり
重い腰を上げてフロントエンドのテストを書き始めたので知見をまとめていこうと思います。
だれかの参考になったら幸いです
次はCookie周りのテストも書けたらいいな、、