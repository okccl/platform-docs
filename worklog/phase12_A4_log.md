# Phase 12 作業ログ（セッション4）

## 実施日
2026-05-05

---

## Step 1: local-ca-secret の SOPS 管理化

```bash
# 現在の local-ca-secret を平文で書き出し
kubectl get secret local-ca-secret -n cert-manager -o yaml \
  | python3 -c "
import sys, yaml
d = yaml.safe_load(sys.stdin)
for k in ['creationTimestamp','resourceVersion','uid','annotations','managedFields']:
    d['metadata'].pop(k, None)
print(yaml.dump(d, default_flow_style=False))
" > secrets/encrypted/local-ca-secret.yaml

# SOPS で暗号化
cd ~/platform-gitops
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops encrypt --in-place secrets/encrypted/local-ca-secret.yaml

# 復号確認
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops decrypt secrets/encrypted/local-ca-secret.yaml \
  | grep "^kind\|^  name\|^  namespace\|^type"

git add secrets/encrypted/local-ca-secret.yaml
git commit -m "feat: SOPS管理化 local-ca-secret"
git push
```

---

## Step 2: Makefile 修正（CA 事前投入・argocd-local-ca 作成）

```bash
# bootstrap-argocd に CA 事前投入を追加（Python で編集）
# 追加内容：
#   - cert-manager namespace 作成
#   - local-ca-secret 事前投入
#   - argocd namespace 作成
#   - argocd-local-ca Secret 作成（ラベル付き）

# bootstrap-sync から update-root-ca.sh 呼び出しを削除

cd ~/platform-infra
git add k3d/Makefile
git commit -m "feat: local-ca SOPS固定管理・update-root-ca.sh廃止"
git push
```

---

## Step 3: update-root-ca.sh の削除

```bash
rm ~/platform-infra/scripts/update-root-ca.sh

cd ~/platform-infra
git add scripts/update-root-ca.sh
git commit -m "chore: update-root-ca.sh 削除（SOPS固定管理に移行）"
git push
```

---

## Step 4: Certificate を GitOps 管理から除外

```bash
# cluster-issuer.yaml から local-ca Certificate リソースを削除
# （local-ca-secret を SOPS 静的管理するため）
cat > ~/platform-gitops/platform/cert-manager/config/cluster-issuer.yaml << 'EOF'
# selfsigned-issuer / local-ca-issuer / platform-local-tls Certificate のみ残す
# local-ca Certificate は削除
EOF

cd ~/platform-gitops
git add platform/cert-manager/config/cluster-issuer.yaml
git commit -m "feat: local-ca Certificate を GitOps管理から除外（SOPS静的管理に移行）"
git push

# ArgoCD で sync
argocd app sync cert-manager-config --server-side

# Certificate が削除されたことを確認
kubectl get certificate -n cert-manager
```

---

## Step 5: argocd-local-ca を現在の CA で更新・ラベル付与

```bash
# argocd-local-ca を現在の local-ca-secret で更新
kubectl get secret local-ca-secret -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d \
  | kubectl create secret generic argocd-local-ca \
    -n argocd \
    --from-file=ca.crt=/dev/stdin \
    --dry-run=client -o yaml | kubectl apply -f -

# ラベル付与（ArgoCD が Secret を認識するために必要）
kubectl label secret argocd-local-ca -n argocd app.kubernetes.io/part-of=argocd

# ArgoCD server 再起動
kubectl rollout restart deployment/argocd-server -n argocd
kubectl rollout status deployment/argocd-server -n argocd --timeout=120s

# SOPS 管理の local-ca-secret.yaml を現在の CA で更新
cd ~/platform-gitops
kubectl get secret local-ca-secret -n cert-manager -o yaml \
  | python3 -c "
import sys, yaml
d = yaml.safe_load(sys.stdin)
for k in ['creationTimestamp','resourceVersion','uid','annotations','managedFields']:
    d['metadata'].pop(k, None)
print(yaml.dump(d, default_flow_style=False))
" > secrets/encrypted/local-ca-secret.yaml
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops encrypt --in-place secrets/encrypted/local-ca-secret.yaml

git add secrets/encrypted/local-ca-secret.yaml
git commit -m "fix: local-ca-secret を現在のCA（12:43生成）で更新"
git push
```

---

## Step 6: Makefile に argocd-local-ca ラベル付与を組み込み

```bash
# argocd-local-ca 作成時に kubectl label --local でラベルを付与するよう変更
cd ~/platform-infra
git add k3d/Makefile
git commit -m "fix: argocd-local-ca に app.kubernetes.io/part-of=argocd ラベルを付与"
git push
```

---

## Step 7: keycloak ExternalSecret・ReferenceGrant を wave -1 に変更

```bash
# keycloak/db-config/external-secret.yaml の wave を -1 に変更
# keycloak/app-config/reference-grant.yaml の wave を -1 に変更

cd ~/platform-gitops
git add platform/keycloak/db-config/external-secret.yaml
git add platform/keycloak/app-config/reference-grant.yaml
git commit -m "fix: keycloak ExternalSecret・ReferenceGrant を wave -1 に変更"
git push
```

---

## Step 8: App-of-Apps を3分割・Makefile で順次制御

```bash
# group-roots の sync-wave アノテーションを削除
# root-app.yaml を削除
# Makefile bootstrap-sync を3つの順次実行に変更：
#   gateway-stack apply → 完了待機
#   auth-stack apply → 完了待機
#   platform apply

cd ~/platform-gitops
git rm platform/argocd/root-app.yaml
git add platform/group-roots/
git commit -m "feat: App-of-Apps を3つに分割・root廃止（bootstrap順序をMakefileで明示制御）"
git push

cd ~/platform-infra
git add k3d/Makefile
git commit -m "feat: bootstrap-sync をgateway-stack→auth-stack→platform の順次実行に変更"
git push
```

---

## Step 9: Application 名・ファイル名・ディレクトリ名を root-名称に統一

```bash
# group-roots の name フィールドを変更
sed -i 's/^  name: gateway-stack$/  name: root-1-gateway/' ~/platform-gitops/platform/group-roots/gateway-stack.yaml
sed -i 's/^  name: auth-stack$/  name: root-2-auth/' ~/platform-gitops/platform/group-roots/auth-stack.yaml
sed -i 's/^  name: platform$/  name: root-3-others/' ~/platform-gitops/platform/group-roots/platform.yaml

# ファイル名変更
cd ~/platform-gitops/platform/group-roots
mv gateway-stack.yaml root-1-gateway.yaml
mv auth-stack.yaml root-2-auth.yaml
mv platform.yaml root-3-others.yaml

# applications ディレクトリ名変更
cd ~/platform-gitops/platform/applications
mv gateway-stack root-1-gateway
mv auth-stack root-2-auth
mv platform root-3-others

# group-roots の path 参照を更新
sed -i 's|platform/applications/gateway-stack|platform/applications/root-1-gateway|' ~/platform-gitops/platform/group-roots/root-1-gateway.yaml
sed -i 's|platform/applications/auth-stack|platform/applications/root-2-auth|' ~/platform-gitops/platform/group-roots/root-2-auth.yaml
sed -i 's|platform/applications/platform|platform/applications/root-3-others|' ~/platform-gitops/platform/group-roots/root-3-others.yaml

cd ~/platform-gitops
git add platform/group-roots/ platform/applications/
git commit -m "feat: app-of-apps を root-1-gateway / root-2-auth / root-3-others にリネーム・ディレクトリ統一"
git push
```

---

## Step 10: gateway routes を分割

```bash
cd ~/platform-gitops

# 新ディレクトリ作成
mkdir -p platform/gateway/routes/keycloak
mkdir -p platform/gateway/routes/platform

# keycloak ルートを移動
mv platform/gateway/config/httproute-keycloak.yaml platform/gateway/routes/keycloak/

# platform ルートを移動
mv platform/gateway/config/httproute-backstage.yaml platform/gateway/routes/platform/
mv platform/gateway/config/httproute-goldilocks.yaml platform/gateway/routes/platform/
mv platform/gateway/config/httproute-grafana.yaml platform/gateway/routes/platform/
mv platform/gateway/config/httproute-rollouts.yaml platform/gateway/routes/platform/
mv platform/gateway/config/reference-grant-monitoring.yaml platform/gateway/routes/platform/
mv platform/gateway/config/reference-grant-argo-rollouts.yaml platform/gateway/routes/platform/

# keycloak-routes Application を root-2-auth に追加
# gateway-routes Application を root-3-others に追加

git add platform/gateway/ platform/applications/
git commit -m "feat: gateway routes を分割（keycloak→root-2-auth / platform→root-3-others）"
git push
```

---

## Step 11: bootstrap の各種修正

```bash
# Envoy Gateway Pod 存在確認ループを追加
# argocd.platform.local 疎通待機ループを追加
# root-2-auth 完了待機に --sync フラグを追加
# keycloak・keycloak-config-cli の Health 待機を追加

cd ~/platform-infra
git add k3d/Makefile
git commit -m "fix: bootstrap-sync の各種待機処理を改善"
git push
```

---

## Step 12: ignoreDifferences を keycloak-routes・gateway-routes に追加

```bash
cd ~/platform-gitops
git add platform/applications/root-2-auth/keycloak-routes.yaml
git add platform/applications/root-3-others/gateway-routes.yaml
git commit -m "fix: keycloak-routes・gateway-routes に ignoreDifferences を追加"
git push
```

---

## Step 13: mise の共通ツール管理を親ディレクトリに移管

```bash
# ディレクトリ構造を変更
cd ~
mkdir -p platform-team sample-app-team
mv platform-infra platform-team/
mv platform-gitops platform-team/
mv platform-charts platform-team/
mv platform-docs platform-team/
mv sample-backend sample-app-team/
mv sample-frontend sample-app-team/

# 親ディレクトリに .mise.toml を作成
cat > ~/platform-team/.mise.toml << 'EOF'
[tools]
kubectl = "1.35.3"
helm    = "3.20.1"
k3d     = "5.8.3"
argocd  = "3.2.9"
sops    = "3.12.2"
age     = "1.3.1"
gh      = "2.87.2"
"ubi:argoproj/argo-rollouts" = { version = "1.8.3", exe = "kubectl-argo-rollouts" }
"github:hatoo/oha" = "1.14.0"
EOF

cat > ~/sample-app-team/.mise.toml << 'EOF'
[tools]
gh = "2.87.2"
EOF

# グローバル設定をクリア
cat > ~/.config/mise/config.toml << 'EOF'
# グローバル共通ツールはここに定義する
# 現在は platform-team / sample-app-team の親 .mise.toml で管理
EOF

# 各リポジトリの .mise.toml をコメントのみに整理してコミット
# direnv allow を実行
direnv allow ~/platform-team/platform-infra
direnv allow ~/platform-team/platform-gitops
direnv allow ~/platform-team/platform-charts
```
