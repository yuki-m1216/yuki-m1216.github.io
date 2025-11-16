# コーディング原則とベストプラクティス

保守性が高く、チームで運用しやすいIaCコードを書くための原則とベストプラクティスを紹介します。

## 基本原則

### 1. DRY原則 (Don't Repeat Yourself)

同じコードを繰り返し書かず、再利用可能な構造にまとめます。

**❌ 悪い例: 重複したコード**

=== "CDK (TypeScript)"
    ```typescript
    // 開発環境のバケット
    const devBucket = new s3.Bucket(this, 'DevBucket', {
      bucketName: 'myapp-dev-data',
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      lifecycleRules: [{
        noncurrentVersionExpiration: cdk.Duration.days(30),
      }],
    });

    // 本番環境のバケット (ほぼ同じコード)
    const prodBucket = new s3.Bucket(this, 'ProdBucket', {
      bucketName: 'myapp-prod-data',
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      lifecycleRules: [{
        noncurrentVersionExpiration: cdk.Duration.days(90),
      }],
    });
    ```

=== "Terraform (HCL)"
    ```hcl
    # 開発環境のバケット
    resource "aws_s3_bucket" "dev_bucket" {
      bucket = "myapp-dev-data"
    }

    resource "aws_s3_bucket_versioning" "dev_bucket" {
      bucket = aws_s3_bucket.dev_bucket.id
      versioning_configuration {
        status = "Enabled"
      }
    }

    resource "aws_s3_bucket_server_side_encryption_configuration" "dev_bucket" {
      bucket = aws_s3_bucket.dev_bucket.id
      rule {
        apply_server_side_encryption_by_default {
          sse_algorithm = "AES256"
        }
      }
    }

    resource "aws_s3_bucket_lifecycle_configuration" "dev_bucket" {
      bucket = aws_s3_bucket.dev_bucket.id
      rule {
        id     = "delete-old-versions"
        status = "Enabled"
        noncurrent_version_expiration {
          noncurrent_days = 30
        }
      }
    }

    # 本番環境のバケット (ほぼ同じコード)
    resource "aws_s3_bucket" "prod_bucket" {
      bucket = "myapp-prod-data"
    }

    resource "aws_s3_bucket_versioning" "prod_bucket" {
      bucket = aws_s3_bucket.prod_bucket.id
      versioning_configuration {
        status = "Enabled"
      }
    }

    resource "aws_s3_bucket_server_side_encryption_configuration" "prod_bucket" {
      bucket = aws_s3_bucket.prod_bucket.id
      rule {
        apply_server_side_encryption_by_default {
          sse_algorithm = "AES256"
        }
      }
    }

    resource "aws_s3_bucket_lifecycle_configuration" "prod_bucket" {
      bucket = aws_s3_bucket.prod_bucket.id
      rule {
        id     = "delete-old-versions"
        status = "Enabled"
        noncurrent_version_expiration {
          noncurrent_days = 90
        }
      }
    }
    ```

**✅ 良い例: 再利用可能なモジュール**

=== "CDK (TypeScript)"
    ```typescript
    // 再利用可能なコンストラクトを定義
    export class DataBucket extends Construct {
      public readonly bucket: s3.Bucket;

      constructor(scope: Construct, id: string, props: {
        environment: string;
        retentionDays: number;
      }) {
        super(scope, id);

        this.bucket = new s3.Bucket(this, 'Bucket', {
          bucketName: `myapp-${props.environment}-data`,
          versioned: true,
          encryption: s3.BucketEncryption.S3_MANAGED,
          lifecycleRules: [{
            noncurrentVersionExpiration: cdk.Duration.days(props.retentionDays),
          }],
        });
      }
    }

    // 使用例
    const devBucket = new DataBucket(this, 'DevBucket', {
      environment: 'dev',
      retentionDays: 30,
    });

    const prodBucket = new DataBucket(this, 'ProdBucket', {
      environment: 'prod',
      retentionDays: 90,
    });
    ```

=== "Terraform (HCL)"
    ```hcl
    # modules/data_bucket/main.tf (再利用可能なモジュール)
    variable "environment" {
      type = string
    }

    variable "retention_days" {
      type = number
    }

    resource "aws_s3_bucket" "this" {
      bucket = "myapp-${var.environment}-data"
    }

    resource "aws_s3_bucket_versioning" "this" {
      bucket = aws_s3_bucket.this.id
      versioning_configuration {
        status = "Enabled"
      }
    }

    resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
      bucket = aws_s3_bucket.this.id
      rule {
        apply_server_side_encryption_by_default {
          sse_algorithm = "AES256"
        }
      }
    }

    resource "aws_s3_bucket_lifecycle_configuration" "this" {
      bucket = aws_s3_bucket.this.id
      rule {
        id     = "delete-old-versions"
        status = "Enabled"
        noncurrent_version_expiration {
          noncurrent_days = var.retention_days
        }
      }
    }

    output "bucket_name" {
      value = aws_s3_bucket.this.bucket
    }

    # 使用例 (main.tf)
    module "dev_bucket" {
      source = "./modules/data_bucket"

      environment    = "dev"
      retention_days = 30
    }

    module "prod_bucket" {
      source = "./modules/data_bucket"

      environment    = "prod"
      retention_days = 90
    }
    ```

### 2. 適切な管理単位の分割

管理単位(CDKのStack、TerraformのState)は**責任**や**ライフサイクル**などを考慮して分割します。

#### 分割の基準

**責任による分割**

- ネットワーク、データベース、アプリケーションなど機能単位で分離

**ライフサイクルによる分割**

- 変更頻度が異なるリソースを分離
- デプロイリスクが異なるリソースを分離
- 削除・再作成の容易性を考慮

**❌ 悪い例: 全てを1つの管理単位に**

=== "CDK (TypeScript)"
    ```typescript
    export class MonolithStack extends Stack {
      constructor(scope: Construct, id: string) {
        super(scope, id);

        // ネットワーク (ライフサイクル: ほぼ変更しない)
        const vpc = new ec2.Vpc(this, 'Vpc');

        // データベース (ライフサイクル: まれに変更、削除は危険)
        const database = new rds.DatabaseInstance(this, 'Database', { /*...*/ });

        // Lambda関数 (ライフサイクル: 頻繁に変更)
        const lambda = new lambda.Function(this, 'Function', { /*...*/ });

        // API Gateway (ライフサイクル: 頻繁に変更)
        const api = new apigateway.RestApi(this, 'Api', { /*...*/ });

        // 問題:
        // - 全てが密結合、変更時の影響範囲が大きい
        // - Lambda変更のたびにVPC・DBも再評価される
        // - 開発環境で試行錯誤しにくい
      }
    }
    ```

=== "Terraform (HCL)"
    ```hcl
    # main.tf (全てが1つのファイル/state)

    # ネットワーク (ライフサイクル: ほぼ変更しない)
    resource "aws_vpc" "main" {
      cidr_block = "10.0.0.0/16"
    }

    # データベース (ライフサイクル: まれに変更、削除は危険)
    resource "aws_db_instance" "main" {
      allocated_storage = 20
      engine            = "mysql"
      # ...
    }

    # Lambda関数 (ライフサイクル: 頻繁に変更)
    resource "aws_lambda_function" "main" {
      function_name = "myapp-function"
      # ...
    }

    # API Gateway (ライフサイクル: 頻繁に変更)
    resource "aws_api_gateway_rest_api" "main" {
      name = "myapp-api"
      # ...
    }

    # 問題:
    # - 全てが1つのstateファイルで管理される
    # - Lambda変更のたびにVPC・DBもplanに含まれる
    # - 開発環境で試行錯誤しにくい
    ```

**✅ 良い例: 責任とライフサイクルで管理単位を分割**

=== "CDK (TypeScript)"
    ```typescript
    // ネットワーク層 (ライフサイクル: 長期間安定)
    export class NetworkStack extends Stack {
      public readonly vpc: ec2.Vpc;

      constructor(scope: Construct, id: string) {
        super(scope, id);
        this.vpc = new ec2.Vpc(this, 'Vpc', {
          maxAzs: 2,
          // 一度構築したらほぼ変更しない
        });
      }
    }

    // データ層 (ライフサイクル: 変更まれ、永続化が重要)
    export class DatabaseStack extends Stack {
      constructor(scope: Construct, id: string, props: {
        vpc: ec2.Vpc;
      }) {
        super(scope, id);
        new rds.DatabaseInstance(this, 'Database', {
          vpc: props.vpc,
          deletionProtection: true,  // 削除保護
          // データは永続化、慎重な変更が必要
        });
      }
    }

    // アプリケーション層 (ライフサイクル: 頻繁に変更)
    export class ApplicationStack extends Stack {
      constructor(scope: Construct, id: string, props: {
        vpc: ec2.Vpc;
      }) {
        super(scope, id);
        // Lambda, API Gateway など
        // 頻繁にデプロイ、開発環境では削除・再作成も容易
      }
    }
    ```

=== "Terraform (HCL)"
    ```hcl
    # network/main.tf (ライフサイクル: 長期間安定)
    resource "aws_vpc" "main" {
      cidr_block = "10.0.0.0/16"
      # 一度構築したらほぼ変更しない
    }

    output "vpc_id" {
      value = aws_vpc.main.id
    }

    # database/main.tf (ライフサイクル: 変更まれ、永続化が重要)
    data "terraform_remote_state" "network" {
      backend = "s3"
      config = {
        bucket = "myapp-terraform-state"
        key    = "network/terraform.tfstate"
      }
    }

    resource "aws_db_instance" "main" {
      vpc_security_group_ids = [data.terraform_remote_state.network.outputs.vpc_id]
      # データは永続化、慎重な変更が必要
      # ...
    }

    # application/main.tf (ライフサイクル: 頻繁に変更)
    data "terraform_remote_state" "network" {
      backend = "s3"
      config = {
        bucket = "myapp-terraform-state"
        key    = "network/terraform.tfstate"
      }
    }

    resource "aws_lambda_function" "main" {
      function_name = "myapp-function"
      # 頻繁にデプロイ、開発環境では削除・再作成も容易
      # ...
    }

    resource "aws_api_gateway_rest_api" "main" {
      name = "myapp-api"
      # ...
    }
    ```

**メリット**:

- ネットワーク変更なしにアプリケーションをデプロイ可能
- データベースを保護しつつ、アプリは自由に変更可能
- 開発環境でアプリケーション層のみ削除・再作成可能

## 命名規則

### リソース名

一貫した命名パターンを使用します。

```typescript
// パターン: {プロジェクト}-{環境}-{用途}-{リソースタイプ}
const bucketName = `myapp-${environment}-data-bucket`;
const functionName = `myapp-${environment}-processor-function`;
const tableName = `myapp-${environment}-users-table`;
```

### モジュール/クラス名

PascalCaseを使用し、用途が明確な名前にします。

=== "CDK (TypeScript)"
    ```typescript
    // ✅ 良い例
    export class UserAuthenticationStack extends Stack { }
    export class DataProcessingPipeline extends Construct { }
    export class MonitoringDashboard extends Construct { }

    // ❌ 悪い例
    export class Stack1 extends Stack { }  // 用途不明
    export class mystack extends Stack { }  // 命名規則違反
    ```

=== "Terraform (HCL)"
    ```hcl
    # ✅ 良い例: モジュール名やディレクトリ名
    modules/user_authentication/
    modules/data_processing_pipeline/
    modules/monitoring_dashboard/

    # ❌ 悪い例
    modules/module1/  # 用途不明
    modules/MyModule/ # snake_caseが推奨
    ```

### 変数名

```typescript
// ✅ 良い例: 用途が明確
const userDataBucket = new s3.Bucket(/*...*/);
const orderProcessorFunction = new lambda.Function(/*...*/);
const customerTable = new dynamodb.Table(/*...*/);

// ❌ 悪い例: 用途不明
const bucket1 = new s3.Bucket(/*...*/);
const func = new lambda.Function(/*...*/);
const tbl = new dynamodb.Table(/*...*/);
```

## パラメータ化

環境ごとの差分は設定ファイルで管理します。

### 設定ファイルの分離

```typescript
// config/dev.ts
export const devConfig = {
  environment: 'dev',
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.SMALL),
  minCapacity: 1,
  maxCapacity: 2,
  dbBackupRetention: 7,
};

// config/prod.ts
export const prodConfig = {
  environment: 'prod',
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.M5, ec2.InstanceSize.LARGE),
  minCapacity: 2,
  maxCapacity: 10,
  dbBackupRetention: 30,
};
```

### 設定の使用

```typescript
import { devConfig } from './config/dev';
import { prodConfig } from './config/prod';

const config = process.env.ENV === 'prod' ? prodConfig : devConfig;

export class ApplicationStack extends Stack {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    new rds.DatabaseInstance(this, 'Database', {
      instanceType: config.instanceType,
      backupRetention: cdk.Duration.days(config.dbBackupRetention),
    });
  }
}
```

## セキュリティベストプラクティス

### シークレット管理

!!! warning "最重要原則"
    **シークレット情報は絶対にIaCコードに含めず、VCS(Git)にコミットしない**

    - パスワード、APIキー、トークンなどはコードに書かない
    - `.env`ファイルなどもGitにコミットしない(`.gitignore`に追加)
    - シークレットは専用のシークレット管理サービスで管理する

**❌ 絶対にやってはいけない**

=== "CDK (TypeScript)"
    ```typescript
    // ❌ コードにシークレットをハードコード
    const apiKey = 'sk-1234567890abcdef';  // 絶対NG! Gitにコミットされる!

    // ❌ 環境変数に直接シークレットを設定
    environment: {
      DATABASE_PASSWORD: 'MyP@ssw0rd',  // 絶対NG! VCS履歴に残る!
    }
    ```

=== "Terraform (HCL)"
    ```hcl
    # ❌ コードにシークレットをハードコード
    variable "api_key" {
      default = "sk-1234567890abcdef"  # 絶対NG! Gitにコミットされる!
    }

    # ❌ 環境変数に直接シークレットを設定
    resource "aws_lambda_function" "main" {
      environment {
        variables = {
          DATABASE_PASSWORD = "MyP@ssw0rd"  # 絶対NG! VCS履歴に残る!
        }
      }
    }
    ```

**✅ 正しい方法: シークレット管理サービスを使用**

IaCコードではシークレット管理サービスのリソースを定義するのみで、実際の値はコードに含めません。

=== "CDK (TypeScript)"
    ```typescript
    // Secrets Managerを使用 (シークレット値自体はコードに含まれない)
    const dbSecret = new secretsmanager.Secret(this, 'DBSecret', {
      generateSecretString: {
        secretStringTemplate: JSON.stringify({ username: 'admin' }),
        generateStringKey: 'password',  // このキーの値が自動生成される
        excludeCharacters: '"@/\\',
      },
    });

    // Lambda関数からシークレットを参照
    const lambda = new lambda.Function(this, 'Function', {
      environment: {
        SECRET_ARN: dbSecret.secretArn,  // ARNのみ渡す(値ではない)
      },
    });

    dbSecret.grantRead(lambda);  // 読み取り権限を付与
    ```

=== "Terraform (HCL)"
    ```hcl
    # Secrets Managerを使用 (シークレット値自体はコードに含まれない)
    resource "aws_secretsmanager_secret" "db_secret" {
      name = "myapp-db-secret"
    }

    resource "aws_secretsmanager_secret_version" "db_secret" {
      secret_id = aws_secretsmanager_secret.db_secret.id
      secret_string = jsonencode({
        username = "admin"
        password = random_password.db_password.result  # ランダム生成
      })
    }

    # パスワードをランダム生成
    resource "random_password" "db_password" {
      length  = 16
      special = true
    }

    # Lambda関数からシークレットを参照
    resource "aws_lambda_function" "main" {
      function_name = "myapp-function"

      environment {
        variables = {
          SECRET_ARN = aws_secretsmanager_secret.db_secret.arn  # ARNのみ渡す
        }
      }
    }

    # 読み取り権限を付与
    resource "aws_iam_role_policy" "lambda_secrets" {
      role = aws_iam_role.lambda_role.id

      policy = jsonencode({
        Version = "2012-10-17"
        Statement = [{
          Effect = "Allow"
          Action = [
            "secretsmanager:GetSecretValue"
          ]
          Resource = aws_secretsmanager_secret.db_secret.arn
        }]
      })
    }
    ```

**ポイント**:

- IaCコードには「シークレットを管理するリソースの定義」のみを記述
- 実際のシークレット値はAWS Secrets Managerが自動生成・安全に保管
- アプリケーションは実行時にシークレットを取得する

## モジュール化と再利用性

DRY原則を実践する上で、**どこまでモジュール化すべきか**の判断が重要です。

### モジュール化すべきケース

以下のような場合は、積極的にモジュール化を検討すべきです:

**1. 組織のセキュリティポリシーやベストプラクティスを強制したい**

- 「全てのS3バケットは暗号化必須」などのポリシーをコードで強制
- デフォルトで安全な設定を適用し、設定漏れを防ぐ

**2. 複数のプロジェクトで共通利用する**

- 社内の複数チームで同じパターンを使い回す
- 一度作成したモジュールを別プロジェクトでも再利用

**3. 複雑な構成をカプセル化してシンプルにしたい**

- 複数リソースの組み合わせを1つのモジュールにまとめる
- 利用側は詳細を気にせず使える

### モジュール化の例: セキュアなS3バケット

組織のセキュリティポリシーを満たすS3バケットを毎回定義するのは手間がかかり、設定漏れのリスクもあります。

=== "CDK (TypeScript)"
    ```typescript
    // constructs/secure-bucket.ts
    // 組織のセキュリティポリシーを強制するモジュール
    export class SecureBucket extends Construct {
      public readonly bucket: s3.Bucket;

      constructor(scope: Construct, id: string, props?: s3.BucketProps) {
        super(scope, id);

        // セキュアな設定をデフォルトで適用(上書き不可)
        this.bucket = new s3.Bucket(this, 'Resource', {
          ...props,
          // 以下は組織ポリシーとして強制
          encryption: s3.BucketEncryption.S3_MANAGED,
          blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
          versioned: true,
          enforceSSL: true,
        });

        // アクセスログを自動で有効化
        const logBucket = new s3.Bucket(this, 'LogBucket', {
          encryption: s3.BucketEncryption.S3_MANAGED,
          blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
        });

        this.bucket.logToBucket(logBucket);
      }
    }

    // 使用例: 開発者はセキュリティ設定を気にせず利用できる
    const dataBucket = new SecureBucket(this, 'DataBucket');
    ```

=== "Terraform (HCL)"
    ```hcl
    # modules/secure_bucket/main.tf
    # 組織のセキュリティポリシーを強制するモジュール
    variable "bucket_name" {
      type = string
    }

    # メインバケット(セキュアな設定を強制)
    resource "aws_s3_bucket" "this" {
      bucket = var.bucket_name
    }

    # バージョニング(組織ポリシーで必須)
    resource "aws_s3_bucket_versioning" "this" {
      bucket = aws_s3_bucket.this.id
      versioning_configuration {
        status = "Enabled"
      }
    }

    # 暗号化(組織ポリシーで必須)
    resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
      bucket = aws_s3_bucket.this.id
      rule {
        apply_server_side_encryption_by_default {
          sse_algorithm = "AES256"
        }
      }
    }

    # パブリックアクセスブロック(組織ポリシーで必須)
    resource "aws_s3_bucket_public_access_block" "this" {
      bucket = aws_s3_bucket.this.id

      block_public_acls       = true
      block_public_policy     = true
      ignore_public_acls      = true
      restrict_public_buckets = true
    }

    # アクセスログバケット
    resource "aws_s3_bucket" "logs" {
      bucket = "${var.bucket_name}-logs"
    }

    resource "aws_s3_bucket_logging" "this" {
      bucket        = aws_s3_bucket.this.id
      target_bucket = aws_s3_bucket.logs.id
      target_prefix = "log/"
    }

    output "bucket_name" {
      value = aws_s3_bucket.this.bucket
    }

    # 使用例 (main.tf)
    # 開発者はセキュリティ設定を気にせず利用できる
    module "data_bucket" {
      source = "./modules/secure_bucket"

      bucket_name = "myapp-data"
    }
    ```

### モジュール化しすぎない

一方で、以下のような場合は無理にモジュール化する必要はありません:

❌ **単一リソースだけの薄いラッパー**

単一のリソースをそのままモジュールにするのは、かえって複雑さを増すだけです。

=== "CDK (TypeScript)"
    ```typescript
    // ❌ 悪い例: 単一リソースをモジュール化(やりすぎ)
    export class MyBucket extends Construct {
      public readonly bucket: s3.Bucket;

      constructor(scope: Construct, id: string, props: s3.BucketProps) {
        super(scope, id);
        // 単にBucketを作るだけ(何も付加価値がない)
        this.bucket = new s3.Bucket(this, 'Bucket', props);
      }
    }

    // ✅ 良い例: 直接書く
    const bucket = new s3.Bucket(this, 'MyBucket', {
      bucketName: 'myapp-data',
      versioned: true,
    });
    ```

=== "Terraform (HCL)"
    ```hcl
    # ❌ 悪い例: 単一リソースをモジュール化(やりすぎ)
    # modules/simple_bucket/main.tf
    variable "bucket_name" {
      type = string
    }

    resource "aws_s3_bucket" "this" {
      bucket = var.bucket_name
    }

    # 使用側 (main.tf)
    module "my_bucket" {
      source = "./modules/simple_bucket"
      bucket_name = "myapp-data"
    }

    # ✅ 良い例: 直接書く
    resource "aws_s3_bucket" "my_bucket" {
      bucket = "myapp-data"
    }
    ```

**なぜダメか**: モジュール化による複雑さ(ファイル分割、インターフェース定義)が、再利用のメリットを上回らない。

❌ **1回しか使わない構成**

- プロジェクト固有の構成を無理に抽象化しない
- 抽象化のコストが再利用のメリットを上回る

❌ **要件が頻繁に変わる部分**

- モジュール化すると変更時の影響範囲が広がる
- 柔軟性が失われる

**ポイント**: モジュール化は「3回同じパターンが出てきたら」検討する、という経験則が有効です。

## コメントとドキュメント

### 意図を記述する

コードの「何をしているか」ではなく、「なぜそうしたか」(設計意図・ビジネス要件)を記述します。

```typescript
/**
 * ログファイル保管用バケット
 *
 * 背景:
 * - 監査要件により90日間のログ保管が義務付けられている
 * - コスト削減のため、古いログは低頻度アクセスストレージへ移行したい
 *
 * 設計判断:
 * - 30日経過後のログはほとんどアクセスされないためGlacierへ移行
 * - 運用チームのオペレーションミスによる誤削除を防ぐためバージョニング有効化
 */
const logBucket = new s3.Bucket(this, 'LogBucket', {
  lifecycleRules: [{
    // 30日経過後はGlacierへ移行(監査要件は満たしつつコスト削減)
    transitions: [{
      storageClass: s3.StorageClass.GLACIER,
      transitionAfter: cdk.Duration.days(30),
    }],
    // 90日経過後は削除(監査要件の保管期間)
    expiration: cdk.Duration.days(90),
  }],

  // 運用チームのオペレーションミス(aws s3 rm誤実行など)からの保護
  versioned: true,
});
```

### 設計判断の記録

```typescript
// NOTE: RDSではなくAurora Serverlessを選択した理由:
// - トラフィックが不定期で自動スケーリングが必要
// - 使用していない時間帯はコスト削減したい
// - マルチAZ構成で高可用性を確保
const cluster = new rds.ServerlessCluster(this, 'Database', {
  engine: rds.DatabaseClusterEngine.AURORA_MYSQL,
  scaling: {
    minCapacity: rds.AuroraCapacityUnit.ACU_2,
    maxCapacity: rds.AuroraCapacityUnit.ACU_16,
    autoPause: cdk.Duration.minutes(10),
  },
});
```

## まとめ

IaCコードの品質を高めるための重要な原則:

- **DRY原則**: 重複を避け、再利用可能な構造に
- **適切な分割**: 管理単位(Stack/State)やモジュールの責務を明確に
- **命名規則**: 一貫したパターンで可読性向上
- **パラメータ化**: 環境差分を設定ファイルで管理
- **セキュリティ**: シークレット管理の徹底
- **モジュール化**: 共通パターンは再利用可能なモジュールに
- **ドキュメント**: 設計意図をコメントで記録

これらの原則を守ることで、保守性が高く、チームで運用しやすいIaCコードを実現できます。

次のセクションでは、IaCコードの品質を保証するテストについて学びます。

[テストの重要性 →](testing.md){ .md-button .md-button--primary }
