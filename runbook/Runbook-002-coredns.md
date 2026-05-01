# Runbook-002: CoreDNS への host.k3d.internal 登録

## 概要

k3d クラスタ内から WSL 上の外部 MinIO（`host.k3d.internal`）への
名前解決ができない問題の原因と対処手順。

---

## 症状

```bash
# クラスタ内から外部 MinIO への疎通確認
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://host.k3d.internal:9000

# → Could not resolve host: host.k3d.internal
```

---

## 原因

k3d はクラスタネットワークとして Docker ネットワーク（`k3d-dev`）を作成する。
このネットワークのゲートウェイ IP が WSL ホストへの出口になるが、
CoreDNS はデフォルトでこのホスト名を解決できない。

```
クラスタ内 Pod → CoreDNS → host.k3d.internal の解決失敗
                           ↑ NodeHosts に登録されていない
```

---

## 対処手順

### 自動対処（bootstrap 時）

`make bootstrap` に `fix-coredns` ターゲットが組み込まれているため、
クラスタ再作成時は自動で適用される。

```bash
cd ~/platform-infra/k3d
make bootstrap
```

### 手動対処（既存クラスタへの適用）

```bash
cd ~/platform-infra/k3d
make fix-coredns
```

`fix-coredns` は以下の処理を行う。

```bash
# k3d ネットワークのゲートウェイ IP を動的に取得
K3D_GW=$(docker network inspect k3d-dev \
  --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}')

# CoreDNS ConfigMap の NodeHosts に追記（冪等）
kubectl get configmap coredns -n kube-system -o json \
  | python3 -c "..." \   # host.k3d.internal が未登録の場合のみ追記
  | kubectl apply -f -

# CoreDNS を再起動して設定を反映
kubectl rollout restart deployment coredns -n kube-system
kubectl rollout status deployment coredns -n kube-system
```

### 動作確認

```bash
# 名前解決の確認
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://host.k3d.internal:9000/minio/health/live

# → HTTP 200 が返れば成功
```

---

## 注意事項

- k3d ネットワークのゲートウェイ IP はクラスタを再作成するたびに変わる可能性がある。
  `fix-coredns` は毎回動的に IP を取得するため、IP 変更に対して冪等に動作する
- CoreDNS の NodeHosts は k3s が管理する ConfigMap のため ArgoCD で管理しない。
  bootstrap スクリプトに組み込む形で冪等性を担保している
- `host.k3d.internal` がすでに登録済みの場合は追記をスキップする設計になっている
