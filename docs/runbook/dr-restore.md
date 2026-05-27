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
- `~/platform-gitops/platform/.../cluster.yaml` → `spec.bootstrap: recovery` に変更、`spec.backup.barmanObjectStore.serverName` を `{cluster_name}-{YYYYMMDD}` に変更
- `~/apps-gitops/apps/.../values.yaml` → `db.recovery.enabled: true` と `db.backup.serverName`（新規書き込み先）・`db.recovery.serverName`（旧バックアップ参照先）を追加

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

`spec.bootstrap` は一度きりの初期化イベントであり、running クラスターの動作には影響しない。gitops が recovery を宣言したまま維持されることで、以後のクラスター再作成も MinIO から自動リストアされる。WAL アーカイブは `generate-dr-manifests` が `backup.serverName` を空の新規パスに向けているため、"Expected empty archive" エラーなく正常に継続される。

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

スクリプトが以下を書き換える:
- `spec.bootstrap` → `recovery` に変更
- `spec.externalClusters` → 旧バックアップの読み込み元を追加（`serverName` は現行の書き込み先パスを自動参照）
- `spec.backup.barmanObjectStore.serverName` → `{cluster_name}-{YYYYMMDD}` に変更（書き込み先を空の新規パスに向ける）

各 gitops リポジトリで commit + push する（シナリオ A の Step 1 と同様）。

### 3. 各クラスターをリストア

> **重要**: ArgoCD の `ignoreDifferences` により `spec.bootstrap` / `spec.externalClusters` は既存クラスターには適用されない。必ず **全対象クラスターを delete して再作成させる**こと。

`make generate-dr-manifests` の出力末尾に、対象クラスターの `kubectl delete` コマンドが自動生成されている。そのコマンドをそのまま実行すること。

```bash
# make generate-dr-manifests の出力に表示されたコマンドを実行する（例）:
kubectl delete cluster keycloak-db -n keycloak
kubectl delete cluster backstage-db -n backstage
kubectl delete cluster sample-backend-db -n sample-app
# ※ PVC は k3d 再作成で消えているため削除不要

# 全クラスターのリストア完了を待機
kubectl get cluster -A -w
# 全クラスターが "Cluster in healthy state" になれば完了
```

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

### 1. SSH 秘密鍵の復元

```bash
mkdir -p ~/.ssh
# パスワードマネージャーから秘密鍵の内容をコピーして貼り付け
cat > ~/.ssh/id_ed25519 << 'EOF'
（ここに秘密鍵の内容を貼り付け）
EOF
chmod 600 ~/.ssh/id_ed25519

# GitHub への接続確認
ssh -T git@github.com
# "Hi okccl! You've successfully authenticated..." と表示されれば OK
```

### 2. 環境構築（bootstrap.sh）

WSL のデフォルト環境には git が入っていないため、まず手動でインストールする。

```bash
sudo apt-get update && sudo apt-get install -y git
git clone git@github.com:okccl/platform-infra.git ~/platform-infra
```

`bootstrap.sh` を実行する前に `~/platform-infra/scripts/repos.txt` を開き、クローン対象リポジトリの追加・削除・名前変更がないか確認する。

```bash
bash ~/platform-infra/scripts/bootstrap.sh
source ~/.bashrc
```

bootstrap.sh が自動で実施すること:
- apt パッケージ・Homebrew・aqua のインストール
- `aqua install`（kubectl / helm / k3d / docker 等）
- Docker daemon の設定・起動
- `.bashrc` 追記
- 全リポジトリのクローン（既存はスキップ）
- Age 秘密鍵の配置（対話式プロンプト）
- `minio-external` コンテナの作成・バケット初期化（既存はスキップ）

### 3. GCS から MinIO へバックアップを復元

```bash
cd ~/platform-infra && make restore-from-gcs
```

> **RPO**: 復元されるデータは最終 GCS 同期時刻（前日 23:00）のスナップショット。WAL は含まれないため、それ以降の変更は復元不可。GCS Object Lifecycle（30 日保持）の範囲内であれば古い時点のバックアップへの復元も可能。

### 4. シナリオ B を実行

MinIO 上のデータが復元できたことを確認した後、**シナリオ B: クラスター全損リストア** を実行する。

---

## トラブルシュート

### WAL アーカイブが "Expected empty archive" で失敗する

**正規の DR 手順（`make generate-dr-manifests` → commit + push → `kubectl delete cluster`）を踏んだ場合、このエラーは発生しない。** スクリプトが `spec.backup.barmanObjectStore.serverName` を日付サフィックス付きの新規名に変更し、書き込み先を空パスに向けるためである。

このエラーが発生するのは以下のケース：
- `generate-dr-manifests` を使わずに CNPG Cluster を手動作成し、`backup.serverName` が既存 WAL のあるパスと一致している場合
- 初回 `make bootstrap` 直後（MinIO が空でなく、かつ `initdb` で起動した場合）

**対処（初回 bootstrap 後の MinIO データ不整合）**:

MinIO の既存バックアップデータをクリアし、新しいベースバックアップを取り直す。これはバックアップデータを「現クラスター」に揃える操作であり、DR（バックアップからの復元）とは無関係。

```bash
CLUSTER_NAME=keycloak-db
NAMESPACE=keycloak

# MinIO の既存データをクリア（警告: 旧バックアップが消える。DR が必要な場合は実行しないこと）
docker exec minio-external mc rm --recursive --force local/cnpg-backup/${CLUSTER_NAME}/

# WAL アーカイブが再開されたことを確認後、手動でベースバックアップをトリガー
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
