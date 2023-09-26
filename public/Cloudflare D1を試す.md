---
title: Cloudflare D1 を試そう with SvelteKit
tags:
  - cloudflare
  - Svelte
  - TypeScript
  - Bun
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
---

# はじめに :sparkles:

Cloudflare D1 は、Cloudflare が提供する、Cloudflare Workers 用に設計されたデータベースです。
その特徴については、下記記事が大変よくまとまっています。
D1 ってなに？って方はまずこちらを読んでみることをおすすめします。

https://qiita.com/yoshii0110/items/d82c158c40bd146e7fcf

今回は、Cloudflare D1 と SvelteKit を組み合わせて、簡単にハンズオンを行ってみたいと思います。

## 検証環境 :computer:

- WSL2 Ubuntu 22.04
- bun 1.0.0

`package.json` は最終的に次のようになりました。

```json
  "devDependencies": {
    "@sveltejs/adapter-auto": "^2.0.0",
    "@sveltejs/adapter-cloudflare-workers": "^1.1.4",
    "@sveltejs/kit": "^1.20.4",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "drizzle-kit": "^0.19.13",
    "drizzle-orm": "^0.28.6",
    "eslint": "^8.28.0",
    "eslint-config-prettier": "^8.5.0",
    "eslint-plugin-svelte": "^2.30.0",
    "prettier": "^2.8.0",
    "prettier-plugin-svelte": "^2.10.1",
    "svelte": "^4.0.5",
    "svelte-check": "^3.4.3",
    "tslib": "^2.4.1",
    "typescript": "^5.0.0",
    "vite": "^4.4.2",
    "wrangler": "^3.9.0"
  },
```

ORM については必須ではありませんが、今回は Drizzle ORM を利用しました。

## プロジェクトの作成 :hammer:

今回は svelte-kit のテンプレートを利用してプロジェクトを作成していきます。

```bash
bun create svelte@latest d1-playground
cd d1-playground
bun install

bun install v1.0.0 (822a00c4)
 + @sveltejs/adapter-auto@2.1.0
 + @sveltejs/kit@1.25.0
 + @typescript-eslint/eslint-plugin@6.7.2
 + @typescript-eslint/parser@6.7.2
 + eslint@8.50.0
 + eslint-config-prettier@8.10.0
 + eslint-plugin-svelte@2.33.2
 + prettier@2.8.8
 + prettier-plugin-svelte@2.10.1
 + svelte@4.2.1
 + svelte-check@3.5.2
 + tslib@2.6.2
 + typescript@5.2.2
 + vite@4.4.9

 218 packages installed [3.60s]
```

次に `wrangler` をインストールします。

```bash
bun add -D wrangler

bun add v1.0.0 (822a00c4)

 installed wrangler@3.9.0 with binaries:
  - wrangler
  - wrangler2


 38 packages installed [2.84s]
```

同時に `wrangler.toml` も作成します。

```toml
name = "d1-playground"

main = "./.cloudflare/worker.js"
site.bucket = "./.cloudflare/public"

build.command = "npm run build"

compatibility_date = "2023-01-01"
workers_dev = true
```

これで準備は完了です。

### データベースの作成 :floppy_disk:

D1 にデータベースを作成する場合は、`wrangler d1`コマンドを利用します。

```bash
bun wrangler d1 create d1-playground

--------------------
🚧 D1 is currently in open alpha and is not recommended for production data and traffic
🚧 Please report any bugs to https://github.com/cloudflare/workers-sdk/issues/new/choose
🚧 To request features, visit https://community.cloudflare.com/c/developers/d1
🚧 To give feedback, visit https://discord.gg/cloudflaredev
--------------------

✅ Successfully created DB 'd1-playground' in region APAC
Created your database using D1's new storage backend. The new storage backend is not yet recommended for
production workloads, but backs up your data via point-in-time restore.

[[d1_databases]]
binding = "DB" # i.e. available in your Worker on env.DB
database_name = "d1-playground"
database_id = "YOUR_DATABASE_ID"
```

`wrangler.toml` にも追記しましょう。

```toml
[[d1_databases]]
binding = "DB" # i.e. available in your Worker on env.DB
database_name = "d1-playground"
database_id = "YOUR_DATABASE_ID"
```

## drizzle cli と drizzle-orm を追加 :wrench:

D1 にテーブルを作成する際に ORM は必須ではありません。
そのため、この章は読み飛ばしていただいて構いません。
今回は Drizzle ORM を利用して、簡単にデータベースを操作できるようにしていきます。
詳しくは下記記事が参考になります。

https://zenn.dev/mizchi/articles/d1-drizzle-orm

```bash
bun add drizzle-kit drizzle-orm -D

bun add v1.0.0 (822a00c4)

 installed drizzle-kit@0.19.13 with binaries:
  - drizzle-kit
 installed drizzle-orm@0.28.6


 44 packages installed [3.27s]
```

スキーマの定義は参考記事よりお借りしています。

```typescript
/*
  DO NOT RENAME THIS FILE FOR DRIZZLE-ORM TO WORK
*/
import { sqliteTable, text, integer } from "drizzle-orm/sqlite-core";

export const users = sqliteTable("users", {
  id: integer("id").primaryKey().notNull(),
  name: text("name").notNull(),
});
```

### マイグレーションの実行 :hammer_and_wrench:

参考記事では`better-sqlite3`がマイグレーションに必要とされていますが、私の環境ではインストールせずに実行できました。
もし理由をご存じのかたがいらっしゃいましたら、ご教授いただけると幸いです。

```bash
bun drizzle-kit generate:sqlite --out migrations --schema src/schema.ts

drizzle-kit: v0.19.13
drizzle-orm: v0.28.6

1 tables
users 2 columns 0 indexes 0 fks

[✓] Your SQL migration file ➜ migrations/0000_fast_patch.sql 🚀
```

マイグレーションの実行

```bash
bun wrangler d1 migrations apply d1-playground --local

▲ [WARNING] Processing wrangler.toml configuration:

    - D1 Bindings are currently in alpha to allow the API to evolve before general availability.
      Please report any issues to https://github.com/cloudflare/workers-sdk/issues/new/choose
      Note: Run this command with the environment variable NO_D1_WARNING=true to hide this message

      For example: `export NO_D1_WARNING=true && wrangler <YOUR COMMAND HERE>`


--------------------
🚧 D1 is currently in open alpha and is not recommended for production data and traffic
🚧 Please report any bugs to https://github.com/cloudflare/workers-sdk/issues/new/choose
🚧 To request features, visit https://community.cloudflare.com/c/developers/d1
🚧 To give feedback, visit https://discord.gg/cloudflaredev
--------------------

Migrations to be applied:
┌─────────────────────┐
│ name                │
├─────────────────────┤
│ 0000_fast_patch.sql │
└─────────────────────┘
✔ About to apply 1 migration(s)
Your database may not be available to serve requests during the migration, continue? … yes
🌀 Mapping SQL input into an array of statements
🌀 Loading YOUR_DATABASE_ID from .wrangler/state/v3/d1
┌─────────────────────┬────────┐
│ name                │ status │
├─────────────────────┼────────┤
│ 0000_fast_patch.sql │ ✅       │
└─────────────────────┴────────┘
```

これで ローカルにテーブルが作成されました。
ここで動作確認をしたい方は、次のコマンドで任意の SQL を実行することが可能です。

```bash
wrangler d1 execute d1-playground --local --command='insert into users (name) values ("test")'
wrangler d1 execute d1-playground --local --command='select * from users'
```

## SvelteKit の構築 :rocket:

まずは Cloudflare Workers 上で実行するための設定を行います。
※ wrangler を利用してローカルで動作確認する場合も必要です。

https://kit.svelte.jp/docs/adapter-cloudflare-workers

### アダプターの追加と設定 :gear:

```bash
bun add -D @sveltejs/adapter-cloudflare-workers
```

```typescript
import adapter from "@sveltejs/adapter-cloudflare-workers";

export default {
  kit: {
    adapter: adapter(),
  },
};
```

さて、ここからは SvelteKit の実装を行っていきます。

### D1 クライアントの作成 :electric_plug:

`drizzle-orm/d1` を利用して D1 クライアントを作成します。

client を取得するためには 次のようなコードを記述します。

```typescript
import { drizzle } from 'drizzle-orm/d1';

export interface Env {
  <BINDING_NAME>: D1Database;
}

export default {
  async fetch(request: Request, env: Env) {
    const db = drizzle(env.<BINDING_NAME>);
    const result = await db.select().from(users).all()
    return Response.json(results);
  },
};
```

https://orm.drizzle.team/docs/quick-sqlite/d1

今回は SvelteKit を利用しているため、環境変数を取得する場合は、`platform` プロパティを経由する必要があります。

```typescript
export async function POST({ platform }) {
  const DB = platform?.env?.DB;
}
```

https://kit.svelte.jp/docs/adapter-cloudflare-workers#environment-variables

また、`platform`から取得する変数の型についても、定義する必要があります。

```typescript app.d.ts
// See https://kit.svelte.dev/docs/types#app
// for information about these interfaces
declare global {
  namespace App {
    // interface Error {}
    // interface Locals {}
    // interface PageData {}
    interface Platform {
      env?: {
        DB: D1Database;
      };
    }
  }
}

export {};
```

D1 クライアントを作成するためには、環境変数から取得した `platform?.env?.DB` を `drizzle` 関数の引数に渡します。
今回は `createD1Client` という関数を作成して、`d1client.ts` にまとめておきます。

```typescript
import { drizzle } from "drizzle-orm/d1";

export const createD1Client = (d1: D1Database | undefined) => {
  if (!d1) {
    throw new Error(`Database not found`);
  }
  return drizzle(d1);
};
```

では、実際にデータ取得、データ更新を実装してみましょう。

### データ取得 :mag:

データ取得は下記のように実装できます。

```typescript
import { createD1Client } from "$lib/server/d1client";
import { users } from "../schema";

const d1Client = createD1Client(platform?.env?.DB);
const result = await d1Client.select().from(users).all();
```

### データ更新 :pencil2:

データ更新は下記のように実装できます。

```typescript
import { createD1Client } from "$lib/server/d1client";
import { users } from "../schema";

const d1Client = createD1Client(platform?.env?.DB);
const result = await d1Client.insert(users).values({ name: "John" }).execute();
```

メソッドチェーンを利用してクエリを記述できる点は、 ORM を利用する場合のメリットですね。

## 全体の実装 :space_invader:

最後に実装全体をまとめておきます。
初期ロード時にデータを取得し、フォームからデータを更新することができるようになりました。

注意としては、`bun dev` ではなく `wrangler dev` を実行する必要があります。
お好みで`Script`を編集するとよいでしょう。

### +page.svelte :flower_playing_cards:

```typescript
<script lang="ts">
  import type { PageData, ActionData } from "./$types";
  export let data: PageData;
  export let form: ActionData;
</script>

<h1>Regist User</h1>

{#if form && !form.success}
  <p>{form.error}</p>
{/if}

<form method="POST">
  <label>Name<input name="name" type="text" /></label>
  <button type="submit">Submit</button>
</form>

{#each data.users as user}
  <p>{user.id}:{user.name}</p>
{/each}
```

### +page.server.ts :art:

```typescript
import type { PageServerLoad, Actions } from "./$types";
import { createD1Client } from "$lib/server/d1client";
import { users } from "../schema";

export const load = (async ({ platform }) => {
  const d1Client = createD1Client(platform?.env?.DB);
  const result = await d1Client.select().from(users).all();
  return { users: result };
}) satisfies PageServerLoad;

export const actions = {
  default: async ({ platform, request }) => {
    const data = await request.formData();
    const name = data.get("name");
    if (!name) {
      return { success: false, error: "Name is required" };
    }
    const d1Client = createD1Client(platform?.env?.DB);
    await d1Client.insert(users).values({ name: name.toString() }).execute();
    return { success: true };
  },
} satisfies Actions;
```

### d1client.ts :tropical_drink:

```typescript
import { drizzle } from "drizzle-orm/d1";

export const createD1Client = (d1: D1Database | undefined) => {
  if (!d1) {
    throw new Error(`Database not found`);
  }
  return drizzle(d1);
};
```

## まとめ :tomato:

今回は Cloudflare D1 と SvelteKit を組み合わせて、簡単な CRUD を実装してみました。
D1 はまだアルファ版ということもあり、プロダクト運用には向いていないかもしれませんが、趣味であれば関係ありません。
是非皆さんも試してみてはいかがでしょうか！
