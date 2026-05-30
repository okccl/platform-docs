# Platform Docs

Platform Engineering ポートフォリオの設計書・ADR・Runbook を管理するドキュメントサイト。

「なぜその技術を選んだか」「なぜその設計にしたか」を記録することで、構築だけでなく設計から判断できることを示す目的で作成しました。

---

## ADR一覧

| No. | タイトル | Status |
|---|---|---|
| [ADR-001](adr/ADR-001-gitops-engine.md) | GitOpsエンジンの選択（ArgoCD） | Accepted |
| [ADR-002](adr/ADR-002-bootstrap.md) | bootstrap順序制御の設計（App-of-Apps + Makefile） | Accepted |
| [ADR-003](adr/ADR-003-helm-library-chart.md) | Kubernetesマニフェスト抽象化方法の選択（Helm Library Chart） | Accepted |
| [ADR-004](adr/ADR-004-secrets-management.md) | Secret管理戦略の選択（SOPS×Age + ESO） | Accepted |
| [ADR-005](adr/ADR-005-crossplane.md) | インフラリソース管理の責務分離（Crossplane vs Terraform） | Draft |
| [ADR-006](adr/ADR-006-postgresql-operator.md) | PostgreSQL Operatorの選択（CloudNativePG） | Accepted |
| [ADR-007](adr/ADR-007-mise-tool-sharing.md) | ツール管理の共有戦略（aqua + AQUA_GLOBAL_CONFIG） | Accepted |
| [ADR-008](adr/ADR-008-observability-stack.md) | Observabilityスタックの選択（LGTMスタック） | Draft |
| [ADR-009](adr/ADR-009-network-ingress.md) | ネットワーク・Ingress設計（Gateway API / Envoy Gateway / Cilium） | Draft |
| [ADR-010](adr/ADR-010-idp.md) | IDP設計（Keycloak / Backstage 選定） | Draft |
| [ADR-011](adr/ADR-011-policy-engine.md) | ポリシーエンジンの選択（Kyverno） | Draft |

---

## 設計書一覧

| タイトル | 概要 |
|---|---|
| [Bootstrap設計書](design/bootstrap-design.md) | クラスタ起動フロー・App-of-Apps構成・sync-wave設計 |
| [DR設計書](design/DR-design.md) | DRシナリオ分類・RTO/RPO目標・GitOpsをDRトリガーとして使う設計 |
| [バックアップ設計書](design/backup-design.md) | PostgreSQL → MinIO → GCS の2層バックアップ設計 |
| [GitHub認証設計書](design/github-auth-design.md) | GitHub App / Fine-grained PAT / OAuth App の使い分け |
| [Golden Path設計書](design/golden-path-design.md) | Scaffolder〜ArgoCD自動sync・Platform自動応答の設計 |
| [リソース管理設計書](design/resource-management-design.md) | ArgoCDアプリケーションとKubernetesリソースの管理方針 |

---

## Runbook一覧

| タイトル | 概要 |
|---|---|
| [DBリストア手順](runbook/dr-restore.md) | PVC破損・クラスタ全損・WSL全損の各シナリオ別DR手順 |
| [シークレット追加・更新手順](runbook/secrets-management.md) | SOPS × ESO を使った Secret の追加・更新・確認手順 |
