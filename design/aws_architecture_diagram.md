# AWS システム構成図

## システム概要
テストケース自動手順生成システムのAWS ECS構成

## 全体構成図

```mermaid
graph TB
    %% External Users
    subgraph "Users"
        U1[社内ユーザー<br/>MVP Phase]
        U2[外部ユーザー<br/>Phase 2]
    end

    %% Internet & Corporate Network
    subgraph "Network"
        CORP[社内ネットワーク<br/>192.168.0.0/16<br/>10.0.0.0/8]
        INET[インターネット]
    end

    %% AWS Cloud
    subgraph "AWS Cloud"
        %% Security Services
        subgraph "Security & DNS"
            WAF[AWS WAF<br/>Phase 2]
            ACM[AWS ACM<br/>TLS証明書<br/>Phase 2]
        end

        %% VPC
        subgraph "VPC"
            %% Public Subnets
            subgraph "Public Subnets"
                ALB[Application Load Balancer<br/>MVP: HTTP:80<br/>Phase2: HTTPS:443]
            end

            %% Private Subnets
            subgraph "Private Subnets"
                %% ECS Cluster
                subgraph "ECS Cluster on EC2"
                    subgraph "ECS Service"
                        subgraph "ECS Task 1"
                            FAST1[FastAPI Container<br/>Port: 8000]
                            OLL1[Ollama Container<br/>Port: 11434<br/>GPU Required]
                        end
                        subgraph "ECS Task 2"
                            FAST2[FastAPI Container<br/>Port: 8000]
                            OLL2[Ollama Container<br/>Port: 11434<br/>GPU Required]
                        end
                        subgraph "ECS Task N"
                            FASTN[FastAPI Container<br/>Auto Scale]
                            OLLN[Ollama Container<br/>Auto Scale]
                        end
                    end
                end
            end

            %% Security Groups
            subgraph "Security Groups"
                SG1[ALB Security Group<br/>MVP: HTTP from 社内IP<br/>Phase2: HTTPS from Internet]
                SG2[ECS Security Group<br/>HTTP:8000 from ALB only]
            end
        end

        %% AWS Services
        subgraph "AWS Services"
            ECR[Amazon ECR<br/>Container Registry<br/>- FastAPI Image<br/>- Ollama Image]
            CWL[CloudWatch Logs<br/>- FastAPI: /aws/ecs/fastapi-app<br/>- Ollama: /aws/ecs/ollama-server<br/>- ALB: /aws/loadbalancer/app]
            SSM[SSM Parameter Store<br/>- Configuration<br/>- Non-secret settings]
            SEC[Secrets Manager<br/>- API Keys<br/>- Credentials]
            EBS[EBS Volumes<br/>- Ollama Models: /root/.ollama<br/>- Persistent Storage<br/>- GP3 (暗号化)]
        end

        %% IAM
        subgraph "IAM"
            ROLE1[ECS Task Execution Role<br/>- ECR Pull権限<br/>- CloudWatch Logs書込み権限]
            ROLE2[ECS Task Role<br/>- SSM Parameter読取り権限<br/>- Secrets Manager読取り権限]
        end
    end

    %% Connections - MVP Phase (社内限定)
    U1 --> CORP
    CORP --> ALB
    ALB --> FAST1
    ALB --> FAST2
    ALB --> FASTN

    %% Connections - Phase 2 (外部公開)
    U2 --> INET
    INET --> WAF
    WAF --> ALB
    ACM --> ALB

    %% Internal Connections (Sidecar Pattern)
    FAST1 -.->|localhost| OLL1
    FAST2 -.->|localhost| OLL2
    FASTN -.->|localhost| OLLN

    %% AWS Service Connections
    FAST1 --> SSM
    FAST1 --> SEC
    FAST2 --> SSM
    FAST2 --> SEC
    FASTN --> SSM
    FASTN --> SEC

    FAST1 --> CWL
    FAST2 --> CWL
    FASTN --> CWL
    OLL1 --> CWL
    OLL2 --> CWL
    OLLN --> CWL

    ECR --> FAST1
    ECR --> FAST2
    ECR --> FASTN
    ECR --> OLL1
    ECR --> OLL2
    ECR --> OLLN

    %% EBS Volume Connections
    EBS -.->|/root/.ollama| OLL1
    EBS -.->|/root/.ollama| OLL2
    EBS -.->|/root/.ollama| OLLN

    ROLE1 --> ECR
    ROLE1 --> CWL
    ROLE2 --> SSM
    ROLE2 --> SEC
    ROLE2 --> EBS

    %% Security Group Associations
    SG1 -.-> ALB
    SG2 -.-> FAST1
    SG2 -.-> FAST2
    SG2 -.-> FASTN

    %% Styling
    classDef mvp fill:#e1f5fe
    classDef phase2 fill:#fff3e0
    classDef container fill:#f3e5f5
    classDef aws fill:#e8f5e8

    class U1,ALB,FAST1,FAST2,FASTN,OLL1,OLL2,OLLN,SG1,SG2 mvp
    class U2,WAF,ACM phase2
    class FAST1,FAST2,FASTN,OLL1,OLL2,OLLN container
    class ECR,CWL,SSM,SEC,ROLE1,ROLE2,EBS aws
```

## アーキテクチャ詳細

### MVP段階（Phase 1：社内限定運用）
- **アクセス制御**: 社内IPアドレス範囲からのHTTPアクセスのみ許可
- **プロトコル**: HTTP通信（社内ネットワークのため安全性確保）
- **セキュリティグループ**: 
  - ALB: 社内IP（192.168.0.0/16, 10.0.0.0/8）からHTTP:80を許可
  - ECS: ALBからのHTTP:8000のみ許可

### Phase 2（外部公開時）
- **TLS終端**: ALBでACM証明書を使用したHTTPS通信
- **WAF**: SQLインジェクション、XSS等の攻撃防御
- **セキュリティ強化**: より厳密なアクセス制御とモニタリング

### ECSタスク構成（サイドカーパターン）
- **FastAPI**: メインアプリケーション（Port 8000）
- **Ollama**: LLM推論エンジン（Port 11434、GPU必須）
- **通信**: 同一タスク内でlocalhost通信による高速連携

### オートスケール設定
- **メトリクス**: CPU使用率、リクエスト数
- **スケール**: 負荷に応じてECSタスク数を自動調整
- **リソース**: GPU対応EC2インスタンス（g4dn.xlarge以上推奨）

### ストレージ・ログ構成
- **EBSボリューム**: 
  - **目的**: Ollamaモデルファイル・設定の永続化
  - **マウント**: `/root/.ollama` (Ollama Dockerfileで定義)
  - **タイプ**: GP3 SSD（暗号化有効）
  - **容量**: 100GB〜（モデルサイズに応じて調整）
  - **バックアップ**: 日次スナップショット自動取得

- **CloudWatch Logs**: 
  - **FastAPI**: `/aws/ecs/fastapi-app`
    - アプリケーションログ、API リクエスト・レスポンス
    - エラーログ、パフォーマンスメトリクス
  - **Ollama**: `/aws/ecs/ollama-server` 
    - LLM推論ログ、モデル読込み状況
    - GPU使用率、推論レイテンシ
  - **ALB**: `/aws/loadbalancer/app`
    - アクセスログ、ヘルスチェック結果
  - **保持期間**: 30日（設定可能）

- **Ollamaファイル構成**:
  ```
  /root/.ollama/
  ├── models/          # LLMモデルファイル
  ├── config.json      # Ollama設定ファイル
  └── logs/            # ローカルログ（CloudWatchと併用）
  ```

### データフロー
1. **リクエスト受信**: ALB → FastAPIコンテナ
2. **LLM推論**: FastAPI → Ollama（localhost）
3. **設定取得**: FastAPI → SSM/Secrets Manager
4. **ログ出力**: 全コンテナ → CloudWatch Logs
5. **イメージ取得**: ECS → ECR
6. **モデル読込**: Ollama → EBSボリューム（/root/.ollama）

### セキュリティ
- **IAM最小権限**: 各コンポーネントに必要最小限の権限のみ付与
- **機密情報管理**: Secrets Managerで安全に管理
- **ネットワーク分離**: プライベートサブネット内でECS実行
- **ログ監視**: CloudWatch Logsでセキュリティイベント追跡

## 変更理由
- **システム全体仕様とECSデプロイ要件に基づいた構成**: 全体仕様暫定.mdとissues/005の要件を完全に反映
- **段階的デプロイ対応**: MVP（社内限定）とPhase 2（外部公開）の両方に対応した設計
- **サイドカーパターン採用**: FastAPIとOllamaの密結合要件に最適な構成
- **GPU対応明記**: Ollama推論に必須のGPUリソース要件を明確化
- **セキュリティ重視**: 最小権限の原則とネットワークセキュリティを適切に設計
- **永続ストレージ追加**: OllamaのDockerfile設定（/root/.ollama）に基づくEBSボリューム設計
- **ログ管理詳細化**: FastAPI・Ollama・ALB各コンポーネントの専用CloudWatch Logsグループ設定
- **運用観測性向上**: 監査ログ・パフォーマンスメトリクス・エラートラッキングを統合管理