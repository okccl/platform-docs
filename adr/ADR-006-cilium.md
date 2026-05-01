# ADR-006: CNI の選択（Cilium）

## Context

k3s のデフォルト CNI は flannel であり、Pod 間の基本的な通信は提供されるが、
以下の点で本番環境での運用には機能不足になりやすい。

- NetworkPolicy は別途インストールが必要
- iptables ベースのルーティングはルール数がノード数・Service 数に比例して増加し、
  大規模クラスタでは性能劣化の要因になることが知られている
- パケットレベルの可観測性がなく、通信障害の原因調査が難しい

現場（Tanzu 基盤）では CNI の選択を意識する機会がなかったため、
実務で eBPF ベースの CNI を導入・評価した経験がない。
Phase 11 では検証目的で CNI を置き換え、Phase 13（EKS）での本格評価につなげることにした。

---

## Options

### Option 1: flannel を継続使用する
- k3s のデフォルトであり、追加設定なしに動作する
- シンプルで軽量だが、機能は最小限
- **問題点:** L7 NetworkPolicy が使えない。可観測性がない。
  大規模環境では iptables のルール数爆発が問題になる

### Option 2: Cilium を採用する
- Linux カーネルの eBPF を使いパケット処理をカーネル空間で直接行う
- L7（HTTP パス・メソッド）レベルの NetworkPolicy に対応
- Hubble による Pod 間通信フローのリアルタイム可視化が組み込まれている
- kube-proxy を eBPF で代替することで大規模環境でのスケーラビリティが向上する
- CNCF Graduated プロジェクト

### Option 3: Calico を採用する
- Cilium と並んで本番環境での採用実績が多い CNI
- BGP を使ったネットワーク設計が得意で、オンプレ環境との親和性が高い
- eBPF 対応は後発であり、Cilium ほど成熟していない

---

## Decision

**Cilium v1.19.3 を採用する。**

ただしローカル環境（k3d）では `kubeProxyReplacement=false` とし、
kube-proxy は k3s に任せる構成とする。kube-proxy 完全置き換えは
Phase 13（EKS）で検証する。

---

## Reasons

1. **eBPF による性能向上の可能性**
   iptables ベースの flannel はルール数がノード・Service 数に比例して増加し、
   大規模クラスタでは性能劣化の原因になる。
   eBPF はカーネル空間で直接パケット処理を行うため、この問題を回避できる。
   ただし今回のローカル環境（4ノード）では性能差を定量的に確認できていない。
   性能比較は Phase 13（EKS）で実施する予定。

2. **L7 NetworkPolicy による細かいアクセス制御**
   通常の NetworkPolicy は L4（IP/Port）までだが、
   Cilium は HTTP のパスやメソッドレベルでアクセス制御できる。
   マイクロサービス間の通信制御をより細粒度で設計できる点は、
   本番環境のセキュリティ設計において有効である。

3. **Hubble による可観測性**
   Pod 間の通信フローをリアルタイムで可視化できる Hubble が組み込まれている。
   「なぜ通信が失敗しているか」の調査が容易になり、
   障害対応コストの削減につながる。

4. **クラウド環境での評価につなげる**
   EKS・GKE での Cilium 採用事例は増えており、
   Phase 13・14 での検証に向けてローカルで構造を把握しておく意図がある。

---

## Consequences

**ポジティブ:**
- L7 NetworkPolicy によるきめ細かいアクセス制御が可能になる
- Hubble による通信フローの可視化でネットワーク障害の調査が容易になる
- Phase 13（EKS）での kube-proxy 完全置き換え検証の土台ができた

**ネガティブ・トレードオフ:**
- ローカル環境（k3d）では `kubeProxyReplacement=true` が使えず、
  eBPF による kube-proxy 代替の恩恵を受けられていない
- flannel と比べてインストール・設定が複雑で、bootstrap の順序制御が必要になった
  （`cluster-create → install-cilium → fix-coredns` の順序を守らないと CoreDNS が起動しない）
- 今回のローカル環境では性能差を定量的に確認できていない。
  flannel から Cilium への置き換えによる実測値の比較は今後の課題である
