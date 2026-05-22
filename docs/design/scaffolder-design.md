# Scaffolder 設計書

> **ステータス**: 作成中

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

（作成中）

### 3.2 teardown テンプレート

（作成中）

---

## 4. 開発環境セットアップの組み込み（aqua.yaml）

（作成中）

---

## 5. GitHub 統合設計

### 5.1 認証方式の分離

Backstage の GitHub 連携は用途ごとに異なる認証方式を使用する。

| 用途 | 方式 | 設定箇所 |
|---|---|---|
| ユーザーログイン | GitHub OAuth App | `auth.providers.github` |
| API アクセス（カタログ読み取り・Scaffolder） | Fine-grained PAT | `integrations.github.token` |

### 5.2 Scaffolder に GitHub App を使わない理由

`integrations.github` に GitHub App（installation token）を使用すると、`publish:github` アクションでのリポジトリ作成が失敗する。

**原因:** GitHub App の installation token（server-to-server）は `POST /user/repos` エンドポイントを呼び出せない。このエンドポイントはパーソナルアカウント配下のリポジトリを作成する際に使用されるが、installation token の権限スコープ外となっている。Organization アカウントであれば `POST /orgs/{org}/repos` が使用されるため問題ないが、`okccl` はパーソナルアカウントであるためこの制約に該当する。

**対処:** `integrations.github` は Fine-grained PAT に切り替え、GitHub App は OAuth ログイン専用とする。Fine-grained PAT は以下の権限のみに限定し、最小権限の原則を維持する。

| Permission | 用途 |
|---|---|
| `Administration: Read & Write` | リポジトリ作成（`publish:github`） |
| `Contents: Read & Write` | コードプッシュ・カタログ読み取り |
| `Workflows: Read & Write` | `.github/workflows/` ファイルのプッシュ |
| `Pull requests: Read & Write` | apps-gitops への PR 作成（`publish:github:pull-request`） |
| `Actions: Read & Write` | teardown テンプレートの workflow dispatch（`github:actions:dispatch`） |

> **将来的な移行:** `okccl` を GitHub Organization に変換した場合は GitHub App に戻すことができる。
