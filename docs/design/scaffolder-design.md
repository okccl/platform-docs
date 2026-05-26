# Scaffolder 設計書

---

## 1. 概要

### 1.1 目的

Backstage Scaffolder は、開発者が Kubernetes を意識せずにアプリケーション環境を払い出せる IDP のエントリーポイントとして機能する。

従来、新規アプリの立ち上げには GitHub リポジトリ作成・CI/CD 設定・GitOps マニフェスト作成・ArgoCD 登録・Namespace 作成・RBAC 設定といった作業が必要だったが、Scaffolder を通じてこれらをすべて自動化する。

この設計書のスコープは以下の通り。

- テンプレートの定義とステップ設計
- 生成されるファイルと各ファイルの役割
- apps-gitops のディレクトリ構造設計
- GitHub 連携の設計意図

### 1.2 テンプレート一覧（fullstack / teardown）

| テンプレート | 説明 | 主要アクション |
|---|---|---|
| fullstack | FastAPI（backend）+ React（frontend）のフルスタックアプリを一括払い出し | `publish:github` / `publish:github:pull-request` / `catalog:register` |
| teardown | fullstack で作成したアプリを一括削除 | `github:actions:dispatch` |

`vcluster` テンプレートも存在するが、現在は実験的位置づけのため本書では扱わない。

---

## 2. apps-gitops ディレクトリ構造

### 2.1 設計方針（論理アプリ単位でグルーピングする理由）

apps-gitops はアプリごとにディレクトリをグルーピングする設計を採っている。

理由は、`user-apps-infra` ApplicationSet が `apps/**/app.yaml` のパターンでファイルを検知し、論理アプリ単位で AppProject と Namespace を自動生成するためである。`app.yaml` 1 ファイルが「このアプリが存在する」という宣言を担い、それ以降のリソース生成はすべてプラットフォームが自動処理する。

backend と frontend を同じディレクトリ配下に置く理由は、論理アプリとして 1 つの AppProject・1 つの Namespace を共有するためである。それぞれを別の論理アプリとして扱うと AppProject / Namespace が倍になり、管理が複雑になる。

### 2.2 ディレクトリ構成

```
apps/
  <app-name>/
    app.yaml                          # 論理アプリ宣言（ApplicationSet が検知）
    <app-name>-backend/
      application.yaml               # ArgoCD Application（backend）
      values.yaml                    # Helm values（common-app chart）
      manifests/
        httproute.yaml               # Gateway API ルーティング
    <app-name>-frontend/
      application.yaml               # ArgoCD Application（frontend）
      values.yaml                    # Helm values（common-app chart）
      manifests/
        httproute.yaml               # Gateway API ルーティング
```

### 2.3 app.yaml スキーマ

```yaml
appName: <アプリ名>    # AppProject 名・Namespace 名に使用
namespace: <アプリ名>  # appName と同値（将来的な独立変更の余地を残す）
owner: <グループ名>    # Keycloak グループ名（ArgoCD RBAC に使用）
```

3 フィールドのみのシンプルな構成とした。ApplicationSet はこの宣言から最低限の情報のみを取り出し、Platform 側で標準的な AppProject と Namespace を生成する。アプリ固有の設定（リソース制限・レプリカ数など）は `values.yaml` で管理し、`app.yaml` は「このアプリが存在する」という意図のみを持つよう設計している。

---

## 3. テンプレート設計

### 3.1 fullstack テンプレート（生成物一覧・各ファイルの役割）

7ステップで構成される。

| ステップ | アクション | 処理 |
|---|---|---|
| fetch-backend | `fetch:template` | backend-skeleton をレンダリングして `./backend/` に展開 |
| publish-backend | `publish:github` | `okccl/<app-name>-backend` リポジトリを作成してプッシュ |
| fetch-frontend | `fetch:template` | frontend-skeleton をレンダリングして `./frontend/` に展開 |
| publish-frontend | `publish:github` | `okccl/<app-name>-frontend` リポジトリを作成してプッシュ |
| fetch-gitops | `fetch:template` | gitops-skeleton をレンダリングして `./gitops/` に展開 |
| publish-gitops | `publish:github:pull-request` | apps-gitops に `app/<app-name>` ブランチで PR を作成 |
| register-backend / register-frontend | `catalog:register` | Backstage カタログにエンティティを登録 |

**backend / frontend リポジトリの生成物**

| ファイル | 役割 |
|---|---|
| `src/` | スターターコード（FastAPI / React + Vite） |
| `Dockerfile` | コンテナイメージビルド設定 |
| `.github/workflows/build.yaml` | CI: `main` push → GHCR push → platform-gitops へ `repository_dispatch` |
| `catalog-info.yaml` | Backstage カタログ登録情報 |
| `aqua.yaml` | 開発ツールバージョン固定（詳細は 4 章参照） |

**apps-gitops の生成物**

| ファイル | 役割 |
|---|---|
| `apps/<app-name>/app.yaml` | 論理アプリ宣言（ApplicationSet が検知） |
| `<app-name>-backend/application.yaml` | ArgoCD Application（backend） |
| `<app-name>-backend/values.yaml` | Helm values（common-app chart） |
| `<app-name>-backend/manifests/httproute.yaml` | Gateway API ルーティング |
| `<app-name>-frontend/` | frontend 側の同構成 |

**設計上の判断**

- **ArgoCD Application は multi-source 構成**: `platform-charts`（Helm chart）+ `apps-gitops`（values.yaml・manifests）の 3 ソースを組み合わせる。chart はプラットフォーム管理で読み取り専用、values と manifests はアプリチームが変更できる領域として分離している。
- **`withDb` フラグ**: DB が不要なアプリに余計なリソースを作らないよう、オプションとして切り出した。`values.yaml` の `db.enabled` に反映され、common-app chart が CNPG Cluster の生成有無を制御する。
- **Argo Rollouts をデフォルト有効**: カナリアデプロイ（20% → pause → 100%）を初期設定として組み込むことで、アプリチームが意識せずに安全なデプロイ戦略を得られる。
- **`catalog:register` は `optional: true`**: カタログ登録の失敗が Scaffolder 全体をロールバックしないようにするため。登録は後から手動でも可能。

### 3.2 teardown テンプレート

`github:actions:dispatch` アクション 1 つで構成される。Backstage 側ではワークフローの起動のみを担い、削除処理の実体は `okccl/platform-gitops` の `teardown.yaml` ワークフローが担う。

ワークフローの処理内容:
1. apps-gitops をチェックアウトして `apps/<app-name>/` を削除し、main に直接 push
2. ArgoCD の prune により Namespace 配下のリソースを削除
3. GHCR パッケージを削除（未作成の場合はスキップ）
4. **GitHub リポジトリ削除は手動**（後述）

Scaffolder 側で `appName` と `confirmName` の 2 回入力を要求する設計にしており、入力の一致チェックをワークフロー側で行う。削除は完全に不可逆のため、このミス防止ステップは除かない方針とした。

---

## 4. 開発環境セットアップの組み込み（aqua.yaml）

fullstack テンプレートの `fetch:template` ステップが、backend-skeleton / frontend-skeleton に含まれる `aqua.yaml` をそのままコピーして生成する。クローン直後に `aqua install` 1 コマンドで開発ツールが揃う状態を初期値として提供する。

```yaml
# 例（実際のファイルは backend-skeleton/aqua.yaml を参照）
packages:
  - name: kubernetes/kubectl@v1.35.3
  - name: helm/helm@v3.20.1
  - name: argoproj/argo-cd@v3.2.9
  # ...
```

**現在の制約**: スケルトンの `aqua.yaml` に記載するバージョンは `platform-infra/aqua.yaml`（PE チームの source of truth）と手動で同期する必要がある。自動同期の仕組みは未実装であり、バージョンアップ時は両ファイルを合わせて更新すること。

ツール管理の設計方針（PE チームとアプリチームのモデルの違い）は `dev-env-design.md` 2 章を参照。

---

## 5. GitHub 統合設計

GitHub 認証・認可の全体設計（認証手段の比較・秘密鍵管理フロー・個人アカウントの制約と将来方針）は [`github-auth-design.md`](github-auth-design.md) にまとめている。本章では Scaffolder の動作に直接関わる部分のみを記述する。

### 5.1 認証方式の分離

Backstage の GitHub 連携は用途ごとに異なる認証方式を使用する。

| 用途 | 方式 | 設定箇所 |
|---|---|---|
| ユーザーログイン | GitHub OAuth App | `auth.providers.github` |
| API アクセス（カタログ読み取り・Scaffolder） | Fine-grained PAT | `integrations.github.token` |

Scaffolder（`publish:github`）に GitHub App を使用しない理由は、個人アカウントへのリポジトリ作成（`POST /user/repos`）が installation token では呼び出せないためである（詳細は `github-auth-design.md` 3.3 節参照）。

### 5.2 新規リポジトリへの CI credentials 配布

`publish:github` の `repoVariables` パラメータを使い、リポジトリ作成と同時に `GITOPS_APP_CLIENT_ID`（非機密の App Client ID）を自動設定する。`GITOPS_APP_PRIVATE_KEY`（秘密鍵）は Scaffolder での自動配布が困難なため、リポジトリ払い出し後に手動設定する運用とした。

個人アカウントのため Organization-level secrets が使えないことが根本原因であり、Organization に移行すれば秘密鍵の自動継承が可能になる（詳細は `github-auth-design.md` 6.1 節参照）。

### 5.3 teardown とリポジトリ削除

GitHub API の制約（`delete_repo` スコープは App installation token では取得不可・アカウント種別に依存しない）により、teardown ワークフローはリポジトリ削除を自動化できない。ワークフロー完了後に手動削除する運用とした（詳細は `github-auth-design.md` 6.2 節参照）。

```bash
gh repo delete okccl/<app-name>-backend --yes
gh repo delete okccl/<app-name>-frontend --yes
```
