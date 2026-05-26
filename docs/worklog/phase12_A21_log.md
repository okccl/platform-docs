# Phase 12-A21 作業ログ：Scenario C 実施前準備・Docker aqua 管理移行

## 冒頭サマリ

| # | 作業内容 |
|---|---|
| 1 | Scenario C 実施前の環境確認（ローカル固有リソース・aqua 管理外ツール調査） |
| 2 | Docker の aqua 管理移行（bootstrap.sh 改修） |

---

## 1. Scenario C 実施前の環境確認

### 背景

WSL 全損リストア（シナリオ C）を次回試みるにあたり、実施前に WSL 環境を調査した。確認内容は（1）Git 未管理のディレクトリや未 push コミットの有無、（2）aqua 管理外ツールの有無の 2 点。

### 発見した問題

| 項目 | 内容 |
|---|---|
| `platform-charts` に未コミットの `Chart.lock` | common-app 0.3.0 → 0.3.1 の更新が 2 ファイル uncommitted のまま残っていた |
| Docker が aqua 未管理 | `/usr/bin/docker`（APT インストール, v29.4.0）が aqua.yaml に記載なく、バージョン管理が 2 箇所に分散していた |

Chart.lock は即時 commit + push した。Docker の問題は次セクションで対処した。

```bash
cd ~/platform-charts
git add charts/sample-backend/Chart.lock charts/sample-frontend/Chart.lock
git commit -m "chore: Chart.lock を common-app 0.3.1 に更新"
git push
```

---

## 2. Docker の aqua 管理移行

### 背景

環境調査で、Docker が APT によるインストールのみで aqua.yaml に記載されていないことが判明した。WSL 全損後の再構築フローにおいて、Docker のバージョン管理が APT（bootstrap.sh）と aqua.yaml の 2 箇所に分散するという問題があった。

### 判断

aqua を CLI ツールの source of truth とする原則に基づき、Docker を aqua 管理に移行する。

Docker は CLI バイナリだけでなく `dockerd`・`runc` も含むフルバンドルとして `docker/cli` パッケージが aqua レジストリに存在することを確認し、`v29.4.0`（現行 APT インストール版と同一）で追加した。

### 実施内容

**aqua.yaml への追加:**

```yaml
- name: docker/cli@v29.4.0
```

**bootstrap.sh の改修:**

改修前は APT で `docker-ce docker-ce-cli containerd.io docker-buildx-plugin` を一括インストールしていた。改修後の構成は以下の通り。

| 処理 | 方法 |
|---|---|
| docker / dockerd / runc バイナリ | aqua install（aqua.yaml で version 固定） |
| containerd.io（dockerd の依存） | APT（docker 公式リポジトリ。aqua 管理外） |
| Docker daemon の systemd サービス設定 | bootstrap.sh が `aqua which dockerd` の実体パスを使って `/etc/systemd/system/docker.service` を生成 |
| docker グループ設定 | bootstrap.sh が `groupadd` / `usermod` を実行 |

また、aqua install を bootstrap.sh 内（Docker daemon 設定の前）で呼ぶよう変更したことで、`make init`（`aqua install` のみの薄いラッパー）は初回セットアップではなくバージョン更新後の再インストール用という位置づけになった。

### 設計上の注意点

systemd サービスの `ExecStart` には `aqua which dockerd` が返すバージョン含みの実体パスを埋め込む。Docker バージョンを aqua.yaml で更新した場合、`aqua install` に加えて `sudo systemctl restart docker` が必要になる。

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-infra/aqua.yaml` | `docker/cli@v29.4.0` を追加 |
| `platform-infra/scripts/bootstrap.sh` | Docker インストールを APT → aqua 管理に移行。aqua install を bootstrap 内で実行するよう変更 |
| `platform-charts/charts/sample-backend/Chart.lock` | common-app 0.3.0 → 0.3.1（未コミット分を commit） |
| `platform-charts/charts/sample-frontend/Chart.lock` | 同上 |
