---
title: "aws-mountpoint-s3-csi-driver v2 アップグレードに関する備忘録"
emoji: "🪣"
type: "tech"
topics: ["aws", "eks", "kubernetes", "s3"]
published: false
publication_name: "atrae"
---

## 概要

EKS Addon の `aws-mountpoint-s3-csi-driver` を v1.x から v2.x にアップグレードしたときに対応が必要だったものを記事にする

## v2 の主な変更点

v2 ではアーキテクチャと認証方式に大きな変更がある。

### 1. アーキテクチャの変更

v2 では Mountpoint プロセスがホスト上の systemd ではなく、**専用の Kubernetes Pod** として動作するようになった。

[UPGRADING_TO_V2.md](https://github.com/awslabs/mountpoint-s3-csi-driver/blob/main/docs/UPGRADING_TO_V2.md) より:

> Prior to v2, Mountpoint processes were spawned on the host using systemd, with v2, Mountpoint processes will be spawned on dedicated and unprivileged Mountpoint Pods.

#### 変更の目的

この変更の主な目的は以下の通り（[Issue #504](https://github.com/awslabs/mountpoint-s3-csi-driver/issues/504) より）:

1. **SELinux 対応**: SELinux が有効な環境（ROSA など）での動作をサポート
2. **リソース分離**: 専用の unprivileged Pod で動作することで、ワークロードの分離が向上
3. **運用改善**: systemd リソースを消費せず、`kubectl logs` でログ確認が可能に
4. **Pod 共有**: 同じノードで同じ認証情報を使う複数ワークロード間で Mountpoint Pod を共有可能

#### v1 と v2 の比較

| 項目 | v1.x | v2.x |
|------|------|------|
| Mountpoint 実行場所 | ホスト上の systemd | 専用 Pod（`mount-s3` namespace） |
| 権限 | root | non-root (unprivileged) |
| ログ確認 | journalctl | `kubectl logs -n mount-s3` |
| SELinux 対応 | 不可 | 可能 |

#### Mountpoint Pod の確認方法

```bash
# Mountpoint Pod の一覧
kubectl get pods -n mount-s3

# 特定ボリュームのログ確認
kubectl logs -n mount-s3 -l s3.csi.aws.com/volume-name=<pv-name>
```

### 2. 認証方式の変更

v1 ではワークロード Pod の ServiceAccount（IRSA）が自動的に使われていたが、v2 ではデフォルトで CSI ドライバーの ServiceAccount が使われるようになった。

| 項目 | v1.x | v2.x |
|------|------|------|
| デフォルト認証 | ワークロード Pod の SA | CSI ドライバーの SA |

#### authenticationSource の選択肢

| 値 | 認証に使う SA | ユースケース |
|----|--------------|-------------|
| `driver`（デフォルト） | CSI ドライバーの SA | クラスター全体で同じ S3 バケットにアクセス |
| `pod` | ワークロード Pod の SA | マルチテナント、Pod ごとに異なるバケット |

## アップグレード時に発生しうる問題

### 問題 1: 認証エラー

IRSA を使用している環境で、以下のエラーが発生する。

```
MountVolume.SetUp failed for volume "xxx-pv" : rpc error: code = Internal desc =
Failed to create S3 client
Caused by: No signing credentials available
```

**原因**: v2 ではデフォルトで CSI ドライバーの SA が使われるため、ワークロード Pod の IRSA が使われなくなった。

**解決策**: PersistentVolume の `volumeAttributes` に `authenticationSource: pod` を追加する。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  csi:
    driver: s3.csi.aws.com
    volumeHandle: s3-csi-driver-volume
    volumeAttributes:
      bucketName: my-bucket
      authenticationSource: pod  # これを追加
```

### 問題 2: Mountpoint Pod のスケジュール失敗

以下のエラーが発生する場合がある。

```
MountVolume.SetUp failed for volume "xxx-pv" : rpc error: code = Internal desc =
Could not mount "bucket-name" at "...": Failed to wait for Mountpoint Pod "mp-xxxxx" to be ready
```

**原因**: v2 では Mountpoint Pod がノードの Pod 数としてカウントされる。Mountpoint Pod はワークロード Pod と同じノードに配置される（node affinity）ため、ノードの Pod 上限（ENI 制限）がギリギリの場合、Mountpoint Pod がスケジュールできない。

```
ワークロード Pod → ノード X にスケジュール (28/29)
                 ↓
CSI driver が Mountpoint Pod を同じノードに作成しようとする
                 ↓
ノードが 29/29 で満杯 → Mountpoint Pod が Pending
                 ↓
マウント失敗
```

**解決策例**: VPC CNI の Prefix Delegation を有効化して Pod 上限を引き上げる。

### 問題 3: authenticationSource: pod 設定時の IAM 権限不足

`authenticationSource: pod` を設定しても、以下のエラーが発生する場合がある。

```
MountVolume.SetUp failed for volume "xxx-pv" : rpc error: code = Internal desc =
Failed to create S3 client
Caused by:
    0: initial ListObjectsV2 failed for bucket xxx in region ap-northeast-1
    1: Client error
    2: Forbidden: User: arn:aws:sts::xxx:assumed-role/xxx-irsa-role/...
       is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::xxx"
       because no identity-based policy allows the s3:ListBucket action
```

**原因**: `authenticationSource: pod` を設定すると、ワークロード Pod の ServiceAccount（IRSA）が使われるが、その IAM ロールに S3 バケットへの必要な権限がない。v1 では動作していたケースでも、v2 では Mountpoint が初期化時に `s3:ListBucket` を実行するため、この権限が必須になっている。

**解決策**: ワークロード Pod の ServiceAccount に紐づく IAM ロールに、S3 バケットへの適切な権限を付与する。

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:ListBucket"
  ],
  "Resource": "arn:aws:s3:::<bucket-name>"
},
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject",
    "s3:DeleteObject"
  ],
  "Resource": "arn:aws:s3:::<bucket-name>/*"
}
```

### 問題 4: Terraform で EKS Addon の IRSA 設定を削除できない

`authenticationSource: pod` への移行に伴い、EKS Addon 自体の IRSA（`service_account_role_arn`）を削除しようとすると、以下のエラーが発生する。

```
Error: updating EKS Add-On (xxx:aws-mountpoint-s3-csi-driver):
operation error EKS: UpdateAddon, https response error StatusCode: 403,
api error AccessDeniedException: Cross-account pass role is not allowed.
```

**原因**: Terraform AWS Provider の既知のバグ（[GitHub Issue #30645](https://github.com/hashicorp/terraform-provider-aws/issues/30645)）。

Provider が `service_account_role_arn` を削除する際に空文字 `""` を送信するが、AWS API はフィールド自体を省略することを期待している。

```json
// Terraform が送信するリクエスト
{"serviceAccountRoleArn": ""}  // ← 空文字

// AWS API が期待する形式
{}  // ← フィールド自体を省略
```

**解決策**: AWS CLI で事前に更新する。`--service-account-role-arn` を指定しないことで、フィールドが省略される。

```bash
aws eks update-addon \
  --cluster-name <cluster-name> \
  --addon-name aws-mountpoint-s3-csi-driver \
  --resolve-conflicts OVERWRITE
```

実行後、`terraform plan` で差分がないことを確認する。

## PV/PVC の再作成

`volumeAttributes` は immutable なので、変更には PV/PVC の削除・再作成が必要。

:::message
PV を削除しても S3 バケット内のデータは影響を受けない（PV は S3 への参照に過ぎない）。
:::

### finalizer による削除ブロック

PVC/PV には `kubernetes.io/pvc-protection` / `kubernetes.io/pv-protection` という finalizer がデフォルトで付与される。これにより、**使用中のリソースが誤って削除されるのを防止**している。

Linux の `umount` と同じイメージで理解できる。

| Linux | Kubernetes |
|-------|------------|
| `umount /mnt/data` | `kubectl delete pvc` |
| `device is busy` エラー | `Terminating` 状態で待機 |
| プロセスがファイルを開いている | Pod が PVC をマウントしている |
| `lsof /mnt/data` で犯人特定 | `kubectl get pods` で犯人特定 |
| プロセス kill → umount 成功 | Pod 削除 → PVC 削除完了 |

Linux では即座にエラーになるが、Kubernetes の finalizer は「削除予約」状態（Terminating）で待機し、使用中の Pod が消えた瞬間に自動で削除が完了する。

### 削除がブロックされるケース

PVC を `kubectl delete` しても `Terminating` 状態のまま消えない場合、**その PVC をマウントしている Pod が残っている**可能性が高い。

```bash
# Terminating 状態の確認
kubectl get pvc -n <namespace>
# STATUS が Terminating のまま

# PVC を使用している Pod を探す
kubectl get pods -n <namespace> --field-selector=status.phase=Running | grep <関連キーワード>

# Pod が PVC をマウントしているか確認
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.volumes[*].persistentVolumeClaim.claimName}'
```

### 再作成手順

1. **PVC をマウントしている Pod を先に削除する**
2. PVC を削除（finalizer が解除され削除される）
3. PV を削除
4. 新しいマニフェストを apply

```bash
# 1. 必要に応じて Pod を削除
kubectl delete pod <pod-name> -n <namespace>

# 2. PVC を削除（Pod 削除後は即座に消える）
kubectl delete pvc <pvc-name> -n <namespace>

# 3. PV を削除
kubectl delete pv <pv-name>

# 4. 再作成
kubectl apply -k <path-to-kustomization>

# 5. 反映確認
kubectl get pv <pv-name> -o jsonpath='{.spec.csi.volumeAttributes}'
```

## 参考

- [UPGRADING_TO_V2.md](https://github.com/awslabs/mountpoint-s3-csi-driver/blob/main/docs/UPGRADING_TO_V2.md)
- [CONFIGURATION.md](https://github.com/awslabs/mountpoint-s3-csi-driver/blob/main/docs/CONFIGURATION.md)
- [Issue #504 - Mountpoint for Amazon S3 CSI Driver v2](https://github.com/awslabs/mountpoint-s3-csi-driver/issues/504)
- [terraform-provider-aws Issue #30645 - aws_eks_addon cannot remove value for service_account_role_arn](https://github.com/hashicorp/terraform-provider-aws/issues/30645)
