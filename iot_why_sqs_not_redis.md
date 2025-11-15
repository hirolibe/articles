## はじめに

「SQSからのshoryukenでいきましょう」

私が未経験からエンジニアに転職して初めての案件打合せで耳にした言葉です。SQS？shoryuken？私には何のことかさっぱり分かりませんでした。

そこでこの記事では、IoTシステムで使用される基本的なシステム構成について、初心者の視点でまとめてみました。

## この記事の対象読者

次のような初心者エンジニアを想定しています。

- Job、Queue、Workerという言葉を聞いたことがあるがいまいち理解できていない
- SQSやshoryukenとは何か、RedisとSidekiqとの違いが分からない

## そもそも「Job」「Queue」「Worker」って何？

私自身、背景ジョブ処理の基本用語の理解が曖昧だったので、超基本的な部分からまとめていきます。

### 用語整理

- **Job**: やるべき仕事の単位
  - 例: 「ユーザーにメール送信」「センサーデータをDBに保存」

- **Queue**: ジョブを順番に並べて保管する場所（待ち行列）
  - 例: Redis、Amazon SQSなど

- **Worker**: キューからジョブを取り出して実行するプログラム
  - 例: Sidekiq、shoryukenなど

### 基本的な仕組み

```
アプリケーション (ジョブを登録)
    ↓
Queue (ジョブを保管)
    ↓
Worker (ジョブを実行)
```

**重要なポイント**: WorkerはAPIを叩かれて動くのではなく、**キューを常時ポーリング（監視）** しています。

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

## Railsの2つのWorker選択肢

Railsで背景ジョブ処理を実装する場合、主に2つの選択肢があります。

### 1. Sidekiq + Redis（定番）

```ruby
# app/workers/notification_worker.rb
class NotificationWorker
  include Sidekiq::Worker

  def perform(user_id)
    user = User.find(user_id)
    NotificationMailer.send_email(user).deliver_now
  end
end

# Webアプリから呼び出し
NotificationWorker.perform_async(current_user.id)
```

- **メリット**: 高速、シンプル、情報が豊富
- **用途**: ユーザーリクエストから発生するジョブ

### 2. shoryuken + SQS（AWS環境）

```ruby
# app/workers/iot_data_processor.rb
class IotDataProcessor
  include Shoryuken::Worker
  shoryuken_options queue: 'iot-data-queue'

  def perform(sqs_msg, body)
    data = JSON.parse(body)
    SensorReading.create!(
      device_id: data['device_id'],
      temperature: data['temperature']
    )
  end
end
```

- **メリット**: AWS統合、スケーラビリティ、可用性
- **用途**: IoTデバイスや外部システムからの大量データ

## 疑問：「IoTシステムでRedis使うって選択肢はないの？」

ここで私の疑問が浮かびます。

**「RedisでもIoTデータを処理できるのでは？」**

結論から言うと、技術的には可能ですが、アーキテクチャ的に問題がありました。

## Redisパターンの何が問題なのか？

### 前提：IoTにおける通信プロトコル

IoTデバイスから大量のデータを送信する際、**MQTT（Message Queuing Telemetry Transport）** が一般的に使われます。

- **軽量**: 組み込みデバイスでも動作する省リソース設計
- **省電力**: バッテリー駆動デバイスに適している
- **双方向通信**: デバイスへのコマンド送信も可能

**重要な制約**: MQTTを使うには**MQTTブローカー**が必要です。通常のWeb App（Rails等）はHTTP/HTTPSのみ対応しており、MQTTを直接処理できません。

**本記事の焦点**: 以下では、プロトコルの違いも含めた**アーキテクチャ全体の違い**を見ていきます。

### Redisパターンのアーキテクチャ

Sidekiq（Redis）を使った場合のアーキテクチャを見てみましょう。

```
IoTデバイス（1,000台）
    ↓ HTTPS（REST API）
Rails Web App ← ★ここがボトルネック★
    ↓ Redisにジョブ登録
Redis (Queue)
    ↓ ジョブを取得
Worker (Sidekiq)
    ↓
RDS (データベース)
```

**このパターンの特徴**:
- IoTデバイスは**HTTPS（REST API）でWeb Appに直接送信**
- Web AppはHTTP/HTTPSのみ対応（MQTTは使えない）
- Web Appがすべてのデータを受け取る必要がある

### 問題点1: Web Appがボトルネックになる

1,000台のIoTデバイスが1秒に1回データを送信する場合、Web Appは**1秒間に1,000リクエストを受け取る**必要があります。

- Web AppはユーザーのUI操作も処理している
- IoTデータ受信のためにWeb Appをスケールするのは本末転倒
- Web Appの責任が多すぎる（UI処理 + IoTデータ受信）

### 問題点2: Web Appダウン = データロスト

Web Appがダウンすると：
- IoTデバイスからのデータが受信できない
- Redisにジョブが登録されない
- データが失われる可能性がある

### 問題点3: アーキテクチャ的に分離できていない

Web AppとIoTデータ処理が密結合しているため：
- Web Appの変更がIoTデータ処理に影響する
- 独立してスケールできない
- 責任の分離ができていない

## SQSパターンが解決する理由

### SQSパターンのアーキテクチャ

ここでSQS（Amazon Simple Queue Service）を使ったアーキテクチャを見てみましょう。

```
IoTデバイス（1,000台）
    ↓ MQTT
AWS IoT Core（MQTTブローカー）
    ↓ IoT Rule（SQL）
SQS (Queue) ← ★Web Appを経由しない★
    ↓ メッセージ取得
Worker (shoryuken on ECS)
    ↓
RDS (データベース)
```

**このパターンの特徴**:
- **AWS IoT CoreがMQTTブローカー**として機能
- IoTデバイスは軽量な**MQTTプロトコル**でIoT Coreに接続
- **Web Appを完全にバイパス**してデータ処理

### 解決策1: アーキテクチャの分離

**2つのパターンのアーキテクチャ比較**:

| パターン | プロトコル | データフロー | Web Appの役割 |
|---------|----------|------------|-------------|
| **Redisパターン** | HTTPS | IoTデバイス → **Web App** → Redis → Worker | IoTデータ受信 + UI処理 |
| **SQSパターン** | MQTT | IoTデバイス → **IoT Core** → SQS → Worker | UI処理のみ |

**アーキテクチャの違いがもたらす利点**:

1. **プロトコルの最適化**: MQTTはIoTデバイスに最適（軽量・省電力）
2. **アーキテクチャの分離**: Web Appをバイパスすることで、それぞれのコンポーネントが独立

SQSパターンでは：
- IoTデバイスがMQTTで効率的にデータ送信
- IoT CoreがMQTTブローカーとして受信
- Web Appはユーザーのリクエスト処理に専念
- IoTデータ処理は独立したWorkerが担当

### 解決策2: 独立したスケーラビリティ

それぞれのコンポーネントを独立してスケールできます。

- Web App: ユーザー数に応じてスケール
- Worker: IoTデータ量に応じてスケール
- **別々の責任を別々にスケール**

### 解決策3: 可用性の向上

Web AppがダウンしてもIoTデータ処理は継続します。

- SQSは永続化されている（メッセージが消えない）
- Workerだけが動いていればデータ処理できる
- システムの一部がダウンしても全体への影響が限定的

### 解決策4: AWS統合によるシンプルさ

AWS環境では追加のメリットがあります。

- IoT CoreからSQSへの直接送信（Lambda不要）
- SQSは自動スケール（手動設定不要）
- CloudWatchでモニタリングが容易

## パターン選択の判断基準

では、どういう場合にRedis（Sidekiq）を選び、どういう場合にSQS（shoryuken）を選ぶべきでしょうか？

| データソース | 推奨パターン | 理由 |
|------------|------------|------|
| **ユーザーのWebリクエスト**<br>（例: メール送信、レポート生成） | Sidekiq + Redis | Web Appから直接ジョブ登録するので、Redisの高速性が活きる。Web Appと同じライフサイクルでOK。 |
| **IoTデバイスからの大量データ**<br>（例: センサーデータ、ログ） | shoryuken + SQS | Web Appをバイパスできる。独立したスケーラビリティが必要。可用性が重要。 |
| **外部Webhookからの大量データ**<br>（例: 決済通知、API連携） | shoryuken + SQS | 同じく分離が望ましい。外部システムのトラフィック変動に対応。 |
| **スケジュールジョブ**<br>（例: 日次バッチ、定期レポート） | Sidekiq + Redis | シンプルでOK。sidekiq-schedulerで十分。 |

**判断基準のポイント**:
1. **データソースがWeb Appの外部にあるか？** → Yes なら SQS
2. **大量データで独立したスケールが必要か？** → Yes なら SQS
3. **Web Appのライフサイクルと分離すべきか？** → Yes なら SQS
4. **それ以外** → Redisでシンプルに

## 実装例：shoryuken + SQS

最後に、実際の実装例を簡単に見てみましょう。

### 1. IoT Core ルール設定（AWSコンソール）

```sql
-- IoT Rule SQL
SELECT temperature, humidity, device_id, timestamp() as received_at
FROM 'iot/sensors/+'
WHERE temperature > 30
```

**アクション**: SQSキューにメッセージ送信

### 2. Rails Worker実装

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

### 3. どうやって動くのか？

1. IoTデバイスが温度データを送信（MQTT）
2. AWS IoT Coreが受信してルールを評価
3. 条件に合致（温度 > 30）したらSQSに送信
4. Workerが常時SQSをポーリング
5. メッセージがあれば取得して処理
6. データベースに保存して完了

**重要**: この全体フローでRails Web Appは一切関与しません。

## まとめ

### 私が理解できたこと

1. **Redisが悪いわけではない**
   - ユースケースによってはRedisがベスト
   - Web Appから発生するジョブならRedisで十分

2. **データソースによってアーキテクチャを変える**
   - Web App外部からの大量データ → SQS
   - Web App内部からのジョブ → Redis

3. **IoTシステムはアーキテクチャ分離が重要**
   - Web Appをバイパスする設計
   - 独立したスケーラビリティ
   - 可用性の向上

### 最初の疑問への回答

**「IoTシステムでRedis使うって選択肢はないの？」**
→ **技術的には可能だが、アーキテクチャ的に推奨されない**

理由は：
- Web Appがボトルネックになる
- 独立したスケールができない
- 可用性が低下する

**「なぜSQSなの？」**
→ **アーキテクチャ分離によるスケーラビリティと可用性の向上**

### 参考になった視点

- 「どのツールが優れているか」ではなく「どのアーキテクチャが適切か」で考える
- ツールの選択は、システム全体の設計思想から逆算する
- 分離できるものは分離する（単一責任の原則）

この記事が、私と同じように「なぜSQSなのか？」と疑問に思っている方の理解の助けになれば幸いです。

## 参考リンク

- [Sidekiq公式ドキュメント](https://github.com/sidekiq/sidekiq)
- [shoryuken公式ドキュメント](https://github.com/ruby-shoryuken/shoryuken)
- [AWS IoT Core - SQS統合](https://docs.aws.amazon.com/iot/latest/developerguide/iot-rule-actions.html)
- [Amazon SQS開発者ガイド](https://docs.aws.amazon.com/sqs/)
