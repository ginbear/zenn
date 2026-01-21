---
title: "Kubernetes クラスタの日次ヘルスチェックを CronJob で実装した"
emoji: "🩺"
type: "tech"
topics:
  - "kubernetes"
  - "argocd"
  - "slack"
  - "cronjob"
published: false
publication_name: "atrae"
---

## はじめに

Kubernetes クラスタを運用していると「気づいたら Pod が CrashLoopBackOff になっていた」「ArgoCD の Sync が失敗したまま放置されていた」ということがあります。

毎朝クラスタの状態を Slack に通知する CronJob を作ったので紹介します。

## できること

- **異常 Pod の検出**: Running/Completed/ContainerCreating/Terminating 以外の Pod を検出
- **ArgoCD アプリの検出**: OutOfSync または Unhealthy なアプリを検出
- **ステータス別グループ化**: エラー種別ごとにまとめて表示

## アーキテクチャ

```
┌─────────────────────────────────────────────────────┐
│  CronJob (毎日 9:00 JST)                            │
│  ┌───────────────────────────────────────────────┐  │
│  │  Pod (bitnami/kubectl)                        │  │
│  │  ┌─────────────────┐                          │  │
│  │  │ ConfigMap       │                          │  │
│  │  │ (check.sh)      │                          │  │
│  │  └─────────────────┘                          │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
         │                          │
         │ kubectl get pods         │ kubectl get applications
         ▼                          ▼
    ┌─────────┐              ┌─────────────┐
    │   Pod   │              │   ArgoCD    │
    │ 状態取得 │              │ Application │
    └─────────┘              └─────────────┘
         │
         ▼
    ┌─────────┐
    │  Slack  │
    │  通知   │
    └─────────┘
```

シンプルに `bitnami/kubectl` イメージで kubectl と curl だけ使っています。

## Slack 通知

### 異常がある場合

```
📋 日次ヘルスチェック (wevox-eks-cluster-dev) 📅 2026-01-20 09:00 JST
────────────────────────────────────────────
🚨 異常 Pod (3件)
*[CrashLoopBackOff]*
• backend/worker-xyz789
• batch/job-runner-def456

*[Error]*
• monitoring/exporter-ghi012

:argo: 異常 ArgoCD Apps (1件)
*[OutOfSync/Healthy]*
• argocd/my-app
```

ステータスごとにグループ化しているので、どんなエラーが何件あるか把握しやすくなっています。

### すべて正常な場合

```
📋 日次ヘルスチェック (wevox-eks-cluster-dev) 📅 2026-01-20 09:00 JST

✅ すべて正常です
```

## ポイント

### 最小権限

ClusterRole は pods と applications の `get`, `list` のみ。書き込み権限は不要です。

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: ["argoproj.io"]
    resources: ["applications"]
    verbs: ["get", "list"]
```

### イメージの固定

`bitnami/kubectl` を sha256 ダイジェストで固定しています。タグだと意図しないバージョンアップが起きる可能性があるので。

```yaml
image: bitnami/kubectl@sha256:b349e60a6ae2969af84a778c0b976b050a0cc77f4fb6a4ad44307cd0a06e9d8f
```

## 今後やりたいこと

- PVC で前回結果を保存して差分表示（EBS CSI Driver 導入後）
- 週次サマリーの追加

## おわりに

Kubernetes は自己修復機能があり堅牢性が高いので、Pod 一つ一つを厳密に監視してアラートを飛ばすのはやりすぎだと感じていました。

かといって放置すると「いつの間にか壊れていた」になりがちです。

毎朝ゆるく状態変化をチェックする、というのがちょうど良い落とし所かなと思ってこの仕組みにしました。
