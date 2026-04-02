---
title: "Argo Workflows の SSO RBAC で namespace 単位の権限制御を実現する"
emoji: "🔐"
type: "tech"
topics: ["argoworkflows", "kubernetes", "sso", "rbac"]
published: true
publication_name: "atrae"
---

## きっかけ

Argo Workflows に SSO (GitHub OAuth) + RBAC を導入して運用していたが、ある日「特定のチームに特定の namespace だけ workflow の実行・操作権限を付与したい」という要件が出た。

既存の構成は ClusterRole + ClusterRoleBinding でクラスタ全体に権限を付与する形だったため、namespace 単位の制御ができない。やろうとすると kustomize の構造的制約にハマって迷走し、最終的に公式の Namespace Delegation 機能にたどり着いた、という話。

## 前提: SSO RBAC の仕組み

Argo Workflows の SSO RBAC は、ServiceAccount のアノテーションでグループをマッピングする仕組みになっている。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: argo
  annotations:
    workflows.argoproj.io/rbac-rule: "'my-admin-team' in groups"
    workflows.argoproj.io/rbac-rule-precedence: "1"
```

SSO ログイン時に、argo namespace 内の ServiceAccount を precedence の高い順に評価し、`rbac-rule` にマッチした SA の権限でユーザーが操作する。

よくある構成はこんな感じ。

| precedence | ServiceAccount | グループ | 権限 |
|---|---|---|---|
| 1 | admin-user | admin チーム | 全 namespace で操作可能 |
| 0 | default-user | 一般開発者 | 全 namespace で閲覧のみ |

参考: [Argo Workflows SSO RBAC ドキュメント](https://argo-workflows.readthedocs.io/en/latest/argo-server-sso/#sso-rbac)

## やりたいこと

一般開発者は閲覧のみのまま、**あるチームに特定の namespace だけ workflow の実行・操作権限を付与**したい。

| チーム | 対象 namespace | その他の namespace |
|-------|---------------|-------------------|
| admin | 操作可能 | 操作可能 |
| 対象チーム | **操作可能** | 閲覧のみ |
| 一般開発者 | 閲覧のみ | 閲覧のみ |

## 案A: rbac-rule に条件追加（シンプルだが namespace 制限なし）

一番シンプルなのは、既存の `admin-user` の rbac-rule に条件を追加する方法。設計が複雑になりすぎるならこれで妥協するつもりだった。

```yaml
workflows.argoproj.io/rbac-rule: "'admin-team' in groups || 'target-team' in groups"
```

1行の変更で済むが、対象チームに**全 namespace の操作権限** が付与されてしまう。特定 namespace に限定できない。まずは namespace 制限を実現できないか試してみることにした。

## 案B: namespace 限定を kustomize で実現しようとした

:::message
案B は迷走の記録なので、結論だけ知りたい方は[案C](#案c%3A-namespace-delegation（公式機能）)まで飛ばしてください。
:::

「argo namespace の kustomization に Role/RoleBinding を追加して対象 namespace に配置すればいいのでは？」と思い、Claude Code と一緒に試行錯誤した。patches、JSON patch、replacements と次々にアイデアを出して対応してくれたが、1つ解決するたびに次の問題が出てくる展開だった。

### 問題1: namespace transformer による上書き

kustomize の `namespace:` ディレクティブは全リソースの namespace を強制的に上書きする。

```yaml
# kustomization.yaml
namespace: argo  # ← これが全リソースに適用される

resources:
- sso-rbac.yaml
```

```yaml
# sso-rbac.yaml で別の namespace を指定しても...
kind: Role
metadata:
  namespace: target-ns  # ← kustomize が argo に書き換えてしまう
```

`kubectl kustomize` はビルドが通るので気付きにくい。出力を grep して初めて namespace が違うことに気付く。

### 問題2: patches でも上書きされる

patches は namespace transformer の**前**に実行されるため、パッチで正しい namespace を設定しても後から上書きされる。

```yaml
# JSON patch で namespace を上書きしようとしても...
patches:
- target:
    kind: Role
    name: my-role
  patch: |
    - op: replace
      path: /metadata/namespace
      value: target-ns     # ← 設定しても namespace transformer が argo に戻す
```

### 問題3: replacements で回避はできるが...

`replacements` は namespace transformer の**後**に実行されるため、上書きされた namespace を再設定できる。

```yaml
replacements:
- source:
    kind: ConfigMap
    name: namespace-config
    fieldPath: data.namespace
  targets:
  - select:
      kind: Role
      name: my-role
    fieldPaths:
    - metadata.namespace
```

ConfigMap から値を注入する形で namespace を再設定できた。が、環境ごと（dev/stg/prd）に ConfigMap のパッチと replacements の再定義が必要になり、ファイルが増殖していく。さらに bases で追加した新しい SA の precedence が overlay でパッチされた既存 SA と衝突する問題も発生し、これも環境ごとにパッチが必要だった。

### 結果

動くものはできたが、PR の diff が複雑になりすぎてレビューが困難な状態になった。やりたいことに対してオーバーエンジニアリングすぎる。

## 案C: Namespace Delegation（公式機能）

案B の複雑さに限界を感じていたところで、Argo Workflows v3.3+ に [SSO RBAC Namespace Delegation](https://argo-workflows.readthedocs.io/en/latest/argo-server-sso/#sso-rbac-namespace-delegation) という機能が存在することを発見した。

### Namespace Delegation とは

`SSO_DELEGATE_RBAC_TO_NAMESPACE=true` を argo-server に設定すると、**リクエスト先の namespace にある ServiceAccount も評価対象になる**。

通常の SSO RBAC は argo namespace の SA だけを評価するが、Namespace Delegation を有効にすると:

1. ユーザーがログイン → argo namespace（cluster レベル）の SA を評価
2. 特定 namespace にリクエスト → **その namespace の SA も評価**
3. namespace SA がマッチし、precedence が高ければそちらを採用
4. マッチしなければ cluster レベルの SA に[フォールバック](https://argo-workflows.readthedocs.io/en/latest/argo-server-sso/#recommended-usage)する

### 本来の設計思想

ドキュメントにはこう書かれている。

> Typically, an organization has a Kubernetes cluster and a central team (the owner of the cluster) manages the cluster. Along with this, there are multiple namespaces which are owned by individual teams. This feature would help **namespace owners to define RBAC for their own namespace**.

つまり:

- **クラスタ管理チーム**: cluster レベルの SA（ログイン + デフォルト権限）を管理
- **各チーム**: 自分の namespace の SA を、自分のマニフェストリポジトリで管理

RBAC の設定権限を namespace オーナーに委譲できる仕組み。

### 実装

argo-server 側は env var を1つ追加するだけ。

```yaml
env:
  - name: SSO_DELEGATE_RBAC_TO_NAMESPACE
    value: "true"
```

namespace 側には SA + RoleBinding を追加する。今回は admin と同じ権限を付与したかったので、**Role を個別に定義せず既存の ClusterRole を RoleBinding から参照**した。namespace ごとに権限を細かく調整したい場合は、個別の Role を定義するのもあり。

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user-ns
  annotations:
    workflows.argoproj.io/rbac-rule: "'target-team' in groups"
    workflows.argoproj.io/rbac-rule-precedence: "1"
---
# K8s 1.24+ では SA トークンが自動作成されないため明示的に作成
apiVersion: v1
kind: Secret
metadata:
  name: admin-user-ns.service-account-token
  annotations:
    kubernetes.io/service-account.name: admin-user-ns
type: kubernetes.io/service-account-token
---
# RoleBinding が ClusterRole を参照すると、権限は RoleBinding の namespace にスコープされる
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin-user-ns-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin-user-cluster-role  # argo namespace で定義済みの ClusterRole を再利用
subjects:
- kind: ServiceAccount
  name: admin-user-ns
```

namespace は kustomization.yaml の `namespace:` で自動付与されるため、マニフェストに書く必要はない。案B で散々悩まされた namespace transformer が、対象 namespace のリポジトリでは素直に正しい namespace を付与してくれる。

### セキュリティ上の考慮

Namespace Delegation を有効にすると、namespace のマニフェストリポジトリに write 権限を持つ人が SA を追加して権限昇格できる可能性がある。
対策として、リポジトリのブランチ保護や、CODEOWNERS で RBAC 関連ファイルに管理者レビューを必須化するなどは必要だろう。

## 最終的な変更量の比較

| アプローチ | 変更量 | namespace 制限 |
|-----------|--------|---------------|
| 案A: rbac-rule に条件追加 | rbac-rule に1条件追加 | なし（全 namespace で操作可能） |
| 案B: kustomize で頑張る | 多数のファイル変更 | あり |
| **案C: Namespace Delegation** | **namespace 側に1ファイル追加 + argo-server に env var 追加** | **あり** |

## おわりに

kustomize で頑張ろうとして迷走したが、最終的に公式の Namespace Delegation 機能に落ち着いた（発見できてよかった）。

振り返ると、「kustomize の `namespace:` ディレクティブが全リソースを上書きする」「patches は namespace transformer の前に実行される」「replacements は後に実行される」といった kustomize の内部動作を理解できたのは良い経験だった。ただ、先に公式ドキュメントを読んでいれば迷走は避けられた。

Argo Workflows v3.3 以降で namespace 単位の権限制御が必要になったら、Namespace Delegation の存在を思い出してもらえれば幸いです。
