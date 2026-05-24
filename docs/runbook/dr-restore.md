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

### 1. DR マニフェストを生成

```bash
cd ~/platform-infra && make generate-dr-manifests
# → k3d/dr/<cluster-name>-recovery.yaml が生成される
```

### 2. 対象クラスターを削除して recovery マニフェストを apply

CNPG の validating webhook により `spec.bootstrap` はイミュータブルなため、クラスターを削除してから recovery マニフェストを apply する。ArgoCD が自動再作成する前（通常 30 秒〜数分の猶予）に apply すること。

```bash
CLUSTER_NAME=keycloak-db   # 対象クラスター名に変更
NAMESPACE=keycloak          # 対象 namespace に変更

# クラスターを削除（PVC は Retain ポリシーで残存）
kubectl delete cluster ${CLUSTER_NAME} -n ${NAMESPACE}

# 破損した PVC を削除
kubectl delete pvc -n ${NAMESPACE} -l cnpg.io/cluster=${CLUSTER_NAME}

# recovery マニフェストを apply
kubectl apply -f ~/platform-infra/k3d/dr/${CLUSTER_NAME}-recovery.yaml
```

### 3. リストア完了を待機

```bash
kubectl get cluster ${CLUSTER_NAME} -n ${NAMESPACE} -w
# "Cluster in healthy state" になれば完了
```

アプリケーション疎通テストでデータ整合性を確認する。

### 4. （任意）GitOps クリーン状態に戻す

DR クラスターを削除すると ArgoCD が `initdb` で再作成する。CNPG は PVC 上の既存データを検知し initdb をスキップするため、データは保全されたまま定常状態に戻る。

```bash
kubectl delete cluster ${CLUSTER_NAME} -n ${NAMESPACE}
```

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

### 2. DR マニフェストを生成

```bash
cd ~/platform-infra && make generate-dr-manifests
```

### 3. 各クラスターをリストア

`keycloak-db` / `backstage-db` と、`db.backup.enabled: true` なユーザーアプリのクラスターが対象。各クラスターに対して以下を実行する。

```bash
CLUSTER_NAME=keycloak-db
NAMESPACE=keycloak

# initdb クラスターを削除（PVC は Retain で残存するが空のため削除）
kubectl delete cluster ${CLUSTER_NAME} -n ${NAMESPACE}
kubectl delete pvc -n ${NAMESPACE} -l cnpg.io/cluster=${CLUSTER_NAME}

# recovery マニフェストを apply
kubectl apply -f ~/platform-infra/k3d/dr/${CLUSTER_NAME}-recovery.yaml

# リストア完了を待機
kubectl get cluster ${CLUSTER_NAME} -n ${NAMESPACE} -w
# "Cluster in healthy state" になれば完了
```

`backstage-db` も同様に実行（`CLUSTER_NAME=backstage-db NAMESPACE=backstage`）。

### 4. データ整合性を確認

- https://keycloak.platform.local — ログイン可能か
- https://backstage.platform.local — ログイン・カタログ表示が正常か
- ユーザーアプリ（存在する場合）— API 疎通確認

### 5. （任意）GitOps クリーン状態に戻す

各 DR クラスターを削除すると ArgoCD が `initdb` で再作成する。CNPG は PVC 上のデータを検知し initdb をスキップするため、クリーンな GitOps 状態に戻る。

```bash
kubectl delete cluster ${CLUSTER_NAME} -n ${NAMESPACE}
```

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
