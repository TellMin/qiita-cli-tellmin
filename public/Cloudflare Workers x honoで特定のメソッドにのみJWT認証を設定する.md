---
title: Cloudflare Workers x HonoでJWT認証を設定する
tags:
  - "CloudflareWorkers"
  - "Hono"
  - "JWT"
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
---

# はじめに :jack_o_lantern:

皆さん Hono はご存知でしょうか？:fire:

Cloudflare Workers 上で動作する軽量な Web フレームワークです。
Web Standard の API に準拠しているため、コードを読むだけでも学習になり、おすすめです。

今回は Hono を利用して CloudFlare Workers 上に OpenAI API のプロキシを作成しました。
JWT 認証を設定する際に、正しい方法が分からず、苦労したのでメモしておきます。

> 大変参考になった記事
> **[Hono と Cloudflare Workers で Azure OpenAI Service の proxy を作った話](https://zenn.dev/calldoctor_blog/articles/deb21728bbe99a)**

成果物はこちら => **[openai-proxy](https://github.com/TellMin/openai-proxy)**

## どう実現したか :zap:

### 1. 環境変数の取得

Hono のテンプレートからプロジェクトを作成した場合、デバッグ時に wrangler を利用していることが分かります。

```json
"dev": "wrangler dev src/index.ts"
```

環境変数を設定する場合、`wrangler.toml`に記載することも可能ですが、今回は`.dev.vars`に設定しました。

設定された環境変数は、次のように Bindings の型を定義することで、`Hono`の context 内で利用できるようになります。

> [Cloudflare Docs - Environmental variables](https://developers.cloudflare.com/workers/wrangler/configuration/#environmental-variables) > [Hono - Bindings](https://hono.dev/getting-started/cloudflare-workers#bindings)

```typescript
type Bindings = {
  OPENAI_API_KEY: string;
  SECRET: string;
};

const app = new Hono<{ Bindings: Bindings }>();
```

### 2. JWT 認証の設定 :old_key:

Hono では、`app.use`を利用した特定のルートにマッチするリクエストに対して、ミドルウェアを設定することができます。

```typescript
// https://hono.dev/middleware/builtin/jwt#usage より引用

const app = new Hono();

app.use(
  "/auth/*",
  jwt({
    secret: "it-is-very-secret",
  })
);

app.get("/auth/page", (c) => {
  return c.text("You are authorized");
});
```

このままでは secret を環境変数から取得することができないため、次のように実装する必要がありました。

```typescript
app.use(
  "*",
  async (
    c: Context<{
      Bindings: Bindings;
    }>,
    next: Next
  ) => jwt({ secret: c.env.SECRET })(c, next)
);
```

> [Hono - Using Variables in Middleware](https://hono.dev/getting-started/cloudflare-workers#using-variables-in-middleware)

typescript で実装したため、型定義を行う必要があります。
要領としてはカスタムミドルウェアを作成する場合と同様らしいです。

### プリフライトリクエストの許可 :small_airplane:

CORS を設定する場合、プリフライトリクエストを許可する必要があります。JWT 認証を行うためには Authorization ヘッダーが必要ですが、クライアントからリクエストを送信する場合、単純リクエストに該当しないため、プリフライトリクエストが送信されます。

> [CORS の理解を深める - Simple Requests(単純リクエスト)とは](<https://zenn.dev/riko/articles/cors_deepen_understanding#simple-requests(%E5%8D%98%E7%B4%94%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88)%E3%81%A8%E3%81%AF>)

そのため、Options メソッドが正しく処理されるように設定するとよいでしょう。

```typescript
app.options("*", (c) => {
  return new Response(null, {
    headers: {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "POST, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Authorization",
    },
  });
});
```

プリフライトリクエストはブラウザが自動的に送信する機能のため、サーバーサイドからのみ呼び出される場合は、この設定は不要です。
認識が誤っていたらご指摘よろしくお願いいたします。

## まとめ :sun_with_face:

JWT 認証の処理について、Hono/JWT の実装を読み込んだのですが、非常に参考になりました。

> [hono/src/middleware/jwt/index.ts](https://github.com/honojs/hono/blob/main/src/middleware/jwt/index.ts)

同様に、他のミドルウェアやコードについても Web Standard に準拠しているため、理解を深められるかと思います。

折角プロキシを作成したので、次は利用するクライアントアプリケーションについて記事を書きたいと思います。

Open AI API 周りの実装については触れませんでしたが、詳しくは Vercel の記事をご覧ください。

> [Docs - OpenAI](https://sdk.vercel.ai/docs/guides/providers/openai)

イカ、よろしくーーー！:octopus:
