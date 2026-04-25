# ADR-003: Secret管理戦略の選択

## Context

Kubernetesクラスタの運用において、SecretはGitで管理するか否かに
関わらず以下の2つの問題を同時に解決する必要がある。

- **問題A（Git管理）:** DBパスワードやAPIキーなどの機密情報を
  Gitリポジトリに平文で置くことはできない
- **問題B（調達元）:** クラスタ上のSecretリソースをどこから・
  どのように生成するか

現場のクラスタでは機密情報の管理ルールが整備されておらず、
マニフェストリポジトリに平文のSecretが残存しているケースがあった。
この状態はリポジトリへのアクセス権を持つ全員が機密情報を
参照できることを意味しており、セキュリティ上の課題として認識している。

ポートフォリオでは最初からSecretをコードとして安全に管理する
仕組みを設計することとし、両問題に対するアプローチを検討した。

---

## Options

### 問題Aへのアプローチ（GitでのSecret管理）

#### Option A-1: Sealed Secrets
- クラスタ上のControllerが復号を担う方式
- `kubeseal`コマンドでSecretを暗号化し、SealedSecretリソースとして
  Gitに置く
- **問題点:** 復号はクラスタ上のControllerが持つ鍵に依存するため、
  クラスタを完全に作り直した場合にControllerの鍵の
  バックアップがなければSecretを復元できない。DRの文脈でクラスタへの
  依存を増やすことになる

#### Option A-2: SOPS × Age
- ファイル単位で暗号化し、Ageの秘密鍵を持つ開発者のみが復号できる方式
- クラスタに依存せずローカルで復号できる
- `.sops.yaml`でファイルパスに応じた暗号化ルールを定義できる
- Gitの差分が暗号化された形で残るため変更履歴を追える

### 問題Bへのアプローチ（クラスタ上のSecret調達）

#### Option B-1: External Secrets Operator（ESO）のみ
- AWS Secrets ManagerやHashiCorp Vaultなどの外部ストアから
  Secretを引っ張り、KubernetesのSecretリソースを自動生成する
- アプリ側はExternalSecretリソースを宣言するだけでよく、
  Gitには平文も暗号化済みSecretも置かない
- **問題点:** 外部ストアのインフラが必要。ローカル検証環境では
  Fakeプロバイダーで代替する必要がある

#### Option B-2: SOPS × Age + ESO の併用
- GitにはSOPSで暗号化したSecretを置き、ArgoCDのdecryption機能
  またはESOのgenerator経由でクラスタ上のSecretを生成する
- Gitを唯一のソースとしつつ暗号化で安全性を担保できる

---

## Decision

**SOPS × Age（問題A）+ ESO（問題B）の併用を採用する。**

---

## Reasons

1. **クラスタ破棄後もAgeキー1つで復元できるためDRと相性がよい**
   Sealed SecretsはクラスタのControllerが復号を担うため、
   クラスタを完全に作り直した場合にControllerの鍵の
   バックアップがなければSecretを復元できない。
   SOPS×AgeはAgeの秘密鍵をローカルに持つだけで復号できるため、
   Phase 10のテーマである「クラスタ全損からの1コマンド復旧」と
   自然に整合する。

2. **SecretをコードとしてGitで管理できる**
   暗号化されたSecretをGitに置くことで、Secretの変更履歴が
   Gitのコミット履歴として残る。誰がいつ何を変更したかを
   追跡でき、レビュープロセスにも乗せられる。
   現場での「マニフェストリポジトリに平文が残存している」
   という課題の根本的な解消策になる。

3. **ESOとの役割分担が明確になる**
   SOPS×AgeはGit上での暗号化（問題A）を担い、
   ESOはクラスタ上でのSecret生成（問題B）を担う。
   将来的に外部ストアをAWS Secrets ManagerやVaultに
   切り替える場合もESO側の設定変更だけで対応でき、
   SOPS×Ageの仕組みはそのまま維持できる。

---

## Consequences

**ポジティブ:**
- AgeキーとGitリポジトリがあればクラスタを完全に再現できる
- SecretをGitで変更履歴・レビュー管理の対象にできる
- ESOの外部ストア設定を変えることでクラウド環境（EKS等）にも
  そのまま移行できる

**ネガティブ・トレードオフ:**
- Ageの秘密鍵の管理が運用者の責任になる。鍵を紛失した場合は
  Gitに積まれた暗号化済みSecretを復元できなくなる
- チーム開発では参加者全員のAgeキーを`.sops.yaml`に登録するか、
  共有鍵の管理ポリシーを別途決める必要がある
- SOPS・Age・ESOと複数ツールの理解が必要で、
  Sealed Secretsの「kubesealで暗号化するだけ」という
  シンプルさと比べると導入コストが高い
- 個人ポートフォリオのスコープではAgeキーの自己管理で
  成立するが、チーム運用ではSecret Storeの外部化
  （AWS Secrets Manager・Azure Key Vault・Vault等）と
  ESOの組み合わせが現実的な選択肢になる
