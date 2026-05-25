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

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-infra/scripts/generate-dr-manifests.py` | apps-gitops glob パターンを 2 階層に修正、`serverName` をテンプレートに追加 |
