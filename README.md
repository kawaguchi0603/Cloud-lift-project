# GCPクラウドリフト・高可用性Webシステム構築プロジェクト

オンプレミス環境（仮想環境）で稼働していたLAMPスタックのWebアプリケーションを、GCP（Google Cloud Platform）上へ移行（クラウドリフト）し、アクセス集中やサーバー障害に耐えうる「高可用性・冗長化ネットワーク」をインフラ自動化ツール（IaC）を用いて構築したプロジェクトです。

---

## 1. システムアーキテクチャ図

<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/223759ed-b765-4280-96ad-c49c528c7173" />

### 主な特徴
* **セキュアな閉域網通信（オーバーレイネットワーク）:** 外部およびサーバー間の通信には **Tailscale（VPN）** を採用し、全サーバーをプライベートIP空間（`100.x.x.x`）に閉じ込めることで、強固なセキュリティを確保。
* **超高速自動フェールオーバー:** LBノードに **Nginx（リバースプロキシ）** を導入。タイムアウト値を1秒にチューニングし、Webマスターサーバーのダウン検知時に1秒以内でスレーブへトラフィックを自動迂回。
* **データのリアルタイム同期:** MySQL 8.0の **GTIDレプリケーション** を採用し、マスター・スレーブ間で安全かつ不整合のないデータ同期を実現。

---

## 2. 技術スタック

| 分類 | 技術・サービス | 役割 |
| :--- | :--- | :--- |
| **Cloud** | GCP (Compute Engine, VPC) | インフラプラットフォームの提供 |
| **Network/VPN** | Tailscale (WireGuard) | 閉域網（VPN）の構築・セキュアな管理接続 |
| **IaC (インフラ自動化)** | Terraform | GCP VPC・VMインスタンス・FWルールのコード管理 |
| **Configuration (構成管理)** | Ansible | ミドルウェアの一括インストール・冗長化設定の自動化 |
| **Middleware** | Nginx, Apache2, MySQL 8.0, PHP | Webサーバー、ロードバランサー、DBレプリケーション |

---

## 3. ディレクトリ構造

* `terraform/` : GCP上のインフラ（土台）を構築する一連のコード
* `ansible/` : 各サーバーにNginx、Apache、MySQL、PHPを自動展開する設定スクリプト

---

## 4. 構築・検証手順

### ① インフラのプロビジョニング (Terraform)
```bash
cd terraform/
terraform init
terraform apply -auto-approve
