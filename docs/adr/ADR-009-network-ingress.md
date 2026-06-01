# ADR-009: ネットワーク・Ingress設計（Gateway API / Envoy Gateway / Cilium）

## Context

クラスタのネットワーク層では、CNIの選定とIngress実装の選定という2つの意思決定が発生した。

CNIについては、現職（Tanzu）ではAntreaを使用しているが、これはVMware環境固有の選択であり、ポートフォリオで同じCNIを使う理由はなかった。将来のOpenShift移行後もデフォルトCNIを使うと予想されるため、現場のキャリアパス上でCiliumが必須になる状況は考えにくい。それでも採用した出発点は単純な興味だが、調べていくとGKE Dataplane V2がCiliumをバックエンドに採用していることがわかり、将来的にローカルk3d環境を開発環境・本番はGKEという構成を想定している観点から、先行して検証する意義を見いだした。

Ingress実装については、ingress-nginxで構築した後にKubernetes公式のメンテナンス終了とGateway APIへの移行推奨を知ったことが移行のきっかけになった。現場でもNSX Advanced Load BalancerによるIngress運用が現役で、OpenShift移行後も独自のRoute（Ingressベース）を使うとされており、技術自体がすぐ廃止されるわけではない。ただし中期的にはGateway API移行タスクが現場でも発生しうると判断し、ポートフォリオで先行検証することにした。

## Options

### CNI

**Option 1: flannel（k3dデフォルト）**
- **Pros:** k3dに同梱されており設定不要。
- **Cons:** シンプルなVXLANオーバーレイ実装であり、ポートフォリオの技術検証目的では選択する積極的な理由がない。

**Option 2: Antrea（現職：Tanzu）**
- **Pros:** 現職での実運用経験がある。
- **Cons:** VMware/Tanzu環境に特化した選択であり、ポートフォリオで使う理由がない。

**Option 3: Cilium**
- **Pros:** eBPFによるカーネルレベルのネットワーク処理。GKE Dataplane V2がCiliumをバックエンドに採用しており、将来のGKE移行で設計知識を活かせる可能性がある。
- **Cons:** ローカル環境ではクラスタ起動手順が複雑化し、bootstrapの改修コストが高い。flannelに比べてリソース消費が増加する。

### Ingress実装

**Option 1: ingress-nginx**
- **Pros:** デファクトスタンダードで情報が豊富。Ingressの仕組み自体は現行Kubernetesでも機能しており、現場でも類似構成（NSX ALB）が現役。
- **Cons:** Kubernetes公式によるメンテナンスが終了し、Gateway APIへの移行が推奨されている。

**Option 2: Gateway API + Envoy Gateway**
- **Pros:** Ingressの後継として標準化が進んでいる。Envoy GatewayはCNCF公式プロジェクト。
- **Cons:** Ingressとはリソース定義の考え方が異なり、移行に学習コストがかかる。

**Option 3: Istio（Gateway APIサポートあり）**
- **Pros:** Gateway API実装に加えてサービスメッシュ機能も統合できる。
- **Cons:** Ingress/Gateway目的のみでの採用はスコープが大きすぎる。現段階で必要な機能ではない。

## Decision

**CNIにCiliumを採用する。Ingress実装はingress-nginxを廃止し、Gateway API + Envoy Gatewayに移行する。**

## Reasons

### flannelを選ばなかった理由

k3dのデフォルトだが、技術検証を目的としたポートフォリオでデフォルトのまま使い続ける理由が薄かった。将来のGKE移行を念頭に、Ciliumによる先行検証を優先した。

### Ciliumを採用した理由

出発点は単純な興味だが、GKE Dataplane V2がCiliumをバックエンドに採用しているという事実が採用を後押しした。将来的に本番環境をGKEに移行することを想定しており、ローカル段階でCiliumの設計を把握しておくことには先行検証としての価値がある。ただし現段階では技術検証としての価値のみで、実際の運用メリットが明確になるのは今後のクラウド移行フェーズになる。

現職（Tanzu/Antrea）でCiliumが直接活きる場面は現時点ではないが、GKEを含むクラウドネイティブ環境を扱う場合はCNI選定の文脈が発生しうる。

### ingress-nginxを廃止した理由

構築後にKubernetes公式のメンテナンス終了とGateway APIへの移行推奨を知ったことが直接のきっかけ。Ingressの仕組み自体はまだ機能しており、現場でもNSX ALBやOpenShift Routeという形でIngressベースの構成が現役のため、即座の対応は必須ではないという認識は変わっていない。ただし中期的に現場でもGateway APIへの移行が発生しうると見込み、ポートフォリオで先行して動かし知見を積んでおくと判断した。

### Envoy Gatewayを採用した理由

CNCFの公式プロジェクトとして位置づけられており、採用の根拠として自然だった。Istioはサービスメッシュとしての機能は認知しているが、Ingress/Gateway目的だけで採用するのはスコープが大きすぎると判断した。

## Consequences

**ポジティブ:**
- Gateway APIのリソース定義（GatewayClass / Gateway / HTTPRoute）を実際に動かして理解できた。将来の現場でのGateway API移行タスクへの備えになっている。
- GKE Dataplane V2との親和性という観点でCiliumの基本的な設計を把握できた。

**ネガティブ・トレードオフ:**
- Ciliumの導入によりローカルクラスタの起動手順が複雑化した。k3d再起動時にCilium eBPFマップがstaleになりClusterIP通信が断絶する問題が発生し、`make cluster-start`へのCiliumNode全削除処理の組み込みが必要になった。bootstrapの設計・改修・トラブルシュートのコストが高かった。
- flannelと比べてリソース消費が増加し、k3dの軽量さを一部損なっている。現状は許容範囲だが、ローカルリソースが逼迫した場合の制約になりうる。
- Gateway APIはIngressとリソース定義の考え方が異なり（GatewayClassによる実装の抽象化など）、移行に学習コストがかかった。学習目的としては達成できている。
