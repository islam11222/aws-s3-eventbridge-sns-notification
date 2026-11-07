# AWS サーバーレス・ファイルアップロード通知システム

**Amazon S3 × Amazon EventBridge × Amazon SNS**  
**Serverless File Upload Notification System**

> 個人学習・検証プロジェクト | AWS リージョン: 東京 (`ap-northeast-1`) | 検証日: 2026年11月7日

**[English documentation / 英語版](README_EN.md)** | [設定・動作確認のスクリーンショット](screenshots/)
[システム構成図を見る](architecture/diagram.md)

## 1. プロジェクト概要

Amazon S3 バケットにファイルがアップロードされた際に、Amazon EventBridge でイベントを検知・フィルタリングし、Amazon SNS を通じて確認済みのメールアドレスへ通知する仕組みを構築しました。

EC2 や常時稼働するサーバーを使用せず、AWS のマネージドサービスを組み合わせたイベント駆動型の個人学習プロジェクトです。

**検証結果:** S3 にテストファイルをアップロードした後、SNS 経由の通知メールを実際に受信できました。

## 2. 想定した業務課題

社内でファイルを共有ストレージに保存する運用では、アップロード後に担当者へ別途連絡する必要があります。連絡忘れや確認の遅れを減らすため、アップロードをきっかけに自動通知する構成を検討しました。

本プロジェクトでは「特定バケットへのファイルアップロード時に、登録した宛先へメール通知する」というシンプルな要件に絞って実装・検証しています。

## 3. アーキテクチャ

```text
[File upload]
      |
      v
[Private Amazon S3 bucket]
      |  Object Created event
      v
[Amazon EventBridge / default event bus]
      |  Rule: source = aws.s3
      |        detail-type = Object Created
      |        bucket name = target bucket
      v
[Input Transformer]
      |  Select file name / bucket / time / region
      v
[Amazon SNS / Standard topic]
      |  Confirmed email subscription
      v
[Email notification]
```

**処理の流れ**

1. S3 にテストファイルをアップロードします。
2. S3 から EventBridge へイベントが送信されます。
3. EventBridge のルールがイベント種別と対象バケット名を照合します。
4. Input Transformer がイベント情報から必要な項目を取り出します。
5. SNS トピックを通じて、承認済みのメール購読先に通知します。

## 4. 使用サービスと選定理由

| サービス | 役割 | 採用理由 |
|---|---|---|
| Amazon S3 | ファイル保存・イベント発生元 | オブジェクトストレージとして利用できるため |
| Amazon EventBridge | イベントの受信・条件による振り分け | 対象バケットと `Object Created` をルールで絞り込めるため |
| Amazon SNS (Standard) | メール通知 | Email サブスクリプションを利用できるため |
| IAM / SNS Topic Access Policy | アクセス権限の制御 | EventBridge から SNS への `sns:Publish` を許可するため |
| EventBridge Input Transformer | 通知内容の整形 | イベント全体ではなく、必要なフィールドを通知するため |

Lambda、EC2、NAT Gateway は、今回の要件を満たすためには追加する必要がないと判断しました。

## 5. 主な設定

### Amazon S3

- リージョン: Asia Pacific (Tokyo) / `ap-northeast-1`
- バケットタイプ: General purpose
- Block all public access: **ON**
- Object Ownership: **Bucket owner enforced（ACL 無効）**
- デフォルト暗号化: **SSE-S3**
- バージョニング: **無効（学習用の簡易構成）**
- Amazon EventBridge へのイベント送信: **ON**

### Amazon EventBridge

- Event bus: `default`
- Rule: `s3-file-upload-notification-rule`
- Rule status: **Enabled**
- Event source: `aws.s3`
- Detail type: `Object Created`
- フィルタ条件: 指定した S3 バケット名
- Target: SNS トピック `business-file-upload-alerts`
- Target input: Input Transformer

学習時のルールを匿名化した例です。`<project-bucket>` を自分のバケット名に置き換えて使用します。

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["<project-bucket>"]
    }
  }
}
```

Input Transformer では、次の JSONPath を利用してイベント情報を取得しました。

```json
{
  "fileName": "$.detail.object.key",
  "bucketName": "$.detail.bucket.name",
  "uploadTime": "$.time",
  "region": "$.region"
}
```

### Amazon SNS

- Topic type: **Standard**
- Topic name: `business-file-upload-alerts`
- Subscription protocol: **EMAIL**
- Subscription status: **Confirmed**
- Topic access policy: `events.amazonaws.com` に対する `sns:Publish` を許可

## 6. セキュリティ設計

- S3 バケットのパブリックアクセスをブロックし、テストファイルを一般公開しない構成にしました。
- S3 の ACL は無効化し、バケット所有者を基準とする設定を使用しました。
- 保存データの暗号化として SSE-S3 を使用しました。
- EventBridge から SNS へ通知するため、SNS トピックのリソースベースポリシーを確認しました。
- GitHub には AWS アカウント ID、個人メールアドレスなどを伏せた検証用画面を掲載します。

## 7. 動作確認

| テスト項目 | 確認結果 |
|---|---|
| S3 へのテストファイルアップロード | 成功 |
| `Object Created` イベントを使った通知 | 成功 |
| 指定 SNS トピックへの通知連携 | 成功 |
| SNS Email サブスクリプション | Confirmed |
| メール受信 | 成功 |
| Input Transformer による必要項目の抽出 | 成功 |

**テストファイル例:** `AWS-Project3-Final-Test.txt`

受信したメールで、ファイル名・バケット名・アップロード日時・AWS リージョンを確認しました。スクリーンショットは [`screenshots/`](screenshots/) を参照してください。

## 8. トラブルシューティングと学び

### IAM 権限不足

作業中、IAM ユーザーに必要な権限が不足し、次のエラーが発生しました。

- `sqs:ListQueues` — コンソール上でキュー一覧の取得が拒否された
- `iam:CreateRole` — EventBridge のデフォルト実行ロールの作成が拒否された
- `iam:ListRoles` — 既存ロールの一覧取得が拒否された

この経験から、**コンソール操作に必要な IAM 権限**と、**AWS サービス間の連携に必要なリソースベースポリシー**を分けて考える重要性を学びました。

SNS ターゲットについては、実行ロールを新規作成しない構成を使用し、SNS トピック側に EventBridge の `sns:Publish` 許可が設定されていることを確認しました。その後、ファイルアップロードからメール受信まで動作確認できました。

### 通知本文のフォーマット

最初は SNS メールにイベント全体の JSON が表示されました。そこで EventBridge Input Transformer を使用し、ファイル名、バケット名、アップロード日時、リージョンを抽出しました。

**現時点の制限:** メール本文の各行に引用符が表示される点は未解決です。必要項目の抽出と通知配信は成功しており、本文の見栄えの改善は今後の課題として整理しました。

## 9. コストを意識した設計

- 常時稼働する EC2 インスタンスを使用しない構成にしました。
- NAT Gateway、Lambda、追加のデータベースを使用していません。
- テストには小さいテキストファイルを使用しました。
- イベント数・通知数・S3 ストレージ量などによって料金が発生する可能性があるため、AWS の料金体系を確認しながら検証しました。

※ 本プロジェクトは「必ず無料で動作する」ことを保証するものではありません。実測のプロジェクト別料金も算出していません。

## 10. 今後の改善案

- 日本語・英語の通知本文をさらに読みやすく整える
- ファイル拡張子やプレフィックスに応じたイベントフィルタリング
- CloudWatch メトリクスによる通知処理の可視化
- Dead-letter queue やエラー時の運用方法の検討
- AWS Infrastructure as Code（CloudFormation / Terraform）による再現性の向上

## 11. このプロジェクトで学んだこと

S3、EventBridge、SNS を組み合わせ、イベント発生からメール通知までの基本的な連携を実際に検証しました。また、権限エラーの切り分けや構成変更後の再テストを通じて、クラウドサービスを設定するだけでなく、**原因を調べて確認し、結果を記録すること**の大切さを学びました。

---

**Project type:** Personal hands-on learning project (not a production deployment)  
**AWS region:** `ap-northeast-1` (Tokyo)
---

## 12. 設定・動作確認のスクリーンショット

以下は、AWS S3・EventBridge・SNS の設定および動作確認の記録です。

### 1. EventBridge ルールの設定

![EventBridge Rule](screenshots/ChatGPT%20Image%20Nov%207%2C%202026%2C%2009_59_55%20PM-1.png)

### 2. SNS メール購読の確認

![SNS Subscription](screenshots/ChatGPT%20Image%20Nov%207%2C%202026%2C%2009_59_57%20PM-2.png)

### 3. EventBridge ルールの有効化

![EventBridge Enabled](screenshots/ChatGPT%20Image%20Nov%207%2C%202026%2C%2009_59_59%20PM-3.png)

### 4. S3 パブリックアクセスのブロック

![S3 Security](screenshots/ChatGPT%20Image%20Nov%207%2C%202026%2C%2010_00_05%20PM-4.png)

### 5. S3 と EventBridge の連携

![S3 EventBridge](screenshots/ChatGPT%20Image%20Nov%207%2C%202026%2C%2010_00_10%20PM-5.png)

### 6. SNS メール通知の受信結果

![SNS Email](screenshots/ChatGPT%20Image%20Nov%207%2C%202026%2C%2010_00_15%20PM-6.png)

### 7. EventBridge と SNS のターゲット設定

![EventBridge Target](screenshots/ChatGPT%20Image%20Nov%207%2C%202026%2C%2010_00_19%20PM-7.png)
