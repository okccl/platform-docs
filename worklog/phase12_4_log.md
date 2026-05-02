# Phase 12-4 作業ログ

## 概要
Backstage を Helm Chart でデプロイし、`backstage.platform.local` で公開した。

---

## 実行コマンド

### 1. SOPS Secret の作成

```bash
# 古い Secret を削除（Step 1 で誤って作成したもの）
kubectl delete secret backstage-secret -n argocd

# DB 認証情報の Secret テンプレート作成（username: app、password は openssl rand -hex 16 で生成）
cat > ~/platform-gitops/secrets/templates/backstage-db-credentials-source.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: backstage-db-credentials-source
  namespace: argocd
type: Opaque
stringData:
  username: app
  password: "<生成した値>"
EOF

# BACKEND_SECRET テンプレート作成（openssl rand -hex 32 で生成）
cat > ~/platform-gitops/secrets/templates/backstage-secret-source.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: backstage-secret-source
  namespace: argocd
type: Opaque
stringData:
  BACKEND_SECRET: "<生成した値>"
EOF

# SOPS で暗号化
AGE_KEY=$(cat ~/.config/sops/age/keys.txt | grep "public key" | awk '{print $4}')
sops --encrypt --age $AGE_KEY \
  ~/platform-gitops/secrets/templates/backstage-db-credentials-source.yaml \
  > ~/platform-gitops/secrets/encrypted/backstage-db-credentials-source.yaml
sops --encrypt --age $AGE_KEY \
  ~/platform-gitops/secrets/templates/backstage-secret-source.yaml \
  > ~/platform-gitops/secrets/encrypted/backstage-secret-source.yaml

# argocd namespace に適用
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops decrypt \
  ~/platform-gitops/secrets/encrypted/backstage-db-credentials-source.yaml \
  | kubectl apply -f -
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops decrypt \
  ~/platform-gitops/secrets/encrypted/backstage-secret-source.yaml \
  | kubectl apply -f -
```

### 2. Git ファイルの作成

```bash
mkdir -p ~/platform-gitops/platform/backstage/db-config
mkdir -p ~/platform-gitops/platform/backstage/app-config
mkdir -p ~/platform-gitops/platform/backstage/config
```

`platform/backstage/db-config/external-secret.yaml`、`cluster.yaml`、
`platform/backstage/app-config/external-secret.yaml`、`reference-grant.yaml`、
`platform/backstage/backstage-app.yaml`、`backstage-db-app.yaml`、`values.yaml`、
`platform/gateway/config/httproute-backstage.yaml` を作成。

### 3. root-app.yaml の exclude に backstage を追加

```bash
# backstage/db-config/* と backstage/app-config/* を exclude に追加
sed -i 's|...|...|' ~/platform-gitops/platform/argocd/root-app.yaml
```

### 4. Kyverno 対応：platform-namespaces.yaml に backstage を追加

```bash
cat >> ~/platform-gitops/platform/policy/platform-namespaces.yaml << 'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: backstage
  labels:
    platform-managed: "true"
EOF
```

### 5. CoreDNS に backstage.platform.local の rewrite を追加

```bash
# Makefile の bootstrap-sync ターゲットに backstage の rewrite ルールを追加（platform-infra）
# 即時反映
EG_SVC=$(kubectl get svc -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -o jsonpath='{.items[0].metadata.name}')
kubectl get configmap coredns -n kube-system -o json \
  | python3 -c "import sys,json
d=json.load(sys.stdin)
cf=d['data']['Corefile']
rule='    rewrite name backstage.platform.local ${EG_SVC}.envoy-gateway-system.svc.cluster.local\n'
d['data']['Corefile']=cf.replace('ready\n', 'ready\n'+rule) if 'rewrite name backstage.platform.local' not in cf else cf
print(json.dumps(d))" \
  | kubectl apply -f -
kubectl rollout restart deployment coredns -n kube-system
kubectl rollout status deployment coredns -n kube-system
```

### 6. Git push

```bash
cd ~/platform-gitops
git add platform/backstage/ platform/gateway/config/httproute-backstage.yaml
git commit -m "feat(phase12-4): add Backstage deployment"
git push origin main

git add secrets/encrypted/backstage-db-credentials-source.yaml \
        secrets/encrypted/backstage-secret-source.yaml
git commit -m "feat(phase12-4): add encrypted secrets for Backstage"
git push origin main

git add platform/argocd/root-app.yaml
git commit -m "fix(phase12-4): exclude backstage sub-dirs from root app"
git push origin main

git add platform/policy/platform-namespaces.yaml
git commit -m "fix(policy): add backstage to platform-managed namespaces"
git push origin main
```

### 7. トラブルシュート：CNPG デッドロック

```bash
# PVC が WaitForFirstConsumer のまま Pod が作られず CNPG がデッドロック
# → Cluster を削除して ArgoCD に再作成させる
kubectl delete cluster backstage-db -n backstage
argocd app sync backstage-db --server-side --grpc-web
```

### 8. トラブルシュート：CREATEDB 権限エラー

```bash
# Backstage が plugin ごとに DB を作成しようとするが app ユーザーに権限なし
# → cluster.yaml の postInitSQL で権限を付与
# platform/backstage/db-config/cluster.yaml に以下を追加：
#   bootstrap.initdb.postInitSQL:
#     - ALTER ROLE app CREATEDB;

# CNPG Cluster を再作成
kubectl delete cluster backstage-db -n backstage
argocd app sync backstage-db --server-side --grpc-web

# Backstage を再起動
kubectl rollout restart deployment backstage -n backstage
kubectl rollout status deployment backstage -n backstage --timeout=120s
```

### 9. トラブルシュート：CSP upgrade-insecure-requests による白画面

```bash
# HTTP 環境で upgrade-insecure-requests が原因でブラウザがスクリプトをブロック
# → values.yaml に以下を追加：
#   backend.csp.upgrade-insecure-requests: false

git add platform/backstage/values.yaml
git commit -m "fix(phase12-4): disable CSP upgrade-insecure-requests for HTTP environment"
git push origin main
argocd app sync backstage --server-side --grpc-web
```

### 10. トラブルシュート：ESO ignoreDifferences

```bash
# ESO がデフォルト値を自動付与するため ArgoCD が OutOfSync と判定する既知問題
# backstage-app.yaml と backstage-db-app.yaml に ignoreDifferences を追加

git add platform/backstage/backstage-app.yaml platform/backstage/backstage-db-app.yaml
git commit -m "fix(phase12-4): ignore ESO default fields in ExternalSecret diff"
git push origin main
argocd app sync backstage backstage-db --server-side --grpc-web
```

---

## 最終確認

```bash
argocd app list --grpc-web | grep backstage
kubectl get cluster,pod,pvc -n backstage
```

## 結果

| コンポーネント | namespace | 状態 |
|---|---|---|
| backstage-db | backstage | Synced / Healthy |
| backstage | backstage | Synced / Healthy |
| backstage.platform.local | - | HTTP アクセス確認済み |
