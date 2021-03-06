---
title: コンテナー内の永続的な記憶域
description: Windows コンテナーで記憶域を保持する方法
keywords: コンテナー, ボリューム, 記憶域, マウント, bindmount
author: cwilhit
ms.topic: conceptual
ms.openlocfilehash: 0e4fcde4fcb0b02b7e1e1b96ad930703207f7b0c
ms.sourcegitcommit: 1bafb5de322763e7f8b0e840b96774e813c39749
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 06/22/2020
ms.locfileid: "85192769"
---
# <a name="persistent-storage-in-containers"></a>コンテナー内の永続的な記憶域

<!-- Great diagram would be great! -->

コンテナー内にデータを保持できる機能が重要なアプリの場合や、コンテナーのビルド時に、コンテナー内に含まれていないファイルを表示する場合があります。 永続的な記憶域をコンテナーに割り当てるには、いくつかの方法があります。

- バインド マウント
- 名前付きボリューム

Docker には、[ボリュームの使用](https://docs.docker.com/engine/admin/volumes/volumes/)に関する概要が用意されているため、まずそのドキュメントをご覧ください。 このページの残りの部分では、Linux と Windows の違いについて説明し、Windows の場合の例を示します。

## <a name="bind-mounts"></a>バインド マウント

[バインド マウント](https://docs.docker.com/engine/admin/volumes/bind-mounts/)を使用すると、コンテナーがディレクトリをホストと共有できます。 この方法は、コンテナーを再起動したときに使用可能なローカル コンピューター上のファイルを保存する場所が必要な場合や、複数のコンテナーと共有する場合に有効です。 コンテナーを複数のコンピューターで実行して同じファイルにアクセスする場合は、代わりに名前付きボリュームまたは SMB マウントを使用する必要があります。

### <a name="permissions"></a>アクセス許可

バインド マウントに使用されるアクセス許可モデルは、コンテナーの分離レベルによって異なります。

**Hyper-V による分離**を使用するコンテナーには、単純な読み取り専用または読み取りおよび書き込みのアクセス許可モデルを使用します。 ホスト上のファイル アクセスには `LocalSystem` アカウントを使用します。 コンテナーに対するアクセスが拒否される場合は、ホスト上のそのディレクトリに `LocalSystem` からのアクセス許可があることを確認してください。 読み取り専用フラグが使用された場合、コンテナー内のボリュームへの変更はホスト上のディレクトリに永続化または表示されません。

**プロセス分離**を使用する Windows コンテナーは、データへのアクセスにコンテナー内のプロセス ID を使用する (ファイルの ACL が優先される) ため、やや異なります。 マウントされたボリューム内のファイルとディレクトリへのアクセスには、`LocalSystem` ではなく、コンテナーで実行されているプロセスの ID (既定で Windows サーバー コアの場合は "ContainerAdministrator"、Nano Server コンテナーの場合は "ContainerUser") が使用され、データを使用するにはアクセス権が必要になります。

これらの ID は、ファイルが格納されているホスト上ではなくコンテナーのコンテキスト内でのみ存在するため、コンテナーへのアクセスを許可する ACL を構成する際には、`Authenticated Users` など既知のセキュリティ グループを使用する必要があります。

> [!WARNING]
> `C:\` などの機密性の高いディレクトリは、信頼されていないコンテナーにバインド マウントしないでください。 このようなバインド マウントを行うと、通常はアクセスできないホスト上のファイルへの変更が許可され、セキュリティ侵害が生じる可能性があります。

使用例:

- `docker run -v c:\ContainerData:c:\data:RO` (読み取り専用アクセス用)
- `docker run -v c:\ContainerData:c:\data:RW` (読み取り/書き込みアクセス用)
- `docker run -v c:\ContainerData:c:\data` (読み取り/書き込みアクセス用、既定)

### <a name="symlinks"></a>symlink

symlink は、コンテナ内で解決されます。 symlink であるコンテナーまたは symlink を含むコンテナーにホスト パスをバインド マウントしても、コンテナーではこれらの symlink にアクセスできません。

### <a name="smb-mounts"></a>SMB マウント

Windows Server Version 1709 以降では、"SMB グローバル マッピング" という機能により、SMB 共有をホストにマウントし、その共有上のディレクトリをコンテナーに渡すことができます。 コンテナーは、特定のサーバー、共有、ユーザー名、またはパスワードを指定して構成する必要はありません。これはすべてホストで処理されます。 コンテナーは、ローカル記憶域がある場合と同様に機能します。

#### <a name="configuration-steps"></a>構成手順

1. コンテナー ホストで、リモート SMB 共有をグローバルにマップします。
    ```
    $creds = Get-Credential
    New-SmbGlobalMapping -RemotePath \\contosofileserver\share1 -Credential $creds -LocalPath G:
    ```
    このコマンドを実行すると、資格情報を使用してリモート SMB サーバーで認証が行われます。 次に、リモート共有パスを G: ドライブ文字にマップします (その他の使用可能なドライブ文字も指定できます)。 このコンテナー ホストで作成されたコンテナーでは、自身のデータ ボリュームを G: ドライブ上のパスにマップできます。

    > [!NOTE]
    > コンテナー用に SMB グローバル マッピングを使用している場合、コンテナー ホストのユーザーはすべて、リモート共有にアクセスできます。 コンテナー ホストで実行されているすべてのアプリケーションにも、マッピングされたリモート共有へのアクセス権があります。

2. グローバルにマウントされた SMB 共有にマップされたデータ ボリュームを使用してコンテナーを作成します: docker run -it --name demo -v g:\ContainerData:c:\AppData1 mcr.microsoft.com/windows/servercore:ltsc2019 cmd.exe

    コンテナー内で、c:\AppData1 はリモート共有の "ContainerData" ディレクトリにマップされます。 グローバルにマップされたリモート共有に格納されているデータは、コンテナー内のアプリケーションで使用できるようになります。 複数のコンテナーが同じコマンドを使用して、この共有データへの読み取り/書き込みアクセス権を獲得することもできます。

この SMB グローバル マッピングは、互換性のある SMB サーバー上で動作できる SMB クライアント側の機能によってサポートされます。次のようなものがあります。

- 記憶域スペース ダイレクト (S2D) 上のスケール アウト ファイル サーバーまたは従来の SAN
- Azure Files (SMB 共有)
- 従来のファイル サーバー
- サード パーティによる SMB プロトコルの実装 (例: NAS アプライアンス)

> [!NOTE]
> SMB グローバル マッピングでは、Windows Server Version 1709 の DFS、DFSN、および DFSR 共有がサポートされていません。

## <a name="named-volumes"></a>名前付きボリューム

名前付きボリュームの機能を使用すると、名前を指定してボリュームを作成し、コンテナーに割り当てて、同じ名前で後で再利用できます。 作成時の実際のパスを追跡する必要はなく、名前だけを使用します。 Windows 上の Docker エンジンには、ローカル コンピューター上にボリュームを作成できる、組み込みの名前付きボリューム プラグインがあります。 複数のコンピューターで名前付きボリュームを使用する場合は、追加のプラグインが必要です。

手順の例:

1. `docker volume create unwound` - 'unwound' という名前のボリュームを作成します
2. `docker run -v unwound:c:\data microsoft/windowsservercore` - c:\data にマップされているボリュームでコンテナーを開始します
3. コンテナー内の c:\data にいくつかのファイルを書き込み、コンテナーを停止します。
4. `docker run -v unwound:c:\data microsoft/windowsservercore` - 新しいコンテナーを開始します
5. 新しいコンテナー内で `dir c:\data` を実行します。ここにはまだファイルが含まれています。

> [!NOTE]
> Windows Server によって、ターゲット パス名 (コンテナー内のパス) は小文字に変換されます。つまり、Linux コンテナー内で `-v unwound:c:\MyData` または `-v unwound:/app/MyData` を実行すると、コンテナー内のディレクトリは Linux コンテナーの `c:\mydata` または `/app/mydata` になり、マッピングされます (存在しない場合は作成されます)。
