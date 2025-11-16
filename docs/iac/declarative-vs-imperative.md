# 宣言的定義 vs 命令的定義

IaCを理解する上で最も重要な概念の一つが、**宣言的定義(Declarative)**と**命令的定義(Imperative)**の違いです。

## 命令的アプローチとは

命令的アプローチでは、**「どうやって」実現するか**を一つ一つの手順として記述します。

### AWS CLIでの例(命令的)

S3バケットを作成し、バージョニングと暗号化を有効にする場合:

```bash
# 1. バケットを作成
aws s3api create-bucket \
  --bucket my-app-data-bucket \
  --region ap-northeast-1 \
  --create-bucket-configuration LocationConstraint=ap-northeast-1

# 2. バージョニングを有効化
aws s3api put-bucket-versioning \
  --bucket my-app-data-bucket \
  --versioning-configuration Status=Enabled

# 3. 暗号化を設定
aws s3api put-bucket-encryption \
  --bucket my-app-data-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# 4. ライフサイクルポリシーを設定
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-app-data-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "Id": "DeleteOldVersions",
      "Status": "Enabled",
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      }
    }]
  }'
```

### 命令的アプローチの課題

❌ **手順の順序依存**

- コマンドの実行順序を間違えるとエラーになる
- バケットが存在しないとバージョニング設定はできない

❌ **冪等性がない**

- 同じコマンドを2回実行するとエラーになる可能性
- 既に存在するリソースへの対応が必要

❌ **現在の状態が不明**

- 「今、どういう状態か」を把握するには別途確認が必要
- 設定漏れや設定ミスに気づきにくい

❌ **変更管理が困難**

- 「何を変更したか」の記録が残らない
- ロールバックが難しい

## 宣言的アプローチとは

宣言的アプローチでは、**「何が欲しいか」(あるべき姿)**を記述します。「どうやって」実現するかはツールが自動で判断します。

### CDKでの例(宣言的)

同じS3バケットをCDKで定義:

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { Construct } from 'constructs';

export class StorageStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // あるべき姿を宣言するだけ
    const bucket = new s3.Bucket(this, 'DataBucket', {
      bucketName: 'my-app-data-bucket',
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      lifecycleRules: [
        {
          noncurrentVersionExpiration: cdk.Duration.days(90),
        },
      ],
    });
  }
}
```

### Terraformでの例(宣言的)

同じ構成をTerraformで定義:

```hcl
resource "aws_s3_bucket" "data_bucket" {
  bucket = "my-app-data-bucket"
}

resource "aws_s3_bucket_versioning" "data_bucket" {
  bucket = aws_s3_bucket.data_bucket.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data_bucket" {
  bucket = aws_s3_bucket.data_bucket.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "data_bucket" {
  bucket = aws_s3_bucket.data_bucket.id

  rule {
    id     = "delete-old-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
}
```

## 宣言的定義のメリット

### 1. 冪等性の保証

何度実行しても同じ結果になります。

```typescript
// 1回目: バケットを作成
cdk deploy

// 2回目: 変更がないので何もしない
cdk deploy

// 設定を変更してデプロイ: 差分だけを適用
bucket.lifecycleRules[0].noncurrentVersionExpiration = cdk.Duration.days(60);
cdk deploy  // ← ライフサイクルルールのみ更新
```

### 2. 状態管理の自動化

現在の状態とあるべき姿を比較し、差分だけを適用します。

```bash
# CDKの場合
$ cdk diff
Stack StorageStack
Resources
[~] AWS::S3::Bucket DataBucket
 └─ [~] LifecycleConfiguration
     └─ [~] .Rules[0].NoncurrentVersionExpiration.NoncurrentDays:
         - 90
         + 60
```

```bash
# Terraformの場合
$ terraform plan
Terraform will perform the following actions:

  # aws_s3_bucket_lifecycle_configuration.data_bucket will be updated in-place
  ~ resource "aws_s3_bucket_lifecycle_configuration" "data_bucket" {
      ~ rule {
          ~ noncurrent_version_expiration {
              ~ noncurrent_days = 90 -> 60
            }
        }
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

### 3. 順序を気にしなくて良い

ツールが依存関係を解決し、正しい順序で実行します。

```typescript
// この順序でも動く
const bucket = new s3.Bucket(this, 'DataBucket', {
  versioned: true,
  encryption: s3.BucketEncryption.S3_MANAGED,
});

// Lambdaがバケットを参照していても、
// CDKが自動的にバケット→Lambda の順序でデプロイ
const lambda = new lambda.Function(this, 'Processor', {
  environment: {
    BUCKET_NAME: bucket.bucketName,  // 依存関係を自動検出
  },
});
```

### 4. 可読性の向上

コードを読めば、現在のインフラ構成が一目でわかります。

```typescript
// このコードを読めば、以下が分かる:
// - バケット名
// - バージョニング有効
// - 暗号化有効
// - 90日でバージョン削除
const bucket = new s3.Bucket(this, 'DataBucket', {
  bucketName: 'my-app-data-bucket',
  versioned: true,
  encryption: s3.BucketEncryption.S3_MANAGED,
  lifecycleRules: [{
    noncurrentVersionExpiration: cdk.Duration.days(90),
  }],
});
```

### 5. 変更の安全性

変更前に差分確認ができ、意図しない変更を防げます。

```bash
# 変更内容を事前確認
$ cdk diff

# 問題なければデプロイ
$ cdk deploy

# 問題があれば即座にロールバック
$ git revert HEAD
$ cdk deploy
```

## 比較表

| 観点 | 命令的(CLI) | 宣言的(IaC) |
|------|------------|------------|
| **記述内容** | 手順(How) | あるべき姿(What) |
| **冪等性** | ❌ なし | ✅ あり |
| **状態管理** | ❌ 手動 | ✅ 自動 |
| **差分確認** | ❌ 困難 | ✅ 簡単(`diff`コマンド) |
| **ロールバック** | ❌ 手動で逆操作 | ✅ 前のコードをデプロイ |
| **可読性** | ❌ 低い(ログを追う必要) | ✅ 高い(コードを読めば分かる) |
| **再現性** | ❌ 低い | ✅ 高い |

## 実際の運用例

### シナリオ: バケットの暗号化アルゴリズムを変更したい

#### 命令的アプローチ(CLI)の場合

```bash
# 1. 現在の設定を確認
aws s3api get-bucket-encryption --bucket my-app-data-bucket

# 2. 新しい設定を適用
aws s3api put-bucket-encryption \
  --bucket my-app-data-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:ap-northeast-1:123456789012:key/xxx"
      }
    }]
  }'

# 3. 変更されたか確認
aws s3api get-bucket-encryption --bucket my-app-data-bucket

# 問題: この変更履歴はどこにも残らない
```

#### 宣言的アプローチ(CDK)の場合

```typescript
// コードを変更
const bucket = new s3.Bucket(this, 'DataBucket', {
  bucketName: 'my-app-data-bucket',
  versioned: true,
  encryption: s3.BucketEncryption.KMS,  // ← ここを変更
  encryptionKey: kmsKey,
  lifecycleRules: [{
    noncurrentVersionExpiration: cdk.Duration.days(90),
  }],
});

// 差分を確認
// $ cdk diff

// デプロイ
// $ cdk deploy

// Gitで変更履歴が残る
// $ git log --oneline
// abc123 Change S3 encryption to KMS
```

## まとめ

- **命令的定義**: 「どうやるか」を手順として記述。柔軟だが管理が大変
- **宣言的定義**: 「何が欲しいか」を記述。ツールが状態管理を自動化

IaCでは宣言的定義を採用することで、インフラの**再現性・可視性・保守性**が大幅に向上します。

次のセクションでは、宣言的定義で書かれたIaCコードの品質を高めるためのコーディング原則について学びます。

[コーディング原則 →](coding-principles.md){ .md-button .md-button--primary }
