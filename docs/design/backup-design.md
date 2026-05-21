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

ストレージクラスに `local-path-retain`（reclaimPolicy: Retain）を使用しているのは、クラスター全損時に PVC が削除されても PV 上のデータを保全するためである。ただし MinIO によるオフサイトバックアップが正式な DR 手段であり、PV の Retain はあくまで補助的な保護に留まる。

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
