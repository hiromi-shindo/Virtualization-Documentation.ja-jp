---
title: Windows ノードの結合
author: daschott
ms.author: daschott
ms.date: 11/02/2018
ms.topic: get-started-article
ms.prod: containers
description: Windows ノードを Kubernetes クラスターに追加する (v 1.14)。
keywords: kubernetes、1.14、windows、はじめに
ms.assetid: 3b05d2c2-4b9b-42b4-a61b-702df35f5b17
ms.openlocfilehash: c9dbfec968d52d9fbc528892f0e3749270e3ff70
ms.sourcegitcommit: 1ca9d7562a877c47f227f1a8e6583cb024909749
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 12/04/2019
ms.locfileid: "74910332"
---
# <a name="joining-windows-server-nodes-to-a-cluster"></a>クラスターへの Windows Server ノードの参加 #
[Kubernetes マスターノードを設定](./creating-a-linux-master.md)し、目的の[ネットワークソリューションを選択](./network-topologies.md)したら、Windows Server ノードに参加してクラスターを形成することができます。 これを行うには、参加する前に[Windows ノードの準備を](#preparing-a-windows-node)行う必要があります。

## <a name="preparing-a-windows-node"></a>Windows のノードの準備 ##
> [!NOTE]  
> Windows セクションでのすべてのコード スニペットは、_管理者特権の_ PowerShell で実行します。

### <a name="install-docker-requires-reboot"></a>Docker のインストール (再起動が必要) ###
Kubernetes は[Docker](https://www.docker.com/)をコンテナーエンジンとして使用するため、Docker をインストールする必要があります。 [Docs の公式手順](../manage-docker/configure-docker-daemon.md#install-docker)または [Docker の手順](https://store.docker.com/editions/enterprise/docker-ee-server-windows)に従うことも、以下の手順を試すこともできます。

```powershell
Install-Module -Name DockerMsftProvider -Repository PSGallery -Force
Install-Package -Name Docker -ProviderName DockerMsftProvider
Restart-Computer -Force
```

プロキシの内側にいる場合は、次の PowerShell 環境変数を定義する必要があります。
```powershell
[Environment]::SetEnvironmentVariable("HTTP_PROXY", "http://proxy.example.com:80/", [EnvironmentVariableTarget]::Machine)
[Environment]::SetEnvironmentVariable("HTTPS_PROXY", "http://proxy.example.com:443/", [EnvironmentVariableTarget]::Machine)
```

再起動後に、次のエラーが表示されます。

![テキスト](media/docker-svc-error.png)

次に、docker サービスを手動で開始します。

```powershell
Start-Service docker
```

### <a name="create-the-pause-infrastructure-image"></a>"一時停止" (インフラストラクチャ) イメージを作成する ###
> [!Important]
> コンテナーイメージの競合に注意することが重要です。期待されるタグがないと、互換性のないコンテナーイメージの `docker pull` が発生し、`ContainerCreating` の状態などの[デプロイの問題](./common-problems.md#when-deploying-docker-containers-keep-restarting)が発生する可能性があります。

`docker` のインストールが完了したため、Kubernetes インフラストラクチャ ポッドを準備するために Kubernetes で使用される "pause" イメージを準備する必要があります。 これには、次の3つの手順があります。 
  1. [イメージをプルする](#pull-the-image)
  2. microsoft/nanoserver: latest として[タグ付け](#tag-the-image)する
  3. [実行中](#run-the-container)


#### <a name="pull-the-image"></a>イメージをプルする ####     
 特定の Windows リリースのイメージを取得します。 たとえば、Windows Server 2019 を実行している場合は、次のようになります。

 ```powershell
docker pull mcr.microsoft.com/windows/nanoserver:1809
 ```

#### <a name="tag-the-image"></a>画像にタグを付ける ####
このガイドの後半で使用する Dockerfiles は、`:latest` image タグを検索します。 次のように、プルした nanoserver イメージにタグを付けます。

```powershell
docker tag mcr.microsoft.com/windows/nanoserver:1809 microsoft/nanoserver:latest
```

#### <a name="run-the-container"></a>コンテナーを実行する ####
コンテナーが実際にコンピューター上で実行されていることを再度確認します。

```powershell
docker run microsoft/nanoserver:latest
```

次のように表示されます。

![テキスト](./media/docker-run-sample.png)

> [!tip]
> コンテナーを実行できない場合は、「コンテナー[ホストのバージョンとコンテナーイメージの一致」を](https://docs.microsoft.com/virtualization/windowscontainers/deploy-containers/version-compatibility#matching-container-host-version-with-container-image-versions)参照してください。


#### <a name="prepare-kubernetes-for-windows-directory"></a>Windows ディレクトリ用に Kubernetes を準備する ####
"Kubernetes for Windows" ディレクトリを作成して、Kubernetes バイナリだけでなく、任意の配置スクリプトと構成ファイルを格納します。

```powershell
mkdir c:\k
```

#### <a name="copy-kubernetes-certificate"></a>Kubernetes 証明書のコピー #### 
Kubernetes 証明書ファイル (`$HOME/.kube/config`) を[master から](./creating-a-linux-master.md#collect-cluster-information)この新しい `C:\k` ディレクトリにコピーします。

> [!tip]
> [Xcopy](https://docs.microsoft.com/windows-server/administration/windows-commands/xcopy)や[winscp](https://winscp.net/eng/download.php)などのツールを使用して、構成ファイルをノード間で転送できます。

#### <a name="download-kubernetes-binaries"></a>Kubernetes バイナリのダウンロード ####
Kubernetes を実行できるようにするには、まず、`kubectl`、`kubelet`、および `kube-proxy` バイナリをダウンロードする必要があります。 これらは、[最新リリース](https://github.com/kubernetes/kubernetes/releases/)の `CHANGELOG.md` ファイルのリンクからダウンロードできます。
 - たとえば、次に示すのは、v1.0[ノードのバイナリ](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.14.md#node-binaries)です。
 - [拡張アーカイブ](https://docs.microsoft.com/powershell/module/microsoft.powershell.archive/expand-archive?view=powershell-6)などのツールを使用してアーカイブを抽出し、バイナリを `C:\k\`に配置します。

#### <a name="optional-setup-kubectl-on-windows"></a>OptionalWindows での kubectl のセットアップ ####
Windows からクラスターを制御する必要がある場合は、`kubectl` コマンドを使用して実行できます。 まず、`C:\k\` ディレクトリ以外で `kubectl` 使用できるようにするには、`PATH` 環境変数を変更します。

```powershell
$env:Path += ";C:\k"
```

この変更を永続化する場合は、コンピューター ターゲット内の変数を変更します。

```powershell
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\k", [EnvironmentVariableTarget]::Machine)
```

次に、[クラスター証明書](#copy-kubernetes-certificate)が有効であることを確認します。 `kubectl` が構成ファイルを検索する場所を設定するには、`--kubeconfig` パラメーターを渡すか、または `KUBECONFIG` 環境変数を変更します。 たとえば、`C:\k\config` に構成がある場合は、次のように設定します。

```powershell
$env:KUBECONFIG="C:\k\config"
```

現在のユーザーのスコープでこの設定を永続化するには、次のように設定します。

```powershell
[Environment]::SetEnvironmentVariable("KUBECONFIG", "C:\k\config", [EnvironmentVariableTarget]::User)
```

最後に、構成が適切に検出されたかどうかを確認するには、次のようにします。

```powershell
kubectl config view
```

接続エラーが発生する場合は、

```
Unable to connect to the server: dial tcp [::1]:8080: connectex: No connection could be made because the target machine actively refused it.
```

Kubeconfig の場所を再確認するか、もう一度コピーしてみてください。

エラーが表示されない場合は、ノードがクラスターに参加する準備ができています。

## <a name="joining-the-windows-node"></a>Windows ノードに参加しています ##
[選択したネットワークソリューション](./network-topologies.md)に応じて、次のことができます。
1. [Windows Server ノードを Flannel (vxlan またはホスト gw) クラスターに参加させる](#joining-a-flannel-cluster)
2. [Windows Server ノードを ToR スイッチを使用してクラスターに参加させる](#joining-a-tor-cluster)

### <a name="joining-a-flannel-cluster"></a>Flannel クラスターへの参加 ###
この[Microsoft リポジトリ](https://github.com/Microsoft/SDN/tree/master/Kubernetes/flannel/overlay)には、このノードをクラスターに参加させるのに役立つ Flannel デプロイスクリプトのコレクションがあります。

[Flannel](https://github.com/Microsoft/SDN/blob/master/Kubernetes/flannel/start.ps1)スクリプトをダウンロードし、その内容を `C:\k`に抽出する必要があります。

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
wget https://raw.githubusercontent.com/Microsoft/SDN/master/Kubernetes/flannel/start.ps1 -o c:\k\start.ps1
```

[Windows ノードを準備](#preparing-a-windows-node)し、`c:\k` ディレクトリが次のようになっていると仮定すると、ノードに参加することができます。

![テキスト](./media/flannel-directory.png)

#### <a name="join-node"></a>ノードの結合 #### 
Windows ノードに参加するプロセスを簡略化するには、1つの Windows スクリプトを実行して、`kubelet`、`kube-proxy`、`flanneld`を起動し、ノードに参加させる必要があります。

> [!Note]
> [開始します](https://github.com/Microsoft/SDN/blob/master/Kubernetes/flannel/start.ps1)。 [ps1 は、`flanneld`](https://github.com/Microsoft/SDN/blob/master/Kubernetes/windows/install.ps1)実行可能ファイルや、[インフラストラクチャポッドの dockerfile](https://github.com/Microsoft/SDN/blob/master/Kubernetes/windows/Dockerfile)などの追加ファイルをダウンロードして、*それらをインストール*します。 オーバーレイネットワークモードでは、ローカル UDP ポート4789の[ファイアウォール](https://github.com/Microsoft/SDN/blob/master/Kubernetes/windows/helper.psm1#L111)が開きます。 複数の powershell ウィンドウが開いているか閉じられている場合があります。また、ネットワークが停止している間に、ポッドネットワーク用の新しい外部 vSwitch が初めて作成されていることもあります。

```powershell
cd c:\k
.\start.ps1 -ManagementIP <Windows Node IP> -NetworkMode <network mode>  -ClusterCIDR <Cluster CIDR> -ServiceCIDR <Service CIDR> -KubeDnsServiceIP <Kube-dns Service IP> -LogDir <Log directory>
```
# <a name="managementiptabmanagementip"></a>[ManagementIP](#tab/ManagementIP)
Windows ノードに割り当てられた IP アドレス。 `ipconfig` を使用すると、これを見つけることができます。

|  |  | 
|---------|---------|
|パラメーター     | `-ManagementIP`        |
|既定値    | n.A. "**必須**"        |

# <a name="networkmodetabnetworkmode"></a>[NetworkMode](#tab/NetworkMode)
ネットワークモード `l2bridge` (flannel host gw)、または[ネットワークソリューション](./network-topologies.md)として選択された `overlay` (flannel vxlan)。

> [!Important] 
> `overlay` ネットワークモード (flannel vxlan) には、Kubernetes v 1.14 のバイナリ (またはそれ以上) と[KB4489899](https://support.microsoft.com/help/4489899)が必要です。

|  |  | 
|---------|---------|
|パラメーター     | `-NetworkMode`        |
|既定値    | `l2bridge`        |


# <a name="clustercidrtabclustercidr"></a>[ClusterCIDR](#tab/ClusterCIDR)
[クラスターサブネットの範囲](./getting-started-kubernetes-windows.md#cluster-subnet-def)。

|  |  | 
|---------|---------|
|パラメーター     | `-ClusterCIDR`        |
|既定値    | `10.244.0.0/16`        |


# <a name="servicecidrtabservicecidr"></a>[Servicが Dr](#tab/ServiceCIDR)
[サービスサブネットの範囲](./getting-started-kubernetes-windows.md#service-subnet-def)。

|  |  | 
|---------|---------|
|パラメーター     | `-ServiceCIDR`        |
|既定値    | `10.96.0.0/12`        |


# <a name="kubednsserviceiptabkubednsserviceip"></a>[KubeDnsServiceIP](#tab/KubeDnsServiceIP)
[KUBERNETES DNS サービス IP](./getting-started-kubernetes-windows.md#plan-ip-addressing-for-your-cluster)。

|  |  | 
|---------|---------|
|パラメーター     | `-KubeDnsServiceIP`        |
|既定値    | `10.96.0.10`        |


# <a name="interfacenametabinterfacename"></a>[InterfaceName](#tab/InterfaceName)
Windows ホストのネットワークインターフェイスの名前。 `ipconfig` を使用すると、これを見つけることができます。

|  |  | 
|---------|---------|
|パラメーター     | `-InterfaceName`        |
|既定値    | `Ethernet`        |


# <a name="logdirtablogdir"></a>[LogDir](#tab/LogDir)
Kubelet および kube ログがそれぞれの出力ファイルにリダイレクトされるディレクトリ。

|  |  | 
|---------|---------|
|パラメーター     | `-LogDir`        |
|既定値    | `C:\k`        |


---

> [!tip]
> [前](./creating-a-linux-master.md#collect-cluster-information)に Linux マスターからクラスターサブネット、サービスサブネット、および kube IP をメモしておきます。

これを実行すると、次のことができるようになります。
  * `kubectl get nodes` を使用して参加している Windows ノードを表示する
  * Powershell ウィンドウを開き、1つ `kubelet`、`flanneld`用、`kube-proxy` もう1つを参照してください。
  * ノードで実行されている `flanneld`、`kubelet`、および `kube-proxy` のホストエージェントプロセスを参照してください。

成功した場合は、[次の手順](#next-steps)に進みます。

## <a name="joining-a-tor-cluster"></a>ToR クラスターへの参加 ##
> [!NOTE]
> [以前](./network-topologies.md#flannel-in-host-gateway-mode)にネットワークソリューションとして Flannel を選択した場合は、このセクションをスキップできます。

これを行うには、[上流の L3 ルーティングトポロジ用に Kubernetes で Windows Server コンテナーを設定](https://kubernetes.io/docs/getting-started-guides/windows/#for-1-upstream-l3-routing-topology-and-2-host-gateway-topology)するための手順に従う必要があります。 これには、ノードに割り当てられたポッド CIDR プレフィックスがそれぞれのノード IP にマップされるように上流ルーターを構成することが含まれます。

新しいノードが `kubectl get nodes`によって "Ready" と表示され、kubelet + kube が実行されていて、アップストリーム ToR ルーターを構成している場合は、次の手順に進むことができます。

## <a name="next-steps"></a>次のステップ ##
このセクションでは、Windows ワーカーを Kubernetes クラスターに参加させる方法について説明します。 これで、手順5の準備ができました。

> [!div class="nextstepaction"]
> [Linux ワーカーの参加](./joining-linux-workers.md)

また、Linux ワーカーがいない場合は、手順 6. に進んでください。

> [!div class="nextstepaction"]
> [Kubernetes リソースのデプロイ](./deploying-resources.md)
