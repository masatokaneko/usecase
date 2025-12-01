# DB-12 DynamoDB/CosmosDBトランザクション補完

## 概要

クラウドネイティブアプリケーションで多用されるNoSQLデータベース（DynamoDBやCosmos DB）は、ベンダーが提供するトランザクション機能にいくつかの制約が存在する。例えばDynamoDBでは、TransactWriteItemsに含められる操作は最大25件（総ペイロード4 MB）までで（参考: [dynomate.io](https://dynomate.io)）、トランザクションは単一AWSリージョン内に限定され、分離レベルを調整することはできない（参考: [dynomate.io](https://dynomate.io)）。Azure Cosmos DBもACIDトランザクションを提供しているが、1つの論理パーティション内に限られ（参考: [learn.microsoft.com](https://learn.microsoft.com)）、トランザクションバッチ全体のサイズは2 MB以下、操作数は最大100までという制限がある（参考: [learn.microsoft.com](https://learn.microsoft.com)）。このユースケースでは、こうしたNoSQL DBのトランザクション制約を補完し、複数テーブルや複数パーティションに跨る処理を可能にする。

## ビジネス課題

- サブスクリプション管理や在庫管理などで複数テーブルを一括更新する必要があるが、DynamoDBのトランザクションは25件／4 MBに制限され（参考: [dynomate.io](https://dynomate.io)）、Cosmos DBは同一パーティション内にしか適用できないため（参考: [learn.microsoft.com](https://learn.microsoft.com)）、要件を満たせない。
- NoSQLは高スケーラビリティを提供する一方、アイテム数やパーティションを跨ぐトランザクションが制限されるため、ピーク時のパフォーマンスや一貫性の確保が難しい。

## システム課題

- DynamoDBでは1回のトランザクションで更新できるアイテム数が25件（最大4 MB）まで、かつ複数リージョンに跨るトランザクションは非対応で isolation level の選択肢もない（参考: [dynomate.io](https://dynomate.io)）。
- Azure Cosmos DBではトランザクションが同一論理パーティションに限定され、バッチサイズは2 MB以下、操作は100件までに制限される（参考: [learn.microsoft.com](https://learn.microsoft.com), [learn.microsoft.com](https://learn.microsoft.com)）。
- これらNoSQL DBと他のRDBMSを併用している場合、テーブルやパーティションを跨いだ一貫性を保証する機構が不足している。

## 対象データ・環境

- DynamoDB/CosmosDBに保存された注文データやログデータ。
- 連携するRDBMSやキャッシュサーバーとの整合性が必要な環境。

## 導入目的・効果

- NoSQLのスケーラブルな特性を維持しつつ、複数テーブル間の一貫したトランザクションを実装する。
- 大量データを扱うバッチ処理や分析処理の効率を高める。

## 解決方法（ソリューション概要）

ScalarDBをNoSQL DBの上位に配置してトランザクションを調停し、DynamoDBやCosmos DB本来の制限を超えた複数テーブル・複数パーティションへの一括更新を実現する。アプリケーションはScalarDBのAPIを通じて操作を発行し、ScalarDBがトランザクション境界を管理することで、バッチサイズやパーティションを跨いだ整合性を保証する。また、RDBMSとNoSQLを併用する場合でも、同じトランザクション境界で更新処理を実行できる。

## システム構成

- アプリケーションからの書き込み要求をScalarDBが受け取り、DynamoDB/CosmosDBのテーブルへの操作をまとめて実行。
- 必要に応じてRDBMSやキャッシュサーバーへの更新も同時に行い、結果をアプリケーションに返却。

## 導入メリット／ROI

- NoSQLの持つスケーラビリティを損なわずに、複数テーブルを跨るトランザクションを実現することで開発者の負担を削減。
- DynamoDB/CosmosDBが持つ制約による性能ボトルネックを解消し、可用性と整合性を両立。

## 類似事例・参考資料

- グローバルECサイトなどで、注文情報の整合性を保つためにScalarDBを用いてDynamoDBとRDBの両方へトランザクションを実行した事例がある。
