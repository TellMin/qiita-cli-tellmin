---
title: GitHub Actions で ECR に Docker イメージをプッシュして ECS にデプロイまで
tags:
  - GitHub
  - AWS
  - C#
  - Docker
private: false
updated_at: '2023-12-12T22:00:05+09:00'
id: acddd8e5794a08334931
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

GitHub Actions で Docker イメージをビルドして ECR にプッシュして ECS にデプロイするまでの手順をまとめます。

### ディレクト構成

今回は以下のようなディレクトリ構成で進めます。
aspnetapp 以下には ASP.NET Core のプロジェクトがあります。

```bash
.
├── .github
│   └── workflows
│       └── deploy.yml
├── deploy
│   └── ecs-task-def.json
├── aspnetapp
│   └── aspnetapp.csproj など
└── Dockerfile
```

.github/workflows 以下に yml ファイルを配置します。
これにより、GitHub Actions を定義することができます。

deploy ディレクトリには、ECS タスク定義の JSON ファイルを配置します。

ASP.NET Core プロジェクトや Dockerfile の作成は省略します。
参考までに公式ドキュメントを貼っておきます。

https://learn.microsoft.com/ja-jp/aspnet/core/host-and-deploy/docker/building-net-docker-images?view=aspnetcore-8.0

### 成果物

先に完成図だけ示します。

```yml
name: ecr push image

on:
  push:
    branches:
      - main

env:
  AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}

jobs:
  push:
    runs-on: ubuntu-latest
    # `permissions` を設定しないと OIDC が使えないので注意
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v3

      # AWS 認証
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ap-northeast-1
          role-to-assume: ${{ secrets.AWS_IAM_ROLE_ARN }} # GitHub Actions 用の IAM ロールの ARN

      # ECR ログイン
      - uses: aws-actions/amazon-ecr-login@v2
        id: login-ecr # outputs で参照するために id を設定

      # Docker イメージを build・push する
      - name: build and push docker image to ecr
        id: build-image # outputs で参照するために id を設定
        env:
          # ECR レジストリを `aws-actions/amazon-ecr-login` アクションの `outputs.registry` から取得
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          # イメージを push する ECR リポジトリ名
          REPOSITORY: ecs-sample
          # 任意のイメージタグ
          IMAGE_TAG: latest
        run: |
          docker build . --tag ${{ env.REGISTRY }}/${{ env.REPOSITORY }}:${{ env.IMAGE_TAG }}
          docker push ${{ env.REGISTRY }}/${{ env.REPOSITORY }}:${{ env.IMAGE_TAG }}
          # outputs で参照するために `image_id` という名前で出力
          echo "image_id=${{ env.REGISTRY }}/${{ env.REPOSITORY }}:${{ env.IMAGE_TAG }}" >> $GITHUB_OUTPUT

      - name: Update ECS task definition and ecs-task-def.json with AWS Account ID
        run: |
          sed -i 's|\[AWS_ACCOUNT_ID\]|${{ env.AWS_ACCOUNT_ID }}|g' ./deploy/ecs-task-def.json
          sed -i 's|\[IMAGE_ID\]|${{ steps.build-image.outputs.image_id }}|g' ./deploy/ecs-task-def.json

      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: deploy/ecs-task-def.json
          container-name: ecs-sample-container
          image: ${{ steps.build-image.outputs.image_id }}

      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ecs-sample-service
          cluster: ecs-sample-service
          wait-for-service-stability: true
          force-new-deployment: true
```

### GitHub Actions の設定

おおまかな流れは以下のようになります。

1. OIDC を使用して AWS の認証を行う
2. Docker イメージをビルド & ECR にプッシュする
3. ECS にデプロイする

#### 1. OIDC を使用して AWS の認証を行う

GitHub Actions で AWS の認証を行うには、OIDC を使用する必要があります。

まずは AWS コンソール上で、ロールを作成します。
作成したロールには、後続の処理を実行するための権限を付与しておきます。

[AWS_ACCOUNT_ID], [ECR_REPOSITORY_NAME] は適宜置き換えてください。

```json
{
	"Version": "2012-10-17",
	"Statement": [
    {
			"Action": "ecr:GetAuthorizationToken",
			"Effect": "Allow",
			"Resource": "*"
		},
		{
			"Action": [
				"ecr:UploadLayerPart",
				"ecr:PutImage",
				"ecr:InitiateLayerUpload",
				"ecr:CompleteLayerUpload",
				"ecr:BatchCheckLayerAvailability"
			],
			"Effect": "Allow",
			"Resource": "arn:aws:ecr:ap-northeast-1:[AWS_ACCOUNT_ID]:repository/[ECR_REPOSITORY_NAME]"
		}
		{
			"Sid": "VisualEditor0",
			"Effect": "Allow",
			"Action": [
				"ecs:DeregisterTaskDefinition",
				"ecs:RegisterTaskDefinition",
				"ecs:ListTaskDefinitions",
				"ecs:DescribeTaskDefinition",
				"ecs:DescribeServices",
				"ecs:UpdateService"
			],
			"Resource": "*"
		},
		{
			"Effect": "Allow",
			"Action": "iam:PassRole",
			"Resource": "arn:aws:iam::[AWS_ACCOUNT_ID]:role/ecsTaskExecutionRole"
		}
	]
}
```

`iam:PassRole`権限は、ECS タスク実行ロールである `ecsTaskExecutionRole` に処理を引き継ぐために必要です。(`ecs-task-def.json`で使用します。同じロール実行させる場合は不要です。)

OIDC については以下の記事が参考になりました。

https://zenn.dev/kou_pg_0131/articles/gh-actions-oidc-aws

yml ファイルには以下のように記述します。

```yml
push:
  runs-on: ubuntu-latest
  # `permissions` を設定しないと OIDC が使えないので注意
  permissions:
    id-token: write
    contents: read
  steps:
    - uses: actions/checkout@v3

    # AWS 認証
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-region: ap-northeast-1
        role-to-assume: ${{ secrets.AWS_IAM_ROLE_ARN }} # GitHub Actions 用の IAM ロールの ARN
```

はじめは記事を参考に `aws-actions/configure-aws-credentials@v1` を使用していましたが、`v4` に変更しました。
GitHub Actions の仕様変更により、すべてのアクションで `Node16` がデフォルトで使用されるにようになったためです。

https://github.blog/changelog/2023-06-13-github-actions-all-actions-will-run-on-node16-instead-of-node12-by-default/

#### 2. Docker イメージをビルド & ECR にプッシュする

Docker イメージをビルドするための処理を記述します。
また、同時に ECR にプッシュする処理も記述します。

```yml
# Docker イメージを build・push する
- name: build and push docker image to ecr
  id: build-image # outputs で参照するために id を設定
  env:
    # ECR レジストリを `aws-actions/amazon-ecr-login` アクションの `outputs.registry` から取得
    REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    # イメージを push する ECR リポジトリ名
    REPOSITORY: ecs-sample
    # 任意のイメージタグ
    IMAGE_TAG: latest
  run: |
    docker build . --tag ${{ env.REGISTRY }}/${{ env.REPOSITORY }}:${{ env.IMAGE_TAG }}
    docker push ${{ env.REGISTRY }}/${{ env.REPOSITORY }}:${{ env.IMAGE_TAG }}
    # outputs で参照するために `image_id` という名前で出力
    echo "image_id=${{ env.REGISTRY }}/${{ env.REPOSITORY }}:${{ env.IMAGE_TAG }}" >> $GITHUB_OUTPUT
```

このとき、`${{ steps.login-ecr.outputs.registry }}` という記述で、`aws-actions/amazon-ecr-login` アクションの 出力を参照しています。
他にも、`$GITHUB_OUTPUT` という変数を利用することでも、出力結果を受け渡すことが可能です。

かつては、`save-state` と `set-output` というコマンドを使用していましたが、現在は非推奨となっています。
そのため、`$GITHUB_OUTPUT` を使用するようにしましょう。

https://github.blog/changelog/2022-10-11-github-actions-deprecating-save-state-and-set-output-commands/

#### 3. ECS にデプロイする

ECS にデプロイするためには タスクの定義が必要です。
今回はリポジトリ内で管理している `ecs-task-def.json` を使用します。

```json
{
  "family": "ecs-sample-task",
  "executionRoleArn": "arn:aws:iam::[AWS_ACCOUNT_ID]:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "ecs-sample-container",
      "image": "[IMAGE_ID]",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 8080
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "/ecs/ecs-sample-task",
          "awslogs-region": "ap-northeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512"
}
```

yml ファイルには以下のように記述します。

```yml
- name: Update ECS task definition and ecs-task-def.json with AWS Account ID
  run: |
    sed -i 's|\[AWS_ACCOUNT_ID\]|${{ env.AWS_ACCOUNT_ID }}|g' ./deploy/ecs-task-def.json
    sed -i 's|\[IMAGE_ID\]|${{ steps.build-image.outputs.image_id }}|g' ./deploy/ecs-task-def.json

- name: Fill in the new image ID in the Amazon ECS task definition
  id: task-def
  uses: aws-actions/amazon-ecs-render-task-definition@v1
  with:
    task-definition: deploy/ecs-task-def.json
    container-name: ecs-sample-container
    image: ${{ steps.build-image.outputs.image_id }}

- name: Deploy Amazon ECS task definition
  uses: aws-actions/amazon-ecs-deploy-task-definition@v1
  with:
    task-definition: ${{ steps.task-def.outputs.task-definition }}
    service: ecs-sample-service
    cluster: ecs-sample-service
    wait-for-service-stability: true
    force-new-deployment: true
```

では順番に解説していきます。

まずは、`ecs-task-def.json` の `[AWS_ACCOUNT_ID]` と `[IMAGE_ID]` についてです。

`ecs-task-def.json` を Git で管理する場合、次の点に注意する必要があります。

- AWS アカウントなどの機密情報の管理
- 動的に変化する値の管理

そのため、以下のようにして、`ecs-task-def.json` に値を埋め込むようにしています。

```
sed -i 's|\[AWS_ACCOUNT_ID\]|${{ env.AWS_ACCOUNT_ID }}|g' ./deploy/ecs-task-def.json
sed -i 's|\[IMAGE_ID\]|${{ steps.build-image.outputs.image_id }}|g' ./deploy/ecs-task-def.json
```

そして `aws-actions/amazon-ecs-render-task-definition@v1`, `aws-actions/amazon-ecs-deploy-task-definition@v1` アクションを使用して、ECS にデプロイします。

注意として、事前に ECS クラスターとサービスを作成しておく必要があります。
また、`force-new-deployment` を `true` にすることで、新しいタスク定義でデプロイするようにしています。

ポート番号はコンテナ内で使用しているポート番号を指定する必要があります。
今回は ASP.NET Core で作成したアプリケーションを使用しているため、`8080` を指定しています。

## まとめ

GitHub Actions で Docker イメージをビルドして ECR にプッシュして ECS にデプロイするまでの手順をまとめました。
誤りや改善点などありましたら、ご指摘いただけると幸いです。
