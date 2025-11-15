## 1. はじめに

### 1-1. Blue/Greenデプロイとは

Blue/Greenデプロイは、本番環境（Blue）と同じ構成の新環境（Green）を並行稼働させ、トラフィックを瞬時に切り替えることで、ダウンタイムゼロのデプロイを実現する手法です。

### 1-2. ローリングアップデートとの違い
ローリングアップデートは段階的に新バージョンへ置き換えるため、デプロイ中に新旧バージョンが混在し、互換性の問題やロールバックの複雑さが生じます。Blue/Greenデプロイは新環境を完全構築後、ロードバランサーやDNSでトラフィックを一括切り替えるため、バージョン混在を回避でき、問題発生時は即座にロールバック可能です。

### 1-3. なぜBlue/Greenデプロイが必要なのか

1. **ゼロダウンタイム**: トラフィックの瞬時切り替えでサービス停止なし
2. **即座のロールバック**: 問題発生時、旧環境へ瞬時に切り戻し可能
3. **安全な検証**: 本番トラフィックを流す前に新環境をテスト可能
4. **リスク最小化**: 段階的な移行でなく一気に切り替えるため、状態管理がシンプル

---

## 2. ECS Blue/Greenデプロイの選択肢

Amazon ECSでBlue/Greenデプロイを実装する方法は2つあります。

### 2-1. CodeDeploy方式（従来）

AWS CodeDeployサービスを利用する方式。

**特徴**:
- CodeDeployリソース（アプリケーション、デプロイメントグループ）が必要
- `appspec.yaml` による設定
- デプロイメントコントローラー: `CODE_DEPLOY`
- 追加のIAMロール、サービスリンクロールが必要

### 2-2. ECS Native方式（2025年7月リリース）

ECSサービス自体にBlue/Green機能が統合された新方式。

**特徴**:
- CodeDeployリソース不要
- ECSサービス定義だけで完結（外部サービス・appspec.yaml不要）
- デプロイメントコントローラー: `ECS`
- シンプルな設定で即座に利用可能

**参考**: [Accelerate safe software releases with new built-in blue/green deployments in Amazon ECS](https://aws.amazon.com/jp/blogs/news/accelerate-safe-software-releases-with-new-built-in-blue-green-deployments-in-amazon-ecs/)

---

## 3. ECS Nativeを選ぶべき理由

### 3-1. 圧倒的なシンプルさ

**CodeDeploy方式**:
```
必要なリソース:
├── CodeDeploy Application
├── CodeDeploy Deployment Group
├── appspec.yaml設定ファイル
├── CodeDeploy用IAMロール
├── CodeDeployサービスリンクロール
└── ECS Service (deployment_controller = CODE_DEPLOY)
```

**ECS Native方式**:
```
必要なリソース:
├── ECS Service (deployment_controller = ECS)
└── Blue/Green設定（ECS Service内に統合）
```

設定ファイルや外部サービスが不要で、ECSサービス定義だけで完結します。

### 3-2. Deployment Circuit Breaker統合

ECS NativeではDeployment Circuit Breakerがシームレスに動作します。

```hcl
resource "aws_ecs_service" "main" {
  # ... 他の設定 ...

  deployment_configuration {
    # Circuit Breaker設定
    deployment_circuit_breaker {
      enable   = true
      rollback = true  # タスク起動失敗時に自動ロールバック
    }
  }
}
```

**Circuit Breakerの役割**:
- ECSタスクの起動失敗を自動検知
- 設定した閾値を超えたら即座にロールバック
- CodeDeployでは別途CloudWatch Alarmsで実装が必要

### 3-3. 機能比較表

| 機能 | ECS Native | CodeDeploy |
|------|-----------|------------|
| **設定の複雑さ** | ⭐️⭐️⭐️⭐️⭐️ シンプル | ⭐️⭐️ 複雑 |
| **ロールバック速度** | ⚡️ 即座（Listener rule変更） | ⚡️ 即座（Listener rule変更） |
| **自動ロールバック設定** | ✅ 簡単（Circuit Breaker標準） | ⚠️ 複雑（CloudWatch Alarms設定必要） |
| **追加コスト** | $0 | $0（運用コストあり） |
| **設定** | ✅ ECSサービス定義のみ | ⚠️ appspec.yamlが必要 |
| **学習コスト** | 低 | 高 |

---

## 4. Terraformでの実装

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

      # オプション: Test Traffic Route（本番前テスト用）
      # test_traffic_route {
      #   listener_arn = aws_lb_listener.test.arn
      # }
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

---

## 5. デプロイフローの詳細

### 5-1. 通常デプロイの流れ

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
   ├── オプション: CloudWatch Alarmsによる監視（7章参照）
   └── 問題なければデプロイ完了

5. デプロイ完了
   ├── Green環境が本番稼働環境になる（次回デプロイ時のBlueの役割）
   └── 旧Blue環境のタスクは終了
```

### 5-2. 自動ロールバックのトリガー

**Circuit Breakerによるロールバック（標準機能）**:
```
条件: ECSタスクの起動失敗
  - コンテナ起動エラー
  - ALBヘルスチェック失敗
  - リソース不足
タイミング: Traffic Shift前（Green環境の起動中）
動作: デプロイ即座にキャンセル、Blue環境維持
```

**CloudWatch Alarmsによるロールバック（オプション・7章参照）**:
```
条件: 設定したアラームがALARM状態（アプリケーションレベル）
  - 5xxエラーが閾値超過
  - レスポンスタイムが閾値超過
  - CPU使用率が閾値超過
タイミング: Traffic Shift後のBake Time期間中
動作: Listener rule即座に変更（Green → Blue）
```

### 5-3. Bake Time期間の役割

Bake Timeは、新環境（Green）に切り替え後の**待機期間**です。

```
Traffic Shift完了
    ↓
[Bake Time: 5分]
    ↓
    ├── 安全確認のための待機期間
    ├── オプション: CloudWatch Alarmsで監視（7章参照）
    ├── 問題検知 → 自動ロールバック
    └── 問題なし → デプロイ完了
```

**重要なポイント**:
- Bake Time中はBlue環境のタスクが維持される（ロールバック可能）
- CloudWatch Alarmsを設定している場合、この期間中に監視
- Bake Time終了後、Blue環境のタスクが終了

---

## 6. まとめ

### 6-1. ECS Nativeがおすすめのケース

✅ **以下の要件に該当する場合、ECS Native一択**:
- シンプルな構成を優先
- CodeDeployの学習コストを避けたい
- 追加サービスの管理を最小化
- Deployment Circuit Breakerを活用したい
- 小〜中規模のプロジェクト

### 6-2. CodeDeployが適しているケース

以下の場合はCodeDeployも検討価値があります:
- 既存のCodeDeployパイプラインがある
- 複雑なデプロイ承認フローが必要
- AWS CodePipelineとの深い統合が必要
- 組織標準としてCodeDeployを採用している

### 6-3. 結論

**2025年以降の新規プロジェクトでは、ECS Nativeを第一選択肢とすべきです。**

理由:
1. **シンプル**: 設定が直感的で理解しやすい
2. **コスト効率**: 追加費用ゼロ、運用負荷最小
3. **十分な機能**: CloudWatch Alarms統合、Circuit Breaker対応
4. **ロールバック性能**: CodeDeployと同等の即座のロールバック

特別な理由がない限り、ECS Nativeで十分な要件を満たせます。

---

## 7. 補足

### 7-1. Lambda Hooks

ECS NativeはデプロイライフサイクルでLambda関数を実行する**Lambda Hooks**にも対応しています（BeforeInstall、AfterInstall、BeforeAllowTraffic、AfterAllowTraffic）。データベースマイグレーションの自動実行や外部システム連携（Slack通知等）が必要な場合に使用します。

### 7-2. CloudWatch Alarmsによるアプリケーション監視

Circuit Breakerは**インフラレベル**（タスク起動失敗、ヘルスチェック失敗等）の問題を検知しますが、**アプリケーションレベル**（5xxエラー、レスポンスタイム、CPU使用率等）の問題も監視したい場合は、CloudWatch Alarmsを追加できます。

Traffic Shift後のBake Time期間中にアラームが発火すると、自動的にロールバックされます。本記事では基本実装に焦点を当てるため省略していますが、本番環境では設定を推奨します。

---

## 8. 参考リンク

- [Accelerate safe software releases with new built-in blue/green deployments in Amazon ECS](https://aws.amazon.com/jp/blogs/news/accelerate-safe-software-releases-with-new-built-in-blue-green-deployments-in-amazon-ecs/)
- [Amazon ECS blue/green deployments](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html)
- [Amazon ECS deployment lifecycle event hooks](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen-hooks.html)
- [AWS Blue/Green Deployments](https://docs.aws.amazon.com/whitepapers/latest/blue-green-deployments/welcome.html)
- [Terraform AWS Provider - aws_ecs_service](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecs_service)
- [Using Amazon CloudWatch alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
