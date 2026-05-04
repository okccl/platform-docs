# Phase 12 クリーンアップ 作業ログ（セッション A2）

## 課題 D：HTTPS / TLS 対応

### cert-manager 設定の移動

```bash
# 新ディレクトリ作成
mkdir -p ~/platform-gitops/platform/cert-manager/config

# 既存ファイルをコピー
cp ~/platform-gitops/platform/ingress/config/cluster-issuer.yaml \
   ~/platform-gitops/platform/cert-manager/config/cluster-issuer.yaml

# ワイルドカード Certificate を追記
cat >> ~/platform-gitops/platform/cert-manager/config/cluster-issuer.yaml << 'EOF'
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: platform-local-tls
  namespace: envoy-gateway-system
  annotations:
    argocd.argoproj.io/sync-wave: "4"
spec:
  secretName: platform-local-tls
  issuerRef:
    name: local-ca-issuer
    kind: ClusterIssuer
    group: cert-manager.io
  commonName: "*.platform.local"
  dnsNames:
    - "*.platform.local"
EOF

# cert-manager-config Application の参照パス変更
sed -i 's|path: platform/ingress/config|path: platform/cert-manager/config|' \
  ~/platform-gitops/platform/applications/cert-manager-config.yaml

# 旧ファイル削除
rm ~/platform-gitops/platform/ingress/config/cluster-issuer.yaml
```

### Gateway に HTTPS リスナーを追加

```bash
cat > ~/platform-gitops/platform/gateway/config/gateway.yaml << 'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "4"
  name: eg
  namespace: envoy-gateway-system
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: platform-local-tls
            namespace: envoy-gateway-system
      allowedRoutes:
        namespaces:
          from: All
EOF
```

### 全 HTTPRoute に sectionName: https を追加

```bash
sed -i 's/      namespace: envoy-gateway-system/      namespace: envoy-gateway-system\n      sectionName: https/' \
  ~/platform-gitops/platform/gateway/config/httproute-*.yaml
```

### ReferenceGrant をファイル分割（httproute-rollouts.yaml から切り出し）

```bash
# httproute-rollouts.yaml を正しい内容に修正
cat > ~/platform-gitops/platform/gateway/config/httproute-rollouts.yaml << 'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "4"
  name: rollouts-dashboard
  namespace: envoy-gateway-system
spec:
  parentRefs:
    - name: eg
      namespace: envoy-gateway-system
      sectionName: https
  hostnames:
    - "rollouts.platform.local"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - kind: Service
          name: argo-rollouts-dashboard
          namespace: argo-rollouts
          port: 3100
EOF

# ReferenceGrant を独立ファイルに切り出し
cat > ~/platform-gitops/platform/gateway/config/reference-grant-argo-rollouts.yaml << 'EOF'
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "4"
  name: reference-grant-argo-rollouts
  namespace: argo-rollouts
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: envoy-gateway-system
  to:
    - group: ""
      kind: Service
EOF
```

### gateway-config の ignoreDifferences に Gateway certificateRefs を追加

```bash
# gateway-config.yaml に ignoreDifferences を追加
# .spec.listeners[]?.tls?.certificateRefs[]?.group
# .spec.listeners[]?.tls?.certificateRefs[]?.kind
```

### HTTP→HTTPS リダイレクト追加

```bash
cat > ~/platform-gitops/platform/gateway/config/httproute-redirect.yaml << 'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "4"
  name: http-to-https-redirect
  namespace: envoy-gateway-system
spec:
  parentRefs:
    - name: eg
      namespace: envoy-gateway-system
      sectionName: http
  hostnames:
    - "*.platform.local"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
EOF
```

### 残存リソースの削除

```bash
# 前セッションの残骸を削除
kubectl delete referencegrant reference-grant-argo-rollouts -n argo-rollouts
```

### push・同期

```bash
cd ~/platform-gitops
git add platform/cert-manager/ \
        platform/ingress/config/cluster-issuer.yaml \
        platform/applications/cert-manager-config.yaml \
        platform/gateway/config/
git commit -m "feat(gateway): HTTPS対応・cert-manager設定をplatform/cert-manager/configへ移動・ReferenceGrantをファイル分割"
git push origin main

argocd app sync cert-manager-config gateway-config --grpc-web
```

---

## 課題 E：Keycloak Realm の宣言的管理

### platform-admin パスワード用 source secret の作成

```bash
kubectl create secret generic keycloak-platform-admin-source \
  -n argocd \
  --from-literal=password=admin123
```

### ファイル作成

```bash
mkdir -p ~/platform-gitops/platform/keycloak/config-cli

# ExternalSecret
cat > ~/platform-gitops/platform/keycloak/config-cli/external-secret.yaml << 'EOF'
# （省略：argocd-keycloak-secret-source / backstage-keycloak-secret-source / keycloak-platform-admin-source を参照）
EOF

# Realm 定義 ConfigMap
cat > ~/platform-gitops/platform/keycloak/config-cli/realm-configmap.yaml << 'EOF'
# （省略：platform Realm / argocd・backstage Client / argocd-admins Group / platform-admin User）
EOF

# values.yaml（config-cli ディレクトリ外に配置）
cat > ~/platform-gitops/platform/keycloak/config-cli-values.yaml << 'EOF'
# KEYCLOAK_URL: http://keycloak-keycloakx-http.keycloak.svc.cluster.local:80/auth
# IMPORT_FILES_LOCATIONS: /config/*
# IMPORT_VAR_SUBSTITUTION_ENABLED: "true"
# existingSecret: keycloak-admin-credentials
# existingConfigMap: keycloak-config-cli-realm
EOF

# Application yaml
cat > ~/platform-gitops/platform/applications/keycloak-config-cli.yaml << 'EOF'
# wave: 6 / chart: jkroepke/keycloak-config-cli 1.3.7
EOF
```

### push・同期

```bash
cd ~/platform-gitops
git add platform/keycloak/ platform/applications/keycloak-config-cli.yaml
git commit -m "feat(keycloak): keycloak-config-cliによるRealm宣言的管理を追加"
git push origin main

argocd app sync argocd-apps --grpc-web
```

### ArgoCD OIDC 設定の修正

```bash
# issuer を https に変更
sed -i 's|issuer: http://keycloak.platform.local/auth/realms/platform|issuer: https://keycloak.platform.local/auth/realms/platform|' \
  ~/platform-gitops/platform/argocd/values.yaml

# argocd-cm url を https に変更
sed -i 's|url: "http://argocd.platform.local"|url: "https://argocd.platform.local"|' \
  ~/platform-gitops/platform/argocd/values.yaml

# ローカル CA 証明書を rootCA として埋め込み
CA_CERT=$(kubectl get secret local-ca-secret -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d)
# values.yaml の insecureSkipVerify: true を rootCA ブロックに置き換え（Python スクリプトで実施）
```

### Backstage metadataUrl の修正

```bash
sed -i 's|metadataUrl: http://keycloak.platform.local/realms/platform|metadataUrl: http://keycloak.platform.local/auth/realms/platform|' \
  ~/platform-gitops/platform/backstage/values.yaml
```

### push・同期

```bash
cd ~/platform-gitops
git add platform/argocd/values.yaml platform/backstage/values.yaml
git commit -m "fix(argocd,backstage): Keycloak URLパスを修正・ローカルCA証明書を登録"
git push origin main

argocd app sync argocd backstage --grpc-web
kubectl rollout restart deployment argocd-server -n argocd
```

---

## 課題 F-1：Backstage catalog-info.yaml

### 各リポジトリに catalog-info.yaml を追加

```bash
# 各リポジトリのルートに catalog-info.yaml を作成
# （sample-backend / sample-frontend / platform-infra / platform-charts / platform-docs）

for repo in sample-backend sample-frontend platform-infra platform-charts platform-docs; do
  cd ~/$repo
  git add catalog-info.yaml
  git commit -m "feat(backstage): catalog-info.yamlを追加"
  git push origin main
done

# platform-gitops は System 定義も含む
cd ~/platform-gitops
git add catalog-info.yaml
git commit -m "feat(backstage): catalog-info.yamlとSystemを追加"
git push origin main
```

### Backstage values.yaml に catalog locations を追加

```bash
# platform/backstage/values.yaml の catalog.locations に6リポジトリの catalog-info.yaml を追記

cd ~/platform-gitops
git add platform/backstage/values.yaml
git commit -m "feat(backstage): カタログlocationsに全リポジトリのcatalog-info.yamlを追加"
git push origin main

argocd app sync backstage --grpc-web
```
