---
title: Windows コンテナー ネットワーク
description: Windows コンテナー ネットワークのアーキテクチャを簡単に紹介します。
keywords: Docker, コンテナー
author: jmesser81
ms.date: 03/27/2018
ms.topic: article
ms.prod: windows-containers
ms.service: windows-containers
ms.assetid: 538871ba-d02e-47d3-a3bf-25cda4a40965
ms.openlocfilehash: 0d43176a07b0ba23f6a893c1b3dcfa1ffddc423d
ms.sourcegitcommit: db508decd9bf6c0dce9952e1a86bf80f00d025eb
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 07/20/2018
ms.locfileid: "2315655"
---
# <a name="windows-container-networking"></a>Windows コンテナー ネットワーク
> ***免責事項: 一般的な Docker ネットワークのコマンド、オプション、構文については、[Docker コンテナーのネットワークに関するページ](https://docs.docker.com/engine/userguide/networking/)をご覧ください。*** [以下](#unsupported-features-and-network-options)に記載されている場合を除き、Docker ネットワーク コマンドはすべて、Linux と同じ構文を使用して、Windows でサポートされます。 ただし、Windows と Linux のネットワーク スタックは異なっており、そのためいくつかの Linux ネットワーク コマンド (ifconfig など) は、Windows ではサポートされていません。


## <a name="basic-networking-architecture"></a>基本的なネットワーク アーキテクチャ
このトピックでは、Docker で Windows 上にホスト ネットワークを作成して管理する方法の概要を示します。 ネットワークに関して言えば、Windows コンテナーの機能は仮想マシンと似ています。 各コンテナーには、Hyper-V 仮想スイッチ (vSwitch) に接続されている仮想ネットワーク アダプター (vNIC) があります。 Windows では、Docker 経由で作成できる、*nat*、*overlay*、*transparent*、*l2bridge*、*l2tunnel* の 5 種類の[ネットワーク ドライバーまたはモード](./network-drivers-topologies.md)をサポートしています。 物理ネットワークのインフラストラクチャと単一または複数のホストのネットワーク要件に応じて、ニーズに最適なネットワーク ドライバーを選択する必要があります。


![テキスト](media/windowsnetworkstack-simple.png)


初めて Docker エンジンを実行したときに、内部 vSwitch と `WinNAT` という名前の Windows コンポーネントを使用する、既定の NAT ネットワーク 'nat' が作成されます。 PowerShell または Hyper-V マネージャーで作成されたホストに既存の外部 vSwitch がある場合、*transparent* ネットワーク ドライバーを使用して Docker でも利用でき、``docker network ls`` コマンドを実行するとに表示できます。  


![テキスト](media/docker-network-ls.png)


> - ***内部*** vSwitch とは、コンテナー ホスト上のネットワーク アダプターに直接接続されていない vSwitch です。 

> - ***外部*** vSwitch とは、コンテナー ホスト上のネットワーク アダプターに直接接続されている__ vSwitch です。  


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
    - HNS API およびスキーマが**近日中に公開されます**。


![テキスト](media/HNS-Management-Stack.png)


 ## <a name="unsupported-features-and-network-options"></a>サポートされていない機能とネットワーク オプション
 次のネットワーク オプションは現在**いない**Windows でサポートされています。
   * IPsec によるコンテナーの通信を暗号化します。
   * コンテナーの HTTP プロキシ サポートします。  この準備の現在価値を追跡する[次のとおり](https://github.com/Microsoft/hcsshim/pull/163)です。
   * HYPER-V コンテナーを実行する端点を添付すると (追加)。
   * 透明なネットワーク ドライバー経由での仮想化された Azure インフラストラクチャで、ネットワーク接続します。

 | コマンド        | サポートされていないオプション   |
 | ---------------|:--------------------:|
 | ``docker run``|   ``--ip6``, ``--dns-option`` |
 | ``docker network create``| ``--aux-address``, ``--internal``, ``--ip-range``, ``--ipam-driver``, ``--ipam-opt``, ``--ipv6``, ``--opt encrypted`` |