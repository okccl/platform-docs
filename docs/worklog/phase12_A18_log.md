# Scaffolder 全面修正

| # | 作業内容 |
|---|---|
| 1 | common-db chart デフォルト値更新 |
| 2 | Scaffolder skeleton ディレクトリ構造変更・app.yaml 追加 |
| 3 | skeleton application.yaml バグ修正 |
| 4 | skeleton values.yaml 充実化 |
| 5 | skeleton に aqua.yaml 追加 |
| 6 | apps-gitops 既存アプリ（sample）移行 |
| 7 | apps-root.yaml の include パターン更新 |
| 8 | user-apps-infra を ApplicationSet に変換 |
| 9 | ClusterExternalSecret 追加（minio-backup-secret 自動配布） |
| 10 | Backstage OIDC 認証障害の対処 |
| 11 | Backstage GitHub 統合を GitHub App から PAT に切り替え |
| 12 | E2E 検証（Scaffolder 全フロー） |
| 13 | fullstack テンプレート: 新規リポジトリへの GITOPS_APP_CLIENT_ID 自動設定 |
| 14 | teardown ワークフローのバグ修正（変数名誤り・rm パス誤り・GHCR エンドポイント誤り・リポジトリ削除を手動対応に変更） |
| 15 | GITOPS_TOKEN 未使用 secret の削除 |
| 16 | ghcr-pull-secret SecretSyncedError 修正（PAT 切り替え時に GITHUB_APP_PRIVATE_KEY が削除されたことが原因） |
| 17 | sample-backend の minio-backup-secret ExternalSecret 削除（ClusterExternalSecret との二重管理解消） |
| 18 | sample-backend OutOfSync 根本原因修正（SSA + ignoreDifferences jsonPointer 配列インデックス問題） |

---

## 1. common-db chart デフォルト値更新

### 背景

新規アプリ払い出し時に明示的な設定なしでバックアップ・モニタリングが有効になるよう、chart のデフォルト値を本番相当に変更した。

| キー | 変更前 | 変更後 |
|---|---|---|
| `db.storageClassName` | `""` | `local-path-retain` |
| `db.backup.enabled` | `false` | `true` |
| `db.backup.endpointURL` | `http://minio.minio.svc.cluster.local:9000` | `http://host.k3d.internal:9000` |
| `db.monitoring.enablePodMonitor` | `false` | `true` |

---

## 2. Scaffolder skeleton ディレクトリ構造変更・app.yaml 追加

### 背景

backend と frontend を 1 つの論理アプリとしてグルーピングし、Platform ApplicationSet が `app.yaml` を検知して AppProject・RBAC を自動生成できる構造に変更した。

### 実施手順

`gitops-skeleton` のディレクトリ構造を以下のように変更した。

```
# 変更前
apps/${{ values.appName }}-backend/
apps/${{ values.appName }}-frontend/

# 変更後
apps/${{ values.appName }}/
  app.yaml                              ← 新規（論理アプリ宣言）
  ${{ values.appName }}-backend/
  ${{ values.appName }}-frontend/
```

`app.yaml` のスキーマ:

```yaml
appName: ${{ values.appName }}
namespace: ${{ values.appName }}
owner: ${{ values.owner }}
```

`namespace.yaml`（backend/manifests 以下）は Platform ApplicationSet が Namespace を管理するため削除。

`template.yaml` には `owner` パラメータを追加し（デフォルト: `app-developer`）、`fetch-gitops` ステップの values に渡すよう変更した。

---

## 3. skeleton application.yaml バグ修正

`gitops-skeleton` 内の application.yaml に以下のバグがあったため修正した。

| 項目 | 変更前（バグ） | 変更後 |
|---|---|---|
| `$values` repoURL（backend/frontend） | `platform-gitops` | `apps-gitops` |
| `project`（backend/frontend） | `user-apps`（存在しない） | `${{ values.appName }}` |
| CNPG `ignoreDifferences`（backend のみ） | なし | `/spec/bootstrap` / `/spec/externalClusters` 追加 |
| values ファイルパス | `apps/${{ values.appName }}-backend/values.yaml` | `apps/${{ values.appName }}/${{ values.appName }}-backend/values.yaml` |
| manifests パス | `apps/${{ values.appName }}-backend/manifests` | `apps/${{ values.appName }}/${{ values.appName }}-backend/manifests` |

---

## 4. skeleton values.yaml 充実化

sample-backend / sample-frontend の実装を参照して skeleton に不足していた設定を追加した。

**backend に追加:**

```yaml
image:
  pullPolicy: IfNotPresent
resources: { requests: { cpu: 100m, memory: 128Mi }, limits: { cpu: 500m, memory: 256Mi } }
probes:
  liveness: { path: /health, initialDelaySeconds: 10, periodSeconds: 10 }
  readiness: { path: /health, initialDelaySeconds: 5, periodSeconds: 5 }
ingress:
  enabled: false
hpa:
  enabled: false
serviceMonitor:
  enabled: true
  path: /metrics
  interval: 30s
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://tempo.tracing.svc.cluster.local:4318"
  - name: OTEL_SERVICE_NAME
    value: "${{ values.appName }}-backend"
rollout:
  enabled: true
  canary:
    steps: [setWeight: 20, pause: {}, setWeight: 100]
```

**frontend に追加:** `image.pullPolicy` / `resources` / `probes` / `ingress` / `hpa` / `serviceMonitor` / `rollout`（OTEL env は除く）

---

## 5. skeleton に aqua.yaml 追加

払い出されたアプリリポジトリでも同一ツールセットを使用できるよう、`backend-skeleton` と `frontend-skeleton` に `aqua.yaml` を追加した。内容は `platform-infra/aqua.yaml` と同一。

---

## 6. apps-gitops 既存アプリ（sample）移行

新ディレクトリ構造に既存 sample アプリを移行した。

```bash
git mv apps/sample-backend apps/sample/sample-backend
git mv apps/sample-frontend apps/sample/sample-frontend
```

`apps/sample/app.yaml` を新規作成:

```yaml
appName: sample
namespace: sample-app   # 既存データ継続のため sample-app を維持（設計原則の例外）
owner: app-developer
```

> **注意:** 設計原則では `namespace = appName` だが、既存 DB データを維持するため `namespace: sample-app` のまま据え置いた。

各 application.yaml のパス参照と `project` を更新した:

| 項目 | 変更前 | 変更後 |
|---|---|---|
| `project` | `sample-apps` | `sample` |
| `$values` パス | `apps/sample-backend/values.yaml` | `apps/sample/sample-backend/values.yaml` |
| `manifests` パス | `apps/sample-backend/manifests` | `apps/sample/sample-backend/manifests` |

---

## 7. apps-root.yaml の include パターン更新

新ディレクトリ構造（`apps/<appName>/<appName>-backend/application.yaml`）に対応するため include パターンを変更した。

```yaml
# 変更前
include: '*/application.yaml'

# 変更後
include: '{**/application.yaml}'
```

**タイミング注意点:** apps-gitops 移行と同時 push を実施した。apps-root の自動 sync が `project: sample`（未作成）を参照する前に T8（ApplicationSet）を完了させる必要があったため、T6+T7 完了後即座に T8 に着手した。

---

## 8. user-apps-infra を ApplicationSet に変換

### 背景

AppProject・Namespace はプラットフォームリソースであり、Scaffolder（開発者側）が直接管理すべきでない。Platform ApplicationSet が `apps-gitops` の `apps/*/app.yaml` を検知し、アプリごとに自動生成する設計に変更した。

### 実施手順

`platform/applications/user-apps-infra.yaml` を Application から ApplicationSet に変換:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  generators:
    - git:
        repoURL: https://github.com/okccl/apps-gitops.git
        revision: HEAD
        files:
          - path: "apps/*/app.yaml"
  template:
    metadata:
      name: "user-apps-infra-{{appName}}"
    spec:
      source:
        path: platform/user-apps-infra
        helm:
          values: |
            appName: "{{appName}}"
            namespace: "{{namespace}}"
```

`platform/user-apps-infra/` を Helm chart 化し、`appproject.yaml` / `namespace.yaml` をテンプレート化:

- `AppProject`: `sourceRepos: https://github.com/okccl/*`、destination: `namespace: {{ .Values.namespace }}`
- `Namespace`: `app-type: user-app` ラベル付き（ClusterExternalSecret のセレクタに使用）

動作確認: `apps/sample/app.yaml` を検知して `user-apps-infra-sample` Application が生成され、`AppProject: sample` と `Namespace: sample-app（app-type: user-app ラベル付き）` が作成された。

---

## 9. ClusterExternalSecret 追加（minio-backup-secret 自動配布）

`app-type: user-app` ラベルを持つ全 Namespace に `minio-backup-secret` を自動配布する `ClusterExternalSecret` を追加した。

```yaml
# platform/secrets/config/user-app-minio-backup.yaml
kind: ClusterExternalSecret
spec:
  namespaceSelector:
    matchLabels:
      app-type: user-app
  externalSecretSpec:
    target:
      name: minio-backup-secret
    data:
      - secretKey: ACCESS_KEY_ID
        remoteRef:
          key: minio-backup-secret-source
          property: ACCESS_KEY_ID
      - secretKey: ACCESS_SECRET_KEY
        remoteRef:
          key: minio-backup-secret-source
          property: ACCESS_SECRET_KEY
```

動作確認: `ClusterExternalSecret` が `Ready: True` になり、`sample-app` namespace に `minio-backup-secret` が存在することを確認済み。

---

## 10. Backstage OIDC 認証障害の対処

### 問題

Backstage にログインしようとすると以下のエラーが返される:

```
connect ECONNREFUSED 10.43.115.161:443
```

### 原因

`10.43.115.161` は Envoy Gateway（`envoy-eg`）の ClusterIP。Backstage pod から TCP/HTTPS 接続を手動テストすると 200 OK が返るため、ネットワーク疎通自体は問題なし。

Envoy Gateway pod が約 43 分前に 10 回リスタートしており、Backstage も 38 分前にリスタートしていた。**Backstage 起動直後の OIDC discovery（`openid-client` ライブラリ）が Envoy 不安定期に失敗し、失敗結果がキャッシュされた**と判断した。

### 対処

```bash
kubectl rollout restart deployment/backstage -n backstage
```

再起動後、Envoy は安定稼働中だったため OIDC discovery が成功し、ログイン可能になった。

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-charts/charts/common-db/values.yaml` | backup 有効化・storageClassName・endpointURL・PodMonitor のデフォルト値変更 |
| `platform-gitops/backstage/templates/fullstack/template.yaml` | `owner` パラメータ追加 |
| `platform-gitops/backstage/templates/fullstack/gitops-skeleton/apps/${{ values.appName }}/app.yaml` | 新規作成 |
| `platform-gitops/backstage/templates/fullstack/gitops-skeleton/apps/${{ values.appName }}/${{ values.appName }}-backend/application.yaml` | repoURL・project・パス・ignoreDifferences 修正 |
| `platform-gitops/backstage/templates/fullstack/gitops-skeleton/apps/${{ values.appName }}/${{ values.appName }}-frontend/application.yaml` | repoURL・project・パス修正 |
| `platform-gitops/backstage/templates/fullstack/gitops-skeleton/apps/${{ values.appName }}/${{ values.appName }}-backend/values.yaml` | resources・probes・rollout 等追加 |
| `platform-gitops/backstage/templates/fullstack/gitops-skeleton/apps/${{ values.appName }}/${{ values.appName }}-frontend/values.yaml` | resources・probes・rollout 等追加 |
| `platform-gitops/backstage/templates/fullstack/backend-skeleton/aqua.yaml` | 新規作成 |
| `platform-gitops/backstage/templates/fullstack/frontend-skeleton/aqua.yaml` | 新規作成 |
| `platform-gitops/platform/applications/user-apps-infra.yaml` | Application → ApplicationSet に変換 |
| `platform-gitops/platform/user-apps-infra/Chart.yaml` | 新規作成（Helm chart 化） |
| `platform-gitops/platform/user-apps-infra/values.yaml` | 新規作成 |
| `platform-gitops/platform/user-apps-infra/templates/appproject.yaml` | 新規作成（テンプレート化） |
| `platform-gitops/platform/user-apps-infra/templates/namespace.yaml` | 新規作成（テンプレート化・app-type ラベル追加） |
| `platform-gitops/platform/secrets/config/user-app-minio-backup.yaml` | 新規作成（ClusterExternalSecret） |
| `apps-gitops/apps/sample/app.yaml` | 新規作成 |
| `apps-gitops/apps/sample/sample-backend/application.yaml` | project・パス修正 |
| `apps-gitops/apps/sample/sample-frontend/application.yaml` | project・パス修正 |
| `platform-infra/k3d/apps-root.yaml` | include パターン変更 |
| `platform-gitops/platform/secrets/sources/backstage-github-app-source.yaml` | githubPat 追加・App 認証情報（appId/clientId/clientSecret/privateKey）削除 |
| `platform-gitops/platform/secrets/config/backstage-secret.yaml` | GITHUB_APP_* マッピング削除・GITHUB_PAT マッピング追加 |
| `platform-gitops/platform/backstage/values.yaml` | integrations.github を apps: → token: に変更・GITHUB_APP_* env 削除・GITHUB_PAT env 追加 |

---

## 11. Backstage GitHub 統合を GitHub App から PAT に切り替え

### 背景

E2E 検証中、Scaffolder の「バックエンドリポジトリ作成」ステップで以下のエラーが発生した。

```
Failed to create the User repository okccl/myapp-backend,
Resource not accessible by integration
```

### 原因

`okccl` は GitHub の**パーソナルアカウント**（Organization ではない）。GitHub App の installation token（server-to-server）は `POST /user/repos`（個人アカウントのリポジトリ作成 API）を呼び出せない仕様となっており、Organization の `POST /orgs/{org}/repos` のみサポートされる。

以前の Scaffolder テストでは新規リポジトリを実際に作成する検証をしていなかったため、この制限が表面化していなかった。

### 対処

`integrations.github` を GitHub App から Fine-grained PAT に切り替え、GitHub App は OAuth ログイン専用（`auth.providers.github`）とした。

**PAT に付与した権限:**

| Permission | 用途 |
|---|---|
| Administration: R/W | リポジトリ作成 |
| Contents: R/W | コードプッシュ・カタログ読み取り |
| Workflows: R/W | `.github/workflows/` ファイルのプッシュ |
| Pull requests: R/W | apps-gitops への PR 作成 |
| Actions: R/W | teardown テンプレートの workflow dispatch |

ユーザーが GitHub で Fine-grained PAT を作成し、SOPS ファイルに `githubPat` を追加（App 認証情報は同時に削除）。

```yaml
# integrations.github の変更
# 変更前
apps:
  - appId: ${GITHUB_APP_ID}
    clientId: ${GITHUB_APP_CLIENT_ID}
    clientSecret: ${GITHUB_APP_CLIENT_SECRET}
    privateKey: ${GITHUB_APP_PRIVATE_KEY}

# 変更後
token: ${GITHUB_PAT}
```

> **将来的な移行:** `okccl` を GitHub Organization に変換した場合は GitHub App に戻すことができる。

### トラブル: ExternalSecret の spec.data 配列が SSA で更新できない

`external-secrets-config` Application に `ServerSideApply=true` が設定されており、`spec.data` 配列からのエントリ削除（GITHUB_APP_* の除去）が SSA では実行できなかった。ArgoCD の sync は "replaced" と報告するが spec は変わらないままの状態が続いた。

根本原因は2つ:
1. `platform-secrets-sources` が未 sync で、ソース Secret（`backstage-github-app-source`）に `githubPat` フィールドが存在していなかった
2. SSA では配列要素の削除が正常に機能しない（`Replace=true` アノテーションを追加したが、ソース Secret 未 sync が先決だった）

`platform-secrets-sources` を明示的に sync した後、ユーザーが ExternalSecret をクラスタ上で手動削除・ArgoCD 再作成することで解消。`Replace=true` アノテーションは解消後に削除した。

---

## 12. E2E 検証（Scaffolder 全フロー）

Backstage Scaffolder でアプリ名 `myapp`、DB 有効で全フローを実行し、以下をすべて確認した。

| 確認項目 | 結果 |
|---|---|
| `myapp-backend` / `myapp-frontend` リポジトリ作成 | ✅ |
| apps-gitops PR 作成・自動マージ | ✅ |
| `apps/myapp/` ディレクトリ構造（app.yaml + backend/frontend） | ✅ |
| ApplicationSet が `user-apps-infra-myapp` を生成 | ✅ |
| `AppProject: myapp` 自動生成 | ✅ |
| `Namespace: myapp`（`app-type: user-app` ラベル付き）自動生成 | ✅ |
| `minio-backup-secret` 自動配布（ClusterExternalSecret 経由） | ✅ |

---

## 13. fullstack テンプレート: 新規リポジトリへの GITOPS_APP_CLIENT_ID 自動設定

### 背景

E2E 検証で作成した `myapp-backend` の initial commit CI が失敗していた。

```
Error: The 'client-id' (or deprecated 'app-id') input must be set to a non-empty string.
```

### 原因

`build.yaml` スケルトンは `vars.GITOPS_APP_CLIENT_ID` と `secrets.GITOPS_APP_PRIVATE_KEY` を参照するが、Scaffolder がリポジトリを作成しても GitHub Actions の Variables / Secrets は自動設定されない。`sample-backend` / `sample-frontend` は手動で設定済みのため動作していたが、新規払い出しリポジトリには設定がなかった。

### 対処

`publish:github` アクションの `repoVariables` パラメータを使い、リポジトリ作成と同時に `GITOPS_APP_CLIENT_ID` を自動設定するよう `template.yaml` に追加した。

```yaml
- id: publish-backend
  action: publish:github
  input:
    repoVariables:
      GITOPS_APP_CLIENT_ID: Iv23liMGIykpOssjwg6a
```

`GITOPS_APP_PRIVATE_KEY`（秘密鍵）は GitHub API の暗号化処理が必要なため自動設定が困難であり、払い出し後に手動設定する運用とした。これは Organization-level secrets が使えないパーソナルアカウントの制約（Variable は平文 API で設定可能、Secret は libsodium 暗号化が必要）に起因する。

---

## 14. teardown ワークフローのバグ修正

### 背景

teardown テンプレートを E2E テストしたところ、複数のバグが判明した。

### 問題と原因・対処

**① 変数名誤り**

```yaml
# 誤り
client-id: ${{ vars.BACKSTAGE_APP_CLIENT_ID }}
private-key: ${{ secrets.BACKSTAGE_APP_PRIVATE_KEY }}

# 正しい（platform-gitops に登録されている名前）
client-id: ${{ vars.GITOPS_APP_CLIENT_ID }}
private-key: ${{ secrets.GITOPS_APP_PRIVATE_KEY }}
```

**② rm パスが実際のディレクトリ構造と不一致**

```bash
# 誤り（apps/<appName>-backend は存在しない）
rm -rf apps/$appName-backend
rm -rf apps/$appName-frontend

# 正しい（論理アプリ単位のディレクトリごと削除）
rm -rf apps/$appName
```

**③ GHCR パッケージ削除エンドポイント誤り**

```bash
# 誤り（Organization 向けエンドポイント）
/orgs/okccl/packages/container/...

# 正しい（パーソナルアカウント向け）
/user/packages/container/...
```

**④ リポジトリ削除が GitHub App token では実行不可**

```
HTTP 403: Resource not accessible by integration
```

`DELETE /repos/{owner}/{repo}` は `delete_repo` という専用 OAuth スコープが必要であり、GitHub App の `Administration: R/W` permission とは独立している。**アカウント種別（個人・Organization）に関わらず** GitHub App installation token ではこのスコープを取得できない。

PAT であれば `delete_repo` スコープを付与可能だが、teardown ワークフロー内の自動処理として不特定リポジトリを削除できる PAT を CI に持たせることはセキュリティリスクが高い。

削除ステップを手動対応に変更し、ワークフローのログに削除コマンドを出力する実装とした。

```bash
# teardown ワークフロー完了後に手動実行
gh repo delete okccl/<app-name>-backend --yes
gh repo delete okccl/<app-name>-frontend --yes
```

---

## 15. GITOPS_TOKEN 未使用 secret の削除

`platform-gitops` に `GITOPS_TOKEN` secret が登録されていたが、どのワークフローからも参照されていなかった。README.md の Phase 6「設計上の決定事項」に PAT を使う旨の記述が残っており、経緯を確認した。

Phase 6 時点ではクロスリポジトリ dispatch に PAT（`GITOPS_TOKEN`）を使用していたが、後から `okccl-gitops` GitHub App に移行した際にワークフローは更新されたものの secret と README が残ったままになっていた。PAT 自体はすでに削除済みであったため、secret エントリを platform-gitops から削除し、README.md の記述を GitHub App 移行済みの内容に更新した。

---

## 変更ファイル一覧（追記分）

| ファイル | 変更内容 |
|---|---|
| `platform-gitops/backstage/templates/fullstack/template.yaml` | publish-backend / publish-frontend に `repoVariables.GITOPS_APP_CLIENT_ID` 追加 |
| `platform-gitops/.github/workflows/teardown.yaml` | 変数名修正・rm パス修正・GHCR エンドポイント修正・リポジトリ削除を手動対応に変更 |
| `platform-gitops/README.md` | Phase 6 の GITOPS_TOKEN 記述を GitHub App 移行済みに更新 |

---

## 16. ghcr-pull-secret SecretSyncedError 修正

### 問題

`ghcr-pull-secret` ExternalSecret が SecretSyncedError になり、5 時間以上 Backstage の GHCR imagePullSecret が更新されていなかった。`external-secrets-config` → `root` と Degraded が連鎖していた。

### 原因

セッション内の作業（11: PAT 切り替え）で `backstage-secret.yaml` から `GITHUB_APP_PRIVATE_KEY` エントリを削除したことが原因。Backstage 本体は PAT に切り替わったため不要になったが、`ghcr-token` GithubAccessToken Generator が `backstage-secret.GITHUB_APP_PRIVATE_KEY` を参照し続けていた。`GITHUB_APP_PRIVATE_KEY` が Secret に存在しないため空値を RSA 鍵としてパースしようとして失敗していた。

```
error parsing RSA private key: invalid key: Key must be a PEM encoded PKCS1 or PKCS8 key
```

### 対処

`backstage-secret.yaml` に `GITHUB_APP_PRIVATE_KEY` → `privateKey`（from `backstage-github-app-source`）のマッピングを復元。

ただし、SSA の spec.data 配列への追加が機能しない問題（11 番と同様の既知問題）が再発した。`Replace=true` アノテーションを追加しても解消せず、ユーザーがクラスタ上の ExternalSecret を手動削除・再作成することで解消した。

---

## 17. sample-backend minio-backup-secret ExternalSecret 二重管理解消

### 問題

`sample-app` namespace に `minio-backup-secret` を target とする ExternalSecret が 2 つ存在し、`user-app-minio-backup-secret` が `target is owned by another ExternalSecret` エラーになっていた。

### 原因

`ClusterExternalSecret`（9 番で追加）による自動配布より前に、apps-gitops の `sample-backend/manifests/minio-backup-secret.yaml` として個別の ExternalSecret が作成されていた。ClusterExternalSecret 導入後は二重管理になっていた。gitops-skeleton（新アプリ用）は manifests に minio-backup-secret.yaml を含まない設計であり、sample アプリだけ旧方式のものが残っていた。

### 対処

`apps-gitops/apps/sample/sample-backend/manifests/minio-backup-secret.yaml` を削除した。ArgoCD の prune により cluster 側の ExternalSecret も削除され、ClusterExternalSecret 側が正常に所有できるようになった。

---

## 18. sample-backend OutOfSync 根本原因修正

### 問題

`sample-backend` が長期間 OutOfSync のまま（image `46e8147` → `ea4b65b` に更新されない）。`argocd app diff` は差分を検出するが、`argocd app sync` を実行しても image は変わらず ResourceVersion も変化しない。

### 原因

`application.yaml` の `ignoreDifferences` に以下の jsonPointer が設定されていた。

```yaml
- group: argoproj.io
  kind: Rollout
  jsonPointers:
    - /spec/template/spec/containers/0/ports/0/protocol
```

`RespectIgnoreDifferences=true` + `ServerSideApply=true` の組み合わせでは、ArgoCD は SSA パッチ送信前に ignoreDifferences で指定したフィールドをマニフェストから除去する。`/spec/template/spec/containers/0/ports/0/protocol` のように**配列インデックス（`0`）を含む jsonPointer** を除去する際に containers 配列全体が SSA パッチから落ちてしまい、`image` フィールドが含まれないパッチが送信されていた。API server は「変更なし」として 200 を返すため sync ログには error が出ず、`serverside-applied` として記録されるが実際には何も変わらない。

`protocol: TCP` の差分は Kubernetes が `containerPort` のデフォルト値として自動付与するためであり、chart 側で明示していなかったことで発生していた。

### 対処

1. `common-app` chart の `_helpers.tpl` に `protocol: TCP` を明示追加（version 0.3.0 → 0.3.1）
2. `helm dependency update` で `sample-backend` / `sample-frontend` の依存 tgz を更新
3. `application.yaml` の `ignoreDifferences` から `protocol` jsonPointer を削除

chart 修正後に protocol の差分が消え、SSA パッチに containers 配列が正しく含まれるようになり、image が `ea4b65b` に更新・Argo Rollouts カナリアが完走して Synced になった。

---

## 変更ファイル一覧（追記分 2）

| ファイル | 変更内容 |
|---|---|
| `platform-gitops/platform/secrets/config/backstage-secret.yaml` | GITHUB_APP_PRIVATE_KEY → privateKey マッピング復元 |
| `apps-gitops/apps/sample/sample-backend/manifests/minio-backup-secret.yaml` | 削除（ClusterExternalSecret に統一） |
| `apps-gitops/apps/sample/sample-backend/application.yaml` | ignoreDifferences から protocol jsonPointer を削除 |
| `platform-charts/charts/common-app/Chart.yaml` | version 0.3.0 → 0.3.1 |
| `platform-charts/charts/common-app/templates/_helpers.tpl` | containerPort に `protocol: TCP` を追加（2 箇所） |
| `platform-charts/charts/sample-backend/Chart.yaml` | common-app 依存を 0.3.1 に更新 |
| `platform-charts/charts/sample-backend/charts/common-app-*.tgz` | 0.3.0 削除・0.3.1 追加 |
| `platform-charts/charts/sample-frontend/Chart.yaml` | common-app 依存を 0.3.1 に更新 |
| `platform-charts/charts/sample-frontend/charts/common-app-*.tgz` | 0.3.0 削除・0.3.1 追加 |
