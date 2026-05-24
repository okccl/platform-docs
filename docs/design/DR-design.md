# DR 設計書

---

## 1. 設計方針

### 1.1 何を守るか

本基盤における DR の主な保護対象は PostgreSQL データ（Keycloak・Backstage・アプリ DB）。
その他のコンポーネント（ArgoCD・Prometheus 等）はすべて GitOps から再構築可能なため、DR 対象外。

### 1.2 障害シナリオの分類

| シナリオ | 内容 | 対応方針 |
|---|---|---|
| **Pod / PV 障害** | CNPG Pod 異常・PVC 破損 | CNPG の自己修復 + WAL リプレイ |
| **クラスター全損** | `k3d cluster delete` 等でクラスター喪失 | MinIO（ローカル）からリストア |
| **WSL 全損** | WSL ディストリビューション全体の消失 | GCS（クラウド）からリストア |

### 1.3 RTO / RPO 目標

| シナリオ | RPO | RTO |
|---|---|---|
| Pod / PV 障害 | WAL ラグ分（数十秒以内） | 数分 |
| クラスター全損 | 最終 ScheduledBackup 時刻（21:00） | 1 時間以内（未計測） |
| WSL 全損 | 最終 GCS 同期時刻（翌日 00:00） | 数時間（未計測） |

> RTO/RPO は現時点では目標値。DR 手順書完成後に実測で検証する。

---

## 2. バックアップ設計

### 2.1 PostgreSQL（CNPG）

CNPG の barman-cloud を使い、MinIO（WSL ローカル）に継続的バックアップを取得する。

| 種別 | 内容 | スケジュール |
|---|---|---|
| WAL アーカイブ | PostgreSQL の変更ログをリアルタイムで転送 | 随時（CNPG が自動管理） |
| ScheduledBackup | ベースバックアップ（フルダンプ相当）| 毎日 21:00 |
| 保持期間 | `retentionPolicy: 7d` | 7 日分 |

```
CNPG Pod
  → barman-cloud-wal-archive（WAL）  ─→ MinIO: s3://cnpg-backup/<cluster-name>/wals/
  → barman-cloud-backup（ベース）    ─→ MinIO: s3://cnpg-backup/<cluster-name>/data/
```

**接続設定**:
- エンドポイント: `http://host.k3d.internal:9000`（k3d コンテナから WSL ホストへ）
- 認証情報: `minio-backup-secret`（ESO 経由で各 namespace に配布）
- `host.k3d.internal` は k3d 起動時に CoreDNS NodeHosts へ動的登録（`make cluster-start` に組み込み済み）

### 2.2 クラウドへのオフサイトバックアップ（MinIO → GCS）

MinIO 上のバックアップを GCS（us-central1）へ rclone で同期し、WSL 全損に備える。

| 種別 | 内容 |
|---|---|
| 同期スクリプト | `k3d/scripts/backup-to-gcs.sh`（`rclone copy` で差分同期） |
| スケジュール | 毎日 23:00（WSL cron）。CNPG ScheduledBackup（21:00）の 2 時間後 |
| 保持期間 | GCS Object Lifecycle で 30 日（MinIO の 7 日と独立して管理） |
| 失敗通知 | Discord Webhook（失敗時のみ通知） |
| 認証情報 | GCP SA キー（SOPS 暗号化）`platform-infra/secrets/gcp-backup-sa-key.enc.json` |

**実装済み**（`make backup-to-gcs`）。

---

## 3. リストア設計

### 3.1 CNPG リストア設計方針

#### bootstrap セクションの性質

CNPG の `spec.bootstrap` はクラスター初回作成時（PVC が空の場合）のみ実行される一度きりの初期化イベントであり、PVC にデータが存在すれば以後は無視される。また、CNPG の validating webhook により **bootstrap のメソッド変更はイミュータブル**（`initdb` ↔ `recovery` の変更は実行時に拒否される）。

この性質から、bootstrap セクションは「GitOps の定常管理対象」ではなく「初期化イベントの記述」として扱う。

#### GitOps 定常状態と DR 手順の分離

| | 役割 | bootstrap |
|---|---|---|
| **GitOps マニフェスト** | クラスターの定常状態を宣言 | `initdb`（PVC が空の場合のデフォルト） |
| **DR マニフェスト** | 障害復旧時のみ使用（ArgoCD 管理外） | `bootstrap.recovery` + `externalClusters` |

GitOps と DR 手順を同一マニフェストで解決しようとしないことが重要。ArgoCD はクラスターの定常運用を管理し、DR 手順は別途スクリプト・Runbook として管理する。

#### ArgoCD ignoreDifferences の設定

DR 実行後、ArgoCD はクラスターの `spec.bootstrap` が `recovery` になっていることを検知し `initdb` に戻そうとするが、CNPG webhook がこれを拒否する。これを防ぐため、対象の ArgoCD Application に以下の設定を追加する（**実装済み**）。

```yaml
spec:
  ignoreDifferences:
    - group: postgresql.cnpg.io
      kind: Cluster
      jsonPointers:
        - /spec/bootstrap
        - /spec/externalClusters
```

`bootstrap` は一度きりの初期化イベントであり GitOps の管轄外という設計思想を ArgoCD に明示するものであり、これは本番標準のパターン。

対象 Application（keycloak-db / backstage-db / sample-backend）に設定済み。`RespectIgnoreDifferences=true` を syncOptions に合わせて設定することで、sync 実行時にも該当フィールドへのパッチを抑止している。

#### PVC Retain ポリシー

DR クラスター（`recovery`）でリストア完了後、DR クラスターを削除して GitOps に戻す際、PVC を残しておくことでデータを保全する。ArgoCD が `initdb` マニフェストでクラスターを再作成した際、CNPG は PVC 上の既存データを検知し bootstrap をスキップする。

**実装済み**:
- `local-path-retain` StorageClass（`reclaimPolicy: Retain`）を GitOps で管理（wave 0 で適用）
- 全 CNPG クラスターの `storage.storageClass: local-path-retain` を設定済み（新規 PVC から自動適用）
- 既存 PV は `kubectl patch` で Retain に変更済み

#### DR マニフェスト生成（実装済み）

DR マニフェストは静的ファイルとして管理せず、`make generate-dr-manifests` で GitOps ソースから動的生成する。これにより、クラスター設定変更時のドリフトを防ぐ。

```bash
# DR 手順の最初のステップとして実行（クラスターが落ちていても動作する）
cd ~/platform-infra
make generate-dr-manifests
# → k3d/dr/<cluster-name>-recovery.yaml を生成
```

生成スクリプト（`k3d/scripts/generate-dr-manifests.py`）は以下を自動スキャンする:
- `platform-gitops/platform/**/*.yaml` — `kind: Cluster` + `barmanObjectStore` を持つもの
- `apps-gitops/apps/*/values.yaml` — `db.backup.enabled: true` のもの

新しいクラスターが追加されても手動メンテナンス不要。生成ファイルは gitignore 済み（`k3d/dr/*.yaml`）。

WSL 全損時は生成後に `endpointURL` を GCS エンドポイントに書き換えてから apply する（Runbook 参照）。

### 3.2 クラスター内障害（Pod / PV 障害）からの復旧

#### CNPG の自動修復

CNPG はほとんどの Pod 障害を自己修復する。手動介入が必要になるのは PVC 破損など CNPG の管轄外の物理障害のみ。

| 障害種別 | CNPG の動作 | 手動介入 |
|---|---|---|
| Pod クラッシュ（プライマリ） | レプリカが自動昇格、新レプリカを補充 | 不要 |
| Pod クラッシュ（レプリカ） | 自動再起動・再同期 | 不要 |
| WAL アーカイブ失敗 | 自動リトライ | アラート確認のみ |
| PVC 破損 | 自動修復不可 | 以下の手順に従う |

Discord 通知（`CNPGWalArchivingFailing` / `CNPGLastBackupFailed` アラート）で障害を検知したら、ArgoCD GUI の CNPG Cluster リソースの `status.conditions` でフェーズを確認する。

#### PVC 破損時の手順

```bash
# 1. DR マニフェストを生成
cd ~/platform-infra/k3d && make generate-dr-manifests

# 2. 対象クラスターを削除
# PVC は Retain ポリシーにより削除されない（ただし破損 PVC は次のステップで削除する）
kubectl delete cluster <name> -n <ns>

# 3. 破損した PVC を削除（recovery クラスターが同名で新規作成する）
kubectl delete pvc -n <ns> -l cnpg.io/cluster=<name>

# 4. リカバリーマニフェストを apply
# CNPG の validating webhook により initdb→recovery の変更はイミュータブルのため、
# クラスター削除後・ArgoCD が再作成する前に apply する。
# ArgoCD は ignoreDifferences により /spec/bootstrap の差分を無視し Synced を維持する。
kubectl apply -f k3d/dr/<name>-recovery.yaml

# 5. リストア完了を待機
kubectl get cluster <name> -n <ns> -w
# → "Cluster in healthy state" フェーズになれば完了

# 6. データ整合性を確認（アプリケーション疎通テスト）

# 7. （任意）GitOps クリーン状態に戻す
# DR クラスターを削除すると ArgoCD が initdb で再作成する。
# CNPG は PVC 上の既存データを検知し initdb をスキップするためデータは保全される。
kubectl delete cluster <name> -n <ns>
```

### 3.3 クラスター全損からの復旧（k3d 再作成）

**前提条件**: `minio-external` Docker コンテナが稼働しており MinIO にバックアップデータが存在すること（MinIO は k3d 外の Docker コンテナのため k3d 再作成の影響を受けない）。

```bash
# 1. クラスターをフルブートストラップ（約 16 分）
# ArgoCD が全 wave を完了するまで待機（GUI: https://argocd.platform.local）
# → CNPG クラスターは initdb で作成されるが、k3d 再作成のため PVC は空
cd ~/platform-infra/k3d && make bootstrap

# 2. DR マニフェストを生成
make generate-dr-manifests

# 3. 対象クラスターごとにリストアを実行
# ※ keycloak-db / backstage-db と、db.backup.enabled: true なユーザーアプリが対象
CLUSTER_NAME=keycloak-db
NAMESPACE=keycloak

# 3-1. initdb クラスターを削除（PVC は Retain で残存するが空のため次のステップで削除）
kubectl delete cluster ${CLUSTER_NAME} -n ${NAMESPACE}

# 3-2. 空の PVC を削除（recovery クラスターが同名で新規 PVC を作成するため）
kubectl delete pvc -n ${NAMESPACE} -l cnpg.io/cluster=${CLUSTER_NAME}

# 3-3. リカバリーマニフェストを apply
# ArgoCD が再作成する前に素早く apply すること（通常 30 秒〜数分の猶予がある）。
# ignoreDifferences により ArgoCD は /spec/bootstrap の差分を無視し Synced を維持する。
kubectl apply -f k3d/dr/${CLUSTER_NAME}-recovery.yaml

# 3-4. リストア完了を待機
kubectl get cluster ${CLUSTER_NAME} -n ${NAMESPACE} -w
# → "Cluster in healthy state" になれば完了

# backstage-db も同様に実行（CLUSTER_NAME=backstage-db NAMESPACE=backstage で 3-1〜3-4 を繰り返す）

# 4. 全クラスター Healthy 確認後、データ整合性を検証
# - https://keycloak.platform.local  : ログイン可能か
# - https://backstage.platform.local : ログイン・カタログ表示が正常か
# - ユーザーアプリ（存在する場合）  : API 疎通確認

# 5. （任意）GitOps クリーン状態に戻す
# DR クラスターを削除すると ArgoCD が initdb で再作成する。
# CNPG は PVC 上のデータを検知し initdb をスキップするため、クリーンな GitOps 状態に戻せる。
kubectl delete cluster ${CLUSTER_NAME} -n ${NAMESPACE}
```

### 3.4 WSL 全損からの復旧

WSL 全損時は MinIO（Docker コンテナ）も失われる。GCS バックアップから MinIO を復元した後、3.3 の手順を実行する。

**前提条件**:
- 全リポジトリを再クローン済み（`platform-infra` / `platform-gitops` / `apps-gitops` 等）
- `~/.config/sops/age/keys.txt`（Age 秘密鍵）を別途バックアップから復元済み
- `aqua install` でツールを再インストール済み
- `minio-external` Docker コンテナを再作成済み（空バケット）

#### GCS から MinIO へバックアップを復元

`backup-to-gcs.sh` の逆方向。GCS から MinIO へ rclone でコピーする。

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

# rclone 設定を生成（backup-to-gcs.sh の逆向き）
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

MinIO 上にバックアップデータが復元できたことを確認した後、**3.3 の手順**を実行する。

> **RPO の注意**: MinIO に復元されるデータは最終 GCS 同期時刻（前日 23:00）時点のスナップショット。WAL は含まれないため最終 GCS 同期以降の変更は復元不可。GCS の Object Lifecycle（30 日保持）の範囲内であれば古い時点のバックアップへの復元も可能。

---

## 4. 実装状況

| 項目 | 状態 | 参照 |
|---|---|---|
| `ignoreDifferences` 設定 | **完了** | 3.1 節 |
| PVC Retain ポリシー設定 | **完了**（`local-path-retain` SC + 既存 PV パッチ） | 3.1 節 |
| DR マニフェスト生成スクリプト | **完了**（`make generate-dr-manifests`） | 3.1 節 |
| クラウドバックアップ実装 | **完了**（`make backup-to-gcs`・毎日 23:00 cron） | 2.2 節 |
| DR 手順書（Runbook）作成 | **完了** | 3.2〜3.4 節 |
| RTO/RPO 実測 | 未実装（DR 手順確立後に計測） | 1.3 節 |
