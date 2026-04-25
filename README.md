# platform-adr

## このリポジトリについて

ポートフォリオ全体の技術選定の意思決定を記録したリポジトリです。

「なぜその技術を選んだか」を Context → Options → Decision → Reasons → Consequences の形式で整理し、
構築できるだけでなく設計から判断できることを示す目的で作成しました。

各ADRは実務経験で直面した課題や比較調査を踏まえた実体験ベースの記録です。

---

## ADR一覧

| No. | タイトル | 対象リポジトリ | Status |
|---|---|---|---|
| [ADR-001](adr/ADR-001-gitops-engine.md) | GitOpsエンジンの選択 | platform-gitops / platform-infra | Accepted |
| [ADR-002](adr/ADR-002-helm-library-chart.md) | Kubernetesマニフェスト抽象化方法の選択 | platform-charts / platform-gitops | Accepted |
| [ADR-003](adr/ADR-003-secrets-management.md) | Secret管理戦略の選択 | platform-gitops | Accepted（一部検証中） |
| [ADR-004](adr/ADR-004-postgresql-operator.md) | PostgreSQL Operatorの選択 | platform-gitops / platform-charts | Accepted |

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
```

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
