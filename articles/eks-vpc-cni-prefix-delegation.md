---
title: "EKS の Prefix Delegation で Pod 数上限を引き上げる"
emoji: "🌐"
type: "tech"
topics: ["aws", "eks", "kubernetes", "vpc"]
published: false
publication_name: "atrae"
---

## 背景

EKS で aws-mountpoint-s3-csi-driver を v2 にアップグレードしたところ、S3 マウントを使用する Job が失敗するようになった。

```
MountVolume.SetUp failed for volume "xxx-pv" :
Failed to wait for Mountpoint Pod "mp-xxxxx" to be ready
```

原因を調査すると、ノードの Pod 数が上限に達していた。

v1 ではドライバー Pod が S3 をマウントしていたが、v2 ではマウントごとに専用の Mountpoint Pod が作成される仕様に変更された。ノードの Pod 数がちょうど上限に達していたため、新しい Pod が Pending になっていた。

この問題を解決するため **Prefix Delegation** を有効化し、ノードあたりの Pod 数上限を引き上げることにした。

## Prefix Delegation とは

AWS VPC CNI の機能で、EKS ノードあたりの Pod 数上限を大幅に引き上げることができる。

## なぜ Pod 数に上限があるのか

EKS では各 Pod に VPC 内の IP アドレスが割り当てられる。この IP は EC2 インスタンスの **ENI (Elastic Network Interface)** を通じて確保される。

### ENI の仕組み

```
EC2 インスタンス (m5.large)
├── ENI 1 (プライマリ)
│   ├── スロット 1: 10.10.1.10 (ノード自身の IP)
│   ├── スロット 2: 10.10.1.11 → Pod A
│   ├── スロット 3: 10.10.1.12 → Pod B
│   └── ... (最大 10 スロット)
├── ENI 2
│   ├── スロット 1: 10.10.1.20 → Pod C
│   └── ... (最大 10 スロット)
└── ENI 3
    └── ... (最大 10 スロット)
```

### インスタンスタイプごとの制限

ENI 数と各 ENI に割り当て可能な IP 数はインスタンスタイプで決まっている。

| インスタンス | 最大 ENI 数 | ENI あたり IP 数 | 最大 Pod 数 |
|-------------|------------|-----------------|-------------|
| t3.medium | 3 | 6 | 3 × 6 - 1 = **17** |
| m5.large | 3 | 10 | 3 × 10 - 1 = **29** |
| m5.xlarge | 4 | 15 | 4 × 15 - 1 = **58** |

※ -1 はノード自身の IP 分

## 通常モードと Prefix Delegation の違い

### 通常モード

ENI の各スロットに **1 IP ずつ** 割り当てる。

```
ENI スロット 1 → 1 IP (10.10.1.11) → 1 Pod
ENI スロット 2 → 1 IP (10.10.1.12) → 1 Pod
ENI スロット 3 → 1 IP (10.10.1.13) → 1 Pod
```

### Prefix Delegation モード

ENI の各スロットに **/28 プレフィックス (16 IP)** を割り当てる。

```
ENI スロット 1 → /28 (10.10.1.0-15)  → 最大 16 Pod
ENI スロット 2 → /28 (10.10.1.16-31) → 最大 16 Pod
ENI スロット 3 → /28 (10.10.1.32-47) → 最大 16 Pod
```

### Pod 数上限の比較

| インスタンス | 通常モード | Prefix Delegation |
|-------------|-----------|-------------------|
| t3.medium | 17 | **110** |
| m5.large | 29 | **110** |
| m5.xlarge | 58 | **250** |

## IP の使われ方

### 通常モード

```
Pod 起動 → 1 IP 確保
Pod 終了 → 1 IP 解放
```

使った分だけ消費される。

### Prefix Delegation モード

```
Pod 起動 → /28 (16 IP) をブロックで確保
           ├── 実際に使うのは 1 IP
           └── 残り 15 IP は「予約済み」状態
```

**実際に使う IP 数は同じでも、「予約」として確保される IP 数が増える。**

### 具体例

| Pod 数 | 通常モード | Prefix Delegation |
|--------|-----------|-------------------|
| 1 | 1 IP 消費 | 16 IP 予約 |
| 10 | 10 IP 消費 | 16 IP 予約 |
| 17 | 17 IP 消費 | 32 IP 予約 (16×2) |
| 30 | 30 IP 消費 | 32 IP 予約 (16×2) |

Pod 密度が高い（ノードあたり多くの Pod が動いている）環境では、Prefix Delegation でも効率よく IP が使われる。

## サブネットの IP 枯渇リスク

Prefix Delegation を有効にすると、サブネットの IP 消費が増える可能性がある。

### 確認すべきこと

1. **サブネットの空き IP 数**

```bash
aws ec2 describe-subnets \
  --subnet-ids <subnet-id> \
  --query 'Subnets[*].[CidrBlock,AvailableIpAddressCount]' \
  --output table
```

2. **クラスターが使用しているサブネット**

```bash
# ノードグループが使用しているサブネットを確認
aws eks describe-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --query 'nodegroup.subnets'
```

### 目安

| サブネット CIDR | 総 IP 数 | Prefix Delegation |
|----------------|---------|-------------------|
| /24 | 256 | 注意が必要 |
| /20 | 4,096 | 余裕あり |
| /19 | 8,192 | 十分 |

小さいサブネット (/24 など) を使っている場合は、IP 枯渇に注意。

## 設定方法

### Terraform での設定

`aws_eks_addon` リソースで vpc-cni の設定を行う。

```hcl
resource "aws_eks_addon" "vpc-cni" {
  cluster_name                = aws_eks_cluster.example.name
  addon_name                  = "vpc-cni"
  addon_version               = "v1.20.1-eksbuild.3"  # 適宜最新版を確認
  resolve_conflicts_on_update = "OVERWRITE"

  configuration_values = jsonencode({
    env = {
      ENABLE_PREFIX_DELEGATION = "true"
      WARM_PREFIX_TARGET       = "1"
    }
  })
}
```

`WARM_PREFIX_TARGET = "1"` は、予備として 1 つの /28 プレフィックスを確保しておく設定。Pod 起動を高速化できる。

### eksctl での設定

```yaml
addons:
  - name: vpc-cni
    configurationValues: |
      env:
        ENABLE_PREFIX_DELEGATION: "true"
        WARM_PREFIX_TARGET: "1"
```

## 適用手順

### 注意: 既存ノードには適用されない

Prefix Delegation の設定は**新規ノードにのみ適用**される。既存ノードは再作成が必要。

### 推奨される適用方法

1. vpc-cni addon の設定を変更して apply
2. 新しいノードグループを作成
3. 旧ノードグループを drain してワークロードを移行
4. 旧ノードグループを削除

Kubernetes のバージョンアップグレード時にノードグループを作り直すタイミングで合わせて適用するのがスムーズ。

## 適用後の確認

### 設定が有効になっているか

```bash
kubectl describe ds -n kube-system aws-node | grep ENABLE_PREFIX_DELEGATION
```

```
ENABLE_PREFIX_DELEGATION:  true
```

### ノードの max-pods

```bash
kubectl get node <node-name> -o jsonpath='{.status.allocatable.pods}'
```

110 や 250 など、通常より大きな値になっていれば OK。

## まとめ

Prefix Delegation を有効化することで、EKS ノードの Pod 数上限を大幅に引き上げられる。

- m5.large: 29 → 110
- m5.xlarge: 58 → 250

設定自体はシンプルだが、既存ノードには適用されない点に注意。ノードの再作成が必要なため、Kubernetes アップグレードのタイミングなどで計画的に適用するのがおすすめ。

## 参考

- [Assign more IP addresses to Amazon EKS nodes with prefixes](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)
- [Prefix Mode for Linux - EKS Best Practices](https://aws.github.io/aws-eks-best-practices/networking/prefix-mode/index_linux/)
- [Prefix Delegation - EKS Workshop](https://www.eksworkshop.com/docs/networking/vpc-cni/prefix/)
