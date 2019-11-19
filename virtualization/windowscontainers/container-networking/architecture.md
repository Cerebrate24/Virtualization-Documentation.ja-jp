---
title: Windows コンテナーネットワーク
description: Windows コンテナー ネットワークのアーキテクチャを簡単に紹介します。
keywords: Docker, コンテナー
author: jmesser81
ms.date: 03/27/2018
ms.topic: article
ms.prod: windows-containers
ms.service: windows-containers
ms.assetid: 538871ba-d02e-47d3-a3bf-25cda4a40965
ms.openlocfilehash: e9d4a9ac88c6853ce019a2469ee80688490b8fdf
ms.sourcegitcommit: bb4ec1f05921f982c00bdb3ace6d9bc1d5355296
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 11/18/2019
ms.locfileid: "10297243"
---
# <a name="windows-container-networking"></a>Windows コンテナーネットワーク

>[!IMPORTANT]
>一般的な docker ネットワークコマンド、オプション、構文については、「 [Docker Container ネットワーク](https://docs.docker.com/engine/userguide/networking/)」を参照してください。 * * * サポートされて[いない機能とネットワークオプション](#unsupported-features-and-network-options)で説明されているケースを除き、すべての docker ネットワークコマンドは、Linux と同じ構文で Windows でサポートされています。 ただし、Windows と Linux のネットワークスタックは異なるため、Windows では一部の Linux ネットワークコマンド (ifconfig など) がサポートされていないことがわかります。

## <a name="basic-networking-architecture"></a>基本的なネットワーク アーキテクチャ

このトピックでは、Docker で Windows 上にホスト ネットワークを作成して管理する方法の概要を示します。 ネットワークに関して言えば、Windows コンテナーの機能は仮想マシンと似ています。 各コンテナーには、Hyper-V 仮想スイッチ (vSwitch) に接続されている仮想ネットワーク アダプター (vNIC) があります。 Windows では、Docker 経由で作成できる、*nat*、*overlay*、*transparent*、*l2bridge*、*l2tunnel* の 5 種類の[ネットワーク ドライバーまたはモード](./network-drivers-topologies.md)をサポートしています。 物理ネットワークのインフラストラクチャと単一または複数のホストのネットワーク要件に応じて、ニーズに最適なネットワーク ドライバーを選択する必要があります。

![テキスト](media/windowsnetworkstack-simple.png)

初めて Docker エンジンを実行したときに、内部 vSwitch と `WinNAT` という名前の Windows コンポーネントを使用する、既定の NAT ネットワーク 'nat' が作成されます。 PowerShell または Hyper-V マネージャーで作成されたホストに既存の外部 vSwitch がある場合、*transparent* ネットワーク ドライバーを使用して Docker でも利用でき、``docker network ls`` コマンドを実行するとに表示できます。  

![テキスト](media/docker-network-ls.png)

- **内部**vSwitch は、コンテナーホストのネットワークアダプターに直接接続されていないものです。
- **外部**vSwitch は、コンテナーホスト上のネットワークアダプターに直接接続されたものです。

![テキスト](media/get-vmswitch.png)

'nat' ネットワークとは、Windows で実行されているコンテナー用の既定ネットワークです。 特定のネットワーク構成を実装するフラグや引数を指定せずに Windows で実行されているすべてのコンテナーは、既定の 'nat' ネットワークに接続され、'nat' ネットワークの内部プレフィックス IP 範囲から自動的に IP アドレスが割り当てられます。 'nat' に使用される既定の内部 IP プレフィックスは、172.16.0.0/16 です。 

## <a name="container-network-management-with-host-network-service"></a>ホスト ネットワーク サービスによるコンテナー ネットワーク管理

ホスト ネットワーク サービス (HNS) とホスト コンピューティング サービス (HCS) は、連携してコンテナーを作成し、エンドポイントをネットワークに接続します。

### <a name="network-creation"></a>ネットワークの作成

- HNS は、各ネットワークの Hyper-V 仮想スイッチを作成します。
- HNS は、必要に応じて NAT プールと IP プールを作成します。

### <a name="endpoint-creation"></a>エンドポイントの作成

- HNS は、コンテナー エンドポイントごとにネットワーク名前空間を作成します。
- HNS/HCS は v(m)NIC をネットワーク名前空間内に配置します。
- HNS は (vSwitch) ポートを作成します。
- HNS は、IP アドレス、DNS 情報、ルートなど (ネットワーク モードに依存) をエンドポイントに割り当てます。

### <a name="policy-creation"></a>ポリシーの作成

- 既定の NAT ネットワーク: HNS は、WinNAT ポート フォワーディング規則/マッピングを、対応する Windows ファイアウォールの ALLOW 規則と共に作成します。
- その他すべてのネットワーク: HNS は、仮想フィルタリング プラットフォーム (VFP) を利用してポリシーを作成します。
    - これには、負荷分散、ACL、カプセル化などが含まれます。
    - [ここで](https://docs.microsoft.com/en-us/windows-server/networking/technologies/hcn/hcn-top)公開されている HNS api とスキーマを参照してください。

![テキスト](media/HNS-Management-Stack.png)

## <a name="unsupported-features-and-network-options"></a>サポートされていない機能とネットワーク オプション

Windows では、現在、次のネットワークオプションがサポートされて**いません**。

- L2bridge、NAT、オーバーレイネットワークに接続された Windows コンテナーは、IPv6 スタック経由の通信をサポートしていません。
- IPsec 経由の暗号化されたコンテナーの通信。
- コンテナーの HTTP プロキシのサポート。
- [ホストモード](https://docs.docker.com/ee/ucp/interlock/config/host-mode-networking/)のネットワーク 
- 透過的なネットワークドライバーを使用した、仮想化された Azure インフラストラクチャのネットワーク。

| コマンド        | サポートされていないオプション   |
|---------------|:--------------------:|
| ``docker run``|   ``--ip6``, ``--dns-option`` |
| ``docker network create``| ``--aux-address``, ``--internal``, ``--ip-range``, ``--ipam-driver``, ``--ipam-opt``, ``--ipv6``, ``--opt encrypted`` |
