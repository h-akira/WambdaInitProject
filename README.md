# WambdaInitProject

WambdaInitProjectの統合リポジトリ。各コンポーネントをGitサブモジュールとして管理します。

## プロジェクト構成

```
WambdaInitProject/
├── Infra/                 # Infrastructure (CDK) - CloudFormation, S3, CloudFront, Route53
├── CSR001_Backend/        # CSR001 Backend (SAM) - API Gateway + Lambda
├── CSR001_Frontend/       # CSR001 Frontend (Vue.js) - SPA
├── SSR001_App/            # SSR001 (SAM) - API Gateway + Lambda + Static files
├── CICD/                  # CI/CD (CloudFormation) - CodeBuild
└── README.md
```

## 初回セットアップ

### 1. リポジトリのクローン

```bash
git clone git@github.com:h-akira/WambdaInitProject.git
cd WambdaInitProject
git submodule update --init --recursive
```

### 2. 設定ファイルの作成

#### Infraの設定

```bash
cd Infra

# config_sample.jsonをコピー
cp config_sample.json config.json

# config.jsonを編集して実際の値を設定
# - account (null のままでも可)
# - region
# - route53.hosted_zone_name
# - csr001.domain_name
# - csr001.acm_certificate_arn
# - csr001.s3_bucket_name
# - ssr001.domain_name
# - ssr001.acm_certificate_arn
# - ssr001.s3_bucket_name
```

#### CICDの設定

```bash
cd ../CICD

# deploy_sample.shをコピー
cp deploy_sample.sh deploy.sh

# deploy.shを編集して実際の値を設定
# - CODESTAR_CONNECTION_ARN
# - CSR001_S3_BUCKET_NAME
# - SSR001_S3_BUCKET_NAME
```

**注意**: `config.json` と `deploy.sh` は `.gitignore` で除外されており、サブモジュール内でのみ管理されます。

## 前提条件

### 1. Route53ホストゾーンの作成

このシステムでは、Route53のホストゾーンが事前に作成されている必要があります。

```bash
# ホストゾーンの作成（初回のみ）
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference $(date +%s) \
  --profile your-profile
```

作成後、ドメインレジストラでドメインのネームサーバーをRoute53のNSレコードに設定してください。

### 2. パラメータストアの設定

CDKをCodeBuildで実行するには、AWS Systems Manager Parameter Storeに設定値を登録する必要があります。

[Infra/gen_parameter_for_codebuild.sh](Infra/gen_parameter_for_codebuild.sh) は、`config.json` から自動的にパラメータストアに設定を登録します。

```bash
cd Infra

# config.jsonを作成・編集後に実行
AWS_PROFILE=wambda ./gen_parameter_for_codebuild.sh
```

このスクリプトは以下のパラメータを作成します：
- `/WambdaInit/Common/Route53/hosted_zone_name` - Route53ホストゾーン名
- `/WambdaInit/Common/ACM/arn` - ACM証明書ARN
- `/WambdaInit/CSR001/CloudFront/domain_name` - CSR001ドメイン名
- `/WambdaInit/CSR001/S3/contents/bucket_name` - CSR001 S3バケット名
- `/WambdaInit/SSR001/CloudFront/domain_name` - SSR001ドメイン名
- `/WambdaInit/SSR001/S3/contents/bucket_name` - SSR001 S3バケット名
- `/WambdaInit/SSR001/DynamoDB/main/table_name` - SSR001 DynamoDBテーブル名

CodeBuildの[buildspec.yml](Infra/buildspec.yml)は、これらのパラメータを読み込んで `config.json` を動的に生成します。

## デプロイ順序

**重要**: SAMでAPI Gatewayを作成し、そのエクスポートをCDKが参照するため、Backend → CDK → Frontendの順序でデプロイする必要があります。

### 1. CI/CD環境のデプロイ

```bash
cd CICD
AWS_PROFILE=wambda ./deploy.sh
```

これにより、以下のCodeBuildプロジェクトが作成されます：
- Infrastructure用CodeBuild
- CSR001 Backend用CodeBuild
- CSR001 Frontend用CodeBuild
- SSR001用CodeBuild

### 2. Backend（SAM）のデプロイ

各Backendリポジトリにpushすると、CodeBuildが自動的にデプロイします。

手動でデプロイする場合：

```bash
# CSR001 Backend
cd CSR001_Backend
sam deploy --config-file samconfig.toml --profile wambda

# SSR001
cd ../SSR001_App
sam deploy --config-file samconfig.toml --profile wambda
```

デプロイされるスタック：
- `stack-wambda-csr001-app` - CSR001のAPI Gateway + Lambda
- `stack-wambda-ssr001-app` - SSR001のAPI Gateway + Lambda

これらのスタックは、API GatewayのURLを`{StackName}-ApiUrl`としてエクスポートします。

### 3. Infrastructure（CDK）のデプロイ

**前提条件**: Backend（SAM）スタックがデプロイ済みであること

Infraリポジトリにpushすると、CodeBuildが自動的に実行されます。

手動でデプロイする場合：

```bash
cd Infra

# CDK Bootstrap（初回のみ）
cdk bootstrap --profile wambda

# デプロイ
cdk deploy --all --profile wambda
```

デプロイされるスタック：
1. Cognito（共通認証）
2. SSR001 DynamoDB
3. SSR001 Main（S3 + CloudFront + Route53）
4. CSR001 Main（S3 + CloudFront + Route53）

CDKスタックは、BackendのAPI Gateway URLをインポートしてCloudFrontのオリジンとして設定します。

### 4. Frontend（Vue.js）のデプロイ

**前提条件**: CSR001 Main（CDK）スタックがデプロイ済みであること

CSR001_Frontendリポジトリにpushすると、CodeBuildが自動的にビルド・デプロイします。

手動でデプロイする場合：

```bash
cd CSR001_Frontend

# ビルド
npm install
npm run build

# S3にアップロード
aws s3 sync dist/ s3://hakira0627-s3-wambda-csr001-main/ --profile wambda

# CloudFront invalidation
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*" \
  --profile wambda
```

### 5. SSR001静的ファイルのデプロイ（2回目）

**前提条件**: SSR001 Main（CDK）スタックがデプロイ済みであること

SSR001はBackendとFrontendが統合されているため、CDKデプロイ後に静的ファイルをS3にアップロードする必要があります。そのため、もう一度CodeBuildを実行します。

手動でデプロイする場合：

```bash
cd SSR001_App

# 静的ファイルをS3にアップロード
aws s3 sync static/ s3://hakira0627-s3-wambda-ssr001-main/CloudFront/ --profile wambda

# CloudFront invalidation
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*" \
  --profile wambda
```

## サブモジュールの更新

```bash
# 全サブモジュールを最新版に更新
git submodule update --remote

# 特定のサブモジュールを更新
git submodule update --remote Infra
```

## アーキテクチャ

### CSR001（Client-Side Rendering）
- Frontend: Vue.js SPA（S3 + CloudFront）
- Backend: API Gateway + Lambda（SAM）
- 認証: Cognito User Pool

### SSR001（Server-Side Rendering）
- Frontend: 静的ファイル（S3 + CloudFront）
- Backend: API Gateway + Lambda（SAM）
- データベース: DynamoDB
- 認証: Cognito User Pool

### 共通
- DNS: Route53
- SSL/TLS: ACM（証明書）
- CI/CD: CodeBuild（自動デプロイ）

## 詳細ドキュメント

各サブモジュールの詳細については、それぞれのREADME.mdを参照してください：
- [Infra/README.md](Infra/README.md)
- [CICD/README.md](CICD/README.md)
- [CSR001_Backend/README.md](CSR001_Backend/README.md)
- [CSR001_Frontend/README.md](CSR001_Frontend/README.md)
- [SSR001_App/README.md](SSR001_App/README.md)
