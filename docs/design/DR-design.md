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

#### GitOps を DR のトリガーとして使う設計

`spec.bootstrap` が immutable である以上、DR 時には「既存クラスターを削除 → recovery bootstrap で再作成」という手順が必須になる。ここで問題になるのが ArgoCD の `selfHeal: true` による即時 reconcile だ。

**旧設計（ArgoCD を迂回する方式）**: DR マニフェストを `kubectl apply` で手動 apply し、ArgoCD の auto-sync を一時停止することでレース条件を回避していた。しかしこれはエラーが起きやすく、一時停止の範囲（apps-root → sample-backend のような App-of-Apps 構造で連鎖が必要）も複雑だった。

**現設計（GitOps を DR のトリガーにする方式）**: `make generate-dr-manifests` が gitops リポジトリのマニフェストを直接 recovery bootstrap に書き換え、push → ArgoCD 自身が recovery bootstrap でクラスターを作成する。

```
make generate-dr-manifests
  → platform-gitops/apps-gitops を recovery に書き換え → push

kubectl delete cluster + pvc
  → ArgoCD が gitops（recovery）から即座に再作成 ← レース条件ゼロ

DR 完了後: git checkout -- . → push（initdb に戻す）
  → ignoreDifferences により running クラスターは影響なし
```

ArgoCD を迂回するのではなく、ArgoCD を DR のエグゼキューターとして活用する。GitOps が一時的に recovery bootstrap を宣言するため、任意のタイミングで delete しても必ず recovery で再作成される。

DR 後 gitops を initdb に戻しても、running クラスターの bootstrap は CNPG webhook で保護されており変更されない。ArgoCD は `ignoreDifferences` でこの差分を無視する（詳細は次節）。

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

PVC Retain ポリシーの主な用途は、クラスター全損（k3d 再作成）時に PVC が自動削除されないようにすることではない（k3d 全損では PVC もなくなる）。正確には以下の 2 つのケースでデータを保全する:

1. **シナリオ A（PVC 破損リストア）**: 破損した PVC を手動削除して recovery でリストアする際、他の PVC（同一クラスターの別インスタンス）が Retain で保護される
2. **誤操作による `kubectl delete cluster`**: クラスターを誤削除した際に PVC が残存し、ArgoCD が `initdb` で再作成した場合に CNPG が既存データを検知して initdb をスキップする

**ただし (2) のケースには WAL アーカイブの問題がある**（後述「WAL アーカイブと bootstrap 方式の関係」参照）。

**実装済み**:
- `local-path-retain` StorageClass（`reclaimPolicy: Retain`）を GitOps で管理（wave 0 で適用）
- 全 CNPG クラスターの `storage.storageClass: local-path-retain` を設定済み（新規 PVC から自動適用）
- 既存 PV は `kubectl patch` で Retain に変更済み

#### DR マニフェスト生成（実装済み）

DR マニフェストは静的ファイルとして管理せず、`make generate-dr-manifests` で GitOps ソースを動的に書き換える。これにより、クラスター設定変更時のドリフトを防ぐ。

スクリプト（`scripts/generate-dr-manifests.py`）は以下を自動スキャンし、該当ファイルを直接上書きする:

| スキャン対象 | 書き換え内容 |
|---|---|
| `platform-gitops/platform/**/*.yaml`（`kind: Cluster` + `barmanObjectStore`） | `spec.bootstrap` を `recovery` に変更・`externalClusters` を追加 |
| `apps-gitops/apps/*/*/values.yaml`（`db.backup.enabled: true`） | `db.recovery.enabled: true` と接続設定を追加 |

apps-gitops のクラスターは `common-db` Helm chart で管理されており、`db.recovery.enabled: true` を設定すると Helm が recovery bootstrap の CNPG Cluster マニフェストを生成する。

新しいクラスターが追加されても手動メンテナンス不要。DR 後は gitops リポジトリで `git checkout -- .` して push するだけで元の `initdb` 設定に戻る（running クラスターは `ignoreDifferences` で保護されており無影響）。

#### WAL アーカイブと bootstrap 方式の関係（実測で判明した重要な制約）

CNPG のバックアップ開始時に `barman-cloud-check-wal-archive` が実行される。このチェックは bootstrap 方式によって動作が異なる。

| bootstrap 方式 | WAL アーカイブ開始時の動作 |
|---|---|
| `initdb`（PVC が空でも既存でも） | `barman-cloud-check-wal-archive` を実行 → MinIO に既存データがあると "Expected empty archive" で失敗 |
| `recovery` | `barman-cloud-check-wal-archive` をスキップ → 既存データがあっても WAL アーカイブ開始 |

この差異から以下の設計上の制約が生じる。

**DR 後にクラスターを `initdb` で再作成してはいけない**: DR 完了後、running クラスターを削除して ArgoCD に `initdb` で再作成させると（旧設計の「GitOps クリーン状態に戻す」ステップ）、MinIO に DR 以前のバックアップデータが残っているため WAL アーカイブが必ず失敗する。

**現設計での解決**: DR 後も running クラスターは `recovery` bootstrap のまま運用し続ける。ArgoCD は `ignoreDifferences` でこの差分を無視する。`recovery` bootstrap のクラスターは WAL アーカイブチェックをスキップするため、MinIO の既存データと共存できる。

**クラスターを誤削除して `initdb` で再作成された場合の対処**: MinIO のバックアップデータをクリアしてから手動でベースバックアップをトリガーする（詳細は Runbook トラブルシュート節を参照）。

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

PVC 破損が発生した場合は MinIO バックアップからの手動リストアが必要。`make generate-dr-manifests` で gitops を recovery bootstrap に書き換えて push すると、ArgoCD が recovery bootstrap でクラスターを再作成する。リストア完了後は gitops を元の `initdb` に戻す（running クラスターは `ignoreDifferences` で保護されているため無影響）。DR 後にクラスターを `initdb` で再作成すると WAL アーカイブが壊れるため、クラスターの削除は行わないこと（3.1 節「WAL アーカイブと bootstrap 方式の関係」参照）。

> **詳細手順**: [Runbook: DB リストア手順 — シナリオ A](../runbook/dr-restore.md)

### 3.3 クラスター全損からの復旧（k3d 再作成）

MinIO は k3d コンテナ外の Docker コンテナ（`minio-external`）として稼働するため、k3d クラスター全損の影響を受けずバックアップデータが保全される。これはローカル 2 層設計の核心で、k3d を誤削除しても MinIO 上のデータは無傷のまま残る。

復旧フローは `make bootstrap` でまず定常状態（ArgoCD + `initdb` クラスター）を再構成し、その後 DR マニフェストで `recovery` bootstrap に切り替える 2 フェーズ構成とした。`make bootstrap` を起点にすることで DR 手順がベースラインの起動フローを完全に再利用でき、独立した再現性が確保される。

> **詳細手順**: [Runbook: DB リストア手順 — シナリオ B](../runbook/dr-restore.md)

### 3.4 WSL 全損からの復旧

WSL 全損時は MinIO（Docker コンテナ）も失われるため、GCS から MinIO を復元した後に 3.3 の手順を実行する 2 ステップ構成となる。

GCS→MinIO→CNPG の 2 ホップを選んだのは、CNPG が GCS S3 互換 API を直接使うには HMAC キー（SA キーとは別管理）が必要で構成が複雑になるためで、MinIO を中継することで認証の複雑さを避けつつ 3.3 の手順をそのまま再利用できる。

GCS に保存されているのは最終同期時刻（毎日 23:00）時点のスナップショットのみで WAL は含まれない。そのため最終 GCS 同期以降の変更は復元不可（最大 RPO = 24 時間）。GCS の Object Lifecycle（30 日保持）の範囲内であれば古い時点のバックアップへの復元も可能。

> **詳細手順**: [Runbook: DB リストア手順 — シナリオ C](../runbook/dr-restore.md)

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
