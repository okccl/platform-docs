# Phase 12 作業ログ（セッション6）

## 実施日
2026-05-05

---

## 1. Secret Pull 型管理への移行（ksops 導入）

### .sops.yaml 更新
```bash
cat > ~/platform-gitops/.sops.yaml << 'EOF2'
creation_rules:
  - path_regex: platform/secrets/sources/.*\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1unp3mcue0u65eu28ls4l037jmn3umv6fusa7ggjeduep59tsvussxdurgs
  - path_regex: secrets/.*\.yaml$
    age: age1unp3mcue0u65eu28ls4l037jmn3umv6fusa7ggjeduep59tsvussxdurgs
EOF2
```

### platform/secrets/sources/ 作成
```bash
mkdir -p ~/platform-gitops/platform/secrets/sources
# kustomization.yaml・secret-generator.yaml を新規作成
```

### source secrets 移行（namespace: argocd → platform-secrets、部分暗号化で再暗号化）
```bash
SOURCE_SECRETS=(
  "argocd-keycloak-secret.yaml"
  "backstage-db-credentials-source.yaml"
  "backstage-github-token-source.yaml"
  "backstage-keycloak-secret-source.yaml"
  "backstage-secret-source.yaml"
  "ghcr-pat.yaml"
  "keycloak-admin-credentials.yaml"
  "keycloak-db-credentials.yaml"
  "keycloak-platform-admin-source.yaml"
)
for f in "${SOURCE_SECRETS[@]}"; do
  SRC="secrets/encrypted/$f"
  DST="platform/secrets/sources/$f"
  SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops decrypt "$SRC" \
    | sed 's/namespace: argocd/namespace: platform-secrets/g' \
    > "$DST"
  SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops -e -i "$DST"
  rm "$SRC"
done
# minio-auth 削除
rm -f secrets/encrypted/minio-auth.yaml
rm -f secrets/templates/minio-auth.yaml
```

### platform/argocd/values.yaml 更新（ksops 導入）
- `configs.cm.kustomize.buildOptions: "--enable-alpha-plugins --enable-exec"` 追加
- `repoServer` セクション追加（ksops init container、sops-age Secret マウント）

### platform-secrets-sources Application 作成
```bash
# platform/applications/root-2-auth/platform-secrets-sources.yaml
# sync-wave: "2"、path: platform/secrets/sources
```

### ClusterSecretStore 更新
```bash
# remoteNamespace: argocd → remoteNamespace: platform-secrets
```

### コミット
```bash
cd ~/platform-gitops
git add .
git commit -m "feat: Pull型Secret管理移行・ksops導入・webhook再有効化"
git push origin main
```

---

## 2. Makefile 更新（platform-infra）

### sops-age Secret 投入を bootstrap-argocd に追加
```makefile
# ArgoCDインストールコメント直前に挿入
kubectl create secret generic sops-age \
    --from-file=keys.txt=$(SOPS_AGE_KEY) \
    -n $(ARGOCD_NS) \
    --dry-run=client -o yaml | kubectl apply -f -
```

### bootstrap-sync から source secrets 手動 apply を削除
- `keycloak-db-credentials` 〜 `backstage-github-token-source` の 8 件の `sops decrypt | kubectl apply` を削除
- `ghcr-pat` の apply を削除

### バグ修正1: タブインデント不足
```bash
# Python で sops-age 行にタブを補完
python3 << 'PYEOF'
f = '/home/ccl/platform-infra/k3d/Makefile'
content = open(f).read()
content = content.replace('\n# sops Age秘密鍵', '\n\t# sops Age秘密鍵')
content = content.replace('\nkubectl create secret generic sops-age', '\n\tkubectl create secret generic sops-age')
open(f, 'w').write(content)
PYEOF
```

### バグ修正2: sops-age の挿入位置が誤っていた（install-cilium レシピ末尾に入っていた）
```bash
python3 << 'PYEOF'
f = '/home/ccl/platform-infra/k3d/Makefile'
lines = open(f).readlines()
out = []
i = 0
inserted = False
while i < len(lines):
    if 'sops Age秘密鍵' in lines[i]:
        i += 5
        continue
    if not inserted and '\thelm repo add argocd' in lines[i]:
        out.append('\t# sops Age秘密鍵をargocd namespaceに投入（ksops用）\n')
        out.append('\tkubectl create secret generic sops-age \\\n')
        out.append('\t\t--from-file=keys.txt=$(SOPS_AGE_KEY) \\\n')
        out.append('\t\t-n $(ARGOCD_NS) \\\n')
        out.append('\t\t--dry-run=client -o yaml | kubectl apply -f -\n')
        inserted = True
    out.append(lines[i])
    i += 1
open(f, 'w').writelines(out)
PYEOF
```

### コミット
```bash
cd ~/platform-infra
git add k3d/Makefile
git commit -m "feat: Makefileにsops-age投入追加・source secrets手動apply削除"
git push origin main
```

---

## 3. external-secrets webhook 再有効化

### failurePolicy: Ignore に変更
```yaml
# platform/applications/root-2-auth/external-secrets.yaml
webhook:
  create: true
  failurePolicy: Ignore   # bootstrap race condition 対策
certController:
  create: true
```

```bash
cd ~/platform-gitops
git add platform/applications/root-2-auth/external-secrets.yaml
git commit -m "fix: ESO webhook failurePolicyをIgnoreに変更（bootstrap race condition対策）"
git push origin main
```

---

## 4. ArgoCD ExternalSecret health check 追加

### platform/argocd/values.yaml に追記
```yaml
configs:
  cm:
    resource.customizations.health.external-secrets.io_ExternalSecret: |
      hs = {}
      if obj.status ~= nil then
        if obj.status.conditions ~= nil then
          for i, condition in ipairs(obj.status.conditions) do
            if condition.type == "Ready" and condition.status == "False" then
              hs.status = "Degraded"
              hs.message = condition.message
              return hs
            end
            if condition.type == "Ready" and condition.status == "True" then
              hs.status = "Healthy"
              hs.message = condition.message
              return hs
            end
          end
        end
      end
      hs.status = "Progressing"
      hs.message = "Waiting for ExternalSecret to be ready"
      return hs
```

```bash
cd ~/platform-gitops
git add platform/argocd/values.yaml
git commit -m "fix: ArgoCD に ExternalSecret health check を追加（keycloak bootstrap race condition 対策）"
git push origin main
```

---

## 5. ディレクトリ構造の巻き戻し

前セッションで platform-team / sample-app-team に移動していたリポジトリを元に戻した。

```bash
mv ~/platform-team/platform-infra ~/
mv ~/platform-team/platform-gitops ~/
mv ~/platform-team/platform-charts ~/
mv ~/platform-team/platform-docs ~/
mv ~/sample-app-team/sample-backend ~/
mv ~/sample-app-team/sample-frontend ~/
mv ~/platform-team/.mise.toml ~/.mise.toml
rm ~/sample-app-team/.mise.toml
rmdir ~/platform-team ~/sample-app-team
echo "" > ~/.config/mise/config.toml
direnv allow ~/platform-infra
direnv allow ~/platform-gitops
direnv allow ~/platform-charts
```

---

## 6. mise 設定整理

### platform-infra を source of truth に
```bash
cat > ~/platform-infra/.mise.toml << 'EOF2'
# platform-team 共通ツール定義（source of truth）
# 他リポジトリの .mise.toml はこのファイルへのシンボリックリンク（../platform-infra/.mise.toml）
# クローン時は platform-infra と同階層に配置すること
[tools]
kubectl = "1.35.3"
helm    = "3.20.1"
# ...
EOF2
cd ~/platform-infra
git add .mise.toml
git commit -m "chore: mise.tomlにシンボリックリンク構成の説明コメントを追記"
git push origin main
```

### 他リポジトリにシンボリックリンクを設定
```bash
for repo in platform-gitops platform-charts sample-backend sample-frontend; do
  cd ~/$repo
  ln -sf ../platform-infra/.mise.toml .mise.toml
  git add .mise.toml
  git commit -m "chore: .mise.tomlをplatform-infra/.mise.tomlへのシンボリックリンクに変更"
  git push origin main
done
```

### platform-gitops の ArgoCD out-of-bounds symlinks 制約への対応
ArgoCD 管理リポジトリではシンボリックリンクが使用不可のため通常ファイルに変更。

```bash
# platform-gitops / platform-charts は通常ファイルとして管理
cd ~/platform-gitops
rm .mise.toml
cat > .mise.toml << 'EOF2'
# 共通ツール定義は platform-infra/.mise.toml が source of truth
# ※ ArgoCD管理リポジトリのためシンボリックリンク不可（out-of-bounds symlinks制約）
[tools]
# ... 同内容をコピー
EOF2
git add .mise.toml
git commit -m "fix: ArgoCD out-of-bounds symlinks制約のためシンボリックリンクから通常ファイルに変更"
git push origin main
```

---

## 7. bootstrap 再検証

```bash
cd ~/platform-infra/k3d
make bootstrap
# → 全自動で完了、ArgoCD オールグリーン、Keycloak OIDC ログインまで到達を確認
```
