---
title: Dev Containers でBlazor Server開発環境構築メモ
tags:
  - 環境構築
  - VSCode
  - ASP.NET_Core
  - Blazor
  - devcontainer
private: false
updated_at: '2023-03-13T22:03:24+09:00'
id: 9f692c66e4a137aa3733
organization_url_name: null
slide: false
---
成果物 => **[blazorserver-devcontainer-playground](https://github.com/TellMin/blazorserver-devcontainer-playground)**

## この記事は? :dango:

VSCodeの拡張機能「Dev Containers」を用いて、ASP.NET Core BlazorのBlazor Serverテンプレートの開発環境構築したので、それをメモしたものです。

### 対象読者 :dark_sunglasses:

- ASP.NET Core アプリケーション開発をDev Containersで行いたい人

### 前提 :cooking: 

Docker、Docker Compose、dotnet CLIやC#のビルド周りについての詳しい説明は省略します💦
間違い、改善点があればご指摘いただけると幸いです。

## 本題 :zap:

### Dev Containersとは？

Dev Containersは、VS Codeの拡張機能の1つであり、Docker環境を利用して開発を行うことができます。 通常のDockerと同様に、ホストのファイルやディレクトリをコンテナにマウントし、開発環境を構築することができます。

今回はこのDev Containersを利用してBlazor Serverの開発環境構築を行います。

### ディレクトリ構造

さっそく dotnet new していくわけですが、先に今回のプロジェクトの最終的なディレクトリ構造について説明します。

```:ディレクトリ構造

Topディレクトリ
│   .gitignore    
│
└───.devcontainer
│       devcontainer.json
│       docker-compose.yml
│       Dockerfile
│
└───source
	|	.dockerignore
	|   docker-compose.yml
	|    
    └───blazorserver_devcontainer_playground
		    // ソースコード一式

```

ざっくり下記位置づけです。

- .devcontainer
	- Dev Continerを利用するための`devcontainer.json`を格納するディレクトリ
	- Dockerfileは必ずしもここに置く必要はない
- source
	- 公開するための`docker-compose.yml`とソース一式を格納
		- ネストさせずにプロジェクト直下に置いても問題ないので、お好みで

さっそく.devcontainerディレクトリ以下を確認していきましょう。

### Dev Containerの設定 :fire:

```json:.devcontainer/devcontainer.json

{
    "dockerComposeFile": ["docker-compose.yml"],
    "service": "web",
	"workspaceFolder": "/blazorserver-devcontainer-playground",
	"customizations": {
		"vscode": {
			"extensions": [
				"ms-dotnettools.csharp",
				"ms-dotnettools.vscode-dotnet-runtime",
				"MS-CEINTL.vscode-language-pack-ja",
				"GitHub.copilot-nightly"
			]
		}
	}
  }

```

Dev Containerで利用するDocker環境の定義にはいくつかの方法があります。今回はdocker-composeを自前で定義して実行します。

> 特にこちらの記事が参考にしました。
> [Docker Compose な開発環境にちょい足し3分で作るVSCode devcontainer](https://zenn.dev/saboyutaka/articles/9cffc8d14c6684)

service の "web" や workspaceFolder の "/blazorserver-devcontainer-playground" はいずれも`docker-compose.yml`にて定義されています。

```yaml:.devcontainer/docker-compose.yml

version: '3.9'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: BlazorContainer
    command: sleep infinity
    volumes:
      - type: bind
        source: ../.
        target: /blazorserver-devcontainer-playground

```

また、ベースイメージはmicrosoftが提供するものをそのまま利用しています。

```Dockerfile:.devcontainer/Dockerfile

FROM mcr.microsoft.com/dotnet/sdk:7.0

```

> imageをそのまま利用しているので、dockerfileを用意する必要はないのですが、学習もかねてdocker-compose.ymlを用意する方法を選択しました。
> 公式ページ => [Create a Dev Container](https://code.visualstudio.com/docs/devcontainers/create-dev-container)

基本的には上記ファイルを用意するだけでdotnetコマンドが実行できる環境が整います。

Dev Containerの実行については、`.devcontainers`ディレクトリが直下に存在する場合にVS Codeから提案してくれるほか、直接コマンドから直接実行することも可能です。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2566826/6312bfc0-ebd0-2f61-0a1f-1d07f80f6e6d.png)

### Dev Container環境ごとGit管理する :coffee: 

```yaml:.devcontainer/docker-compose.yml

volumes:
  - type: bind
    source: ../.
    target: /blazorserver-devcontainer-playground

```

上記のように、sourceとして `../.`を指定し、Topディレクトリ全体をマウントさせています。
これはDev Containerの設定ごとGit管理を行うためです。

こうすることによって、開発環境だけでなくVS Codeの拡張機能についても共通化することができます。

```json:.devcontainer/devcontainer.json

"customizations": {
    "vscode": {
        "extensions": [
            "ms-dotnettools.csharp",
            "ms-dotnettools.vscode-dotnet-runtime",
            "MS-CEINTL.vscode-language-pack-ja",
            "GitHub.copilot-nightly"
        ]
    }
}

```

複数言語を触る方にとっても、最小限の拡張機能で開発が行えるので、（競合を心配することもなく）良さそうです。

## まとめ :panda_face: 

今回の構築を通じてC#のDocker環境構築にある程度慣れることができました。実際に手を動かすことでC#だけでなくDockerについても理解を深めることができますので、ぜひ皆さんも試してみてはいかがでしょうか。
