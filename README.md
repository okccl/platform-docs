# platform-docs

## このリポジトリについて

ADRや作業手順書など、platformに関係するドキュメント類を保管しています。

---

## ADR一覧

ポートフォリオ全体の技術選定の意思決定を記録しています。
実務経験なども踏まえつつ、「なぜその技術を選んだか」を Context → Options → Decision → Reasons → Consequences の形式で整理し、
構築できるだけでなく設計から判断できることを示す目的で作成しました。

| No. | タイトル | 対象リポジトリ | Status |
|---|---|---|---|
| [ADR-001](adr/ADR-001-gitops-engine.md) | GitOpsエンジンの選択 | platform-gitops / platform-infra | Accepted |
| [ADR-002](adr/ADR-002-helm-library-chart.md) | Kubernetesマニフェスト抽象化方法の選択 | platform-charts / platform-gitops | Accepted |
| [ADR-003](adr/ADR-003-secrets-management.md) | Secret管理戦略の選択 | platform-gitops | Accepted |
| [ADR-004](adr/ADR-004-postgresql-operator.md) | PostgreSQL Operatorの選択 | platform-gitops / platform-charts | Accepted |
| [ADR-005](adr/ADR-005-crossplane.md) | インフラリソース管理の責務分離（Crossplane vs Terraform） | platform-gitops / platform-infra | Draft |
| [ADR-006](adr/ADR-006-cilium.md) | CNIの選択（Cilium） | platform-infra | Draft |
| [ADR-007](adr/ADR-007-local-to-cloud.md) | Local→Cloud 移行パス設計方針 | platform-infra / platform-gitops | Draft |

---

## 意思決定の全体像

```
ローカル基盤
└─ ADR-001: ArgoCD を GitOps エンジンに選択
      └─ pull型によるdrift検出・GUI・App of Appsパターン
アプリデプロイ抽象化
└─ ADR-002: Helm Library Chart を抽象化レイヤーに選択
      └─ テンプレート一元管理・ガードレール・Golden Path
セキュリティ
└─ ADR-003: SOPS×Age + ESO を Secret 管理に選択
      └─ Secrets as Code・クラスタ非依存・DR整合性
データ管理
└─ ADR-004: CloudNativePG を PostgreSQL Operator に選択
      └─ ライセンス・k8sネイティブ・クラウド親和性
インフラ管理
└─ ADR-005: Terraform と Crossplane を用途で使い分け
      └─ クラスタ基盤はTerraform / 開発者向けリソースはCrossplane
ネットワーク
└─ ADR-006: Cilium を CNI に選択
      └─ eBPF・L7 NetworkPolicy・Hubble可観測性
クラウド移行
└─ ADR-007: Local→Cloud 移行パス設計方針（Draft）
      └─ 何をマネージドに置き換えるか / 何を持ち込むか
```

---

## Runbook一覧

プラットフォーム運用中に発生した既知の問題と対処手順を記録しています。

| No. | タイトル | 関連コンポーネント |
|---|---|---|
| [Runbook-001](runbook/Runbook-001-cnpg-wal-error.md) | CNPG 旧 primary WAL エラー対処 | CloudNativePG |
| [Runbook-002](runbook/Runbook-002-coredns.md) | CoreDNS への host.k3d.internal 登録 | k3d / CoreDNS |

---

## 関連リポジトリ

| リポジトリ | 役割 |
|---|---|
| [platform-infra](https://github.com/okccl/platform-infra) | k3d クラスタ IaC・mise・Terraform（EKS） |
| [platform-gitops](https://github.com/okccl/platform-gitops) | ArgoCD による GitOps 管理 |
| [platform-charts](https://github.com/okccl/platform-charts) | Helm Library Chart（common-app / common-db） |
| [sample-backend](https://github.com/okccl/sample-backend) | FastAPI + PostgreSQL（Golden Path利用例） |
| [sample-frontend](https://github.com/okccl/sample-frontend) | React + Vite（Golden Path利用例） |

ポートフォリオ全体の概要は [okccl/okccl](https://github.com/okccl/okccl) を参照してください。
