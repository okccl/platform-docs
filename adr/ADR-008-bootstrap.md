# ADR-008: bootstrap 順序制御の設計（App-of-Apps 分割と Makefile による明示的制御）

## Context

プラットフォームの bootstrap では、コンポーネント間に明確な依存関係がある。
cert-manager → Envoy Gateway → Keycloak → Backstage という順で起動する必要があり、
前段のコンポーネントが Ready になる前に後段を sync しようとすると
webhook 未登録・namespace 未作成などのエラーが発生する。

当初は ArgoCD の App-of-Apps パターンを2段階で構成し、
`sync-wave` アノテーションで順序を制御しようとしていた。

```
root（App-of-Apps）
└── group-roots/（App-of-Apps × 3：wave 1/2/3）
    └── applications/（実際の App）
```

group-roots に `sync-wave: "1"`, `"2"`, `"3"` を設定することで
gateway-stack → auth-stack → platform の順序を期待していたが、
実際には想定通りに動かないケースが発生した。

トラブルシュートの過程で、ArgoCD の wave の動作について重要な制約を理解した。
**wave が保証するのは「root が sync する際の Application リソースの作成順序」のみであり、
各 Application に `automated: selfHeal: true` が設定されている場合、
一度作成された後はそれぞれが独立して自動 sync を走らせる。**
つまり root による初回 sync では正しく順序制御されるが、
その後 auth-stack と platform が並行して自動 sync を走らせるため、
CNPG の webhook が完全に起動する前に backstage-db が sync されるといった
タイミング問題が発生していた。

加えて、既存の構成（root → group-roots → applications）は App-of-Apps が
2段階になっており、wave による順序制御を機能させようとするほど
構造が複雑になっていた。group-roots の存在意義を整理したところ、
「bootstrap の初回実行時にだけ順序制御が必要」という事実に気づき、
それならば ArgoCD の wave に頼るより Makefile で明示的に制御する方が
シンプルかつ確実だという結論に至った。

---

## Options

### Option 1: wave の設定を精緻化して順序を制御し続ける

- group-roots の wave を調整し、health check の条件を厳格化する
- **問題点:** `automated: selfHeal: true` がある限り、
  初回 sync 後の並行実行は防げない。wave は「いつ Application リソースを作るか」
  を制御するだけで「いつ Application の中身が健全になるか」は制御できない。
  根本的な解決にならない。

### Option 2: App-of-Apps を3つに分割し、Makefile で順次実行する

- root を廃止し、root-1-gateway / root-2-auth / root-3-others の
  3つの App-of-Apps を作成する
- Makefile がそれぞれを順番に apply し、完了（Synced かつ Healthy）を
  確認してから次を実行する
- 完了判定は ArgoCD の health check ではなく、重要コンポーネント
  （keycloak, keycloak-config-cli）の Pod 起動を個別に待機する

### Option 3: automated sync を無効にして Makefile がすべての sync を制御する

- 各 Application の `automated` を削除し、Makefile の中でコンポーネントごとに
  `argocd app sync` を順番に実行する
- **問題点:** bootstrap 完了後の自動修復（selfHeal）が失われる。
  誰かが直接 kubectl でリソースを変更してもドリフトが自動修正されなくなり、
  GitOps のメリットが損なわれる。

---

## Decision

**Option 2 を採用する。App-of-Apps を3つに分割し、Makefile で順次実行する。**

---

## Reasons

1. **「bootstrap 時の順序保証」と「運用時の自動修復」を分離できる**

   bootstrap 時の順序制御は Makefile が担い、
   bootstrap 完了後の状態維持は ArgoCD の selfHeal が担う、
   という役割分担が明確になる。
   Option 1 では両方を ArgoCD の wave に押し込もうとしていたため
   構造が複雑化していた。Option 3 では後者が失われてしまう。

2. **ArgoCD の GUI を「最初の安全網」として保証できる**

   bootstrap の中で最優先したいのは、
   「何か問題が発生したときに ArgoCD の GUI にアクセスできる状態」を
   いち早く確保することと考えている。
   現場での経験上、障害時やハング時のトラブルシュートにおいて
   ArgoCD の GUI は CLI より圧倒的に情報が把握しやすい。
   CLI では sync 中に別の操作をしようとすると「another operation is already in progress」
   と返されるだけで、待つべきか強制終了すべきかがわかりにくい。
   GUI であれば処理がどこで詰まっているかが一目でわかり、
   TERMINATE して再実行するといった判断がすぐにできる。

   Makefile で root-1-gateway（ArgoCD の HTTPRoute を含む）を先に完了させることで、
   `https://argocd.platform.local` へのアクセスが保証された状態で
   以降の作業に進める。

3. **root の2段階構造に存在意義がなかった**

   既存の `root → group-roots → applications` という2段階の App-of-Apps は、
   wave による順序制御のために設けられていたが、
   automated sync がある以上その効果は初回限定だった。
   分割後は `root-1-gateway / root-2-auth / root-3-others` が
   直接 `applications/` を指す1段階の構造になり、見通しが改善した。

---

## Consequences

**ポジティブ:**

- bootstrap の順序制御が Makefile に集約され、挙動が予測しやすくなった
- ArgoCD の GUI へのアクセスが bootstrap の早い段階で保証される
- bootstrap 完了後は `automated: selfHeal: true` によるドリフト自動修復が維持される
- App-of-Apps の階層が1段階減り、ArgoCD GUI 上での見通しが改善した

**ネガティブ・トレードオフ:**

- bootstrap は必ず `make bootstrap` 経由で実行する必要がある。
  ArgoCD GUI から手動で root を sync する、という操作は存在しなくなった。
- root-appにあたるものが3つ存在し、bootstrapのためという背景を知らないと構成がわかりづらい。

**留意事項:**
- 一部application（KEDAなど）でbootstrap時にSyncが失敗し、手動Syncもしくはretryを待つ必要がある問題が残っている。
  原因としては、初回起動時に内部のwebhookサーバの証明書が自動生成されるのだが、その完了を待たずにinit podが動き出すため、webhookの応答がなくその後の処理が進まない。
  いずれタイムアウトしてSync失敗となりArgocdがretryをかけて解消するが、それなりに時間がかかるため、手動でSync停止→再実行が早い。
  似たような初回起動時のみの構造上の欠陥は他のアプリでも存在する。
  ただ、今回はこれらの問題には特に対応しないことにした。
  理由としては、「時間はかかっても自動で収束する」「Argocd GUI へのアクセスには影響しない（→動いてなくてもトラブルシュートが可能）」
  「根本解決としては証明書を事前に発行しておくことが考えられるが、管理コストが増えてしまう」といった判断がある。
  完璧な初回 sync を目指して過剰な対処をするより、
  収束することが確認できている問題は許容範囲と見なした。
