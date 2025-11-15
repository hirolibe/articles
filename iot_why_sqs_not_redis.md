## 1. はじめに

「SQSからのshoryukenでいきましょう」

私が未経験からエンジニアに転職して初めての案件打合せで耳にした言葉です。
SQS？shoryuken？私には何のことかさっぱり分かりませんでした・・・。

そこでこの記事では、Railsをバックエンドとして構築するIoTシステムで使用される基本的なシステム構成について、初心者の視点でまとめてみました。

#### この記事の対象読者

次のような初心者エンジニアを想定しています。

- Job、Queue、Workerという言葉を聞いたことがあるがいまいち理解できていない
- SQSやshoryukenとは何か、RedisとSidekiqとの違いが分からない

## 2. そもそも「Job」「Queue」「Worker」って何？

私自身、背景ジョブ処理の基本用語の理解が曖昧だったので、超基本的な部分からまとめていきます。

### 2-1. 用語整理

#### Job
- やるべき仕事の単位
- 例：「ユーザーにメール送信」「センサーデータをDBに保存」

#### Queue
- ジョブを順番に並べて保管する場所（待ち行列）
- 例：Redis、Amazon SQSなど

#### Worker
- キューからジョブを取り出して実行するプログラム
- 例：Sidekiq、shoryukenなど

### 2-2. 基本的な仕組み

```
アプリケーション (ジョブを登録)
    ↓
Queue (ジョブを保管)
    ↓
Worker (ジョブを実行)
```

**ポイント**: WorkerはAPIを叩かれて動くのではなく、**キューを常時ポーリング（監視）** しています。

```ruby
# Workerの動作イメージ（擬似コード）
loop do
  job = queue.poll()  # キューをチェック
  if job
    process(job)      # ジョブがあれば処理
    job.delete()      # 完了後削除
  end
  sleep(1)            # 1秒待機して再度チェック
end
```

### 2-3. Queueの選択肢

IoTシステムでジョブ処理を実装する場合、**Redis**もしくは**SQS**という選択肢があります。
まず最初に、IoTシステムでRedisを使用することを考えてみます。

## 3. IoTシステムでRedisを使用する場合

### 3-1. IoTにおける通信プロトコルの制約

IoTデバイスから大量のデータを送信する際、以下の理由から**MQTT（Message Queuing Telemetry Transport）** が一般的に使われます。

- **軽量**: 組み込みデバイスでも動作する省リソース設計
- **省電力**: バッテリー駆動デバイスに適している
- **双方向通信**: デバイスへのコマンド送信も可能

しかし、RedisはHTTP/HTTPSのみ対応しており、MQTTを直接処理できないため、プロトコルの変換が必要となります。

### 3-2. アーキテクチャ

仮にRedisを使った場合のアーキテクチャを考えてみましょう。

```
IoTデバイス
    ↓ MQTT
AWS IoT Core（MQTTブローカー）
    ↓ Lambda（MQTT→HTTP/HTTPS変換してRedis登録）
Redis (Queue)
    ↓ ジョブを取得
Worker
    ↓
RDS (データベース)
```

### 3-3. 問題点

上記のアーキテクチャから、下記の問題点があることがわかりました。
- IoT CoreからRedisへの直接送信不可 → Lambda等での変換処理が必須
- 4段階構成（IoT Core → Lambda → Redis → Worker）になり、障害ポイントが増加
- デバッグ・監視・メンテナンスコストの増大

## 4. IoTシステムでSQSを使用する場合

### 4-1. アーキテクチャ

SQSを利用する場合のアーキテクチャは次のようになります。

```
IoTデバイス
    ↓ MQTT
AWS IoT Core（MQTTブローカー）
    ↓
SQS (Queue)
    ↓ ジョブを取得
Worker
    ↓
RDS (データベース)
```

### 4-2. 主なメリット

**シンプルな構成**
- IoT CoreからSQSへ直接送信可能（Lambda不要）
- Redis構成の4段階 → SQS構成の3段階に削減

**高い信頼性**
- マネージドサービス（99.9%以上のSLA）
- サーバー監視・障害対応・スケーリングが不要

### 4-3. 実装例

最後に、SQSを使用した場合の実装例についてまとめます。

#### IoT Core ルール設定

IoT CoreからSQSにメッセージを送信します。

```sql
-- IoT Rule SQL
SELECT temperature, humidity, device_id, timestamp() as received_at
FROM 'iot/sensors/+'
WHERE temperature > 30
```

#### Workerの実装

```ruby
# app/workers/sensor_data_processor.rb
class SensorDataProcessor
  include Shoryuken::Worker
  shoryuken_options queue: 'sensor-data-queue', auto_delete: true

  def perform(sqs_msg, body)
    data = JSON.parse(body)

    # データベースに保存（冪等性を考慮）
    SensorReading.find_or_create_by!(
      device_id: data['device_id'],
      received_at: data['received_at']
    ) do |reading|
      reading.temperature = data['temperature']
      reading.humidity = data['humidity']
    end

    # 閾値超過なら通知
    if data['temperature'] > 40
      AlertService.notify_high_temperature(data)
    end
  end
end
```

## 5. コスト削減の可能性について

ここまでSQS + Worker（shoryuken）の構成を紹介してきましたが、Workerの代わりにLambdaを使用することで、コスト削減が可能になる場合があります。

参考リンク：[SQS + Lambda実装パターン (Speaker Deck)](https://speakerdeck.com/pikosan0000/sqs-lambda-devday2022)

### 5-1. Lambda構成のメリット

```
IoTデバイス → IoT Core → SQS → Lambda（イベント駆動） → RDS
```

**主なメリット**
- **従量課金**: 実際の処理時間のみ課金（Worker構成はEC2の常時起動が必要）
- **サーバー管理不要**: EC2インスタンスの監視・パッチ適用が不要
- **自動スケーリング**: トラフィック変動に応じて自動でスケール

特に**データ送信量が少ない**または**時間帯により変動が大きい**IoTシステムでは、Lambda構成によるコスト削減効果が期待できます。

### 5-2. 構成の選択基準

**Worker構成が適するケース**
- 大量・連続的な処理が24時間発生する
- 15分以上かかる長時間処理

**Lambda構成が適するケース**
- データ送信がバースト的、または間欠的
- 短時間（数秒以内）で完了する処理

システムの特性に応じて、最適な構成を選択することが重要です。Lambda構成の詳細な実装やコスト試算については、別の記事でまとめる予定です。

## 6. まとめ

### 6-1. Queueの選択について

**RedisとSQSの違い**
- Redis: MQTTに非対応 → Lambda変換が必要（4段階構成）
- SQS: IoT Coreから直接連携可能（3段階構成）

**IoTシステムでSQSを選ぶ理由**
- シンプルな構成でメンテナンスコストを削減
- マネージドサービスとして高い信頼性を提供
- デバイス数増加に応じた柔軟なスケーリング

### 6-2. WorkerとLambdaの選択について
- Worker構成からLambda構成への移行でコスト削減の可能性
- システムの特性（処理量・頻度）に応じた構成選択が重要

IoTシステムでは、MQTTプロトコルとの親和性を重視し、SQSを選択することでシンプルかつ堅牢なアーキテクチャを構築できます。 

## 7. 参考リンク

- [Sidekiq公式ドキュメント](https://github.com/sidekiq/sidekiq)
- [shoryuken公式ドキュメント](https://github.com/ruby-shoryuken/shoryuken)
- [AWS IoT Core - SQS統合](https://docs.aws.amazon.com/iot/latest/developerguide/iot-rule-actions.html)
- [Amazon SQS開発者ガイド](https://docs.aws.amazon.com/sqs/)
- [SQS + Lambda実装パターン (Speaker Deck)](https://speakerdeck.com/pikosan0000/sqs-lambda-devday2022)
