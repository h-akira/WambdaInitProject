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

## デプロイ順序

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

### 2. Infrastructure（CDK）のデプロイ

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

**注意**: SSR001 MainとCSR001 Mainスタックは、それぞれのBackend（SAM）スタックがデプロイされた後に実行してください。

### 3. Backend（SAM）のデプロイ

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

### 4. Frontend（Vue.js）のデプロイ

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

## サブモジュールの更新

```bash
# 全サブモジュールを最新版に更新
git submodule update --remote

# 特定のサブモジュールを更新
git submodule update --remote Infra
```

## トラブルシューティング

### S3バケットが既に存在するエラー

既存のS3バケットを削除してから再デプロイしてください：

```bash
# バケット内のオブジェクトを削除
aws s3 rm s3://hakira0627-s3-wambda-csr001-main --recursive --profile wambda
aws s3 rm s3://hakira0627-s3-wambda-ssr001-main --recursive --profile wambda

# バケットを削除
aws s3 rb s3://hakira0627-s3-wambda-csr001-main --profile wambda
aws s3 rb s3://hakira0627-s3-wambda-ssr001-main --profile wambda
```

### Route53のアクセス権限エラー

CloudFormation実行ロールにRoute53の権限が付与されているか確認してください。`Infra/init/cfn-execution-policies.yaml` を確認し、必要に応じて更新してください。

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
