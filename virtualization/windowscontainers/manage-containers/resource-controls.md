---
title: "リソース コントロールの実装"
description: "Windows コンテナーのリソース コントロールに関する詳細情報"
keywords: "docker, コンテナー, cpu, メモリ, ディスク, リソース"
author: taylorb-microsoft
ms.date: 11/21/2017
ms.topic: article
ms.prod: windows-containers
ms.service: windows-containers
ms.assetid: 8ccd4192-4a58-42a5-8f74-2574d10de98e
ms.openlocfilehash: bc36f1f59ed339b2cc3dd3372a5cd5119f470c7c
ms.sourcegitcommit: 64f5f8d838f72ea8e0e66a72eeb4ab78d982b715
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 11/22/2017
---
# <a name="implementing-resource-controls-for-windows-containers"></a>Windows コンテナーのリソース コントロールの実装
リソース コントロールには、コンテナー単位およびリソース単位で実装できるものがいくつかあります。  既定では、一般的な Windows リソース管理 (通常はフェアシェア ベース) によって、実行されるコンテナーが決まりますが、リソース コントロールを構成することで、開発者や管理者はリソース使用の制限または調整を行うことができます。  コントロール可能なリソースには、CPU/プロセッサ、メモリ/RAM、ディスク/記憶域、ネットワーク/スループットなどがあります。
Windows コンテナーでは、各コンテナーに関連付するプロセスのグループ化と追跡に、[ジョブ オブジェクト]( https://msdn.microsoft.com/en-us/library/windows/desktop/ms684161(v=vs.85).aspx)が使用されます。  リソース コントロールは、コンテナーに関連付けられた親ジョブ オブジェクトに実装されます。  [Hyper-V による分離](https://docs.microsoft.com/en-us/virtualization/windowscontainers/about/index#windows-container-types)の場合、リソース コントロールは仮想マシンにも、仮想マシン内で実行されているコンテナーのジョブ オブジェクトにも自動的に適用されます。これにより、コンテナー内で実行されているプロセスがジョブ オブジェクトのコントロールをバイパスまたはエスケープした場合も、定義されているリソース コントロールを超過しないよう仮想マシンによって制御されます。

## <a name="resources"></a>リソース
このセクションでは、各リソースについて、リソース コントロールの使用例として Docker コマンド ライン インターフェイス (オーケストレータまたはその他のツールによって構成される場合もあります) および対応する Windows ホスト コンピューティング サービス (HCS) API を示します。また、Windows によるリソース コントロールの一般的な実装方法も示します (ここに示す説明は概要であり、基になる実装は変わることがあります)。

|  | |
| ----- | ------|
| *メモリ* ||
| Docker インターフェイス | [--memory](https://docs.docker.com/engine/admin/resource_constraints/#memory) |
| HCS インターフェイス | [MemoryMaximumInMB]( https://github.com/Microsoft/hcsshim/blob/b144c605002d4086146ca1c15c79e56bfaadc2a7/interface.go#L67) |
| 共有カーネル | [JOB_OBJECT_LIMIT_JOB_MEMORY](https://msdn.microsoft.com/en-us/library/windows/desktop/ms684147(v=vs.85).aspx) |
| Hyper-V による分離 | バーチャル マシン メモリ |
| ||
| *CPU (数)* ||
| Docker インターフェイス | [--cpus](https://docs.docker.com/engine/admin/resource_constraints/#cpu) |
| HCS インターフェイス | [ProcessorCount]( https://github.com/Microsoft/hcsshim/blob/b144c605002d4086146ca1c15c79e56bfaadc2a7/interface.go#L67) |
| 共有カーネル | [JOB_OBJECT_CPU_RATE_CONTROL_HARD_CAP](https://msdn.microsoft.com/en-us/library/windows/desktop/hh448384(v=vs.85).aspx)* でシミュレート |
| Hyper-V による分離 | 公開されている仮想プロセッサの数 |
| ||
| *CPU (パーセント)* ||
| Docker インターフェイス | [--cpu-percent](https://docs.docker.com/engine/admin/resource_constraints/#cpu) |
| HCS インターフェイス | [ProcessorMaximum](https://github.com/Microsoft/hcsshim/blob/b144c605002d4086146ca1c15c79e56bfaadc2a7/interface.go#L67) |
| 共有カーネル | [JOB_OBJECT_CPU_RATE_CONTROL_HARD_CAP](https://msdn.microsoft.com/en-us/library/windows/desktop/hh448384(v=vs.85).aspx) |
| Hyper-V による分離 | 仮想プロセッサ上のハイパーバイザー制限 |
| ||
| *CPU (共有)* ||
| Docker インターフェイス | [--cpu-shares](https://docs.docker.com/engine/admin/resource_constraints/#cpu) |
| HCS インターフェイス | [ProcessorWeight](https://github.com/Microsoft/hcsshim/blob/b144c605002d4086146ca1c15c79e56bfaadc2a7/interface.go#L67) |
| 共有カーネル | [JOB_OBJECT_CPU_RATE_CONTROL_WEIGHT_BASED](https://msdn.microsoft.com/en-us/library/windows/desktop/hh448384(v=vs.85).aspx) |
| Hyper-V による分離 | ハイパーバイザーの仮想プロセッサの重み |
| ||
| *記憶域 (イメージ)* ||
| Docker インターフェイス | [--io-maxbandwidth/--io-maxiops]( https://docs.docker.com/edge/engine/reference/commandline/run/#usage) |
| HCS インターフェイス | [StorageIOPSMaximum と StorageBandwidthMaximum](https://github.com/Microsoft/hcsshim/blob/b144c605002d4086146ca1c15c79e56bfaadc2a7/interface.go#L67) |
| 共有カーネル | [JobObjectIoRateControlInformation](https://msdn.microsoft.com/en-us/library/windows/desktop/hh448384(v=vs.85).aspx) |
| Hyper-V による分離 | [JobObjectIoRateControlInformation](https://msdn.microsoft.com/en-us/library/windows/desktop/hh448384(v=vs.85).aspx) |
| ||
| *記憶域 (ボリューム)* ||
| Docker インターフェイス | [--storage-opt size=]( https://docs.docker.com/edge/engine/reference/commandline/run/#set-storage-driver-options-per-container) |
| HCS インターフェイス | [StorageSandboxSize](https://github.com/Microsoft/hcsshim/blob/b144c605002d4086146ca1c15c79e56bfaadc2a7/interface.go#L67) |
| 共有カーネル | [JobObjectIoRateControlInformation](https://msdn.microsoft.com/en-us/library/windows/desktop/hh448384(v=vs.85).aspx) |
| Hyper-V による分離 | [JobObjectIoRateControlInformation](https://msdn.microsoft.com/en-us/library/windows/desktop/hh448384(v=vs.85).aspx) |

## <a name="additional-notes-or-details"></a>追加の注意事項または詳細情報
### <a name="memory"></a>メモリ
Windows コンテナーは、各コンテナーでシステム プロセスを実行します。これらのシステム プロセスは一般的に、ユーザー管理やネットワーキングなど、コンテナー単位の機能を提供します。 また、プロセスに必要なメモリの多くは複数のコンテナー間で共有されますが、メモリ容量はこれらに対応できる十分な量にする必要があります。  [システム要件](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/system-requirements#memory-requirments)に関するドキュメントに、各基本イメージのタイプと Hyper-V による分離の有無に応じた容量が示されています。

### <a name="cpu-shares-without-hyper-v-isolation"></a>CPU 共有 (Hyper-V による分離なし)
CPU 共有を使用する場合、基になる実装 (Hyper-V による分離を使用しない場合) によって [JOBOBJECT_CPU_RATE_CONTROL_INFORMATION](https://msdn.microsoft.com/en-us/library/windows/desktop/hh448384(v=vs.85).aspx) が構成されます。具体的には、コントロール フラグが JOB_OBJECT_CPU_RATE_CONTROL_WEIGHT_BASED に設定され、適切な重みが指定されます。  ジョブ オブジェクトの有効な重みは 1 ～ 9、既定値は 5 で、ホスト コンピューティング サービスの値 1 ～ 10000 より精度は低くなります。  たとえば、共有の重みが 7500 であれば値は 7、共有の重みが 2500 であれば値は 2 になります。
