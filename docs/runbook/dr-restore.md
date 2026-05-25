# DR リストア手順（CNPG）

## シナリオ判定

| 状況 | 実行するシナリオ |
|---|---|
| k3d は動いているが特定の CNPG クラスターが壊れた / PVC が破損した | **A: PVC 破損リストア** |
| k3d クラスターが消えた / 再作成が必要な状態 | **B: クラスター全損リストア** |
| WSL を再インストールした / WSL ごと消えた | **C: WSL 全損リストア**（→ B に続く） |

---

## シナリオ A: PVC 破損リストア

**前提**: k3d クラスター・ArgoCD は稼働中。特定の CNPG クラスターの PVC が破損している。

### 1. GitOps を recovery bootstrap に書き換える

```bash
cd ~/platform-infra && make generate-dr-manifests
```

スクリプトが以下を自動で書き換える:
- `~/platform-gitops/platform/.../cluster.yaml` → `spec.bootstrap: recovery` に変更
- `~/apps-gitops/apps/.../values.yaml` → `db.recovery.enabled: true` を追加

変更内容を確認して commit + push する:

```bash
cd ~/platform-gitops
git diff                  # 変更内容を確認
git add -A && git commit -m "dr: activate recovery bootstrap for ${CLUSTER_NAME}" && git push

cd ~/apps-gitops
git diff                  # （apps-gitops クラスターが対象の場合）
git add -A && git commit -m "dr: activate recovery bootstrap for ${CLUSTER_NAME}" && git push
```

### 2. 対象クラスターと破損 PVC を削除する

ArgoCD が gitops の recovery manifest を検知してクラスターを自動再作成するが、
CNPG webhook により既存クラスターへの bootstrap 変更は拒否される（これは想定内）。
クラスターを削除すると ArgoCD が recovery bootstrap で即座に再作成する。

```bash
CLUSTER_NAME=keycloak-db   # 対象クラスター名に変更
NAMESPACE=keycloak          # 対象 namespace に変更

# クラスターを削除（ArgoCD が recovery bootstrap で自動再作成する）
kubectl delete cluster ${CLUSTER_NAME} -n ${NAMESPACE}

# 破損した PVC を削除
kubectl delete pvc -n ${NAMESPACE} -l cnpg.io/cluster=${CLUSTER_NAME}
```

> **ポイント**: ArgoCD の auto-sync を止める必要はない。ArgoCD 自身が recovery bootstrap でクラスターを作成するため、race condition が発生しない。

### 3. リストア完了を待機

```bash
kubectl get cluster ${CLUSTER_NAME} -n ${NAMESPACE} -w
# "Cluster in healthy state" になれば完了
```

アプリケーション疎通テストでデータ整合性を確認する。

### 4. データ整合性チェック

recovery bootstrap は initdb をスキップするため、バックアップ取得後に変更があったフィールドは手動で同期が必要。

**DB ユーザーパスワードの確認**（バックアップ取得後に Secret が変更された場合）:

```bash
# 例: backstage-db
PASS=$(kubectl get secret backstage-db-credentials -n backstage \
  -o jsonpath='{.data.password}' | base64 -d)
kubectl exec backstage-db-1 -n backstage -c postgres -- \
  psql -U postgres -c "ALTER ROLE app WITH PASSWORD '${PASS}';"
```

### 5. DR 完了確認

gitops は recovery bootstrap のまま維持する（`git checkout` は不要）。

`spec.bootstrap` は一度きりの初期化イベントであり、running クラスターの動作には影響しない。gitops が recovery を宣言したまま維持されることで、以後のクラスター再作成も MinIO から自動リストアされる。WAL アーカイブは recovery bootstrap のクラスターが "Expected empty archive" チェックをスキップするため、正常に継続される。

---

## シナリオ B: クラスター全損リストア

**前提**: k3d クラスターが消えた / 再作成後。`minio-external` コンテナはデータを保持している。

### 1. フルブートストラップ

```bash
cd ~/platform-infra/k3d
make bootstrap
```

全 wave 完了まで約 16 分。ArgoCD GUI（https://argocd.platform.local）で全 App が Healthy になったことを確認する。

> CNPG クラスターは `initdb` で作成されるが、k3d 再作成のため PVC は空。次のステップでリストアする。

### 2. GitOps を recovery bootstrap に書き換える

```bash
cd ~/platform-infra && make generate-dr-manifests
```

各 gitops リポジトリで commit + push する（シナリオ A の Step 1 と同様）。

### 3. 各クラスターをリストア

`keycloak-db` / `backstage-db` と、`db.backup.enabled: true` なユーザーアプリのクラスターが対象。
各クラスターに対して以下を実行する。

```bash
CLUSTER_NAME=keycloak-db
NAMESPACE=keycloak

# initdb クラスターを削除（ArgoCD が recovery bootstrap で自動再作成する）
kubectl delete cluster ${CLUSTER_NAME} -n ${NAMESPACE}
# PVC は k3d 再作成で消えているため削除不要

# リストア完了を待機
kubectl get cluster ${CLUSTER_NAME} -n ${NAMESPACE} -w
# "Cluster in healthy state" になれば完了
```

`backstage-db`（`CLUSTER_NAME=backstage-db NAMESPACE=backstage`）、
ユーザーアプリのクラスターも同様に実行する。

### 4. データ整合性を確認

- https://keycloak.platform.local — ログイン可能か
- https://backstage.platform.local — ログイン・カタログ表示が正常か
- ユーザーアプリ（存在する場合）— API 疎通確認

DB ユーザーパスワードの確認も実施すること（シナリオ A Step 4 参照）。

### 5. DR 完了確認

シナリオ A Step 5 と同様。gitops は recovery bootstrap のまま維持する。

---

## シナリオ C: WSL 全損リストア

**前提**: WSL の再インストール・環境再構築後。GCS バケット `ccl-platform-cnpg-backup` にバックアップが存在する。

### 1. 環境の前提を確認

以下がすべて完了していることを確認する。

- [ ] 全リポジトリを再クローン済み（`platform-infra` / `platform-gitops` / `apps-gitops` 等）
- [ ] `~/.config/sops/age/keys.txt`（Age 秘密鍵）を復元済み
- [ ] `cd ~/platform-infra && aqua install` でツールを再インストール済み
- [ ] `minio-external` Docker コンテナを再作成済み（空バケット）

### 2. GCS から MinIO へバックアップを復元

`backup-to-gcs.sh` の逆方向。SOPS から認証情報を復号し rclone で GCS → MinIO へコピーする。

```bash
SOPS_AGE_KEY_FILE="${HOME}/.config/sops/age/keys.txt"
WORK_DIR=$(mktemp -d)
trap "rm -rf ${WORK_DIR}" EXIT

# MinIO 認証情報を SOPS から復号
MINIO_SECRET_YAML=$(SOPS_AGE_KEY_FILE="${SOPS_AGE_KEY_FILE}" sops decrypt \
  "${HOME}/platform-gitops/platform/secrets/sources/minio-backup-secret-source.yaml")
ACCESS_KEY_ID=$(echo "${MINIO_SECRET_YAML}" | python3 -c \
  "import sys, yaml; d=yaml.safe_load(sys.stdin); print(d['stringData']['ACCESS_KEY_ID'])")
ACCESS_SECRET_KEY=$(echo "${MINIO_SECRET_YAML}" | python3 -c \
  "import sys, yaml; d=yaml.safe_load(sys.stdin); print(d['stringData']['ACCESS_SECRET_KEY'])")

# GCP SA キーを SOPS から復号
SOPS_AGE_KEY_FILE="${SOPS_AGE_KEY_FILE}" sops decrypt \
  "${HOME}/platform-infra/secrets/gcp-backup-sa-key.enc.json" > "${WORK_DIR}/gcp-sa-key.json"

# rclone 設定を生成
cat > "${WORK_DIR}/rclone.conf" <<EOF
[gcs]
type = google cloud storage
service_account_file = ${WORK_DIR}/gcp-sa-key.json
bucket_policy_only = true

[minio]
type = s3
provider = Minio
endpoint = http://localhost:9000
access_key_id = ${ACCESS_KEY_ID}
secret_access_key = ${ACCESS_SECRET_KEY}
no_check_bucket = true
EOF

# GCS から MinIO へコピー
rclone copy \
  --config "${WORK_DIR}/rclone.conf" \
  --transfers 4 \
  --progress \
  "gcs:ccl-platform-cnpg-backup" \
  "minio:cnpg-backup"
```

> **RPO**: 復元されるデータは最終 GCS 同期時刻（前日 23:00）のスナップショット。WAL は含まれないため、それ以降の変更は復元不可。GCS Object Lifecycle（30 日保持）の範囲内であれば古い時点のバックアップへの復元も可能。

### 3. シナリオ B を実行

MinIO 上のデータが復元できたことを確認した後、**シナリオ B: クラスター全損リストア** を実行する。

---

## トラブルシュート

### WAL アーカイブが "Expected empty archive" で失敗する

クラスターが `initdb` bootstrap で再作成され、MinIO に既存のバックアップデータがある場合に発生する。
DR 後の GitOps 復元ステップでクラスターを削除・再作成した場合に起きる。

**対処**: MinIO バックアップデータをクリアしてから手動でベースバックアップを取得する。

```bash
# MinIO データをクリア（警告: バックアップが消える）
docker exec minio-external mc rm --recursive --force local/cnpg-backup/${CLUSTER_NAME}/

# WAL アーカイブ再開を確認後、手動バックアップをトリガー
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: ${CLUSTER_NAME}-manual-$(date +%Y%m%d)
  namespace: ${NAMESPACE}
spec:
  method: barmanObjectStore
  cluster:
    name: ${CLUSTER_NAME}
EOF
```

### ArgoCD の sync が OutOfSync のまま（CNPG Cluster）

`ignoreDifferences` が `spec.bootstrap` / `spec.externalClusters` に設定されているため、これらのフィールドの差分は OutOfSync としてカウントされない。もし OutOfSync が表示される場合は `spec.bootstrap` 以外のフィールドに差分があるため、`argocd app diff <app-name>` で内容を確認すること。
