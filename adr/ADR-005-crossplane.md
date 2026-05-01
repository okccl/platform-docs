# ADR-005: インフラリソース管理の責務分離（Crossplane vs Terraform）

## Context

Platform Engineering の文脈では、インフラリソース（DB・ネットワーク・ストレージなど）の
プロビジョニングをどのレイヤーで管理するかという設計判断が重要になる。

一般的な選択肢として Terraform が広く使われているが、Terraform は「実行した時点での
あるべき状態への収束」を保証するものであり、実行とクラスタ状態は分離している。
つまり、Terraform 実行後に誰かが手動でリソースを変更しても Terraform は検知しない。
これは ADR-001 で課題として挙げた「push 型 CD とドリフト検知」の問題と同じ構造である。

また現場では、開発者がデータベースや環境を必要とするたびにインフラチームへ依頼する
フローが発生しがちで、プラットフォームチームの作業ボトルネックになるケースがある。

---

## Options

### Option 1: Terraform で統一管理する
- 既存の IaC ツールとして最も普及しており、学習コストが低い
- HCL で宣言的に記述できるが、実行は手動またはパイプライン経由
- **問題点:** 実行後のドリフトを自動検知・修正する機能がない。
  push 型 CD と同様に「デプロイした瞬間の状態」しか保証できない

### Option 2: Crossplane を採用する
- Kubernetes の Operator パターンでクラウドリソースを管理する
- `kubectl apply` でリソースを宣言すると、Crossplane が継続的に現実をあるべき状態に収束させる
- ArgoCD と組み合わせることで、Git → ArgoCD → Crossplane → クラウドリソース という
  一貫した GitOps フローを構築できる
- **問題点:** Terraform に比べてエコシステムの成熟度が低く、
  Provider によってはサポートされないリソースがある

### Option 3: Terraform と Crossplane を用途で使い分ける
- クラスタ基盤（VPC・EKS クラスタ自体）は Terraform で管理
- 開発者向けのセルフサービスリソース（DB・キュー など）は Crossplane で管理
- それぞれの得意領域に特化させる

---

## Decision

**Terraform と Crossplane を用途で使い分ける方針を採用する。**

- クラスタ基盤（VPC・EKS・IAM など）は Terraform で管理する
- 開発者がセルフサービスで利用するリソース（RDS など）は Crossplane で管理する

Phase 11 では provider-helm を使ってローカル環境で Crossplane の構造を実証した。
Phase 13（EKS）で provider-aws を使いクラウドリソースの管理を検証する予定。

---

## Reasons

1. **Terraform とドリフト問題**
   Terraform は実行時点での状態収束を保証するが、その後の手動変更を検知しない。
   インフラチームが厳密に運用ルールを守れる範囲（クラスタ基盤）では問題になりにくいが、
   開発者が頻繁に変更・再作成するリソースでは drift が発生しやすい。
   Crossplane の継続的 reconciliation はこの問題を構造的に解決できる。

2. **開発者のセルフサービス化**
   開発者が DB や環境を必要とするたびにインフラチームへ依頼するフローは、
   プラットフォームチームのボトルネックになる。Crossplane を使えば、
   開発者が Kubernetes リソースを apply するだけで DB が払い出される仕組みを作れる。
   インフラチームへの依頼フローをなくし、開発速度を上げることがプラットフォームの目的の1つである。

3. **GitOps との一貫性**
   ArgoCD による GitOps フローをアプリだけでなくインフラリソースにも統一できる。
   クラウドリソースの状態も Git が Single Source of Truth になり、
   DR 時の復旧手順が簡潔になる。

4. **ローカルでの構造検証**
   provider-helm を使って「Git（Release CR）→ ArgoCD → Crossplane → Helm release」
   という責務分離の構造をローカル環境で実証した。GUI での状態確認は ArgoCD の方が
   見やすいため人間が操作する場面では ArgoCD で十分だが、
   Crossplane はスクリプトやパイプラインからの操作・自動化との親和性が高い。

---

## Consequences

**ポジティブ:**
- インフラ基盤（Terraform）とアプリ向けリソース（Crossplane）の責務が明確に分離される
- クラウドリソースも GitOps の管理下に置け、ドリフトを自動修正できる
- 開発者がインフラチームを介さずにリソースを調達できるセルフサービス基盤の土台になる

**ネガティブ・トレードオフ:**
- Crossplane の導入・運用コストが増える。Provider の成熟度もツールによって差がある
- 2つのツールを使い分ける境界線の設計・チームへの説明コストが発生する
- Phase 13 での provider-aws 検証が未完了のため、実際のクラウドリソース管理における
  有効性はまだ実証段階である
