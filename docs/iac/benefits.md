# IaCのメリット

Infrastructure as Code (IaC) を導入することで、従来の手作業によるインフラ管理と比較して多くのメリットが得られます。

## 1. 再現性と一貫性

### 従来の課題

手作業でのインフラ構築では、以下のような問題が発生します:

- 同じ手順書を見ても、人によって微妙に異なる設定になる
- 環境ごと(開発、ステージング、本番)に設定の差異が生まれる
- 「この設定、誰がいつ変更したんだっけ?」が分からない

### IaCによる解決

```typescript
// 同じコードから同じインフラが必ず作られる
const bucket = new s3.Bucket(this, 'DataBucket', {
  versioned: true,
  encryption: s3.BucketEncryption.S3_MANAGED,
  lifecycleRules: [{
    expiration: Duration.days(90),
  }],
});
```

- **同じコードからは必ず同じインフラが生成される**
- 環境差分はパラメータで明示的に管理
- 「いつ・誰が・何を」変更したかがGit履歴で明確

## 2. バージョン管理と履歴追跡

### Gitによる変更管理

IaCコードをGitで管理することで、ソフトウェア開発と同じメリットが得られます:

```bash
# 変更履歴の確認
$ git log --oneline
a1b2c3d Add lifecycle policy to S3 bucket
d4e5f6g Update Lambda memory to 512MB
g7h8i9j Initial infrastructure setup

# 差分の確認
$ git diff
- memory: 256,
+ memory: 512,

# 問題があれば即座にロールバック
$ git revert a1b2c3d
```

### メリット

- **変更理由の記録**: コミットメッセージで「なぜ」変更したかを記録
- **レビューの実施**: Pull Requestでインフラ変更をレビュー
- **ロールバック**: 問題発生時に簡単に以前の状態に戻せる
- **ブランチ戦略**: feature/fix ブランチで安全に変更を試せる

## 3. レビュー可能性

### コードレビューによる品質向上

手作業での変更は事後確認が困難ですが、IaCではPull Requestによる事前レビューが可能です。

```typescript
// レビュワーが指摘できる例
const database = new rds.DatabaseInstance(this, 'Database', {
  engine: rds.DatabaseInstanceEngine.postgres({ version: rds.PostgresEngineVersion.VER_15 }),
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
  vpc,
  // ❌ レビュー指摘: 本番環境ではバックアップが必須
  // backupRetention: Duration.days(7),
});
```

**レビューで確認できる項目**:

- セキュリティ設定の不備(パブリックアクセス、暗号化etc)
- コスト最適化の機会
- ベストプラクティスからの逸脱
- 命名規則の遵守

## 4. 自動化とCI/CD統合

### CI/CDパイプラインでのデプロイ

IaCコードをCI/CDパイプラインに統合することで、完全自動化が実現できます。

```yaml
# GitHub Actions の例
name: Deploy Infrastructure
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Node.js
        uses: actions/setup-node@v3
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test
      - name: Deploy to AWS
        run: npm run cdk deploy
```

### 自動化のメリット

- **人的ミスの削減**: 手作業によるコピペミスや設定漏れを防止
- **高速なデプロイ**: 数分〜数十分で完全なインフラを構築
- **テストの自動実行**: デプロイ前に必ず品質チェック
- **セキュリティスキャン**: 脆弱性検出を自動化

## 5. ドキュメントとしての役割

### 最新状態が常に明確

IaCコードは「実行可能なドキュメント」として機能します。

```typescript
// このコードを読めば、現在のインフラ構成が分かる
export class NetworkStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // VPC: 10.0.0.0/16
    const vpc = new ec2.Vpc(this, 'MainVpc', {
      maxAzs: 2,
      natGateways: 1,
      subnetConfiguration: [
        {
          name: 'Public',
          subnetType: ec2.SubnetType.PUBLIC,
          cidrMask: 24,
        },
        {
          name: 'Private',
          subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
          cidrMask: 24,
        },
      ],
    });
  }
}
```

### 従来のドキュメント管理の課題

- **更新漏れ**: Excelやconfluenceのドキュメントは古くなりがち
- **実態との乖離**: 「ドキュメントには書いてあるけど、実際は違う」
- **メンテナンスコスト**: ドキュメント更新に時間がかかる

### IaCによる解決

- **常に最新**: コード=実態なので乖離しない
- **理解しやすい**: コメントと合わせて設計意図を明示
- **自動生成**: コードから図や一覧表を自動生成可能

## 6. コスト最適化

### 環境の迅速な作成・破棄

```bash
# 開発環境を一時的に作成
$ cdk deploy DevEnvironment

# 作業完了後、即座に削除してコスト削減
$ cdk destroy DevEnvironment
```

- **不要環境の即座削除**: 使わない時は削除してコスト削減
- **コスト試算**: デプロイ前にコストを見積もり
- **リソース最適化**: コードレビューでコスト面も検討

## 7. コラボレーションの促進

### チーム開発での恩恵

- **並行開発**: 複数メンバーが異なるブランチで同時に作業
- **知識の共有**: コードレビューを通じて設計知識が共有される
- **標準化**: チーム共通のモジュールやConstructsで一貫性を保つ
- **オンボーディング**: 新メンバーがコードを読めば構成を理解できる

## まとめ

| 観点 | 従来の手作業 | IaC |
|------|-------------|-----|
| 再現性 | ❌ 人により異なる | ✅ 常に同じ結果 |
| 履歴管理 | ❌ 変更履歴が不明瞭 | ✅ Git履歴で完全追跡 |
| レビュー | ❌ 事後確認のみ | ✅ 事前レビュー可能 |
| 自動化 | ❌ 手作業が必須 | ✅ 完全自動化可能 |
| ドキュメント | ❌ 更新漏れ・乖離 | ✅ コード=ドキュメント |
| コスト | ❌ 削除忘れリスク | ✅ 簡単に作成・破棄 |

次のセクションでは、IaCの核心概念である「宣言的定義」について、命令的定義と比較しながら学びます。

[宣言的定義 vs 命令的定義 →](declarative-vs-imperative.md){ .md-button .md-button--primary }
