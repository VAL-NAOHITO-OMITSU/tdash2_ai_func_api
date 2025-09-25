# Issue #007: AWS ECSデプロイ構成の実装

## タイトル
AWS ECSデプロイ構成の実装

## 概要
AWS ECS on EC2でのコンテナデプロイメント構成を実装する。FastAPIとOllamaのサイドカー構成、オートスケール設定、セキュリティ設定を含む。

## 詳細
- Dockerfileの作成（FastAPI、Ollama）
- docker-compose.ymlの作成（ローカル開発用）
- ECS Task Definitionの設計
- FastAPIとOllamaのサイドカー構成
- ECS Serviceの設定（オートスケール）
- ALB（Application Load Balancer）設定
- VPC、セキュリティグループ設定
- SSM Parameter Store / Secrets Manager連携
- TLS証明書設定（ACM）
- IAMロール・ポリシー設定

## 受け入れ条件
- [ ] DockerイメージがビルドされECRに登録される
- [ ] ECSタスクが正常に起動する
- [ ] FastAPIとOllamaが同一タスク内で連携動作する
- [ ] ALBを通じてAPIにアクセス可能
- [ ] オートスケールが動作する
- [ ] セキュリティグループが適切に設定される
- [ ] 設定情報がSSM/Secrets Managerから取得される

## 関連仕様
- システム全体像（AWS/ECS）
- セキュリティ／公開範囲
- 運用・構成管理（設定管理）
- 非機能要件（可用性、拡張性）

## 推定工数
6-8日