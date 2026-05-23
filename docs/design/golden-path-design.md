# Golden Path 設計書

---

## 1. 概要

### 1.1 Golden Path の定義と目的

Golden Path とは、開発者がアプリケーション環境を「正しく・速く」立ち上げられる標準化された経路を指す。
Backstage Scaffolder がエントリーポイントとなり、リポジトリ作成・CI/CD 設定・GitOps マニフェスト生成・
ArgoCD 登録・Namespace/RBAC 設定をすべて自動化する。

開発者が行う操作は Backstage 上でのフォーム入力のみであり、
クローン直後から `aqua install` でツールが揃い、コードを書き始められる状態を初期値として提供する。

**設計上の判断**

Scaffolder の導入目的は「手順の自動化」だけではない。
プラットフォームが「何を標準とするか」を明示する手段として機能する。
Argo Rollouts によるカナリアデプロイ・MinIO バックアップ・Kyverno ポリシーによるガードレールを
デフォルトで組み込むことで、開発者が意識しなくても本番相当の運用品質が確保される設計とした。

### 1.2 設計方針

**Scaffolder の責務をファイル生成に限定する**

Scaffolder はリポジトリへのファイルプッシュと PR 作成のみを行い、クラスタへの直接操作は一切行わない。
クラスタへの反映は ArgoCD が Git の変化を検知して行う。

**設計上の判断**

Scaffolder がクラスタを直接操作すると、Scaffolder の失敗時にクラスタが中途半端な状態になるリスクがある。
GitOps の原則（Git = クラスタ状態）を守ることで、失敗しても Git を巻き戻すだけで済む。
また、Scaffolder の完了とクラスタへの反映を非同期にすることで、Scaffolder のロールが単純になり障害点が減る。

**Platform の自動応答で「暗黙の標準」を提供する**

開発者が明示的に要求しなくても、app.yaml のコミットをトリガーに AppProject・Namespace・Secret・
監視設定がプラットフォーム側で自動生成される。開発者が知るべきことは最小限にする。

---

## 2. 払い出しフロー全体図

### 2.1 フロー概要

```
開発者
  │
  ├─ Backstage Scaffolder（フォーム入力）
  │    ├─ <app>-backend リポジトリ作成・スターターコード push
  │    ├─ <app>-frontend リポジトリ作成・スターターコード push
  │    └─ apps-gitops に PR 作成（app.yaml / application.yaml / values.yaml / httproute.yaml）
  │
  ├─ auto-merge ワークフロー（platform-gitops）
  │    └─ apps-gitops PR を自動 squash merge
  │
  └─ ArgoCD（Git 変化を検知）
       ├─ user-apps-infra ApplicationSet
       │    └─ app.yaml 検知 → AppProject・Namespace（app-type: user-app ラベル）自動生成
       ├─ ClusterExternalSecret
       │    └─ Namespace ラベルを検知 → minio-backup-secret を自動配布
       └─ user-apps App-of-Apps
            └─ application.yaml 検知 → ArgoCD Application 生成 → 初期イメージでデプロイ
```

コード push 後の CI/CD フロー:

```
開発者が main push
  │
  ├─ CI（GitHub Actions: build.yaml）
  │    └─ イメージビルド → GHCR push → platform-gitops へ repository_dispatch
  │
  └─ platform-gitops（update-gitops.yaml）
       └─ apps-gitops に values.yaml 更新 PR 作成 → auto-merge
            └─ ArgoCD が新イメージを検知 → Argo Rollouts カナリア（20% → pause → 100%）
```

### 2.2 設計上の判断

**auto-merge を採用した理由**

apps-gitops への PR を手動マージにすると、開発者がマージ操作を覚える必要が生じる。
Golden Path の目的（開発者がインフラを意識しない）と矛盾するため、
`app/*` ブランチへの PR は自動 squash merge とした。

`main` への直接 push でなく PR 経由にしているのは、ArgoCD の prune が誤動作した場合に
Git の履歴から意図を追えるようにするためと、将来的にレビューを挟む選択肢を残すためである。

**repository_dispatch 経由でイメージ更新する理由**

イメージタグを直接 apps-gitops に push するのではなく、
platform-gitops のワークフロー（update-gitops.yaml）が受け取って PR を作る設計にしている。
これはイメージ更新の操作権限を platform-gitops ワークフローに集約し、
apps-gitops への直接 push 権限をアプリリポジトリに持たせないためである。

---

## 3. Scaffolder の責務（何を生成するか）

### 3.1 生成物の全体像

Scaffolder が生成するのはファイルのみであり、クラスタへの操作は行わない。
生成物は 3 種類のリポジトリに分かれる。

| リポジトリ | 生成物 | 役割 |
|---|---|---|
| `<app>-backend` | スターターコード・Dockerfile・CI ワークフロー・aqua.yaml・catalog-info.yaml | アプリ開発の出発点 |
| `<app>-frontend` | 同上（React + Vite） | 同上 |
| `apps-gitops` | app.yaml・application.yaml・values.yaml・httproute.yaml | Platform への環境払い出し宣言 |

各ファイルの詳細は `scaffolder-design.md` 3 章を参照。

### 3.2 設計上の判断

**withDb フラグで DB 不要なアプリに余計なリソースを作らない**

DB が必要かどうかはアプリによって異なる。
`withDb` フラグを false にすると CNPG Cluster・PodMonitor・MinIO バックアップ設定が生成されない。
「フルスタックテンプレートだが DB はオプション」という設計は、
将来的にテンプレートを frontend-only などに細分化せずに済む利点もある。

**catalog-info.yaml をスケルトンに含める理由**

Backstage でアプリ一覧を管理できるようにするため、各リポジトリに `catalog-info.yaml` を配置する。
Scaffolder の `catalog:register` ステップがこのファイルを Backstage カタログに登録し、
払い出し完了後すぐにアプリが Backstage 上に表示される。

---

## 4. Platform の自動応答

### 4.1 AppProject・RBAC の自動生成（user-apps-infra ApplicationSet）

apps-gitops の `apps/**/app.yaml` を ApplicationSet が検知し、
以下のリソースを自動生成する。

| リソース | 生成内容 |
|---|---|
| AppProject | `spec.sourceRepos` にアプリ用リポジトリ・`spec.destinations` にアプリ Namespace を設定 |
| Namespace | `app-type: user-app` ラベルを付与（ClusterExternalSecret のターゲット条件として使用） |
| RoleBinding | app-developer グループに Namespace スコープの権限を付与 |

### 4.2 Secret の自動配布（ClusterExternalSecret）

`app-type: user-app` ラベルを持つ Namespace が作成されると、
ClusterExternalSecret が `minio-backup-secret` を自動配布する。

開発者は Secret の存在を意識する必要がなく、DB バックアップが初期状態から有効になる。

**設計上の判断**

全アプリに共通して配布する Secret（MinIO 認証情報）を各アプリ App に分散させると、
Secret の更新時に全アプリの変更が必要になる。ClusterExternalSecret の一元管理により、
プラットフォーム側の更新のみで全アプリに反映される。

個別アプリの ExternalSecret を apps-gitops に持たせる設計も検討したが採用しなかった。
追加の理由として、ClusterSecretStore（wave 11）が起動する前にアプリ App（wave 13 以降）が
ExternalSecret を作ろうとすると fresh bootstrap 時に失敗するという起動順の問題もある。

### 4.3 バックアップ・監視のデフォルト適用（common-db chart）

`withDb: true` の場合、common-db chart が以下を自動生成する。

| リソース | 内容 |
|---|---|
| CNPG Cluster | PostgreSQL 17 HA（3ノード）・MinIO バックアップ・WAL アーカイブ設定 |
| PodMonitor | Prometheus によるメトリクス収集 |
| PrometheusRule | WAL アーカイブ失敗・バックアップ失敗アラート |

**設計上の判断**

バックアップ・監視設定を開発者が明示的に有効化する設計にすると、
設定忘れによる「気づいたら監視されていなかった」が発生する。
platform chart（common-db）に組み込み、`withDb: true` にするだけで全て有効になる設計とした。

---

## 5. 開発者が受け取るもの（ゴール状態）

（作成中）

---

## 6. アプリのライフサイクル管理（teardown）

（作成中）

---

## 7. 設計上の判断

（作成中）
