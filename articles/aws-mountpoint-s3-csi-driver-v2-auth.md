---
title: "aws-mountpoint-s3-csi-driver v2 へのアップグレードで S3 マウントが失敗する問題"
emoji: "🪣"
type: "tech"
topics: ["aws", "eks", "kubernetes", "s3"]
published: false
---

## 概要

EKS Addon の `aws-mountpoint-s3-csi-driver` を v1.x から v2.x にアップグレードしたところ、S3 マウントが失敗するようになった。

```
MountVolume.SetUp failed for volume "xxx-pv" : rpc error: code = Internal desc =
Failed to create S3 client
Caused by: No signing credentials available
```

## 原因

v2.0 でアーキテクチャが大きく変更され、**認証方式のデフォルト動作が変わった**。

| 項目 | v1.x | v2.x |
|------|------|------|
| Mountpoint 実行場所 | ホスト上の systemd | 専用 Pod（`mount-s3` namespace） |
| デフォルト認証 | ワークロード Pod の SA | CSI ドライバーの SA |

v1 ではワークロード Pod の ServiceAccount（IRSA）が自動的に使われていたが、v2 ではデフォルトで **CSI ドライバーの ServiceAccount** が使われるようになった。

## 解決策

PersistentVolume の `volumeAttributes` に `authenticationSource: pod` を追加する。

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

これにより、v1 と同様にワークロード Pod の IRSA 認証情報が使用される。

## authenticationSource の選択肢

| 値 | 認証に使う SA | ユースケース |
|----|--------------|-------------|
| `driver`（デフォルト） | CSI ドライバーの SA | クラスター全体で同じ S3 バケットにアクセス |
| `pod` | ワークロード Pod の SA | マルチテナント、Pod ごとに異なるバケット |

既存で IRSA を使っている場合は `authenticationSource: pod` を指定する必要がある。

## 注意点

- `volumeAttributes` は immutable なので、変更には PV の削除・再作成が必要
- PV を削除しても S3 バケット内のデータは影響を受けない（PV は S3 への参照に過ぎない）

## 参考

- [UPGRADING_TO_V2.md](https://github.com/awslabs/mountpoint-s3-csi-driver/blob/main/docs/UPGRADING_TO_V2.md)
- [CONFIGURATION.md](https://github.com/awslabs/mountpoint-s3-csi-driver/blob/main/docs/CONFIGURATION.md)
