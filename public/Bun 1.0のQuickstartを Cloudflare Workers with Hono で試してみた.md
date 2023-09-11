---
title: Bun 1.0 の Quickstart を Cloudflare Workers with Hono で試してみた
tags:
  - "Bun"
  - "Hono"
  - "cloudflare"
  - "CloudflareWorkers"
  - "TypeScript"
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
---

## はじめに :rainbow:

Bun の 1.0 がリリースされたということで、Quickstart を試してみました。

https://bun.sh/docs/quickstart

ただなぞるだけだけ、というのも面白くないので

> 『Hono は Bun サポートを「謳った」初めての Web フレームワークです。』

https://zenn.dev/yusukebe/articles/efa173ab4b9360

ということで、Cloudflare Workers with Hono コードを置き換えて deploy までしてみました。
作業ログの側面もあり、実行結果を多めに載せています。

それでは、やっていきましょうか。[^1]

[^1]: インターネットの皆さんこんにちは by [のばまんゲームス](https://dic.pixiv.net/a/%E3%81%AE%E3%81%B0%E3%81%BE%E3%82%93%E3%82%B2%E3%83%BC%E3%83%A0%E3%82%B9)

### 環境 :earth_asia:

- `WSL2` Ubuntu 22.04
- `Bun` 1.0
- `figlet` 1.6.0
- `hono` 3.5.8
  - 現在 2023/09/11 の最新バージョンは 3.6.0
- `wrangler` 3.7.0

### 成果物 :trophy:

https://github.com/TellMin/bun-hono

それでは、やっていきましょう。

## Bun セットアップ 🧑‍🍳

まずは Bun の インストールから。

https://bun.sh/docs/installation

筆者は WSL2 Ubuntu 22.04 を利用しているので `curl` でインストールします。

```Batch
$ curl -fsSL https://bun.sh/install | Batch

error: unzip is required to install bun (see: https://github.com/oven-sh/bun#unzip-is-required)
```

ということで `unzip` 、ついでに `zip` もインストールします。

```Batch
$ sudo apt-get install zip unzip

[sudo] password for ユーザー名:
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  unzip zip
0 upgraded, 2 newly installed, 0 to remove and 44 not upgraded.
Need to get 350 kB of archives.
After this operation, 929 kB of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 unzip amd64 6.0-26ubuntu3.1 [174 kB]
Get:2 http://archive.ubuntu.com/ubuntu jammy/main amd64 zip amd64 3.0-12build2 [176 kB]
Fetched 350 kB in 2s (191 kB/s)
Selecting previously unselected package unzip.
(Reading database ... 31435 files and directories currently installed.)
Preparing to unpack .../unzip_6.0-26ubuntu3.1_amd64.deb ...
Unpacking unzip (6.0-26ubuntu3.1) ...
Selecting previously unselected package zip.
Preparing to unpack .../zip_3.0-12build2_amd64.deb ...
Unpacking zip (3.0-12build2) ...
Setting up unzip (6.0-26ubuntu3.1) ...
Setting up zip (3.0-12build2) ...
Processing triggers for man-db (2.10.2-1) ...
Processing triggers for mailcap (3.70+nmu1ubuntu1) ...
```

再度 Bun のインストールを試みます。

```Batch
$ curl -fsSL https://bun.sh/install | Batch

######################################################################## 100.0%
bun was installed successfully to ~/.bun/bin/bun

Added "~/.bun/bin" to $PATH in "~/.Batchrc"

To get started, run:

 source /home/tellmin/.Batchrc
  bun --help
```

このまま bun コマンドを実行しようとすると下記エラーが発生します。

```Batch
$ bun --help
Command 'bun' not found, did you mean:
  command 'zun' from deb python3-zunclient (4.4.0-0ubuntu1)
  command 'bus' from deb atm-tools (1:2.5.1-4build2)
  command 'ben' from deb ben (0.9.2ubuntu5)
  command 'bup' from deb bup (0.32-3build2)
Try: sudo apt install <deb name>
```

ここは `Added "~/.bun/bin" to $PATH in "~/.Batchrc"` のメッセージに従い、パスの追加を行います。

```Batch
export BUN_INSTALL="/home/YOUR_USERNAME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"
```

パスの追加方法については、次の記事を参考にしました。

https://stackoverflow.com/questions/72902668/bun-not-found-after-running-installation-script

パスの追加が完了したら、再度 bun コマンドを実行します。

```Batch
$ bun --help

        -h, --help                              Display this help and exit.
        -b, --bun                               Force a script or package to use Bun's runtime instead of Node.js (via symlinking node)
            --cwd <STR>                         Absolute path to resolve files & entry points from. This just changes the process' cwd.
        -c, --config <PATH>?                    Config file to load bun from (e.g. -c bunfig.toml
            --extension-order <STR>...          Defaults to: .tsx,.ts,.jsx,.js,.json
            --jsx-factory <STR>                 Changes the function called when compiling JSX elements using the classic JSX runtime
            --jsx-fragment <STR>                Changes the function called when compiling JSX fragments
            --jsx-import-source <STR>           Declares the module specifier to be used for importing the jsx and jsxs factory functions. Default: "react"
            --jsx-runtime <STR>                 "automatic" (default) or "classic"
        -r, --preload <STR>...                  Import a module before other modules are loaded
            --main-fields <STR>...              Main fields to lookup in package.json. Defaults to --target dependent
            --no-summary                        Don't print a summary (when generating .bun)
        -v, --version                           Print version and exit
            --revision                          Print version with revision and exit
            --tsconfig-override <STR>           Load tsconfig from path instead of cwd/tsconfig.json
        -d, --define <STR>...                   Substitute K:V while parsing, e.g. --define process.env.NODE_ENV:"development". Values are parsed as JSON.
        -e, --external <STR>...                 Exclude module from transpilation (can use * wildcards). ex: -e react
        -l, --loader <STR>...                   Parse files with .ext:loader, e.g. --loader .js:jsx. Valid loaders: js, jsx, ts, tsx, json, toml, text, file, wasm, napi
        -u, --origin <STR>                      Rewrite import URLs to start with --origin. Default: ""
        -p, --port <STR>                        Port to serve bun's dev server on. Default: "3000"
            --smol                              Use less memory, but run garbage collection more often
            --minify                            Minify (experimental)
            --minify-syntax                     Minify syntax and inline data (experimental)
            --minify-whitespace                 Minify whitespace (experimental)
            --minify-identifiers                Minify identifiers
            --no-macros                         Disable macros from being executed in the bundler, transpiler and runtime
            --target <STR>                      The intended execution environment for the bundle. "browser", "bun" or "node"
            --inspect <STR>?                    Activate Bun's Debugger
            --inspect-wait <STR>?               Activate Bun's Debugger, wait for a connection before executing
            --inspect-brk <STR>?                Activate Bun's Debugger, set breakpoint on first line of code and wait
            --hot                               Enable auto reload in bun's JavaScript runtime
            --watch                             Automatically restart bun's JavaScript runtime on file change
            --no-install                        Disable auto install in bun's JavaScript runtime
        -i                                      Automatically install dependencies and use global cache in bun's runtime, equivalent to --install=fallback
            --install <STR>                     Install dependencies automatically when no node_modules are present, default: "auto". "force" to ignore node_modules, fallback to install any missing
            --prefer-offline                    Skip staleness checks for packages in bun's JavaScript runtime and resolve from disk
            --prefer-latest                     Use the latest matching versions of packages in bun's JavaScript runtime, always checking npm
            --silent                            Don't repeat the command for bun run
            --dump-environment-variables        Dump environment variables from .env and process as JSON and quit. Useful for debugging
            --dump-limits                       Dump system limits. Useful for debugging

-------

Bun: a fast JavaScript runtime, package manager, bundler and test runner. (1.0.0)

  run       ./my-script.ts       Run JavaScript with Bun, a package.json script, or a bin
  test                           Run unit tests with Bun
  x         nuxi                 Install and execute a package bin (bunx)
  repl                           Start a REPL session with Bun

  init                           Start an empty Bun project from a blank template
  create    next-app             Create a new project from a template (bun c)

  install                        Install dependencies for a package.json (bun i)
  add       hono                 Add a dependency to package.json (bun a)
  remove    browserify           Remove a dependency from package.json (bun rm)
  update    react                Update outdated dependencies & save to package.json
  link                           Link an npm package globally
  unlink                         Globally unlink an npm package
  pm                             More commands for managing packages

  build     ./a.ts ./b.jsx       Bundle TypeScript & JavaScript into a single file

  upgrade                        Get the latest version of Bun
  bun --help                     Show all supported flags and commands

  Learn more about Bun:          https://bun.sh/docs
  Join our Discord community:    https://bun.sh/discord

```

よさそうですね。
では Quickstart を試してみましょう。

## Quickstart :stopwatch:

### プロジェクトの初期化 :clapper:

まずはプロジェクトの初期化を行います。
ここでは `bun-hono` というプロジェクト名で作成しました。

```Batch
$ bun init

bun init helps you get started with a minimal project and tries to guess sensible defaults. Press ^C anytime to quit

package name (bun-hono): bun-hono
entry point (index.ts): index.ts

Done! A package.json file was saved in the current directory.
 + index.ts
 + .gitignore
 + tsconfig.json (for editor auto-complete)
 + README.md

To get started, run:
  bun run index.ts
```

### サーバーが起動するまで :movie_camera:

まずはそのまま`index.ts` を実行してみます。

```Batch
$ bun run index.ts
Hello via Bun!
```

`Hello via Bun!` と表示されました。
続いて、`index.ts` を編集します。

```typescript
const server = Bun.serve({
  port: 3000,
  fetch(req) {
    return new Response(`Bun!`);
  },
});

console.log(`Listening on http://localhost:${server.port} ...`);
```

`index.ts` を編集したら、再度実行してみます。

```Batch
$ bun index.ts
Listening at http://localhost:3000 ...
```

これだけでサーバーが起動するのはなんだか凄い気がします。
わくわくしてきましたね。 :sparkles:

### package.json に scripts を追加 :milky_way:

`npm` を利用するときと同様、`package.json` に `scripts` を追加することで、任意の command を登録できます。

```javascript
{
  "scripts": {
    "start": "bun run index.ts"
  }
}
```

### Install a package :inbox_tray:

お次はパッケージのインストールです。
`npm` と比べ、かなり速いという噂ですので、期待が高まります。

まずはアスキーアートを表示するためのパッケージをインストールします。

```Batch
$ bun add figlet

bun add v1.0.0 (822a00c4)

 installed figlet@1.6.0 with binaries:
  - figlet


 1 packages installed [809.00ms]

$ bun add -d @types/figlet # TypeScript users only

bun add v1.0.0 (822a00c4)

 installed @types/figlet@1.5.6


 1 packages installed [1433.00ms]
```

速いです。:bullettrain_side:
普段`pnpm`を利用しているのですが、パッケージ管理だけ `Bun` に寄せるのもありかもしれませんね。

さて、チュートリアルに従って `index.ts` を編集します。

```diff_typescript
+import figlet from "figlet";

 const server = Bun.serve({
   fetch() {
+    const body = figlet.textSync('Bun!');
+    return new Response(body);
-    return new Response(`Bun!`);
   },
   port: 3000,
 });
```

実行すると、アスキーアートが表示されました。

```Batch
  ____              _
 | __ ) _   _ _ __ | |
 |  _ | | | | '_ | |
 | |_) | |_| | | | |_|
 |____/ __,_|_| |_(_)
```

Good! :+1:
（エディタ上ではきれいに表示されていませんが、、、）

## Cloudflare Workers with Hono に置き換える 🔄

さて、本題の `Hono` を導入してみましょう。
`bun create hono my-app` をする方が都合が良い気もしますが、今回は後から`Hono`を導入する形で進めてみます。

https://hono.dev/getting-started/bun

### Hono のインストール :fire:

まずは`Hono`をインストールします。
同時に`@cloudflare/workers-types`, `wrangler`もインストールします。

```Batch
$ bun add hono

bun add v1.0.0 (822a00c4)

 installed hono@3.5.8


 1 packages installed [1256.00ms]

$ bun add -d @cloudflare/workers-types wrangler

bun add v1.0.0 (822a00c4)

 installed @cloudflare/workers-types@4.20230904.0
 installed wrangler@3.7.0 with binaries:
  - wrangler
  - wrangler2


 106 packages installed [4.49s]
```

`scripts`も、`wrangler`を利用するように変更しておきましょう。

```diff_javascript
 {
   "scripts": {
-    "start": "bun run index.ts"
+    "start": "wrangler dev index.ts",
+    "deploy": "wrangler deploy --minify index.ts"
   }
 }
```

### index.ts を Hono に置き換える :dizzy:

では `index.ts`を編集し、`Hono`を利用するようにします。

```typescript
import figlet from "figlet";
import { Hono } from "hono";

const app = new Hono();

const body = figlet.textSync("Bun!");

app.get("/", (c) => c.text(body));

export default app;
```

また、`wrangler.toml`を生成しておきましょう。
私の場合は直接ファイルを作成しました。

```toml
name = "bun-hono"
compatibility_date = "2023-01-01"
```

これで準備は完了、のはずでしたが、`Bun run dev` を実行したところ、下記エラーが発生しました。

```Batch
$ Bun run dev

service core:user:bun-hono: Uncaught Error: synchronous font loading is not implemented for the browser
  at index.js:978:15 in me.loadFontSync
  at index.js:847:43 in me.textSync
  at index.js:2610:34
✘ [ERROR] MiniflareCoreError [ERR_RUNTIME_FAILURE]: The Workers runtime failed to start. There is likely additional logging output above.
```

どうやら、cloudflare workers では同期的なフォントの読み込みができないようです。
ランタイムがブラウザライクであることによる制約のようです。
このあたりについてはもう少し学習が必要そうです。

今回は github の公式 Doc を参考に、あらかじめフォントを読み込んでおくことにしました。

https://github.com/patorjk/figlet.js/#getting-started---webpack--react

```diff_typescript
 import figlet from "figlet";
+ import standard from "figlet/importable-fonts/Standard";
 import { Hono } from "hono";

 const app = new Hono();

+ figlet.parseFont("Standard", standard);
 const body = figlet.textSync("Bun!");

 app.get("/", (c) => c.text(body));

 export default app;
```

このままでは 型情報が足りずに警告が発生してしまいます。
今回は次のような型定義ファイルを作成しました。

```typescript: figlet.d.ts
declare module "figlet/importable-fonts/Standard" {
  const font: string;
  export = font;
}
```

### Deploy :rocket:

折角ですので Cloudflare Workers に deploy してみましょう。

```Batch
$ bun run deploy

 ⛅️ wrangler 3.7.0
------------------
Total Upload: 60.80 KiB / gzip: 17.08 KiB
Uploaded bun-hono (1.04 sec)
Published bun-hono (3.88 sec)
  https://~
Current Deployment ID: ~
```

無事に deploy できました。
アクセスしてみると

![Bun.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2566826/34173cfd-dee8-4cd8-06a3-f9dd653693fc.png)

無事にアスキーアートが表示されました。:tada:

## まとめ :memo:

Node.js を過去の物にする最速の肉まん、恐るべし :scream:
(最速の肉まんという語感、好きです)

https://qiita.com/shadowTanaka/items/5fb99819629dcaab3e05

`Bun` 専用のデプロイ環境もロードマップにはあるそうです。
この先も目が離せませんね。

次は `SvelteKit` プロジェクト を `Bun` , `Docekr`, `ECS` あたりで deploy してみたいと思います。

それでは、また今度 :wave:
