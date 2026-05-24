# クラウドバックアップ GCP セットアップ

| # | 作業内容 |
|---|---|
| 1 | gcloud CLI・rclone を aqua に追加 |
| 2 | GCP プロジェクト作成・請求先アカウントリンク |
| 3 | GCS バケット作成・Lifecycle 設定 |
| 4 | Service Account 作成・バケット権限付与 |
| 5 | org policy 除外・SA キー作成・SOPS 暗号化 |

---

## 1. gcloud CLI・rclone を aqua に追加

gcloud と rclone のいずれも aqua 標準レジストリに登録されていることを確認してから追加した。gcloud は `twistedpair/google-cloud-sdk`（`gcloud` / `gsutil` / `bq` を含む）、rclone は `rclone/rclone` で管理する。

```yaml
# aqua.yaml に追加
- name: twistedpair/google-cloud-sdk@562.0.0
- name: rclone/rclone@v1.74.2
```

```bash
aqua install
```

---

## 2. GCP プロジェクト作成・請求先アカウントリンク

**GUI 手順:** コンソール上部のプロジェクト選択 → 「新しいプロジェクト」→ プロジェクト名・組織・場所（組織直下）を入力 → 「作成」

```bash
gcloud projects create <PROJECT_ID> \
  --name="<PROJECT_ID>" \
  --organization=<ORG_ID>
gcloud config set project <PROJECT_ID>
```

### トラブルシュート：バケット作成時に請求先エラー

バケット作成を試みたところ以下のエラーが発生した。新規プロジェクトに請求先アカウントが未リンクだったため。

```
ERROR: The billing account for the owning project is disabled in state absent.
```

**GUI 手順:** お支払い → 「マイプロジェクト」タブ → プロジェクト行の「請求先アカウントを変更」→ 対象アカウントを選択

```bash
gcloud billing projects link <PROJECT_ID> \
  --billing-account=<BILLING_ACCOUNT_ID>
```

---

## 3. GCS バケット作成・Lifecycle 設定

**GUI 手順（バケット）:** Cloud Storage → 「バケットを作成」→ バケット名・リージョン（us-central1）・Standard・均一アクセス制御 → 「作成」

```bash
gcloud storage buckets create gs://<BUCKET_NAME> \
  --project=<PROJECT_ID> \
  --location=us-central1 \
  --default-storage-class=standard \
  --uniform-bucket-level-access
```

**GUI 手順（Lifecycle）:** バケット選択 → 「ライフサイクル」タブ → 「ルールを追加」→ アクション「削除」・条件「経過日数 30 日」→「作成」

lifecycle.json を一時ファイルとして作成してから渡す（プロセス置換は gcloud では使用不可）。

```bash
cat > /tmp/lifecycle.json << 'EOF'
{"rule": [{"action": {"type": "Delete"}, "condition": {"age": 30}}]}
EOF

gcloud storage buckets update gs://<BUCKET_NAME> \
  --lifecycle-file=/tmp/lifecycle.json

rm /tmp/lifecycle.json
```

---

## 4. Service Account 作成・バケット権限付与

**GUI 手順（SA 作成）:** IAM と管理 → サービス アカウント → 「サービス アカウントを作成」→ 名前・説明を入力 → ロールなしで作成

**GUI 手順（権限付与）:** Cloud Storage → バケット選択 → 「権限」タブ → 「プリンシパルを追加」→ SA メールを入力 → ロール「Storage オブジェクト管理者」

```bash
gcloud iam service-accounts create <SA_NAME> \
  --display-name="CNPG Backup SA" \
  --description="MinIO-GCS バックアップ同期用（バケット限定 objectAdmin）" \
  --project=<PROJECT_ID>

gcloud storage buckets add-iam-policy-binding gs://<BUCKET_NAME> \
  --member="serviceAccount:<SA_EMAIL>" \
  --role="roles/storage.objectAdmin"
```

---

## 5. org policy 除外・SA キー作成・SOPS 暗号化

### 背景

`ccl-org` には `constraints/iam.disableServiceAccountKeyCreation` が設定されており、SA キーの作成がデフォルトで禁止されていた。

ADC（`gcloud auth application-default login`）への切り替えも検討したが、WSL cron 環境では ADC も `~/.config/gcloud/application_default_credentials.json` に平文保存されるため安全性は SA キーと同等であり、さらに Git 管理できず bootstrap 再現性も失われる。SA キー + SOPS を採用し、このプロジェクトのみ org policy から除外することにした。詳細な判断根拠は `backup-design.md` 4.6 節に記載。

### 問題 1：Organization Policy API が未有効化

```
ERROR: Organization Policy API has not been used in project <PROJECT_ID> before or it is disabled.
```

org policy を操作するには `orgpolicy.googleapis.com` を事前に有効化する必要がある。

```bash
gcloud services enable orgpolicy.googleapis.com --project=<PROJECT_ID>
```

### 問題 2：orgpolicy.policyAdmin 権限が不足

```
ERROR: Permission 'orgpolicy.policies.create' denied on resource
```

`roles/resourcemanager.organizationAdmin` には `orgpolicy.policies.create` が含まれていない。組織レベルで `roles/orgpolicy.policyAdmin` を別途付与する必要があった。

**GUI 手順:** IAM と管理 → 組織の IAM → 「アクセスを許可」→ ユーザーを追加 → ロール「組織ポリシー管理者」

```bash
gcloud organizations add-iam-policy-binding <ORG_ID> \
  --member="user:<USER_EMAIL>" \
  --role="roles/orgpolicy.policyAdmin"
```

### 対処：org policy 除外設定・SA キー作成

**GUI 手順（ポリシー除外）:** IAM と管理 → 組織のポリシー → `iam.disableServiceAccountKeyCreation` → 「管理」→ プロジェクトレベルで「適用しない」に変更

```bash
gcloud org-policies set-policy /dev/stdin --project=<PROJECT_ID> << 'EOF'
name: projects/<PROJECT_ID>/policies/iam.disableServiceAccountKeyCreation
spec:
  rules:
  - enforce: false
EOF
```

org policy の反映には数分かかった。反映後、SA キーを作成した。

**GUI 手順（SA キー）:** IAM と管理 → サービス アカウント → SA 選択 → 「キー」タブ → 「鍵を追加」→「新しい鍵を作成」→ JSON → 「作成」（自動ダウンロード）

```bash
gcloud iam service-accounts keys create /tmp/gcp-backup-sa-key.json \
  --iam-account=<SA_EMAIL> \
  --project=<PROJECT_ID>
```

キーを SOPS で暗号化して platform-infra の Git で管理する。`trap EXIT` による一時ファイル削除はスクリプト実行時に行うが、ここではキー作成直後に手動削除した。

```bash
SOPS_AGE_KEY_FILE="${HOME}/.config/sops/age/keys.txt" \
  sops encrypt \
  --age <AGE_PUBLIC_KEY> \
  /tmp/gcp-backup-sa-key.json > ~/platform-infra/secrets/gcp-backup-sa-key.enc.json

rm /tmp/gcp-backup-sa-key.json
```

---

| ファイル | 変更内容 |
|---|---|
| `platform-infra/aqua.yaml` | gcloud CLI（twistedpair/google-cloud-sdk 562.0.0）・rclone（v1.74.2）を追加 |
| `platform-infra/secrets/gcp-backup-sa-key.enc.json` | GCP SA キーを SOPS 暗号化して追加（新規） |
