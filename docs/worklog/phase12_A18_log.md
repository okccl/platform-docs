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
