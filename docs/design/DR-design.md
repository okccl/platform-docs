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

### 1.3 RTO / RPO 目標・実測値

| シナリオ | RPO | RTO（目標） | RTO（実測） |
|---|---|---|---|
| Pod / PV 障害 | WAL ラグ分（数十秒以内） | 数分 | — |
| クラスター全損 | 最終 ScheduledBackup 時刻（毎日 02:00） | 1 時間以内 | **31〜67 秒**（シナリオ A 実測） |
| WSL 全損 | 最終 GCS 同期時刻（前日 23:00） | 数時間 | 未計測 |

**シナリオ A（PVC 破損リストア）実測値 — 2026-05-26**

| クラスター | RTO | 最終バックアップ（RPO 基準） |
|---|---|---|
| keycloak-db（1 インスタンス） | **41 秒** | 2026-05-10 12:02 UTC |
| backstage-db（1 インスタンス） | **31 秒** | 2026-05-11 20:21 UTC |
| sample-backend-db（2 インスタンス） | **67 秒** | 2026-05-11 20:02 UTC |

> 実測は k3d ローカル環境（WSL2）での計測。`kubectl apply` から "Cluster in healthy state" までの時間。

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

PVC 破損が発生した場合は MinIO バックアップからの手動リストアが必要。`make generate-dr-manifests` で生成した `recovery` マニフェストを apply し、リストア完了後に DR クラスターを削除すると ArgoCD が `initdb` で再作成する。このとき PVC Retain ポリシー（3.1 節）によりデータが保全され、CNPG が既存 PVC を検知して initdb をスキップする。

> **詳細手順**: [Runbook-002: DB リストア手順 — シナリオ A](../runbook/dr-restore.md)

### 3.3 クラスター全損からの復旧（k3d 再作成）

MinIO は k3d コンテナ外の Docker コンテナ（`minio-external`）として稼働するため、k3d クラスター全損の影響を受けずバックアップデータが保全される。これはローカル 2 層設計の核心で、k3d を誤削除しても MinIO 上のデータは無傷のまま残る。

復旧フローは `make bootstrap` でまず定常状態（ArgoCD + `initdb` クラスター）を再構成し、その後 DR マニフェストで `recovery` bootstrap に切り替える 2 フェーズ構成とした。`make bootstrap` を起点にすることで DR 手順がベースラインの起動フローを完全に再利用でき、独立した再現性が確保される。

> **詳細手順**: [Runbook-002: DB リストア手順 — シナリオ B](../runbook/dr-restore.md)

### 3.4 WSL 全損からの復旧

WSL 全損時は MinIO（Docker コンテナ）も失われるため、GCS から MinIO を復元した後に 3.3 の手順を実行する 2 ステップ構成となる。

GCS→MinIO→CNPG の 2 ホップを選んだのは、CNPG が GCS S3 互換 API を直接使うには HMAC キー（SA キーとは別管理）が必要で構成が複雑になるためで、MinIO を中継することで認証の複雑さを避けつつ 3.3 の手順をそのまま再利用できる。

GCS に保存されているのは最終同期時刻（毎日 23:00）時点のスナップショットのみで WAL は含まれない。そのため最終 GCS 同期以降の変更は復元不可（最大 RPO = 24 時間）。GCS の Object Lifecycle（30 日保持）の範囲内であれば古い時点のバックアップへの復元も可能。

> **詳細手順**: [Runbook-002: DB リストア手順 — シナリオ C](../runbook/dr-restore.md)

---

## 4. 実装状況

| 項目 | 状態 | 参照 |
|---|---|---|
| `ignoreDifferences` 設定 | **完了** | 3.1 節 |
| PVC Retain ポリシー設定 | **完了**（`local-path-retain` SC + 既存 PV パッチ） | 3.1 節 |
| DR マニフェスト生成スクリプト | **完了**（`make generate-dr-manifests`・GitOps 直接書き換え方式） | 3.1 節 |
| `common-db` Helm chart recovery 対応 | **完了**（`db.recovery.enabled` values）| — |
| クラウドバックアップ実装 | **完了**（`make backup-to-gcs`・毎日 23:00 cron） | 2.2 節 |
| DR 手順書（Runbook）作成 | **完了** | `runbook/dr-restore.md` |
| RTO/RPO 実測 | **完了**（シナリオ A: 31〜67 秒） | 1.3 節 |
