# Phase 12-A20 作業ログ：DR 実測（シナリオ A）

## 冒頭サマリ

| # | 作業内容 |
|---|---|
| 1 | generate-dr-manifests.py のバグ修正（glob パターン・serverName 欠落） |
| 2 | シナリオ A（PVC 破損リストア）の RTO 実測（3 クラスター） |
| 3 | データ整合性の確認・修復 |
| 4 | ArgoCD auto-sync の一時停止と復元 |
| 5 | WAL アーカイブの正常化（MinIO クリア → 手動バックアップ） |
| 6 | DR 設計・Runbook の課題発見 |
| 7 | 手動ベースバックアップのトリガー（全 3 クラスター） |
| 8 | ArgoCD auto-sync 回避策の設計検討（GitOps-first 方式を採用） |
| 9 | GitOps-first DR 実装（common-db chart 拡張・generate-dr-manifests 全面改修・k3d/dr 廃止） |

---

## 1. generate-dr-manifests.py のバグ修正

### 背景

DR 実測前に `make generate-dr-manifests` を実行したところ、`keycloak-db` と `backstage-db` の 2 クラスターしか生成されなかった。`sample-backend-db` が누락。

### 問題①: apps-gitops の glob パターンが 1 階層分しか探索しない

#### 原因

apps-gitops のディレクトリ構造は `apps/<app-name>/<app-name>-backend/values.yaml`（2 階層）だが、スクリプトの glob が `apps/*/values.yaml`（1 階層）になっていた。

#### 対処

```python
# 修正前
for values_path in sorted(glob.glob(f"{APPS_GITOPS}/apps/*/values.yaml")):

# 修正後
for values_path in sorted(glob.glob(f"{APPS_GITOPS}/apps/*/*/values.yaml")):
```

コメント行も同様に修正。

### 問題②: recovery マニフェストに `serverName` が欠落

#### 原因

barman-cloud はバックアップを `{destinationPath}/{server_name}/base/` に格納する。`server_name` は barman がバックアップを取ったときのクラスター名（例: `keycloak-db`）。

recovery マニフェストの `externalClusters[].name` を `minio-backup` にしていたため、CNPG は `s3://cnpg-backup/keycloak-db/minio-backup/base/` を探すが、実際のデータは `s3://cnpg-backup/keycloak-db/keycloak-db/base/` にある。

これにより `no target backup found` エラーが発生。

#### 対処

テンプレートに `serverName: "{name}"` を追加し、バックアップ時のサーバー名（クラスター名）を明示する。

```yaml
# 修正後（externalClusters[].barmanObjectStore に追加）
serverName: "{name}"
```

---

## 2. シナリオ A：RTO 実測

### 背景

5/11 にクラスターを新規作成した際、DR リストア手順を実施しなかったため、各 CNPG クラスターは空の状態で MinIO には 5/11 以前のバックアップが残存。"Expected empty archive" エラーにより WAL アーカイブも失敗中。

MinIO のデータが正であり現在の DB データは捨ててよい状態のため、DR 実測のタイミングとして実施した。

### 実測手順（Runbook シナリオ A に準拠）

1. `make generate-dr-manifests` で recovery マニフェストを生成
2. 各クラスターを削除 → PVC 削除 → recovery マニフェスト apply
3. "Cluster in healthy state" までの時間を RTO として計測

### RTO 実測結果

| クラスター | apply 時刻（UTC） | 完了時刻（UTC） | RTO | 備考 |
|---|---|---|---|---|
| keycloak-db | 14:16:35 | 14:17:16 | **41 秒** | 1 インスタンス |
| backstage-db | 14:30:40 | 14:31:11 | **31 秒** | 1 インスタンス |
| sample-backend-db | 14:39:19 | 14:40:26 | **67 秒** | 2 インスタンス（primary + replica） |

### RPO（最終バックアップ時刻）

| クラスター | 最終バックアップ |
|---|---|
| keycloak-db | 2026-05-10 12:02 UTC |
| backstage-db | 2026-05-11 20:21 UTC |
| sample-backend-db | 2026-05-11 20:02 UTC |

---

## 3. ArgoCD auto-sync との競合（トラブルシュート）

### 問題

`backstage-db` / `sample-backend-db` の recovery apply 時に以下のエラーが発生。

```
The Cluster "backstage-db" is invalid: spec.bootstrap: Forbidden:
Only one bootstrap method can be specified at a time
```

#### 原因

ArgoCD の `selfHeal: true` により、クラスター削除を検知した直後に ArgoCD が `initdb` bootstrap のクラスターを再作成してしまう。`kubectl apply -f recovery.yaml` が既存の `initdb` クラスターへのパッチ適用として扱われ、CNPG webhook が `initdb` + `recovery` の共存を拒否。

`keycloak-db` が成功したのは、Pod の force-delete に時間がかかったため偶然 ArgoCD の reconcile サイクルに隙間ができたため。再現性はない。

#### 対処

recovery apply 前に対象 ArgoCD Application の auto-sync を一時停止する。

`backstage-db` は独立した Application のため `argocd app set backstage-db --sync-policy none` で対応。

`sample-backend` は多段 App-of-Apps 構造のため以下の手順が必要だった：

1. `apps-root` Application が `sample-backend` Application の `syncPolicy` を常に復元するため、まず `apps-root` の auto-sync を停止

```bash
kubectl patch application apps-root -n argocd --type=json \
  -p '[{"op":"remove","path":"/spec/syncPolicy/automated"}]'
```

2. その後 `sample-backend` の auto-sync を停止

```bash
kubectl patch application sample-backend -n argocd --type=json \
  -p '[{"op":"remove","path":"/spec/syncPolicy/automated"}]'
```

3. recovery apply 完了後、auto-sync を復元

```bash
kubectl patch application apps-root -n argocd --type=merge \
  -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
kubectl patch application backstage-db -n argocd --type=merge \
  -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
```

（`sample-backend` は `apps-root` の auto-sync 復元により自動的に元の設定に戻る）

#### Runbook への反映（必要）

DR 手順の各クラスター作業前に、対象 App の auto-sync を停止するステップを追加する。

---

## 4. GitOps クリーン状態復元後の WAL アーカイブ破綻（トラブルシュート）

### 問題

Runbook Step 4「DR クラスター削除 → ArgoCD が initdb で再作成 → CNPG は既存 PVC を検知して initdb をスキップ」を実施したところ、全クラスターで WAL アーカイブが再び失敗。

```
WAL archive check failed for server keycloak-db: Expected empty archive
```

### 原因

CNPG の WAL アーカイブ動作の違いによる設計上の問題。

| bootstrap 方式 | WAL アーカイブ開始時の動作 |
|---|---|
| `recovery` | バックアップの継続と認識 → "Expected empty archive" チェックをスキップ |
| `initdb`（既存 PVC あり・initdb スキップ） | 新規クラスターとして扱い → "Expected empty archive" チェックを実行 → 既存データがあるため失敗 |

つまり **Runbook Step 4（GitOps クリーン状態への復元）を実施すると、WAL アーカイブが必ず壊れる**。

### 対処

以下の 2 ステップで正常化した。

**MinIO バックアップデータをクリア**（"Expected empty archive" チェックをパスさせる）

```bash
docker exec minio-external mc rm --recursive --force local/cnpg-backup/keycloak-db/
docker exec minio-external mc rm --recursive --force local/cnpg-backup/backstage-db/
docker exec minio-external mc rm --recursive --force local/cnpg-backup/sample-backend-db/
```

WAL アーカイブ再開を確認後、各クラスターのベースバックアップを手動トリガーする。

### Runbook / 設計への反映（必要）

- **Runbook Step 4（任意）を削除または警告追記**：このステップは WAL アーカイブを破壊するため実施しない。recovery bootstrap のクラスターを `ignoreDifferences` に委ねてそのまま運用する。
- **DR 後の MinIO クリア + 手動バックアップ手順**を新たな Step として追加する。

---

## 5. データ整合性の確認・修復

### Backstage DB パスワード不一致

recovery 後に Backstage が DB に接続できなかった。

```
password authentication failed for user "app"
```

#### 原因

CNPG の `initdb` 時に `backstage-db-credentials` Secret に記載のパスワードで `app` ユーザーを作成する。バックアップ取得時（5/11）のパスワードと、クラスター再作成時（5/25 bootstrap）のパスワードが異なっていたため、リストア後の DB（5/11 時点のパスワード）と現在の Secret が不一致になった。

#### 対処

現在の Secret のパスワードで DB を更新した。

```bash
PASS=$(kubectl get secret backstage-db-credentials -n backstage \
  -o jsonpath='{.data.password}' | base64 -d)
kubectl exec backstage-db-1 -n backstage -c postgres -- \
  psql -U postgres -c "ALTER ROLE app WITH PASSWORD '${PASS}';"
```

#### 本対応（将来方針）

DR 手順に「recovery 後は対象クラスターの DB ユーザーパスワードを現在の Secret に合わせて更新する」ステップを追加する。または、SOPS 管理のパスワードを固定値にして再作成時もパスワードが変わらないようにする。

### Keycloak `jgroups_ping` テーブル欠落

recovery 後に Keycloak DB で以下のエラーが継続。

```
relation "jgroups_ping" does not exist
```

5/10 バックアップ時点ではこのテーブルが存在しなかったため（Keycloak が起動後に作成するテーブル）。Keycloak は正常稼働（OIDC エンドポイント 200 応答）しており、Keycloak 自身がエラーを処理してテーブルを自動作成する見込み。現時点では Keycloak の機能に影響なし。

### Keycloak Pod の再起動（catch-22 解消）

recovery 中に keycloak-db が一時停止したため、Keycloak Pod が "Keycloak cluster health check" DOWN となり Not Ready 状態に。StatefulSet rolling update が Not Ready Pod を待つため rollout restart が詰まった。

Pod を強制削除して StatefulSet が再作成することで解消。

```bash
kubectl delete pod keycloak-keycloakx-0 -n keycloak --force --grace-period=0
```

---

---

## 7. 手動ベースバックアップのトリガー

WAL アーカイブ正常化（セクション 5）後、MinIO のバックアップデータをクリアしたため新しいベースバックアップが存在しない状態だった。3 クラスター分の `Backup` リソースを apply してベースバックアップをトリガーし、全て completed になったことを確認した。

```bash
kubectl apply -f - <<'EOF'
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: keycloak-db-manual-20260526
  namespace: keycloak
spec:
  method: barmanObjectStore
  cluster:
    name: keycloak-db
---
# backstage-db / sample-backend-db も同様
EOF
```

3 クラスター全て `completed` に達したことを確認。

---

## 8. ArgoCD auto-sync 回避策の設計検討

セクション 3 の課題（ArgoCD との race condition）に対して、auto-sync 停止より良い回避策を検討した。

### 問題の構造

`spec.bootstrap` が CNPG webhook で immutable なため、bootstrap 変更には delete → recreate が必須。削除の瞬間に ArgoCD（selfHeal）が initdb で再作成するため、その後 recovery マニフェストを apply しても webhook に弾かれる。

### 検討した案

| 案 | 概要 | 採否 |
|---|---|---|
| 案1: `ignoreDifferences` のみ | ArgoCD が bootstrap 差分を無視するが、delete 直後の「ArgoCD が initdb で作成する」ことは防げない。レース条件は残る | 不採用（部分的解決のみ） |
| **案2: GitOps を recovery に書き換えてから delete** | gitops に recovery bootstrap を push → ArgoCD 自身が recovery でクラスターを作成 → レース条件ゼロ | **採用** |
| 案3: `resource.exclusions` で CNPG を一時除外 | ArgoCD ConfigMap の変更が必要で侵襲的 | 不採用 |

### 案2の採用理由

「ArgoCD を迂回するのではなく、ArgoCD を DR のエグゼキューターとして活用する」という考え方が GitOps の設計思想と一致している。また、全 ArgoCD Application にすでに `ignoreDifferences`（`spec.bootstrap` + `spec.externalClusters`）と `RespectIgnoreDifferences=true` が設定済みであったため、「DR 後に gitops を initdb に戻しても running クラスターは影響を受けない」という条件がすでに満たされていた。

---

## 9. GitOps-first DR 実装

### common-db Helm chart への recovery bootstrap 対応追加

`platform-charts/charts/common-db` の library chart（`_helpers.tpl`）に `db.recovery.enabled` 値を追加。`true` のとき `spec.bootstrap.recovery` と `spec.externalClusters` を生成し、`false`（デフォルト）のとき従来通り `spec.bootstrap.initdb` を生成する。

```yaml
# values.yaml に追加したデフォルト値
db:
  recovery:
    enabled: false
    source: minio-backup
    endpointURL: "http://host.k3d.internal:9000"
    bucketName: "cnpg-backup"
    secretName: "minio-backup-secret"
```

apps-gitops で `db.recovery.enabled: true` を設定すると Helm が recovery bootstrap のマニフェストを生成するため、platform-charts の変更だけで apps-gitops 側のすべてのアプリに対応できる。

### generate-dr-manifests.py の全面改修

旧動作（`k3d/dr/` に生成 → 手動 `kubectl apply`）から、gitops リポジトリを直接書き換える方式に変更。

| スキャン対象 | 旧動作 | 新動作 |
|---|---|---|
| `platform-gitops/.../cluster.yaml` | `k3d/dr/` に YAML 生成 | ファイルを直接上書き（`spec.bootstrap: recovery` + `externalClusters` を追加） |
| `apps-gitops/.../values.yaml` | `k3d/dr/` に YAML 生成 | `db.recovery.enabled: true` をテキスト挿入（他フィールドのフォーマットを変えない） |

apps-gitops の values.yaml はフォーマット保持のため `ruamel.yaml` ではなくテキスト挿入方式を採用（`ruamel.yaml` が環境にないため）。`db:` セクションの末尾（次の top-level キーの直前）に `db.recovery` ブロックを挿入する。

### k3d/dr/ ディレクトリの廃止

gitops ファイル自体が DR マニフェストになったため `k3d/dr/` は冗長。`.yaml` はもともと gitignore 済みで未追跡だったため、`README.md` と `.gitignore` のみ削除して廃止した。

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-infra/scripts/generate-dr-manifests.py` | ①apps-gitops glob パターンを 2 階層に修正、`serverName` 追加（バグ修正） ②GitOps 直接書き換え方式に全面改修（k3d/dr/ 廃止） |
| `platform-infra/k3d/dr/` | 廃止（README.md・.gitignore を削除） |
| `platform-charts/charts/common-db/templates/_helpers.tpl` | `db.recovery.enabled: true` で recovery bootstrap・externalClusters を生成する分岐を追加 |
| `platform-charts/charts/common-db/values.yaml` | `db.recovery` デフォルト値追加（`enabled: false`） |
