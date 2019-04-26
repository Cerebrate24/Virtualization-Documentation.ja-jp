---
title: Hyper-V と Windows PowerShell の使用
description: Hyper-V と Windows PowerShell の使用
keywords: windows 10、hyper-v
author: scooley
ms.date: 05/02/2016
ms.topic: article
ms.prod: windows-10-hyperv
ms.service: windows-10-hyperv
ms.assetid: 6d1ae036-0841-4ba5-b7e0-733aad31e9a7
ms.openlocfilehash: 3991895d381897f27c34aa7840fde44d2cedae7e
ms.sourcegitcommit: 0deb653de8a14b32a1cfe3e1d73e5d3f31bbe83b
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 04/26/2019
ms.locfileid: "9576563"
---
# <a name="working-with-hyper-v-and-windows-powershell"></a>Hyper-V と Windows PowerShell の使用

Hyper-V の展開、仮想マシンの作成、仮想マシンの管理の基本を確認できたので、次は PowerShell でこれらの作業の大半を自動化する方法について説明します。

### <a name="return-a-list-of-hyper-v-commands"></a>HYPER-V のコマンドの一覧を返します

1.  Windows の [スタート] ボタンをクリックし、「**PowerShell**」と入力します。
2.  次のコマンドを実行すると、Hyper-V PowerShell モジュールで利用できる PowerShell コマンドの検索可能な一覧が表示されます

 ```powershell
Get-Command -Module hyper-v | Out-GridView
```
  次のような一覧になります。

  ![](media\command_grid.png)

3. 特定の PowerShell コマンドの詳細を確認するには、`Get-Help` を使用します。 たとえば、次のコマンドを実行すると、`Get-VM` Hyper-V コマンドに関する情報が返されます。

  ```powershell
Get-Help Get-VM
```
 コマンドを構築する方法、必須と任意のパラメーター、および使用できるエイリアスが出力されます。

 ![](media\get_help.png)


### <a name="return-a-list-of-virtual-machines"></a>仮想マシンの一覧を返す

仮想マシンの一覧を返すには、`Get-VM` コマンドを使用します。

1. PowerShell で次のコマンドを実行します。
 
 ```powershell
Get-VM
```
 次のような出力が表示されます。

 ![](media\get_vm.png)

2. 電源がオンになっている仮想マシンのみの一覧を返す場合、`Get-VM` コマンドにフィルターを追加します。 フィルターは `Where-Object` コマンドを使用して追加できます。 フィルター処理の詳細については、[Where-Object の使用](https://technet.microsoft.com/en-us/library/ee177028.aspx)に関するドキュメントをご覧ください。   

 ```powershell
 Get-VM | where {$_.State -eq 'Running'}
 ```
3.  電源のすべての仮想マシンの一覧を表示する、状態を無効には、次のコマンドを実行します。 このコマンドは手順 2. のコマンドのコピーで、フィルターを "Running" から "Off" に変更したものです。

 ```powershell
 Get-VM | where {$_.State -eq 'Off'}
 ```

### <a name="start-and-shut-down-virtual-machines"></a>起動し、仮想マシンをシャットダウン

1. 特定のバーチャル マシンを開始するには、仮想マシンの名前で、次のコマンドを実行します。

 ```powershell
 Start-VM -Name <virtual machine name>
 ```

2. 現在電源がオフになっているすべての仮想マシンを起動するには、そのようなマシンの一覧を取得し、`Start-VM` コマンドに対して一覧を渡します。

  ```powershell
 Get-VM | where {$_.State -eq 'Off'} | Start-VM
 ```
3. 実行中のすべての仮想マシンをシャットダウンするには、次を実行します。
 
  ```powershell
 Get-VM | where {$_.State -eq 'Running'} | Stop-VM
 ```

### <a name="create-a-vm-checkpoint"></a>VM チェックポイントを作成する

PowerShell を使用してチェックポイントを作成するには、`Get-VM` コマンドを使用して仮想マシンを選択し、それを `Checkpoint-VM` コマンドに渡します。 最後に、`-SnapshotName` を使用してチェックポイントに名前を付けます。 完全なコマンドは次のようになります。

 ```powershell
 Get-VM -Name <VM Name> | Checkpoint-VM -SnapshotName <name for snapshot>
 ```
### <a name="create-a-new-virtual-machine"></a>新しい仮想マシンを作成する

次の例は、PowerShell Integrated Scripting Environment (ISE) で新しい仮想マシンを作成する方法を示すものです。 これは単純な例であり、拡張して PowerShell 機能やより高度な VM 展開を追加できます。

1. PowerShell ISE を開くには、[開始] をクリックし、「**PowerShell ISE**」と入力します。
2. 次のコードを実行し、仮想マシンを作成します。 `New-VM` コマンドの詳細については、[New-VM](https://technet.microsoft.com/en-us/library/hh848537.aspx) ドキュメントを参照してください。

  ```powershell
 $VMName = "VMNAME"

 $VM = @{
     Name = $VMName 
     MemoryStartupBytes = 2147483648
     Generation = 2
     NewVHDPath = "C:\Virtual Machines\$VMName\$VMName.vhdx"
     NewVHDSizeBytes = 53687091200
     BootDevice = "VHD"
     Path = "C:\Virtual Machines\$VMName"
     SwitchName = (Get-VMSwitch).Name
 }

 New-VM @VM
  ```

## <a name="wrap-up-and-references"></a>ラップし、参照

このドキュメントはいくつかのサンプル シナリオと同様に、HYPER-V の PowerShell モジュールに、エクスプ ローラーにいくつかの簡単な手順を説明しました。 Hyper-V PowerShell モジュールの詳細については、[Windows PowerShell リファレンスの Hyper-V コマンドレット](https://technet.microsoft.com/%5Clibrary/Hh848559.aspx)をご覧ください。  
 
