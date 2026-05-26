# GitHub 認証・認可設計書

---

## 1. 概要

GitHub との連携には目的ごとに異なる認証方式を使用する。1 つの方式に統一しなかった理由は、各方式が得意とするユースケースが異なるためである。

| 方式 | 用途 |
|---|---|
| **okccl-gitops GitHub App** | GHCR imagePullSecret（ESO 経由）・CI/CD 自動化 |
| **Fine-grained PAT** | Backstage API アクセス・Scaffolder リポジトリ操作 |
| **GitHub OAuth App** | Backstage ユーザーサインイン |
| **GITHUB_TOKEN（組み込み）** | apps-gitops PR 自動マージ |

---

## 2. okccl-gitops GitHub App

### 2.1 役割と用途

| 用途 | 仕組み | 参照箇所 |
|---|---|---|
| Backstage pod 用 GHCR imagePullSecret | ESO `GithubAccessToken` generator（`packages: read`、30 分ごとに更新） | `platform/secrets/generators/backstage-ghcr-token.yaml` |
| イメージタグ更新 PR 作成・マージ | GitHub Actions で `actions/create-github-app-token` を使用 | `platform-gitops/.github/workflows/update-gitops.yaml` |
| teardown 時の GHCR パッケージ削除・apps-gitops マニフェスト削除 | 同上 | `platform-gitops/.github/workflows/teardown.yaml` |

### 2.2 識別情報

| 項目 | 値 |
|---|---|
| App ID | 3638469 |
| Client ID | `Iv23liMGIykpOssjwg6a` |
| Installation ID | 131473848 |

### 2.3 秘密鍵の管理フロー

```
gitops-github-app-source.yaml  (SOPS + Age 暗号化)
  └─ ESO ClusterSecretStore (kubernetes-store)
       └─ backstage-secret.GITHUB_APP_PRIVATE_KEY
            ├─ ESO GithubAccessToken generator → ghcr-pull-secret (30min 更新)
            └─ GitHub Actions secrets.GITOPS_APP_PRIVATE_KEY (手動登録)
```

秘密鍵は 1 本で ESO 経由（GHCR）と GitHub Actions 経由（CI/CD）の両方に使用する。

### 2.4 生成アプリリポジトリへの Client ID 配布

Scaffolder（`publish:github`）は新規リポジトリを作成する際に `repoVariables` パラメータで `GITOPS_APP_CLIENT_ID` を自動設定する。生成されたリポジトリの GitHub Actions はこの変数を使って App トークンを生成し、apps-gitops へのイメージタグ更新 PR を作成する。

`GITOPS_APP_PRIVATE_KEY`（秘密鍵）は暗号化処理が必要なため Scaffolder での自動設定が困難であり、リポジトリ払い出し後に手動設定する運用とした。

> **Organization 移行後**: Organization-level secrets により秘密鍵の自動配布が可能になる（「6. 個人アカウントの制約」参照）。

---

## 3. Fine-grained PAT（GITHUB_PAT）

### 3.1 役割

Backstage が GitHub API を直接呼び出す際に使用する。環境変数 `GITHUB_PAT` として Backstage に渡され、`integrations.github.token` に設定される。

| 用途 | アクション |
|---|---|
| Scaffolder: リポジトリ作成・コードプッシュ | `publish:github` |
| Scaffolder: apps-gitops への初回 PR 作成 | `publish:github:pull-request` |
| Backstage: カタログ読み込み（GitHub 上の `catalog-info.yaml`） | Backstage catalog backend |
| teardown テンプレート: ワークフロー起動 | `github:actions:dispatch` |

### 3.2 必要スコープ

| Permission | 用途 |
|---|---|
| `Administration: Read & Write` | リポジトリ作成（`publish:github`） |
| `Contents: Read & Write` | コードプッシュ・カタログ読み取り |
| `Workflows: Read & Write` | `.github/workflows/` ファイルのプッシュ |
| `Pull requests: Read & Write` | apps-gitops への PR 作成（`publish:github:pull-request`） |
| `Actions: Read & Write` | teardown テンプレートの workflow dispatch（`github:actions:dispatch`） |

### 3.3 Scaffolder に GitHub App を使わない理由

`integrations.github` に GitHub App（installation token）を設定すると、`publish:github` アクションでのリポジトリ作成が失敗する。

**原因:** GitHub App の installation token（server-to-server）は `POST /user/repos` エンドポイントを呼び出せない。このエンドポイントはパーソナルアカウント配下のリポジトリを作成する際に使用されるが、installation token の権限スコープ外となっている。Organization アカウントであれば `POST /orgs/{org}/repos` が使用可能なため問題ないが、`okccl` はパーソナルアカウントであるためこの制約に該当する。

### 3.4 シークレット管理フロー

```
backstage-github-app-source.yaml  (SOPS + Age 暗号化、githubPat フィールド)
  └─ ESO ClusterSecretStore (kubernetes-store)
       └─ backstage-secret.GITHUB_PAT
            └─ Backstage 環境変数 GITHUB_PAT
                 └─ integrations.github.token
```

---

## 4. GitHub OAuth App

### 4.1 役割

Backstage へのユーザーサインインに使用する。`auth.providers.github` に設定される。

| 項目 | 内容 |
|---|---|
| 用途 | ユーザーログイン・PR カード・SCM 連携表示 |
| 設定箇所 | `auth.providers.github.development.clientId/clientSecret` |

### 4.2 シークレット管理フロー

```
backstage-github-app-source.yaml  (SOPS + Age 暗号化)
  ├─ githubOauthClientId
  └─ githubOauthClientSecret
       └─ ESO → backstage-secret.GITHUB_OAUTH_CLIENT_ID / GITHUB_OAUTH_CLIENT_SECRET
```

---

## 5. GITHUB_TOKEN（GitHub Actions 組み込み）

apps-gitops の PR 自動マージにのみ使用する。カスタムトークン不要。

| ファイル | 用途 |
|---|---|
| `apps-gitops/.github/workflows/auto-merge-app-pr.yaml` | `app/*` ブランチの PR を自動 squash マージ |

Scaffolder（PAT 経由）が作成した `app/<app-name>` ブランチ PR を検知して自動マージする。

---

## 6. 個人アカウントの制約と将来方針

本プロジェクトでは個人アカウント（`okccl`）を使用する必要があり、以下の制約が生じている。

### 6.1 Organization 移行で解消される制約

| 制約 | 影響 | 現在の対処 |
|---|---|---|
| GitHub App installation token が `POST /user/repos` を呼べない | Scaffolder でのリポジトリ作成が失敗する | Fine-grained PAT に切り替え（3 章参照） |
| Organization-level secrets が使えない | Scaffolder が新規作成したリポジトリに `GITOPS_APP_PRIVATE_KEY` を自動配布できない | 払い出し後に手動設定 |

**将来方針:** `okccl` を GitHub Organization に変換することで両制約が解消される。`integrations.github` を GitHub App に戻すことができ、新規リポジトリへの secrets 自動継承も可能になる。

### 6.2 Organization 移行では解消されない制約

| 制約 | 影響 | 現在の対処 |
|---|---|---|
| GitHub App installation token で `delete_repo` スコープを取得できない | teardown ワークフローからリポジトリを自動削除できない | ワークフロー完了後に手動削除 |

`DELETE /repos/{owner}/{repo}` は `delete_repo` スコープを要求する。このスコープは GitHub App の permission model（`Administration: R/W` 等）とは独立した OAuth スコープであり、**アカウント種別に関わらず** App installation token では取得できない。PAT であれば付与できるが、不特定のリポジトリを削除できる PAT を CI に持たせることはセキュリティリスクが高いため、手動削除の運用とした。
