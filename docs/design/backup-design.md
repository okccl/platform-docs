# バックアップ設計書

> **ステータス**: 作成中（1〜3章のみ記述済み）

---

## 1. 概要

### 1.1 目的

本基盤の保護対象は PostgreSQL データ（Keycloak DB・Backstage DB）。ArgoCD・Prometheus 等のステートレスコンポーネントはすべて GitOps から再構築できるためバックアップ対象外とする。

単一ストレージへのバックアップはその障害が即 DR 不能に直結する。そのため、障害シナリオごとに対応できる2層構成を採用した。

| 層 | バックアップ先 | カバーするシナリオ |
|---|---|---|
| 第1層 | ローカル MinIO（WSL ホスト） | クラスター全損（k3d 再作成等） |
| 第2層 | GCS（Google Cloud Storage） | WSL 全損（PC 故障・OS 再インストール等） |

### 1.2 バックアップチェーン全体図

```
PostgreSQL（CNPG Pod）
  │
  ├─ WAL アーカイブ（随時）─────────────────────┐
  └─ ベースバックアップ（毎日 21:00）───────────┤
                                                 ▼
                                  MinIO（minio-external コンテナ）
                                  s3://cnpg-backup/
                                  ├── backstage-db/wals/
                                  ├── backstage-db/data/
                                  ├── keycloak-db/wals/
                                  └── keycloak-db/data/
                                                 │
                                  rclone copy（毎日 23:00、WSL cron）
                                                 ▼
                                  GCS: gs://ccl-platform-cnpg-backup/
                                  （Object Lifecycle: 30 日後に自動削除）
```

---

## 2. バックアップ対象

| DB | namespace | PostgreSQL バージョン | ストレージ |
|---|---|---|---|
| Backstage DB | backstage | 17 | 1Gi（local-path-retain） |
| Keycloak DB | keycloak | 17 | 2Gi（local-path-retain） |

バックアップのデータ種別は以下の2種類。

| 種別 | 内容 | タイミング |
|---|---|---|
| WAL アーカイブ | PostgreSQL の Write-Ahead Log。変更をリアルタイムで転送 | 随時（CNPG が自動管理） |
| ベースバックアップ | フルダンプ相当。WAL リプレイの起点となる | 毎日 21:00（ScheduledBackup） |

ストレージクラスに `local-path-retain`（reclaimPolicy: Retain）を使用しているのは、MinIO 復元フロー中の切り戻し時に PV データを保全するためである。k3d クラスター全損（`k3d cluster delete`）では Docker ボリュームごと消えるため reclaimPolicy は関係ない。

Retain が必要な場面は、recovery クラスターを削除して ArgoCD の initdb クラスターに切り戻す際である。

```
1. recovery クラスターを削除
   ├─ [Retain あり] PV が Released 状態で残る → データ保全
   └─ [Retain なし] PV ごと削除 → 復元データが消える
2. ArgoCD の initdb クラスターが PVC を作成 → 既存 PV にバインド
3. CNPG が PV 上のデータを検知 → bootstrap をスキップ → 通常運用に復帰
```

---

## 3. 第1層：PostgreSQL → MinIO（ローカルバックアップ）

### 3.1 MinIO をクラスター外に配置する理由

CNPG のバックアップ先をクラスター内（PVC 等）に置くと、クラスター全損と同時にバックアップも失われる。DR の意味をなさない。

そのため MinIO を WSL ホスト上の Docker コンテナ（`minio-external`）として稼働させ、クラスターのライフサイクルから独立させた。k3d クラスターを削除・再作成しても MinIO のデータは保全される。

k3d コンテナから WSL ホストの MinIO への接続は `http://host.k3d.internal:9000` で行う。`host.k3d.internal` は k3d 起動時に CoreDNS NodeHosts へ動的登録されており（`make cluster-start` に組み込み済み）、クラスター再起動後も疎通が保たれる。

### 3.2 WAL アーカイブ

CNPG の `barmanObjectStore` 設定により、WAL セグメントが生成されるたびに `barman-cloud-wal-archive` が自動でアップロードする。スケジュール設定は不要で CNPG が管理する。

WAL アーカイブの効果は「ベースバックアップ間のポイントインタイムリカバリ（PITR）を可能にする」ことである。ベースバックアップ単体では取得時刻までしか復旧できないが、WAL を組み合わせることで任意の時点へのリストアが可能になる。

```yaml
# barmanObjectStore（各 CNPG Cluster の spec.backup に設定）
endpointURL: "http://host.k3d.internal:9000"
destinationPath: "s3://cnpg-backup/<cluster-name>"
wal:
  compression: gzip
data:
  compression: gzip
```

圧縮に gzip を採用しているのはストレージ使用量削減のため。CNPG がデフォルトでサポートしており追加実装コストがない。

### 3.3 ベースバックアップ（ScheduledBackup）

CNPG の `ScheduledBackup` リソースにより毎日 21:00 にベースバックアップを取得する。21:00 を選んだのは、第2層のクラウド同期が 23:00 であり、その前にベースバックアップが完了するよう余裕を持たせるためである。

```yaml
# ScheduledBackup（各 DB に定義）
schedule: "0 21 * * *"
backupOwnerReference: self
```

### 3.4 保持期間

`retentionPolicy: 7d`（7日間）を設定している。7日を超えた古いベースバックアップは CNPG が自動削除する。

7日を選んだ理由は明確な業務要件があるわけではなく、ローカルストレージ容量の節約と、本番相当の要件がない個人ポートフォリオという性質から判断した。7日より長期の保全は第2層（GCS）で対応する。

| ストレージ | 保持期間 | 削除方法 |
|---|---|---|
| MinIO（ローカル） | 7日 | CNPG の retentionPolicy による自動削除 |
| GCS（クラウド） | 30日 | Object Lifecycle ルールによる自動削除 |

### 3.5 認証情報管理

MinIO の認証情報（`ACCESS_KEY_ID` / `ACCESS_SECRET_KEY`）は SOPS × Age で暗号化して Git 管理する。クラスター起動後は ESO が Secret を自動生成するため、手動で kubectl apply する必要はない。

```
platform-gitops/platform/secrets/sources/minio-backup-secret-source.yaml  ← SOPS 暗号化 Secret
  ↓ ESO ClusterSecretStore（kubernetes-store）
  ↓ ExternalSecret（backstage-minio-backup.yaml / keycloak-minio-backup.yaml）
各 namespace の minio-backup-secret（CNPG Cluster が参照）
```

認証情報を各 namespace に ExternalSecret で配布しているのは、CNPG Cluster の `spec.backup.barmanObjectStore.s3Credentials` が同一 namespace の Secret しか参照できないためである。Secret の実体は `platform-secrets` namespace に1つだけ存在し、ESO がコピーを配布する形をとることで、認証情報のソースを一元管理している。

---

## 4. 第2層：MinIO → GCS（クラウドバックアップ）

### 4.1 クラウドバックアップを追加する理由

MinIO は WSL ホスト上の Docker コンテナ（`minio-external`）として稼働しているため、WSL 全損（PC 故障・OS 再インストール等）が発生した場合は MinIO のデータも失われる。第1層だけでは WSL 全損シナリオに対応できない。

GCS にオフサイトコピーを持つことで、ローカル環境が完全に失われた場合でも DB データを復旧できる。

### 4.2 コンポーネント構成

| コンポーネント | 役割 | 備考 |
|---|---|---|
| rclone | MinIO → GCS の同期ツール | aqua で管理 |
| GCS バケット | バックアップ保存先 | Always Free: 5GB / 月 |
| GCP Service Account | GCS への書き込み認証 | バケット限定の最小権限 |
| `backup-to-gcs.sh` | バックアップ実行スクリプト | `platform-infra/scripts/` |
| WSL crontab | 日次自動実行 | 毎日 23:00 |
| `make backup-to-gcs` | 手動実行 Makefile ターゲット | |

### 4.3 同期方式の選択（copy vs sync）

`rclone copy` を採用し、`rclone sync` は採用しない。

MinIO 側では CNPG の `retentionPolicy: 7d` により古いバックアップが自動削除される。`rclone sync` にすると、MinIO の削除が GCS にも伝播し、クラウド側でも 7 日分しか保持できなくなる。これでは WSL 全損時の復旧窓口がローカルと変わらず、クラウドバックアップの意義が薄れる。

`rclone copy` にすることで GCS 側には累積保存し、GCS の Object Lifecycle ルールで 30 日後に自動削除する。ローカル 7 日 / クラウド 30 日という差別化した保持期間を実現している。

### 4.4 GCS 設定（リージョン・クラス・ライフサイクル）

| 項目 | 値 | 理由 |
|---|---|---|
| バケット名 | `ccl-platform-cnpg-backup` | |
| リージョン | `us-central1` | GCS Always Free（5GB / 月）の対象リージョン |
| ストレージクラス | Standard | Always Free 対象。Nearline / Coldline は小容量では最低保存期間の縛りがあり割高になる |
| Object Lifecycle | Age: 30 日 → Delete | ローカル 7 日より長期保持しつつコスト増を抑える |
| バージョニング | 無効 | 同名ファイルは上書き運用のため不要 |

Always Free の 5GB / 月制限について：現在のバックアップサイズは約 314MB。30 日累積でも 5GB 以内に収まる見込みのため、GCS の利用コストは実質ゼロを想定している。

### 4.5 rclone を選択した理由

MinIO（S3 互換 API）と GCS（Google Cloud Storage）の両方をソース・デスティネーションとして扱えるツールが必要になる。候補と評価は以下。

| ツール | 評価 |
|---|---|
| gsutil | GCS 公式 CLI だが MinIO（S3 互換）からの読み取りには対応していない |
| mc（MinIO Client） | MinIO 公式 CLI だが GCS への書き込みは S3 互換エンドポイント経由のみで信頼性が低い |
| rclone | S3 互換と GCS の両方をネイティブサポート。設定ファイルなしでコマンドライン引数だけで完結できる |

rclone はコマンドライン引数でバックエンドを指定できるため、設定ファイルをファイルシステムに残さずに済む。これにより SA キーのような認証情報の漏洩リスクを軽減できる。

### 4.6 認証情報管理（SOPS × Age）

GCP Service Account のキー（JSON）は SOPS × Age で暗号化して platform-infra の Git で管理する。

スクリプト実行時に SOPS で復号して `/tmp` に一時展開し、rclone 終了後に `trap EXIT` で削除する。

```
platform-infra/secrets/gcp-backup-sa-key.enc.json  ← SOPS 暗号化 SA キー
  ↓ スクリプト実行時に SOPS 復号
  /tmp/gcp-backup-sa-key.json（一時ファイル）
  ↓ rclone の --gcs-service-account-file に渡す
  ↓ 終了後に trap EXIT で削除
```

SA 権限は `roles/storage.objectAdmin`（対象バケット限定の最小権限）。

MinIO の認証情報は platform-gitops で管理している SOPS 暗号化ファイル（`minio-backup-secret-source.yaml`）からスクリプト実行時に復号して取得する。MinIO コンテナには `MINIO_ROOT_PASSWORD_FILE` が設定されており `docker inspect` では正確な値が取れないためこの方式を採用した。

### 4.7 実行スケジュール

| 時刻 | 処理 |
|---|---|
| 毎日 21:00 | CNPG ScheduledBackup（MinIO へのベースバックアップ） |
| 毎日 23:00 | WSL cron：`backup-to-gcs.sh` 実行（MinIO → GCS） |

21:00 の ScheduledBackup が完了してから十分な余裕を持って 23:00 に GCS へ同期する設計になっている。

PC の電源がオフの場合は cron が実行されないため、バックアップがスキップされる可能性がある。個人 PC での運用という制約上、これは許容する。
