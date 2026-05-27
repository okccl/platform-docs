# Phase 12-A21 作業ログ：DR シナリオ C 実施前準備（bootstrap.sh 拡張・GCS バックアップ）

## 冒頭サマリ

| # | 作業内容 |
|---|---|
| 1 | bootstrap.sh に環境構築の全ステップを統合（リポジトリ clone・Age 鍵配置・minio-external 作成） |
| 2 | クローン対象リポジトリを `repos.txt` に外出し |
| 3 | GCS バックアップ実行（シナリオ C 実測の RPO 基準確定） |

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

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-infra/scripts/bootstrap.sh` | セクション 7〜9 を追加（リポジトリ clone・Age 鍵配置・minio-external 作成）。完了メッセージを DR 手順書へのポインタに更新 |
| `platform-infra/scripts/repos.txt` | 新規作成。クローン対象リポジトリ 8 件を列挙 |
