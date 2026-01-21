---
title: "EKS の Prefix Delegation で Pod 数上限を引き上げる"
emoji: "🌐"
type: "tech"
topics: ["aws", "eks", "kubernetes", "vpc"]
published: false
publication_name: "atrae"
---

## Prefix Delegation とは

[AWS VPC CNI の機能](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)で、EKS ノードあたりの Pod 数上限を大幅に引き上げることができる。

Pod 数上限が上がることで、1 ノードあたりの Pod 収容効率が向上し、より少ないノード数で同じワークロードを運用できる可能性がある。結果としてコスト削減にも繋がる。

## なぜ Pod 数に上限があるのか

EKS では各 Pod に VPC 内の IP アドレスが割り当てられる。この IP は EC2 インスタンスの **ENI (Elastic Network Interface)** を通じて確保される。

### ENI とは

[ENI (Elastic Network Interface)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html) は、VPC 内の仮想ネットワークカード。EC2 インスタンスにアタッチすることで、VPC 内の他のリソースやインターネットと通信できるようになる。

各 ENI には複数の IP アドレス（プライベート IPv4 アドレス）を割り当てることができ、EKS ではこの仕組みを利用して Pod に IP を割り当てている。

### ENI の仕組み

```
EC2 インスタンス (m7i.large)
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

ENI 数と各 ENI に割り当て可能な IP 数は[インスタンスタイプで決まっている](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)。

| インスタンス | 最大 ENI 数 | ENI あたり IP 数 | 最大 Pod 数 |
|-------------|------------|-----------------|-------------|
| m7i.large | 3 | 10 | 3 × 10 - 1 = **29** |
| m7i.xlarge | 4 | 15 | 4 × 15 - 1 = **58** |
| m7i.2xlarge | 4 | 15 | 4 × 15 - 1 = **58** |

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
| m7i.large | 29 | **110** |
| m7i.xlarge | 58 | **250** |
| m7i.2xlarge | 58 | **250** |

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

Pod 密度が高い（ノードあたり多くの Pod が動いている）環境では、Prefix Delegation でも効率よく IP が使われる。詳細は [EKS Best Practices - Prefix Mode](https://aws.github.io/aws-eks-best-practices/networking/prefix-mode/index_linux/) を参照。

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

## 適用後の確認

### 設定が有効になっているか

```bash
kubectl describe ds -n kube-system aws-node | grep ENABLE_PREFIX_DELEGATION
```

```
ENABLE_PREFIX_DELEGATION:               true
```

### ノードの max-pods

```bash
kubectl get node <node-name> -o jsonpath='{.status.allocatable}' | jq .
```

```json
{
  "cpu": "3920m",
  "ephemeral-storage": "95393112644",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "memory": "15076412Ki",
  "pods": "110"
}
```

`pods` が 110 や 250 など、通常より大きな値になっていれば OK。

## まとめ

Prefix Delegation を有効化することで、EKS ノードの Pod 数上限を大幅に引き上げられる。

- m7i.large: 29 → 110
- m7i.xlarge: 58 → 250

設定自体はシンプルだが、既存ノードには適用されない点に注意。ノードの再作成が必要なため、Kubernetes アップグレードのタイミングなどで計画的に適用するのがおすすめ。
