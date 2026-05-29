# platform-docs

## このリポジトリについて

Platform Engineering ポートフォリオの設計書・ADR・Runbook を管理するリポジトリ。

「なぜその技術を選んだか」「なぜその設計にしたか」を記録することで、構築だけでなく設計から判断できることを示す目的で作成しました。Context → Options → Decision → Reasons → Consequences の形式で意思決定を整理しています。

Backstage TechDocs でのドキュメントサイト表示に対応しています。

---

## 設計書一覧

各コンポーネントの設計意図・構成・制約を記録しています。

| タイトル | 概要 |
|---|---|
| [Bootstrap設計書](docs/design/bootstrap-design.md) | クラスタ起動フロー・App-of-Apps構成・sync-wave設計・カスタムヘルスチェック |
| [DR設計書](docs/design/DR-design.md) | DRシナリオ分類・RTO/RPO目標・GitOpsをDRトリガーとして使う設計 |
| [バックアップ設計書](docs/design/backup-design.md) | PostgreSQL → MinIO → GCS の2層バックアップ設計・認証情報管理 |
| [開発環境セットアップ設計書](docs/design/dev-env-design.md) | aquaによるツール管理・PEチームとアプリチームのモデルの違い |
| [GitHub認証設計書](docs/design/github-auth-design.md) | GitHub App / Fine-grained PAT / OAuth App の使い分けと個人アカウントの制約 |
| [Golden Path設計書](docs/design/golden-path-design.md) | Scaffolder〜ArgoCD自動sync・Platform自動応答（AppProject/Namespace/Secret）の設計 |
| [Scaffolder設計書](docs/design/scaffolder-design.md) | fullstack/teardownテンプレートの設計・生成物一覧・apps-gitopsディレクトリ構造 |
| [リソース管理設計書](docs/design/resource-management-design.md) | ArgoCDアプリケーションとKubernetesリソースの管理方針（作成中） |

---

## ADR一覧

| No. | タイトル | 対象リポジトリ | Status |
|---|---|---|---|
| [ADR-001](docs/adr/ADR-001-gitops-engine.md) | GitOpsエンジンの選択（ArgoCD） | platform-gitops / platform-infra | Accepted |
| [ADR-002](docs/adr/ADR-002-bootstrap.md) | bootstrap順序制御の設計（App-of-Apps + Makefile） | platform-gitops / platform-infra | Accepted |
| [ADR-003](docs/adr/ADR-003-helm-library-chart.md) | Kubernetesマニフェスト抽象化方法の選択（Helm Library Chart） | platform-charts / platform-gitops | Accepted |
| [ADR-004](docs/adr/ADR-004-secrets-management.md) | Secret管理戦略の選択（SOPS×Age + ESO） | platform-gitops | Accepted |
| [ADR-005](docs/adr/ADR-005-crossplane.md) | インフラリソース管理の責務分離（Crossplane vs Terraform） | platform-gitops / platform-infra | Draft |
| [ADR-006](docs/adr/ADR-006-postgresql-operator.md) | PostgreSQL Operatorの選択（CloudNativePG） | platform-gitops / platform-charts | Accepted |
| [ADR-007](docs/adr/ADR-007-mise-tool-sharing.md) | ツール管理の共有戦略（aqua + AQUA_GLOBAL_CONFIG） | platform-infra | Accepted |
| [ADR-008](docs/adr/ADR-008-observability-stack.md) | Observabilityスタックの選択（LGTMスタック） | platform-gitops | 未着手 |
| [ADR-009](docs/adr/ADR-009-network-ingress.md) | ネットワーク・Ingress設計（Gateway API / Envoy Gateway / Cilium） | platform-gitops / platform-infra | 未着手 |
| [ADR-010](docs/adr/ADR-010-idp.md) | IDP設計（Keycloak / Backstage 選定） | platform-gitops / backstage | 未着手 |
| [ADR-011](docs/adr/ADR-011-policy-engine.md) | ポリシーエンジンの選択（Kyverno） | platform-gitops | 未着手 |

---

## 意思決定の全体像

```
GitOps 基盤
└─ ADR-001: ArgoCD を GitOps エンジンに選択
      └─ pull型によるdrift検出・GUI・App of Appsパターン
bootstrap
└─ ADR-002: App-of-Apps + Makefile による順序制御
      └─ CR health check 問題の回避・ArgoCD GUI の早期確保
アプリデプロイ抽象化
└─ ADR-003: Helm Library Chart を抽象化レイヤーに選択
      └─ テンプレート一元管理・ガードレール・Golden Path
セキュリティ / シークレット管理
└─ ADR-004: SOPS×Age + ESO を Secret 管理に選択
      └─ Secrets as Code・クラスタ非依存・DR整合性
インフラ管理
└─ ADR-005: Terraform と Crossplane を用途で使い分け（Draft）
      └─ クラスタ基盤はTerraform / 開発者向けリソースはCrossplane
データ管理
└─ ADR-006: CloudNativePG を PostgreSQL Operator に選択
      └─ Apache 2.0・k8sネイティブ・クラウド親和性
ツール管理
└─ ADR-007: aqua + AQUA_GLOBAL_CONFIG でツール定義を一元管理
      └─ platform-infra を source of truth / ArgoCD制約回避
Observability
└─ ADR-008: LGTMスタック選定（未着手）
Networking
└─ ADR-009: Gateway API / Envoy Gateway / Cilium 選定（未着手）
IDP
└─ ADR-010: Keycloak / Backstage 選定（未着手）
ポリシー管理
└─ ADR-011: Kyverno 選定（未着手）
```

---

## Runbook一覧

| タイトル | 概要 |
|---|---|
| [DBリストア手順](docs/runbook/dr-restore.md) | PVC破損・クラスタ全損・WSL全損の各シナリオ別DR手順 |
| [シークレット追加・更新手順](docs/runbook/secrets-management.md) | SOPS × ESO を使った Secret の追加・更新・確認手順 |

---

## 関連リポジトリ

| リポジトリ | 役割 |
|---|---|
| [platform-infra](https://github.com/okccl/platform-infra) | k3d クラスタ IaC・aqua・Terraform（EKS） |
| [platform-gitops](https://github.com/okccl/platform-gitops) | ArgoCD による GitOps 管理 |
| [platform-charts](https://github.com/okccl/platform-charts) | Helm Library Chart（common-app / common-db） |
| [sample-backend](https://github.com/okccl/sample-backend) | FastAPI + PostgreSQL（Golden Path利用例） |
| [sample-frontend](https://github.com/okccl/sample-frontend) | React + Vite（Golden Path利用例） |
| [backstage](https://github.com/okccl/backstage) | Backstage カスタムイメージのソース |

ポートフォリオ全体の概要は [okccl](https://github.com/okccl) を参照してください。
