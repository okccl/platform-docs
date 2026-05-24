# シークレット追加・更新手順

## 構成概要

```
Git（SOPS暗号化）
  ↓ ArgoCD + ksops
platform-secrets NS（source secrets）
  ↓ ESO ClusterSecretStore（kubernetes-store）
各コンポーネント NS（ExternalSecret経由で生成）
```

- **管理場所**: `platform-gitops/platform/secrets/sources/`（12ファイル）
- **暗号化対象フィールド**: `data` / `stringData` のみ（キー名・構造は平文）
- **テンプレート**: `platform-gitops/secrets/templates/` に新規追加の雛形あり

---

## 既存シークレットの更新

```bash
cd ~/platform-gitops

# エディタが開き、復号された状態で編集できる
# 保存・終了すると自動で再暗号化される
sops platform/secrets/sources/<ファイル名>.yaml
```

編集後:

```bash
git add platform/secrets/sources/<ファイル名>.yaml
git commit -m "secret: <変更内容>"
git push origin main
```

ArgoCD が自動 sync する。即時反映したい場合:

```bash
argocd app sync platform-secrets-sources
```

ESO による Secret 再生成を確認:

```bash
kubectl get externalsecret -n platform-secrets
# READY が True になれば反映完了
```

---

## 新規シークレットの追加

### 1. テンプレートをコピー

```bash
cd ~/platform-gitops

# テンプレートが存在する場合はコピー（この時点では値はプレースホルダーのまま）
cp secrets/templates/<近いもの>.yaml platform/secrets/sources/<新ファイル>.yaml

# なければ既存ファイルを参考に作成
# 形式: apiVersion/kind/metadata/stringData の通常の Secret マニフェスト
```

### 2. 先に暗号化してからエディタで実際の値を入力

```bash
# まず暗号化（プレースホルダーが暗号化される。平文の実値はまだ入れない）
sops --encrypt --in-place platform/secrets/sources/<新ファイル>.yaml

# エディタで開いてプレースホルダーを実際の値に書き換える
# 保存・終了すると自動で再暗号化される
sops platform/secrets/sources/<新ファイル>.yaml
```

`.sops.yaml` のルールにより `stringData` / `data` フィールドが自動で暗号化される。

> **注意**: テンプレートをコピーした直後は SOPS メタデータがないため `sops <file>` は "sops metadata not found" エラーになる。必ず `--encrypt --in-place` を先に実行すること。

### 3. kustomization.yaml に追加

```bash
sops platform/secrets/sources/kustomization.yaml
# resources: に新ファイルを追記
```

### 4. push・sync

```bash
git add platform/secrets/sources/<新ファイル>.yaml platform/secrets/sources/kustomization.yaml
git commit -m "secret: <新シークレット名> 追加"
git push origin main

argocd app sync platform-secrets-sources
kubectl get secret <シークレット名> -n platform-secrets
```

### 5. ExternalSecret の追加（他 NS に配布する場合）

対象 NS の ExternalSecret マニフェストを追加し、該当 Application を sync。

---

## 確認コマンド

```bash
# source secrets の一覧
kubectl get secret -n platform-secrets

# ESO の同期状態
kubectl get externalsecret -A

# 特定シークレットの内容確認
kubectl get secret <名前> -n <namespace> -o jsonpath='{.data.<キー>}' | base64 -d
```

---

## 注意事項

- `secrets/encrypted/local-ca-secret.yaml` は bootstrap 専用の push 型シークレット（cert-manager 起動前に必要）。このファイルはすべてのフィールドが暗号化されており、通常の sources/ とは管理方法が異なる。更新手順は bootstrap 手順書を参照。
- `sops <file>` は `$EDITOR` を使用する。未設定の場合は `export EDITOR=vim`（または `nano`）を先に実行。
- 平文ファイルを一時的にディスクに書き出す方法（`sops -d > plain.yaml`）は平文が残るリスクがあるため避けること。
