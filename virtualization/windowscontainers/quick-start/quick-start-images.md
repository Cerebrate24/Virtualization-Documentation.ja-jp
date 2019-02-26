---
title: コンテナー展開のクイック スタート - イメージ
description: コンテナー展開のクイック スタート
keywords: Docker, コンテナー
author: cwilhit
ms.date: 09/26/2016
ms.topic: article
ms.prod: windows-containers
ms.service: windows-containers
ms.assetid: 479e05b1-2642-47c7-9db4-d2a23592d29f
ms.openlocfilehash: a28b9ced2c9669b65bfa02bd02c1061226dcea7e
ms.sourcegitcommit: f172d14ef1ebfb5a9df1b3529e0722d9103e0eba
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 02/26/2019
ms.locfileid: "9112919"
---
# <a name="automating-builds-and-saving-images"></a>ビルドの自動化とイメージの保存

前の Windows Server クイック スタートでは、以前に作成した .NET Core サンプルから Windows コンテナーを作成しました。 ここでは、Dockerfile から独自のコンテナー イメージを作成し、Docker ハブ パブリック レジストリにコンテナーの画像を保存する方法を示します。

このクイック スタートでは、Windows Server 2019 または Windows Server 2016 で Windows Server コンテナーをし、Windows Server Core コンテナーの基本イメージを使用します。 このページの左側の目次に追加のクイック スタート文書があります。

## <a name="prerequisites"></a>前提条件

次の要件を満たしていることを確認してください。

- 1 台のコンピューター システムが実行されている Windows Server 2019 (物理または仮想) または Windows Server 2016 します。
- Windows コンテナー機能と Docker でこのシステムを構成します。 これらの手順のチュートリアルについては、「[Windows Containers on Windows Server](./quick-start-windows-server.md)」 (Windows Server の Windows コンテナー) を参照してください。
- Docker ID。コンテナー イメージを Docker Hub にプッシュするために使用されます。 Docker ID がない場合は、[Docker Cloud](https://cloud.docker.com/) でサインアップしてください。

## <a name="container-image---dockerfile"></a>コンテナーの画像 - Dockerfile

コンテナーは手動で作成、変更して、新しいコンテナー イメージにキャプチャすることもできますが、Docker には、Dockerfile を使用してこのプロセスを自動化するためのメソッドが含まれています。 この演習には Docker ID が必要です。 Docker ID がない場合は、[Docker Cloud]( https://cloud.docker.com/) でサインアップしてください。

コンテナー ホストで、ディレクトリ `c:\build` を作成し、このディレクトリ内に `Dockerfile` という名前のファイルを作成します。 注記 – このファイルにはファイル拡張子を与えません。

```console
powershell new-item c:\build\Dockerfile -Force
```

Dockerfile をメモ帳で開きます。

```console
notepad c:\build\Dockerfile
```

Dockerfile に次のテキストをコピーし、ファイルを保存します。 これらのコマンドは、`microsoft/iis` を基礎として使用し、新しいイメージを作成するように Docker に指示します。 dockerfile は次に、`RUN` の指示に指定されているコマンドを実行します。この場合、index.html ファイルが新しいコンテンツで更新されます。

Dockerfile の詳細については、「[Dockerfiles on Windows](../manage-docker/manage-windows-dockerfile.md)」 (Windows 上の Dockerfile) を参照してください。

```dockerfile
FROM microsoft/iis
RUN echo "Hello World - Dockerfile" > c:\inetpub\wwwroot\index.html
```

`docker build` コマンドはイメージ ビルド プロセスを開始します。 `-t` パラメーターは、新しいイメージに `iis-dockerfile` という名前を付けるようにビルド プロセスに指示します。 **'user' は Docker アカウントのユーザー名で置き換えます**。 Docker にアカウントがない場合は、[Docker Cloud](https://cloud.docker.com/) でサインアップしてください。

```console
docker build -t <user>/iis-dockerfile c:\Build
```

完了したら、`docker images` コマンドを使用して、イメージが作成されたことを確認できます。

```console
docker images

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
iis-dockerfile      latest              8d1ab4e7e48e        2 seconds ago       9.483 GB
microsoft/iis       windowsservercore   c26f4ceb81db        2 weeks ago         9.48 GB
windowsservercore   10.0.14300.1000     dbfee88ee9fd        8 weeks ago         9.344 GB
windowsservercore   latest              dbfee88ee9fd        8 weeks ago         9.344 GB
```

次のコマンドでコンテナーを展開します (ここでも user を Docker ID で置き換えます)。

```console
docker run -d -p 80:80 <user>/iis-dockerfile ping -t localhost
```

コンテナーが作成されたら、コンテナー ホストの IP アドレスを参照します。 hello world アプリケーションが表示されるはずです。

![](media/dockerfile2.png)

コンテナー ホストに戻り、`docker ps` を使用してコンテナーの名前を取得し、`docker rm` を使用してコンテナーを削除します。 注記 – この例のコンテナー名を実際のコンテナー名に置き換えます。

コンテナー名を取得します。

```console
docker ps

CONTAINER ID   IMAGE            COMMAND               CREATED              STATUS              PORTS                NAMES
c1dc6c1387b9   iis-dockerfile   "ping -t localhost"   About a minute ago   Up About a minute   0.0.0.0:80->80/tcp   cranky_brown
```

コンテナーを停止します。

```console
docker stop <container name>
```

コンテナーを削除します。

```console
docker rm -f <container name>
```

## <a name="docker-push"></a>Docker プッシュ

Docker コンテナー イメージはコンテナー レジストリに保存できます。 イメージをレジストリに保存すると、後で多数のコンテナー ホストから取得して使用できるようになります。 Docker には、[Docker Hub](https://hub.docker.com/) にコンテナー イメージを保存できるパブリック レジストリがあります。

この演習では、カスタムの hello world イメージを Docker Hub の自分のアカウントにプッシュします。

まず、`docker login command` を使用して Docker アカウントにログインします。

```console
docker login

Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.

Username: user
Password: Password

Login Succeeded
```

ログインしたら、コンテナー イメージを Docker Hub にプッシュできます。 この場合は、`docker push` コマンドを使用します。 **'user' は Docker ID で置き換えます**。 

```console
docker push <user>/iis-dockerfile
```

Docker Docker ハブまでには、各レイヤーをプッシュと docker は Docker ハブで、またはその他のレジストリ (外部レイヤー) に既に存在しているレイヤーをスキップします。  たとえば、Microsoft コンテナー レジストリ、または、プライベート企業レジストリからレイヤーでホストされている最新のバージョンの Windows Server Core をスキップするといない Docker ハブにプッシュするとします。

これで、`docker pull` を使用して、コンテナー イメージを Docker Hub から任意の Windows コンテナー ホストにダウンロードできるようになります。 このチュートリアルでは、既存のイメージを削除し、Docker Hub からプルします。 

```console
docker rmi <user>/iis-dockerfile
```

`docker images` を実行すると、イメージが削除されたことが表示されます。

```console
docker images

REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
modified-iis              latest              51f1fe8470b3        5 minutes ago       7.69 GB
microsoft/iis             latest              e4525dda8206        3 hours ago         7.61 GB
```

最後に、Docker Pull を使用して、イメージをコンテナー ホストにプルすることができます。 'user' は Docker アカウントのユーザー名で置き換えます。 

```
docker pull <user>/iis-dockerfile
```

## <a name="next-steps"></a>次の手順

サンプル ASP.NET アプリケーションをパッケージ化する方法については、下のリンクで Windows 10 のチュートリアルをご覧ください。

> [!div class="nextstepaction"]
> [Windows 10 のコンテナー](./quick-start-windows-10.md)
