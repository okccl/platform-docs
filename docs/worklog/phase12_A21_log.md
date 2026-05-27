# Phase 12-A21 作業ログ：DR シナリオ C 実施・バグ修正・RTO 計測

## 冒頭サマリ

| # | 作業内容 |
|---|---|
| 1 | bootstrap.sh に環境構築の全ステップを統合（リポジトリ clone・Age 鍵配置・minio-external 作成） |
| 2 | クローン対象リポジトリを `repos.txt` に外出し |
| 3 | GCS バックアップ実行（シナリオ C 実測の RPO 基準確定） |
| 4 | DR シナリオ C 実施（k3d 再作成・全クラスター recovery bootstrap） |
| 5 | generate-dr-manifests.py 改善（kubectl delete コマンド自動生成） |
| 6 | sample-backend Application の CNPG ignoreDifferences 削除 |
| 7 | トラブルシュート: sample-backend-db が常に initdb で作成される（tgz 未コミット・serverName 誤設定） |
| 8 | トラブルシュート: backstage-db パスワード認証失敗 |

---

## 1. bootstrap.sh への環境構築ステップ統合

### 背景

シナリオ C（WSL 全損）の手順書に「全リポジトリのクローン」「Age 秘密鍵の配置」「minio-external の再作成」が個別ステップとして列挙されていた。WSL 全損と新規マシンセットアップはほぼ同一の作業であり、両方に対応する単一のエントリポイントとして `bootstrap.sh` に統合した。

### 実施内容

`bootstrap.sh` に以下の 3 セクションを追加した（既存はすべてスキップ）。

**セクション 7: リポジトリのクローン**

`platform-infra` は実行前にクローン済みのため除外。他 8 リポジトリはディレクトリが存在しない場合のみ clone する。

**セクション 8: Age 秘密鍵の配置**

`~/.config/sops/age/keys.txt` が存在しない場合、対話式プロンプトでパスワードマネージャーからの貼り付けを促す。

```bash
echo "[INPUT] Age 秘密鍵をパスワードマネージャーからコピーして貼り付け、Ctrl+D で確定してください。"
cat > "$AGE_KEY_FILE"
chmod 600 "$AGE_KEY_FILE"
```

**セクション 9: minio-external コンテナの作成**

Docker グループはスクリプト内では有効にならないため `sudo -E` を使用（`-E` で aqua の PATH を引き継ぐ）。コンテナが既存の場合はスキップ。

```bash
sudo -E docker run -d \
  --name minio-external \
  --restart unless-stopped \
  -p 9000:9000 -p 9001:9001 \
  -v "$HOME/minio-data:/data" \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD=minioadmin123 \
  quay.io/minio/minio:latest \
  server /data --console-address ":9001"
sleep 3
sudo -E docker exec minio-external /usr/bin/mc alias set local \
  http://localhost:9000 minioadmin minioadmin123
sudo -E docker exec minio-external /usr/bin/mc mb local/cnpg-backup
```

---

## 2. クローン対象リポジトリを repos.txt に外出し

### 背景

bootstrap.sh にリポジトリ名をハードコードすると、追加・名前変更時にスクリプト本体の修正が必要になる。頻繁に bootstrap を実行するわけではないため、動的取得（GitHub API）は不要だが、リストの管理場所を明示的に分離する方が保守性が高い。

### 実施内容

`scripts/repos.txt` を新規作成し、コメントで「実行前に内容を確認すること」を明記した。bootstrap.sh はこのファイルを読む形に変更した。

```bash
REPOS_FILE="$(dirname "$0")/repos.txt"
mapfile -t repos < <(grep -v '^\s*#' "$REPOS_FILE" | grep -v '^\s*$')
```

リポジトリの追加・変更時は `repos.txt` だけ編集すればよく、bootstrap.sh 本体は変更不要。

---

## 3. GCS バックアップ実行

シナリオ C 実測の直前に `make backup-to-gcs` を実行し、MinIO の最新データを GCS に同期した。

| 項目 | 値 |
|---|---|
| 完了時刻 | 2026-05-27 22:20:48 JST |
| 転送量 | 428.5 MiB / 350 ファイル |
| RPO 基準 | この時刻以降の変更は復元不可 |

この時刻が シナリオ C の RPO 基準時刻となる。

---

---

## 4. DR シナリオ C 実施

### 実施内容

k3d クラスターを削除・再作成した状態から `make bootstrap` を実行し、全 wave 完了後に `make generate-dr-manifests` → commit + push → 各クラスター delete で recovery bootstrap を起動した。

```bash
# k3d 再作成（ユーザー手動）
cd ~/platform-infra/k3d && make bootstrap

# recovery マニフェスト生成・push
cd ~/platform-infra && make generate-dr-manifests
cd ~/platform-gitops && git add -A && git commit -m "dr: activate recovery bootstrap" && git push
cd ~/apps-gitops && git add -A && git commit -m "dr: activate recovery bootstrap" && git push

# 全クラスター削除（ArgoCD が recovery bootstrap で自動再作成）
kubectl delete cluster keycloak-db -n keycloak
kubectl delete cluster backstage-db -n backstage
kubectl delete cluster sample-backend-db -n sample-app
```

keycloak-db・backstage-db は正常に recovery bootstrap で起動。sample-backend-db は後述のバグにより複数回失敗した。

**実測 RTO: 85 分**（DR-design.md 1.3 節に記録）

---

## 5. generate-dr-manifests.py 改善

### 背景

DR 実施時に「全対象クラスターを kubectl delete する」手順を手動で把握する必要があり、削除漏れのリスクがあった。

### 実施内容

スクリプト末尾の「次の手順」出力に、`kubectl get cluster -A` で取得した実際の namespace を使った `kubectl delete cluster` コマンドを自動生成するよう改修した。

```python
def get_cluster_namespace(cluster_name):
    result = subprocess.run(
        ["kubectl", "get", "cluster", "-A", "--no-headers",
         "-o", "custom-columns=NS:.metadata.namespace,NAME:.metadata.name"],
        capture_output=True, text=True, timeout=10
    )
    for line in result.stdout.splitlines():
        parts = line.split()
        if len(parts) >= 2 and parts[1] == cluster_name:
            return parts[0]
    return None
```

出力例:

```
3. 全対象クラスターを削除する:
  kubectl delete cluster backstage-db -n backstage
  kubectl delete cluster keycloak-db -n keycloak
  kubectl delete cluster sample-backend-db -n sample-app
```

あわせて `dr-restore.md` シナリオ B Step 3 を「スクリプト出力のコマンドをそのまま実行する」旨に更新した。

---

## 6. sample-backend Application の CNPG ignoreDifferences 削除

### 背景

`application.yaml` に `spec.bootstrap` / `spec.externalClusters` に対する `ignoreDifferences` が設定されていた。ArgoCD は `ServerSideApply=true` 環境では `ignoreDifferences` に指定したフィールドを SSA パッチから除外するため、Helm multi-source アプリでは新規クラスター作成時にも `bootstrap.recovery` が送信されないことが判明した。

### 実施内容

`apps-gitops/apps/sample/sample-backend/application.yaml` から以下の `ignoreDifferences` エントリを削除した。

```yaml
# 削除したエントリ
- group: postgresql.cnpg.io
  kind: Cluster
  jsonPointers:
    - /spec/bootstrap
    - /spec/externalClusters
```

platform-gitops の keycloak-db / backstage-db は直接 YAML 管理（Helm 非使用）のため影響なし。

---

## 7. トラブルシュート: sample-backend-db が常に initdb で作成される

### 問題

ignoreDifferences 削除・Redis キャッシュ削除・クラスター＆PVC 削除を繰り返しても、sample-backend-db が常に `bootstrap.initdb` で作成される。`argocd app manifests sample-backend` でも desired state が `initdb` を示した。ローカルで `helm template` を実行すると `bootstrap.recovery` が正しく出力される。

### 原因①: common-db-0.1.0.tgz の更新コミット漏れ

ArgoCD は git checkout したチャートディレクトリ（`charts/sample-backend/charts/common-db-0.1.0.tgz`）を使って `helm template` を実行する。このファイルは git コミット `db96453` 時点のもの（1522 bytes）で、その後 `932f166`（recovery 追加）・`ef7833c`（serverName 追加）で `charts/common-db/_helpers.tpl` を更新したが **tgz の再生成・コミットを行っていなかった**。ローカルの tgz は `helm dependency update` 後に更新済み（1900 bytes）だったため、ローカル実行では正しく動作していた。

```
git show 716ee058:charts/sample-backend/charts/common-db-0.1.0.tgz \
  | tar -xzf - -O common-db/templates/_helpers.tpl | grep "bootstrap"
→ 17:  bootstrap:
   18:    initdb:   ← recovery 分岐がない旧テンプレート
```

`helm dependency update` を再実行して tgz を更新し、コミット・プッシュした（`platform-charts 0cc2c53`）。

### 原因②: recovery.serverName に存在しないパスが指定されていた

tgz 修正後、recovery bootstrap が起動したが `barman-cloud-check-wal-archive` が "Expected empty archive" で失敗した。MinIO の実パスを確認したところ：

```
cnpg-backup/sample-backend-db/sample-backend-db/base/...   ← 旧バックアップ（serverName = クラスター名）
cnpg-backup/sample-backend-db/sample-backend-db-20260527/  ← initdb クラスターが汚染したパス
```

旧バックアップは chart に `serverName` フィールドがなかった時代（CNPG がクラスター名をデフォルト使用）に作成されたため、serverName は `sample-backend-db`（日付サフィックスなし）だった。一方 `apps-gitops` の values.yaml には `recovery.serverName: "sample-backend-db-20260526"` が設定されており、このパスは MinIO に存在しなかった。また `backup.serverName: "sample-backend-db-20260527"` は initdb クラスターが WAL を書き込んでいたため空ではなく、新規書き込み先チェックでも失敗した。

`recovery.serverName: "sample-backend-db-20260526"` → `"sample-backend-db"`（実在するパス）  
`backup.serverName: "sample-backend-db-20260527"` → `"sample-backend-db-20260528"`（新規空パス）

に修正して push し、クラスターを再作成したところ正常に recovery 完了した（`apps-gitops ad082f3`）。

> `generate-dr-manifests.py` 自体は `backup.get("serverName") or db_name` のロジックで正しく `serverName = "sample-backend-db"` を導出できる実装になっていた。DR 実施時点ではスクリプトに serverName 対応が未実装だったため recovery ブロックに serverName が書かれず、その後の手動修正で誤った日付サフィックス付きの値が設定されたのが直接の原因。

---

## 8. トラブルシュート: backstage-db パスワード認証失敗

### 問題

backstage-db の recovery 完了後、Backstage アプリが `password authentication failed for user "app"` で全プラグインの起動に失敗し、readiness probe が 503 を返し続けた（107 分間 Ready にならず）。

### 原因

recovery bootstrap はバックアップ時点のデータをそのまま復元するため、`initdb` フローで行われる「CNPG 管理 Secret のパスワードで DB ロールを更新する」処理がスキップされる。バックアップ取得後に ExternalSecret（`backstage-db-credentials`）のパスワードが更新されていた場合、DB ロールの実パスワードとアプリが使うパスワードがズレる。

```bash
# 確認: 2 つの Secret のパスワードが不一致だった
CNPG: backstage-db-app        → b4FI5lNl...
App:  backstage-db-credentials → 9a4f3c1e...  ← 不一致
```

### 対処

アプリが使う Secret（`backstage-db-credentials`）のパスワードに DB ロールを合わせた後、Deployment を再起動した。

```bash
PASS=$(kubectl get secret backstage-db-credentials -n backstage \
  -o jsonpath='{.data.password}' | base64 -d)
kubectl exec backstage-db-1 -n backstage -c postgres -- \
  psql -U postgres -c "ALTER ROLE app WITH PASSWORD '${PASS}';"
kubectl rollout restart deployment/backstage -n backstage
```

`dr-restore.md` トラブルシュートセクションに本事象の確認・修正手順を追記した。

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-infra/scripts/bootstrap.sh` | セクション 7〜9 を追加（リポジトリ clone・Age 鍵配置・minio-external 作成）。完了メッセージを DR 手順書へのポインタに更新 |
| `platform-infra/scripts/repos.txt` | 新規作成。クローン対象リポジトリ 8 件を列挙 |
| `platform-infra/scripts/generate-dr-manifests.py` | 「次の手順」出力に kubectl delete コマンドを自動生成する処理を追加 |
| `apps-gitops/apps/sample/sample-backend/application.yaml` | CNPG Cluster の ignoreDifferences エントリ（spec.bootstrap / spec.externalClusters）を削除 |
| `apps-gitops/apps/sample/sample-backend/values.yaml` | recovery.serverName を `sample-backend-db-20260526` → `sample-backend-db` に、backup.serverName を `sample-backend-db-20260527` → `sample-backend-db-20260528` に修正 |
| `platform-charts/charts/sample-backend/charts/common-db-0.1.0.tgz` | helm dependency update で再生成（recovery bootstrap・serverName 対応を含む最新版に更新） |
| `platform-charts/charts/sample-backend/charts/common-app-0.3.1.tgz` | helm dependency update で再生成 |
| `platform-docs/docs/design/DR-design.md` | シナリオ C RTO 実測値（85 分）を記録 |
| `platform-docs/docs/runbook/dr-restore.md` | シナリオ B Step 3 更新・パスワード不一致トラブルシュート追加 |
