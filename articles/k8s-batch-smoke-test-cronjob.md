---
title: "Kubernetes バッチジョブの起動エラーを事前検知する Smoke Test CronJob を作った"
emoji: "🧪"
type: "tech"
topics:
  - "kubernetes"
  - "argoevents"
  - "slack"
  - "cronjob"
published: true
published_at: 2026-02-06 09:30
publication_name: "atrae"
---

## はじめに

Kubernetes でイベント駆動型バッチ（Argo Events / Sensor）を運用していると、「dev 環境でバッチが何週間も動かないうちに ConfigMap の環境変数が足りなくなっていた」「イメージ更新後に起動しなくなっていたが、次のトリガーまで気づけなかった」ということがあります。

バッチの起動エラーを事前に検知する Smoke Test CronJob を作ったので紹介します。

## 背景

dev 環境ではユーザー操作が少なく、Sensor トリガーのバッチが長期間実行されないことがあります。その間に以下のような問題が発生しても気づけません。

- ConfigMap に新しい環境変数が追加されたが、dev 用の ConfigMap に反映されていない（`KeyError`, `ENV.fetch` 失敗）
- Docker イメージが更新されたが、起動時のバリデーションで落ちるようになった
- Secret が削除・変更されて `CreateContainerConfigError` になった

これらは本番リリース前に dev で踏みたい問題です。しかしトリガーされるまで分からない、というのが課題でした。

## できること

- **起動エラーの事前検知**: 各バッチの実行イメージ + ConfigMap/Secret の組み合わせで軽量コマンドを実行し、起動できるかを検証
- **イメージの動的解決**: Sensor / Deployment / CronJob から実行時に最新のイメージタグを取得。CI/CD でタグが更新されても tests.conf の変更は不要
- **Slack 通知**: 失敗時はイメージ情報とエラー行を表示

## アーキテクチャ

```text
CronJob (お好きなスケジュールで)
  └─ check.sh
       ├─ tests.conf を読み込み
       ├─ テスト定義ごとに:
       │    ├─ image_source からイメージを動的解決
       │    │    └─ Sensor / Deployment / CronJob から最新タグを取得
       │    ├─ 短命 Job を作成
       │    │    └─ 取得した image + envFrom で smoke コマンドを実行
       │    └─ 成功/失敗を記録
       ├─ 結果を集約
       └─ Slack に通知（失敗時はイメージ情報+エラー行を表示）
```

やっていることはシンプルで、**実際のバッチと同じイメージ・同じ環境変数で軽量コマンドを実行する Job を作成**し、起動できるかを確認しています。

- **Rails**: `bundle exec rails runner "puts 'OK'"` で全 initializer の `ENV.fetch` を検証
- **Go**: `./binary --help` で起動時バリデーションを検証
- **Python**: `python3 -c "print('OK')"` で起動可否を検証
  - ※ Rails と異なり import は実行されないため、環境変数の検証には `python3 -c "from myapp import main"` のように明示的な import が必要

## テスト定義

テスト定義はパイプ区切りのテキストファイル（ConfigMap）です。

```text
# name | namespace | image_source | configmap | secret | command | env (optional)

# Rails バッチ（Sensor からイメージ取得）
my-rails-batch | batch-dev | sensor/my-sensor/my-app | my-batch-config-env | my-secret,aws-secret | bundle exec rails runner "puts 'smoke test OK'"

# Go バッチ（Sensor からイメージ取得、env で個別 Secret キーを指定）
my-go-batch | batch-dev | sensor/my-go-sensor/my-go-app | my-go-config-env | my-secret,aws-secret | ./my-go-app --help | DB_PASSWORD=secret:db-secret:PASSWORD

# Rails バッチ（別 namespace の Deployment からイメージ取得）
my-admin-batch | batch-dev | deploy/my-admin@admin-dev | my-admin-config-env | my-secret,aws-secret | bundle exec rails runner "puts 'smoke test OK'"
```

テストの追加・削除はこのファイルに行を追加・削除するだけです。

### env 列（個別環境変数の指定）

`envFrom` では ConfigMap/Secret 全体を環境変数として読み込みますが、**特定のキーだけを別名で設定したい**場合は 7列目の `env` を使います。

```text
# 形式: ENV_NAME=secret:secretName:key または ENV_NAME=configmap:cmName:key
# 複数指定はカンマ区切り

my-go-batch | ... | ./my-go-app --help | DB_PASSWORD=secret:front-db-secret:PASSWORD,API_KEY=secret:api-secret:KEY
```

これは Kubernetes の `env.valueFrom.secretKeyRef` / `configMapKeyRef` に相当します。`envFrom` で読み込む Secret と別の Secret から特定のキーだけ取得したい場合に便利です。

## Slack 通知

Slack 通知は 2 つの attachment で構成されています。

1. **サマリー**: 結果の概要、失敗したテストの詳細、成功一覧
2. **詳細ログ**: 各テストの image_source と解決されたイメージ（折りたたみ表示）

### 失敗がある場合

失敗したテストにはイメージ情報とログから抽出したエラー行が表示されます。

```text
🧪 バッチ Smoke Test (my-eks-cluster-dev) 📅 2026-01-29 10:00 JST
結果: ✅ 3 passed / ❌ 2 failed / 📊 5 total (⏱ 3m45s)
────────────────────────────────────────
❌ 失敗したテスト:
• my-python-batch (batch-dev)
  image: my-python-app:1.6.3
  KeyError: 'S3_BUCKET_NAME'

✅ 成功: my-rails-batch, my-go-batch, my-admin-batch
```

イメージタグが表示されるので、「どのバージョンで壊れたか」がすぐに分かります。

### 詳細ログ（折りたたみ）

2 つ目の attachment には各テストの詳細が表示されます。Slack では長いテキストは自動的に折りたたまれるため、必要なときだけ展開して確認できます。

```text
[1] my-rails-batch (batch-dev)
    source: sensor/my-sensor/my-app
    image: my-app:c1f415a
    -> PASSED
[2] my-go-batch (batch-dev)
    source: sensor/my-go-sensor/my-go-app
    image: my-go-app:2995428
    -> PASSED
...
=== Results: 3 passed, 2 failed / 5 total (3m45s) ===
```

`source` でどのリソースからイメージを取得したかが分かるので、問題発生時のデバッグに役立ちます。

### すべて成功した場合

```text
🧪 バッチ Smoke Test (my-eks-cluster-dev) 📅 2026-01-29 10:00 JST
結果: ✅ 5 passed / ❌ 0 failed / 📊 5 total (⏱ 4m15s)

✅ 成功: my-rails-batch, my-go-batch, my-admin-batch, my-admin-batch-2, my-python-batch
```

## ポイント

### イメージの動的解決

一番工夫したところです。バッチのイメージタグは CI/CD で更新されることも多く、tests.conf にタグを直書きすると、すぐ古くなって意味のないテストになります。

そこで `image_source` フィールドで取得元を指定し、実行時にクラスタ上のリソースから最新のイメージを取得するようにしました。

| 形式 | 説明 |
|------|------|
| `sensor/<name>/<repo>` | Sensor リソースからリポジトリ名で一致するイメージを取得 |
| `deploy/<name>` | Deployment の先頭コンテナイメージを取得 |
| `deploy/<name>@<ns>` | 別 namespace の Deployment から取得 |
| `cronjob/<name>` | CronJob のコンテナイメージを取得 |
| `cronjob/<name>@<ns>` | 別 namespace の CronJob から取得 |
| `<image-url>` | 直接指定（更新頻度が低いもの向け） |

Sensor からの取得は少し工夫が必要でした。Sensor リソースは Argo Events 固有の CRD で、イメージが深くネストした位置にあります。jq の再帰探索 (`..`) ですべての `.image` フィールドを取り出し、リポジトリ名のパターンでフィルタしています。

```bash
kubectl get sensor "$sensor_name" -n "$ns" -o json \
  | jq -r '.. | .image? // empty' \
  | grep "$repo_pattern" \
  | head -1
```

### distroless イメージへの対応

Go の distroless イメージには `sh` が存在しません。最初はすべてのコマンドを `command: ["sh", "-c"]` でラップしていましたが、distroless で失敗しました。

コマンドが `./` や `/` で始まる場合はバイナリ直接実行と判断し、`sh` を経由しないようにしています。

```bash
if [[ "$command" == ./* || "$command" == /* ]]; then
  # distroless 対応: バイナリ直接実行
  # ./my-app --help → command: ["./my-app"], args: ["--help"]
  read -ra CMD_PARTS <<< "$command"
  CMD_YAML='["'"${CMD_PARTS[0]}"'"]'
  # ... args を組み立て
else
  # sh -c ラッパーで実行
  CMD_YAML='["sh", "-c"]'
  ARGS_YAML="[\"${escaped_command}\"]"
fi
```

### RBAC 最小権限

Smoke Test の Job 作成・監視・削除に加え、イメージ動的取得のために Sensor / Deployment / CronJob の `get` 権限が必要です。

```yaml
rules:
  # Job 作成・監視・削除
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["create", "get", "list", "watch", "delete"]
  # イメージ動的取得
  - apiGroups: ["argoproj.io"]
    resources: ["sensors"]
    verbs: ["get"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get"]
  - apiGroups: ["batch"]
    resources: ["cronjobs"]
    verbs: ["get"]
```

### イメージの固定

[前回の記事](https://zenn.dev/atrae/articles/k8s-health-report-cronjob)と同じく、`bitnami/kubectl` を sha256 ダイジェストで固定しています。

```yaml
image: bitnami/kubectl@sha256:b349e60a6ae2969af84a778c0b976b050a0cc77f4fb6a4ad44307cd0a06e9d8f
```

## ハマったところ

### `xargs` がダブルクォートを食べる

tests.conf から読み込んだ各フィールドの前後空白を除去するために `xargs` を使っていました。

```bash
command=$(echo "$command" | xargs)
```

ところが `xargs` はクォートを構文として解釈するため、`python3 -c "print('smoke test OK')"` が `python3 -c print('smoke test OK')` になってしまいます。ダブルクォートが消えた結果、`sh -c` に渡されるコマンドが壊れて失敗していました。

`command` フィールドだけ `sed` での空白除去に変更して解決しました。

```bash
command=$(echo "$command" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
```

### ConfigMap の YAML ブロックスカラと変数展開

check.sh 内で Job マニフェストを生成する際、最初はヒアドキュメント (`cat <<EOF`) を使っていました。しかし ConfigMap の `|` ブロックスカラ内にヒアドキュメントを書くと、変数展開のタイミングやインデントの問題で YAML パースエラーになります。

文字列変数に組み立てて `echo "$JOB_YAML" | kubectl apply -f -` する方式に変更して解決しました。

## 今後やりたいこと

- prd 環境への展開（本番のイメージ + ConfigMap の組み合わせも検証）
- テスト失敗時の自動 Issue 作成

## おわりに

イベント駆動バッチは「トリガーされるまでエラーに気づけない」という性質上、問題の発見が遅れがちです。特に dev 環境はトリガー頻度が低いので、何週間も壊れたまま放置されていたこともありました。

毎朝軽量コマンドで起動可否を確認する、というシンプルなアプローチですが、ConfigMap の環境変数不足やイメージの不整合を本番リリース前に検知できるようになりました。
