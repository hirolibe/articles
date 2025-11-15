## 1. はじめに

### 1-1. 本記事の執筆背景

現在、実務案件にてAmazon ECSのBlue/Greenデプロイを検討しています。
弊社では従来よりCodeDeployによる手法を採用してきました。

2025年7月にECS Nativeという比較的新しいデプロイ手法がリリースされていることを知り、その実装方法についてまとめてみました。

### 1-2. デプロイ手法の種類

本記事で扱うデプロイ手法はBlue/Greenデプロイのみとなりますが、ECSには下記のように複数のデプロイ手法があります。

#### リプレイスデプロイ
- 既存タスクをすべて停止→新タスクを起動
- **ダウンタイムが発生**
- 最もシンプルな方式
- 開発環境や緊急時の強制再起動に使用

#### ローリングアップデート
- 段階的に新バージョンへ置き換え
- デプロイ中に新旧バージョンが混在
- **互換性の問題やロールバックの複雑さが発生**
- リソース効率は良い（追加インスタンス不要）

#### Blue/Greenデプロイ（本記事で取り扱う手法）
- 新環境を完全構築後、トラフィックを一括切り替え
- バージョン混在を回避
- 問題発生時は即座にロールバック可能
- **一時的に2倍のリソースが必要**

### 1-3. Blue/Greenデプロイの選定理由

今回のプロジェクトでは、本番環境でゼロダウンタイムと即座のロールバックを実現し、デプロイリスクを最小化することを最優先としたため、Blue/Greenデプロイ手法を選定しました。

## 2. Blue/Greenデプロイの仕組み

### 2-1. 通常デプロイの流れ

```
1. デプロイ開始
   ├── 新しいTask Definition作成
   └── Green環境にタスク起動

2. Green環境の起動とヘルスチェック
   ├── ECS Tasks起動
   ├── ALB Health Checkパス
   └── Circuit Breaker監視

3. Traffic Shift（瞬時切り替え）
   ├── Listener rule変更（Blue → Green）
   ├── 100%のトラフィックがGreenへ
   └── Blue環境は待機状態

4. Bake Time（監視期間 5分）
   ├── デプロイ完了まで待機
   ├── オプション: CloudWatch Alarmsによる監視（補足参照）
   └── 問題なければデプロイ完了

5. デプロイ完了
   ├── Green環境が本番稼働環境になる（次回デプロイ時のBlueの役割）
   └── 旧Blue環境のタスクは終了
```

### 2-2. 自動ロールバックのトリガー

**Circuit Breakerによるロールバック（標準機能）**:
```
条件: ECSタスクの起動失敗
  - コンテナ起動エラー
  - ALBヘルスチェック失敗
  - リソース不足
タイミング: Traffic Shift前（Green環境の起動中）
動作: デプロイ即座にキャンセル、Blue環境維持
```

**CloudWatch Alarmsによるロールバック（オプション・補足参照）**:
```
条件: 設定したアラームがALARM状態（アプリケーションレベル）
  - 5xxエラーが閾値超過
  - レスポンスタイムが閾値超過
  - CPU使用率が閾値超過
タイミング: Traffic Shift後のBake Time期間中
動作: Listener rule即座に変更（Green → Blue）
```

### 2-3. Bake Time期間の役割

Bake Timeは、新環境（Green）に切り替え後の**待機期間**です。

```
Traffic Shift完了
    ↓
[Bake Time: 5分]
    ↓
    ├── 安全確認のための待機期間
    ├── オプション: CloudWatch Alarmsで監視
    ├── 問題検知 → 自動ロールバック
    └── 問題なし → デプロイ完了
```

**重要なポイント**:
- Bake Time中はBlue環境のタスクが維持される（ロールバック可能）
- CloudWatch Alarmsを設定している場合、この期間中に監視
- Bake Time終了後、Blue環境のタスクが終了

## 3. Blue/Greenデプロイの実装方式

Amazon ECSでBlue/Greenデプロイを実装する方法は2つあります。

### 3-1. CodeDeploy方式

AWS CodeDeployサービスを利用する方式。

**特徴**
- CodeDeployリソース（アプリケーション、デプロイメントグループ）が必要
- `appspec.yaml` による設定
- デプロイメントコントローラー: `CODE_DEPLOY`
- 追加のIAMロール、サービスリンクロールが必要

### 3-2. ECS Native方式

Amazon ECSサービス自体にBlue/Green機能が統合された新方式（2025年7月リリース）。

**特徴**:
- CodeDeployリソース不要
- ECSサービス定義だけで完結（`appspec.yaml`不要）
- デプロイメントコントローラー: `ECS`
- シンプルな設定で即座に利用可能
- [AWSの公式ドキュメント](https://aws.amazon.com/jp/blogs/news/accelerate-safe-software-releases-with-new-built-in-blue-green-deployments-in-amazon-ecs/)で導入が推奨されている


### 3-3. 本記事で扱う方式

本記事では、冒頭で述べた通り**ECS Native方式の実装方法**を次章で詳しく解説します。

※ CodeDeploy方式の実装方法については本記事では割愛します。

## 4. ECS Nativeの実装方法（Terraform）

### 4-1. 使用バージョン

- Terraform 1.13.5以上
- AWS Provider 6.4.0以上（ECS Native Blue/Green対応）

### 4-2. 前提となるネットワーク構成

本記事では、Blue/Greenデプロイの実装に焦点を当てるため、以下のネットワーク構成は既存のものを使用する前提としています。

**前提リソース**:
- VPC
- パブリック/プライベートサブネット
- インターネットゲートウェイ/NATゲートウェイ
- セキュリティグループ
- IAMロール（ECSタスク実行用、タスク用）

これらの基本的なネットワーク構成の構築方法については、AWS公式ドキュメントやTerraform VPCモジュールを参照してください。

### 4-3. ディレクトリ構成例

```
terraform/
└── modules/
    ├── alb/
    │   ├── main.tf        # Blue/Greenターゲットグループ + Listener
    │   ├── variables.tf
    │   └── outputs.tf
    └── ecs/
        ├── main.tf        # ECS Service（Blue/Green設定）
        ├── variables.tf
        └── outputs.tf
```

**注**: 本記事ではBlue/Green設定に焦点を当てるため、`main.tf` のみ詳細コードを記載します。

### 4-4. ALB + デュアルターゲットグループ

Blue/Green用に2つのターゲットグループを作成します。

```hcl:modules/alb/main.tf
# Blue Target Group
resource "aws_lb_target_group" "blue" {
  name        = "${var.project_name}-blue-tg"
  port        = var.container_port
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }

  deregistration_delay = 30

  tags = {
    Name = "${var.project_name}-blue-tg"
  }
}

# Green Target Group
resource "aws_lb_target_group" "green" {
  name        = "${var.project_name}-green-tg"
  port        = var.container_port
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }

  deregistration_delay = 30

  tags = {
    Name = "${var.project_name}-green-tg"
  }
}

# ALB Listener（初期状態はBlueへ）
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.blue.arn
  }
}
```

### 4-5. ECS Service（Blue/Green設定）

ECS Nativeの核となる設定です。

```hcl:modules/ecs/main.tf
resource "aws_ecs_service" "main" {
  name            = "${var.project_name}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.main.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"

  # ECS Nativeを指定（重要）
  deployment_controller {
    type = "ECS"  # "CODE_DEPLOY" ではない
  }

  # Blue/Green設定
  deployment_configuration {
    # Blue/Green戦略を指定
    deployment_option = "WITH_TRAFFIC_CONTROL"

    minimum_healthy_percent = 100
    maximum_percent         = 200

    # Circuit Breaker統合
    deployment_circuit_breaker {
      enable   = true
      rollback = true
    }

    # Blue/Green詳細設定
    blue_green_deployment_config {
      # 本番トラフィックのListener設定
      production_traffic_route {
        listener_arn = aws_lb_listener.http.arn
      }

      # Blue/Greenターゲットグループ指定
      target_group {
        blue  = aws_lb_target_group.blue.name
        green = aws_lb_target_group.green.name
      }

      # Bake Time（Green環境の監視期間）
      bake_time_in_minutes = 5

      # オプション: CloudWatch Alarmsによる自動ロールバック（4-6参照）
      # alarms = [
      #   aws_cloudwatch_metric_alarm.target_5xx_errors.arn,
      #   aws_cloudwatch_metric_alarm.response_time.arn
      # ]
    }
  }

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = true
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.blue.arn  # 初期状態
    container_name   = var.container_name
    container_port   = var.container_port
  }

  depends_on = [
    aws_lb_listener.http,
    aws_iam_role_policy_attachment.ecs_task_execution_role_policy,
  ]
}
```

### 4-6. 【オプション】CloudWatch Alarmsによるアプリケーション監視

Circuit Breakerは**インフラレベル**（タスク起動失敗、ヘルスチェック失敗等）の問題を検知しますが、**アプリケーションレベル**（5xxエラー、レスポンスタイム、CPU使用率等）の問題も監視したい場合は、CloudWatch Alarmsを追加できます。

4-5のコード内の `alarms` パラメータでCloudWatch Alarmを設定することで、Traffic Shift後のBake Time期間中にアラームが発火した場合、自動的にロールバックされます。本記事では基本実装に焦点を当てるため詳細は省略していますが、本番環境では設定を推奨します。

## 5. まとめ

本記事では、Amazon ECSにおけるBlue/Greenデプロイの仕組みと、2025年7月にリリースされたECS Native方式のTerraform実装方法を解説しました。

今回の実装を通じて、**CodeDeployよりも比較的簡単に実装でき、AWS公式の推奨もあることから、新規プロジェクトではECS Native方式を検討する価値が十分にある**と感じました。

## 6. 参考リンク

- [Accelerate safe software releases with new built-in blue/green deployments in Amazon ECS](https://aws.amazon.com/jp/blogs/news/accelerate-safe-software-releases-with-new-built-in-blue-green-deployments-in-amazon-ecs/)
- [Amazon ECS blue/green deployments](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html)
- [Terraform AWS Provider - aws_ecs_service](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecs_service)
- [Using Amazon CloudWatch alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
