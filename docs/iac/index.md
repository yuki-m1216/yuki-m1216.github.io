# Infrastructure as Code (IaC)

Infrastructure as Code（IaC）は、インフラストラクチャをコードで定義・管理する手法です。手作業での設定やクリックオペレーションの代わりに、プログラミング言語や宣言的な記述でインフラを構築します。

## このセクションで学べること

このセクションでは、IaCの基礎から実践までを体系的に学べます。

### 📚 基礎知識

- **[IaCのメリット](benefits.md)**: なぜIaCが必要なのか、従来の手法との違い
- **[宣言的定義 vs 命令的定義](declarative-vs-imperative.md)**: IaCの核心概念を理解する

### 🛠️ 実践スキル

- **[コーディング原則](coding-principles.md)**: 保守性の高いIaCコードを書くための原則
- **[テストの重要性](testing.md)**: インフラコードの品質を保証するテスト手法 🚧 **編集中**
- **[ディレクトリ構成](directory-structure.md)**: 実プロジェクトでの推奨ディレクトリ構成 🚧 **編集中**
- **[管理単位のベストプラクティス](management-units.md)**: スタック/stateの適切な分割方法 🚧 **編集中**

### 💡 実例で学ぶ

- **[CDK基本例](examples/cdk-basic.md)**: シンプルな構成から始める 🚧 **編集中**
- **[CDK応用例](examples/cdk-advanced.md)**: 本番レベルの複雑な構成 🚧 **編集中**
- **[Terraform基本例](examples/terraform-basic.md)**: シンプルな構成から始める 🚧 **編集中**
- **[Terraform応用例](examples/terraform-advanced.md)**: 本番レベルの複雑な構成 🚧 **編集中**

## IaCツールの位置づけ

このセクションでは主に以下のツールを取り扱います:

### AWS CDK (Cloud Development Kit)

- **言語**: TypeScript, Python, Java等のプログラミング言語
- **対象**: AWS専用
- **特徴**: プログラミング言語の表現力を活かせる、型安全性

### Terraform

- **言語**: HCL (HashiCorp Configuration Language)
- **対象**: マルチクラウド対応
- **特徴**: 豊富なプロバイダー、広範なコミュニティ

## なぜIaCを学ぶのか

現代のクラウドインフラ運用において、IaCは以下の理由から必須のスキルとなっています:

- **再現性**: 同じコードから同じインフラを何度でも構築可能
- **バージョン管理**: Gitで変更履歴を追跡、レビュー、ロールバック
- **自動化**: CI/CDパイプラインに統合し、デプロイを自動化
- **ドキュメント**: コード自体が最新のインフラ構成のドキュメント
- **コラボレーション**: チーム開発におけるレビューと共同作業

次のページで、これらのメリットを詳しく見ていきましょう。

[IaCのメリットを学ぶ →](benefits.md){ .md-button .md-button--primary }
