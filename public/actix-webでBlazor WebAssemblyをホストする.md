---
title: actix-webでBlazor WebAssemblyをホストする
tags:
  - C#
  - Rust
  - Blazor
  - actix-web
private: false
updated_at: "2023-12-22T23:12:09+09:00"
id: 9e7f1e576a195103aa92
organization_url_name: null
slide: false
ignorePublish: false
---

# なんの記事？ :sparkles:

C#の Web フレームワーク [ASP.NET Core Blazor](https://learn.microsoft.com/ja-jp/aspnet/core/blazor/?view=aspnetcore-8.0&WT.mc_id=dotnet-35129-website) の WebAssembly プロジェクトを Rust の Web フレームワーク [actix-web](https://actix.rs/) でホストしてみました。

https://learn.microsoft.com/ja-jp/aspnet/core/blazor/hosting-models?view=aspnetcore-8.0#blazor-webassembly

あらかじめお断りしておきますが、バックエンドには ASP.NET Core アプリを使うのがおすすめです。
もし絶対にフロントエンドは C#で、バックエンドは rust で書きたいという場合は参考になるかもしれません。
ではどうぞ。

成果物

https://github.com/TellMin/actix-ecs

あ、参考になりましたら是非いいねをお願いします :heart:

## 開発環境

- cargo 1.74.0
- rustc 1.74.0
- rustup 1.26.0 (rustup だけ古いことに今気づきました...)
- actix-web = "4"
- actix-files = "0.6.2"
- .NET SDK 8.0.100

## Blazor WebAssembly について :cooking:

WebAssembly とは、簡単に言えばブラウザ上で動作するバイナリフォーマットです。

https://webassembly.org/

Blazor WebAssembly は、実行に必要なランタイムすらも wasm で実装することで、ブラウザ上で C# を動かすことができるようになります。
つまり完全にブラウザ上で実行される SPA アプリケーションが開発できます。

> Blazor アプリは、サーバーを使用せずに、ブラウザーで WebAssembly で実行されているアプリを完全にターゲットにすることができます。 "スタンドアロン Blazor WebAssembly アプリ" の場合、資産は、静的コンテンツをクライアントに提供できる Web サーバーまたはサービスに静的ファイルとして展開されます。

ということで、S3 や GitHub Pages, Cloudflare Pages などの静的ホスティングサービスにデプロイすることができます。

今回は出来心で actix-web によるホストを試してみてみようと思います。

## actix-web について :floppy_disk:

Rust の Web フレームワークです。

https://actix.rs/

静的ファイルをホストする方法については次のような記載があります。

https://actix.rs/docs/static-files/

`actix_web` に加え `actix_files` が必要ですので、`Cargo.toml` に追記してください。

では実装に移りましょう。

## 実装方針 :chocolate_bar:

ディレクトリ構成はざっくり次のようになっています。

```
.
├── src
│   └──main.rs  // actix-web のエントリポイント
├── aspnetwasm
│       ├── aspnetwasm.csproj
│       ├── Pages
│       │   ├── Counter.razor
│       │   ├── FetchData.razor
│       │   └── Index.razor
│       └──  Program.cs  // Blazor WebAssembly のエントリポイント
├── Cargo.lock
├── Cargo.toml
└── README.md
```

Blazor WebAssembly プロジェクトは `aspnetwasm` ディレクトリにあります。
ディレクトリ直下で `dotnet publish aspnetwasm.csproj -c Release` コマンドを実行することで、`wwwroot` ディレクトリに静的ファイルが出力されます。
このとき、`-o .././` を引数に渡すことで、`wwwroot` ディレクトリをプロジェクト直下に出力することができます。

Blazor WebAssembly をビルド後のディレクトリ構成は次のようになります。

```diff
.
├── src
│   └──main.rs  // actix-web のエントリポイント
├── aspnetwasm
│       ├── aspnetwasm.csproj
│       ├── Pages
│       │   ├── Counter.razor
│       │   ├── FetchData.razor
│       │   └── Index.razor
│       └──  Program.cs  // Blazor WebAssembly のエントリポイント
├── Cargo.lock
├── Cargo.toml
├── README.md
+ └── wwwroot // ビルド後の静的ファイル
+     ├── _framework
+     │   ├── blazor.boot.json
+     │   ├── blazor.webassembly.js
+     │   ├── dotnet.timezones.blat
+     │   ├── dotnet.wasm
+     │   └── などなど...
+     ├── css
+     │   └── app.css
+     └── index.html
```

さて、これらの静的ファイルを actix-web でホストするのですが、ここで問題があります。
Blazor WebAssembly は、`index.html` にアセットのパスを記載しています。
そこから必要なアセットを読み込むことで、アプリケーションが動作します。
そのため、SPA アプリケーション上の遷移は、`index.html` に、それ以外のリクエストには、`index.html` 以外のアセットを返すように実装する必要があります。

> Blazor WebAssembly アプリ内のページ コンポーネントに対するルーティング要求は、Blazor Server アプリでのルーティング要求のように単純なものではありません。 ~ 中略 ~
> ブラウザーではクライアント側ページの要求がインターネットベースのホストに対して行われるため、Web サーバーとホスティング サービスでは、サーバー上に物理的に存在しないリソースに対する index.html ページへのすべての要求を、書き換える必要があります。 index.html が返されると、アプリの Blazor ルーターがそれを受け取り、正しいリソースで応答します。

https://learn.microsoft.com/ja-jp/aspnet/core/blazor/host-and-deploy/webassembly?view=aspnetcore-8.0#rewrite-urls-for-correct-routing

上記課題を解決するために、静的ファイルのリクエスト以外には、`index.html` を返す方針で実装してみます。

## 実装 :zap:

ということで、`actix-web` と `actix-files` を使って実装してみます。

```rust
use actix_files as fs;
use actix_web::{get, App, HttpRequest, HttpServer, Result};
use std::path::PathBuf;

#[get("/")]
async fn index() -> Result<fs::NamedFile> {
    Ok(fs::NamedFile::open("wwwroot/index.html")?)
}

#[get("/{filename}")]
async fn pages(req: HttpRequest) -> Result<fs::NamedFile> {
    let path: PathBuf = req.match_info().query("filename").parse().unwrap();
    let file_name = path.file_name().unwrap().to_str().unwrap();

    // 拡張子がある場合はファイルとして扱う
    if file_name.contains(".") {
        let root_path = PathBuf::from("wwwroot");
        let file = fs::NamedFile::open(root_path.join(path)); // 要求を wwwroot/ にマッピングする
        if let Ok(file) = file {
            return Ok(file);
        }
    }
    Ok(fs::NamedFile::open("wwwroot/index.html")?)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(index) // 不要かもしれない
            .service(pages)
            .service(fs::Files::new("/", "wwwroot")) // 静的ファイルをホストする
    })
    .bind(("0.0.0.0", 8080))? // コンテナで動かすために 0.0.0.0 を指定してます
    .run()
    .await
}
```

キモは `pages` 関数です。
`{filename}` にマッチするリクエストは、`.` を含む場合はファイル要求として扱い、それ以外は `index.html` を返すことで Blazor ルーターに処理させます。

あとは `cargo run` で実行してアクセスできれば成功です！

## おまけの コンテナ化 :icecream:

マルチステージビルドを使って、コンテナ化してみます。

```Dockerfile
# https://hub.docker.com/_/microsoft-dotnet
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /source

# copy everything else and build app
COPY aspnetwasm/. ./aspnetwasm/
WORKDIR /source/aspnetwasm
RUN dotnet publish -c release -o /app

# final stage/image
FROM rust:1.71 as builder
WORKDIR /app
COPY . .
RUN cargo build --release
COPY --from=build /app/wwwroot ./wwwroot
EXPOSE 8080
ENTRYPOINT ["./target/release/actix-ecs"]
```

いろいろとまずい部分があり、サイズも大きめとなっています。
これらは先の課題としておきます :sweat_smile:

## おわりに :tada:

actix-web で Blazor WebAssembly をホストすることができました。
actix-web 側に api を実装して、Blazor WebAssembly から呼び出すこともできるかと思います。
実用的な構成ではないとは思いますが、参考になれば幸いです。
