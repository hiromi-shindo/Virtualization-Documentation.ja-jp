---
title: Linux ノードの結合
author: daschott
ms.author: daschott
ms.date: 02/09/2018
ms.topic: get-started-article
ms.prod: containers
description: Linux ノードを Kubernetes クラスターに追加する (v 1.14)。
keywords: kubernetes、1.14、windows、はじめに
ms.assetid: 3b05d2c2-4b9b-42b4-a61b-702df35f5b17
ms.openlocfilehash: 88207939c82bfe8ffa0b088cfd61cf4ab22cb10a
ms.sourcegitcommit: 1ca9d7562a877c47f227f1a8e6583cb024909749
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 12/04/2019
ms.locfileid: "74909952"
---
# <a name="joining-linux-nodes-to-a-cluster"></a>クラスターへの Linux ノードの参加

[Kubernetes マスターノードを設定](creating-a-linux-master.md)し、目的の[ネットワークソリューションを選択](network-topologies.md)したら、Linux ノードをクラスターに参加させることができます。 これを行うには、参加する前に[Linux ノードの準備を](joining-linux-workers.md#preparing-a-linux-node)行う必要があります。
> [!tip]
> Linux の手順は、 **Ubuntu 16.04**に合わせて調整されています。 Kubernetes を実行するために認定された他の Linux ディストリビューションにも、代替として使用できる同等のコマンドが用意されています。 また、Windows との相互運用も正常に行われます。

## <a name="preparing-a-linux-node"></a>Linux ノードの準備

> [!NOTE]
> 明示的に指定されている場合を除き、**管理者特権のルートユーザーシェル**で任意のコマンドを実行します。

まず、ルートシェルにアクセスします。

```bash
sudo –s
```

コンピューターが最新の状態であることを確認します。

```bash
apt-get update && apt-get upgrade
```

## <a name="install-docker"></a>Docker のインストール

コンテナーを使用できるようにするには、Docker などのコンテナーエンジンが必要です。 最新バージョンを入手するには、Docker のインストールに関する[次の手順](https://docs.docker.com/install/linux/docker-ce/ubuntu/)を使用します。 `hello-world` イメージを実行して、docker が正しくインストールされていることを確認できます。

```bash
docker run hello-world
```

## <a name="install-kubeadm"></a>Kubeadm のインストール

Linux ディストリビューションのバイナリをダウンロードして、クラスターを初期化します。 `kubeadm` します。

> [!Important]  
> Linux ディストリビューションによっては、以下の `kubernetes-xenial` を正しい[コードネーム](https://wiki.ubuntu.com/Releases)に置き換える必要がある場合があります。

``` bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update && apt-get install -y kubelet kubeadm kubectl 
```

## <a name="disable-swap"></a>スワップを無効にする

Linux 上の Kubernetes では、スワップ領域を無効にする必要があります。

``` bash
nano /etc/fstab  # (remove a line referencing 'swap.img' , if it exists)
swapoff -a
```

## <a name="flannel-only-enable-bridged-ipv4-traffic-to-iptables"></a>(Flannel のみ)Iptables への IPv4 トラフィックのブリッジを有効にする

ネットワークソリューションとして Flannel を選択した場合は、iptables チェーンへのブリッジされた IPv4 トラフィックを有効にすることをお勧めします。 [これはマスターに対して既に実行](network-topologies.md#flannel-in-host-gateway-mode)されている必要があります。また、これを Linux ノードに参加させるために繰り返す必要があります。 これは、次のコマンドを使用して行うことができます。

``` bash
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

## <a name="copy-kubernetes-certificate"></a>Kubernetes 証明書のコピー

**通常の (非ルート) ユーザーとして**、次の3つの手順を実行します。

1. Linux ディレクトリ用の Kubernetes を作成します。

```bash
mkdir -p $HOME/.kube
```

2. [マスターから](./creating-a-linux-master.md#collect-cluster-information)Kubernetes 証明書ファイル (`$HOME/.kube/config`) をコピーし、ワーカーに `$HOME/.kube/config` として保存します。

> [!tip]
> [Winscp](https://winscp.net/eng/download.php)などの scp ベースのツールを使用して、構成ファイルをノード間で転送できます。

3. コピーした構成ファイルのファイル所有権を次のように設定します。

``` bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## <a name="joining-node"></a>ノードの結合

最後に、クラスターに参加するには、[先ほどメモ](./creating-a-linux-master.md#initialize-master)した `kubeadm join` コマンドを**root として**実行します。

```bash
kubeadm join <Master_IP>:6443 --token <some_token> --discovery-token-ca-cert-hash <some_hash>
```

成功した場合は、次のような出力が表示されます。

![テキスト](./media/node-join.png)

## <a name="next-steps"></a>次のステップ

このセクションでは、Linux ワーカーを Kubernetes クラスターに参加させる方法について説明します。 これで、手順6の準備ができました。
> [!div class="nextstepaction"]
> [Kubernetes リソースのデプロイ](./deploying-resources.md)