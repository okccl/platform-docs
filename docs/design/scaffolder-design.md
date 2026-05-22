# Scaffolder 設計書

> **ステータス**: 作成中

---

## 1. 概要

### 1.1 目的

（作成中）

### 1.2 テンプレート一覧（fullstack / teardown）

（作成中）

---

## 2. apps-gitops ディレクトリ構造

### 2.1 設計方針（論理アプリ単位でグルーピングする理由）

（作成中）

### 2.2 ディレクトリ構成

（作成中）

### 2.3 app.yaml スキーマ

（作成中）

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
