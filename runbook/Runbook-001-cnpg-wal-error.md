# Runbook-001: CNPG 旧 primary WAL エラー対処

## 概要

CNPG クラスタでフェイルオーバーが発生した後、旧 primary が replica として復帰する際に
`Expected empty archive` エラーで startup probe が永続的に失敗し、Ready にならない問題の対処手順。

---

## 症状

```bash
kubectl get pods -n sample-app
# NAME                   READY   STATUS    RESTARTS   AGE
# sample-backend-db-1    0/1     Running   0          10m   ← startup probe が失敗し続ける
# sample-backend-db-2    1/1     Running   0          5m    ← 新 primary（昇格済み）

kubectl logs sample-backend-db-1 -n sample-app | grep "empty archive"
# Expected empty archive
```

---

## 原因

CNPG は旧 primary 再起動時に、WAL アーカイブ先（MinIO）が空であることを確認する。
しかしフェイルオーバー前の WAL がすでに MinIO に存在するため、この確認が必ず失敗する。
startup probe の失敗が続くことで Pod は永続的に Ready にならない。

---

## 対処手順（dev 環境）

### 1. 問題の Pod と PVC を特定する

```bash
# Pod の状態確認
kubectl get pods -n sample-app -l cnpg.io/cluster=sample-backend-db

# startup probe の失敗を確認
kubectl describe pod <旧primary-pod> -n sample-app | grep -A5 "Startup"
```

### 2. 旧 primary の Pod と PVC を削除する

```bash
# Pod 削除
kubectl delete pod <旧primary-pod> -n sample-app

# PVC 削除（CNPG が新インスタンスを自動作成する）
kubectl delete pvc <旧primary-pvc> -n sample-app
```

### 3. CNPG が新インスタンスを作成するのを待つ

```bash
# CNPG はインスタンス番号を単調増加で管理する（削除した番号は再利用されない）
# 例: db-1 を削除 → 次は db-3 が作成される
kubectl get pods -n sample-app -w
```

### 4. クラスタの状態を確認する

```bash
kubectl get cluster sample-backend-db -n sample-app
# STATUS が Cluster in healthy state になれば復旧完了

kubectl get pods -n sample-app
# 全 Pod が Ready になっていることを確認
```

---

## 注意事項

- CNPG はインスタンス番号を単調増加で管理するため、削除した番号は再利用されない
  （例：db-1 を削除 → db-3 が作成される）
- PVC を削除すると当該インスタンスのデータは失われるが、
  CNPG が新インスタンスを稼働中の primary からクローンして再構築するため
  クラスタ全体のデータは失われない
- 本番環境では PVC 削除前にデータの退避（別 Pod へのマウント + pg_dump）を検討すること

---

## 参考：本番環境での対処方針

1. 問題の PVC を別 Pod にマウントしてデータを直接救出
2. `pg_resetwal` で WAL をリセットして強制起動 → `pg_dump` でエクスポート
3. 稼働中インスタンスに差分をインポート

詳細な判断基準は ADR-004 の Consequences セクションも参照。
