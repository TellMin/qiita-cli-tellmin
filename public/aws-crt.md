---
title: Next.js の App Router 利用時 に 'aws-crt' が Module not found Can't resolve となる件
tags:
  - "Next.js"
  - "AWS"
  - "TypeScript"
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# なんの記事？

Next.js の App Router で AWS SDK を利用している場合に発生する警告について、暫定対応を記載します。

## 環境

以下に package.json の中身を記載します。

```json package.json
{
  "name": "nextjs-aws",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "@aws-sdk/client-dynamodb": "^3.431.0",
    "@aws-sdk/lib-dynamodb": "^3.431.0",
    "@aws-sdk/util-dynamodb": "^3.431.0",
    "next": "13.5.4",
    "next-auth": "^4.23.2",
    "react": "^18",
    "react-dom": "^18"
  },
  "devDependencies": {
    "typescript": "^5",
    "@types/node": "^20",
    "@types/react": "^18",
    "@types/react-dom": "^18",
    "autoprefixer": "^10",
    "postcss": "^8",
    "tailwindcss": "^3",
    "eslint": "^8",
    "eslint-config-next": "13.5.4"
  }
}
```

## どうすればいいの？

- AWS SDK の仕様上の問題
- `next.config.js` に特定の AWS SDK 関連のリソースを無視する設定を追加する
- 改善は次のメジャーリリースになる模様

## 詳細

aws-amplify のリポジトリに対し、次のような Issue が上がっています。

https://github.com/aws-amplify/amplify-js/issues/11030

恐らく、Next.js の App Router で AWS SDK を利用する場合に発生する問題だと思われます。

次の対応が提案されており、実際に私の環境でエラーが抑制されることが確認できました。

> ```javascript next.config.js
> /** @type {import('next').NextConfig} */
> const config = {
>   // …
>   webpack: (config, { webpack, isServer, nextRuntime }) => {
>     // Avoid AWS SDK Node.js require issue
>     if (isServer && nextRuntime === "nodejs")
>       config.plugins.push(
>         new webpack.IgnorePlugin({ resourceRegExp: /^aws-crt$/ })
>       );
>     return config;
>   },
>   // …
> };
>
> module.exports = config;
> ```

https://github.com/aws-amplify/amplify-js/issues/11030#issuecomment-1598207365

AWS エンジニアの方が次のようにコメントしています。

> Hello Josh, we apologize for the delay in resolving this issue. Based on our investigations, making the necessary enhancement requires breaking changes. We are currently working on introducing an improved experience for App Router as part of our next major version.

https://github.com/aws-amplify/amplify-js/issues/11030#issuecomment-1695900992

ということでメジャーアップデートで対応されるのを待つしかなさそうです。

## 私の場合

ちなみに私がこの問題に遭遇した際のエラーは次の通りです。

```bash
⚠ ./node_modules/@aws-sdk/util-user-agent-node/dist-cjs/is-crt-available.js
Module not found: Can't resolve 'aws-crt' in '/home/tellmin/dev/nextjs-aws/node_modules/@aws-sdk/util-user-agent-node/dist-cjs'

Import trace for requested module:
./node_modules/@aws-sdk/util-user-agent-node/dist-cjs/is-crt-available.js
./node_modules/@aws-sdk/util-user-agent-node/dist-cjs/index.js
./node_modules/@aws-sdk/client-dynamodb/dist-cjs/runtimeConfig.js
./node_modules/@aws-sdk/client-dynamodb/dist-cjs/DynamoDBClient.js
./node_modules/@aws-sdk/client-dynamodb/dist-cjs/index.js
./lib/dynamoDBClient.ts
./app/api/user/route.ts
{
  '$metadata': {
  httpStatusCode: 200,
  requestId: **************,
  extendedRequestId: undefined,
  cfId: undefined,
  attempts: 1,
  totalRetryDelay: 0
  }
}
```

`@aws-sdk/client-dynamodb` と `@aws-sdk/lib-dynamodb` を利用して DynamoDB に書き込みを実施しようとした際に発生しました。

次のブランチにコードを置いています。
参考までにどうぞ。

https://github.com/TellMin/nextjs-aws/tree/dynamoDB

## まとめ

全てが App Router に対応というわけではなさそうです。
`next.config.js`の設定や `webpack` についてもっと詳しくなりたいと思いました。
