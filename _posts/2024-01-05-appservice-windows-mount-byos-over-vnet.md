---
title: "App Service に VNet 経由でストレージアカウントをマウントする際の注意点"
author_name: "Takeharu Oshida"
tags:
    - App Service
---


# はじめに
お世話になっております。App Service サポート担当の押田です。

App Service では組み込みの [File Servers](https://jpazpaas.github.io/blog/2023/05/10/Inside-the-Azure-App-Service-Architecture.html#file-servers)のほかに、独自のストレージアカウントをマウントすることが可能となっています。

[App Service でローカル共有として Azure Storage をマウントする](https://learn.microsoft.com/ja-jp/azure/app-service/configure-connect-to-azure-storage?tabs=basic%2Cportal&pivots=code-windows)

この機能は Linux、Windows Container、Windows(コード)環境それぞれで制限や構成方法に違いがあります。

本記事では、Windows(コード)環境における、Azure Files のマウントについて記載します。

# 想定される構成

Azure Files をマウントするにあたって、Azure Files はできるだけセキュアに保つことは一般的な構成としてよく見受けられるものと存じます。

Azure Files の観点においては Private Endpoint を構成し、パブリックアクセスを無効にすることが可能となっています。

[Azure Files ネットワーク エンドポイントの構成](https://learn.microsoft.com/ja-jp/azure/storage/files/storage-files-networking-endpoints?tabs=azure-portal)

以下のようにApp Service がストレージとして Azure Files をマウントする際に、VNet を経由してPrivate Endpoint を利用した接続にてマウントする方法、トラブルシューティング方法を紹介します。

![image-75f4013a-a858-4cb2-834a-6b9eef663784.png]({{site.baseurl}}/media/2024/01/image-75f4013a-a858-4cb2-834a-6b9eef663784.png)

# Windows(コード)環境でマウントする際に Private Endpoint を利用する方法

手順としては基本的には下記ドキュメントに沿って実施ください。

[App Service でローカル共有として Azure Storage をマウントする](
https://learn.microsoft.com/ja-jp/azure/app-service/configure-connect-to-azure-storage?tabs=basic%2Cportal&pivots=code-windows)

## `WEBSITE_CONTENTOVERVNET` を必ず設定する

注意点として、Windows(コード)環境で VNet 経由すなわち Private Endpoint を経由したマウントを行うためには、App Settings として `WEBSITE_CONTENTOVERVNET` に `1` を設定する必要があります。
ドキュメントの最下部に記載されているのみとなり、見落としがちなポイントとなります。

![image-5e2208d7-b19e-4c43-8133-6d6587c2e182.png]({{site.baseurl}}/media/2024/01/image-5e2208d7-b19e-4c43-8133-6d6587c2e182.png)

上記設定がない場合、Private Endpoint を利用したマウントが行われないことになります。

# 検証
App Service からは /mounts/myfiles というパスでストレージアカウントを利用するように構成します。

![image-729f95e3-198d-45c7-a451-6464deb16b6d.png]({{site.baseurl}}/media/2024/01/image-729f95e3-198d-45c7-a451-6464deb16b6d.png)

![image-d3e28375-0616-47f8-88ae-7b99e538f508.png]({{site.baseurl}}/media/2024/01/image-d3e28375-0616-47f8-88ae-7b99e538f508.png)

## VNet 統合を用いない場合

まず、VNet 統合をしない状態、すなわち Private Endpoint を利用しない状態での App Service から Azure Files へのアクセスを見てみます。

VNET 統合未実施場合、下記の通り `WEBSITE_PRIVATE_IP` が未設定、 `nameresolver` の結果パブリックIPが取得できる状態です。
HTTPアクセスも疎通は可能です。（SAS未指定のためエラーとなっています。）

![image-139cb7ba-e4bd-43e9-8199-68e8fdedcf6b.png]({{site.baseurl}}/media/2024/01/image-139cb7ba-e4bd-43e9-8199-68e8fdedcf6b.png)

この時のストレージ側の診断ログを見てみます。
以下の通り、SMB アクセス、HTTPS アクセスいずれも `CallerIpAddress` は `10.11.0.122` となっています。
VNet 統合していないにもかかわらず、プライベート IP が用いられています。

![image-33c4751e-9798-4811-a973-03e8009f7751.png]({{site.baseurl}}/media/2024/01/image-33c4751e-9798-4811-a973-03e8009f7751.png)

このアドレスは、下記のドキュメントにおける「プライベートAzure IPアドレス」と考えられます。

[Azure Storage ファイアウォールおよび仮想ネットワークを構成する](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-network-security?tabs=azure-portal)

![image-6646e121-1cf8-496d-9e2c-71dd3d43b55f.png]({{site.baseurl}}/media/2024/01/image-6646e121-1cf8-496d-9e2c-71dd3d43b55f.png)

[前述のApp Service 側のドキュメント](
https://learn.microsoft.com/ja-jp/azure/app-service/configure-connect-to-azure-storage?tabs=basic%2Cportal&pivots=code-windows)には以下の記載があります。

> アプリで VNET 統合を使用すると、マウントされたドライブは、VNET からの IP アドレスではなく RFC1918 IP アドレスを使用します。

> アプリと Azure Storage アカウントが同じリージョンにある場合に App Service IP アドレスからのアクセスを Azure Storage ファイアウォール構成で許可しても、それらの IP 制限は適用されません。

参考

- [Azure VM からストレージ アカウントへアクセスする際の挙動とアクセス元制御](https://jpaztech.github.io/blog/storage/storageFirewall-accesscontroll/)

この状態でストレージアカウント側のファイアウォールにてパブリックアクセスを無効にした場合、マウントした領域は見えなくなってしまいます。

![image-b29aa5b1-2089-4b7e-bd84-179f830df0a9.png]({{site.baseurl}}/media/2024/01/image-b29aa5b1-2089-4b7e-bd84-179f830df0a9.png)


## VNet 統合を用いてPrivate Endpoint 経由でのアクセスを行う

App Service を Private Endpoint にアクセス可能な VNet に統合し、アプリケーション設定に `WEBSITE_CONTENTOVERVNET` = 1 を追加します。

以下のように `WEBSITE_PRIVATE_IP` としてサブネットから `10.0.1.254` が割り当てられており、ストレージアカウントの名前解決も `10.0.0.13` と PrivateEndpoint の IP を取得できる状態でマウントができています。

![image-4d5c6140-0927-4989-a7b6-9e1320257964.png]({{site.baseurl}}/media/2024/01/image-4d5c6140-0927-4989-a7b6-9e1320257964.png)

ストレージの診断ログ側では以下ように、`CallerIpAddress` は `10.0.1.254` とインスタンスに割り当てられた Private IP からのアクセスになっていることが確認できます。

![image-7731aba5-a29d-4aa6-b83c-142f50ab8cbf.png]({{site.baseurl}}/media/2024/01/image-7731aba5-a29d-4aa6-b83c-142f50ab8cbf.png)

また、ここでの注目点として、`Uri` が、`\\toshidatempstorage.file.core.windows.net-10.0.1.254\myfiles\` と<ストレージアカウントのFQDN>-<AppServiceインスタンスのプライベートIP>と変わっていることが確認できます。

これはどういうことかというと、アプリケーション上でのマウントは直接行われているわけではなく、App Service のプラットフォームにてマッピングが行われていることを示しております。
したがって、複数インスタンスが存在する場合などは、それぞれのIPで各インスタンス上でマッピングされることになります。

`C:\local > storageAccountMappings.txt` からこの情報を確認することができます。

![image-ca2921ad-574f-4b87-b6c8-63a680e1b86c.png]({{site.baseurl}}/media/2024/01/image-ca2921ad-574f-4b87-b6c8-63a680e1b86c.png)

以下のようなファイルアクセスも可能となります。
ただし、最下部で実施のように、PrivateIPを含まない場合でのアクセスはできません。

![image-a39d9c5c-0bef-49f2-883c-f298cd240b05.png]({{site.baseurl}}/media/2024/01/image-a39d9c5c-0bef-49f2-883c-f298cd240b05.png)

また、`10.0.1.254` と別の Private IP を持つインスタンスでは以下のようになっており、`\\toshidatempstorage.file.core.windows.net-10.0.1.254\myfiles\`　にアクセスすることはできません。

![image-dc84f0c7-2103-436d-8e6d-5ca0c4a698ea.png]({{site.baseurl}}/media/2024/01/image-dc84f0c7-2103-436d-8e6d-5ca0c4a698ea.png)

診断ログも以下のように、`10.0.1.253` `10.0.1.254` それぞれのアクセスがあることがわかります。

![image-bfa8415f-4672-4d06-b01f-f8c8bdb511ae.png]({{site.baseurl}}/media/2024/01/image-bfa8415f-4672-4d06-b01f-f8c8bdb511ae.png)

# トラブルシューティングのポイント

## Private Endpoint を利用できる状態にあるか

まずは VNet 統合が正しくできていること、Private Endpoint にアクセス可能かどうかの見極めとして、
下記資料を参考に `nameresolver` や `curl` を用いてご確認くださいませ。

[仮想ネットワークとAzure App Serviceの統合のトラブルシューティング](https://learn.microsoft.com/ja-jp/troubleshoot/azure/app-service/troubleshoot-vnet-integration-apps)

## `WEBSITE_CONTENTOVERVNET` = 1 が設定されているか

Windows(コード)環境において、VNet 経由でのマウントを行う場合必須の設定となります。
ドキュメント上は見落としやすいためご注意ください。

## UNCパスでファイルを参照していないか

Private Endpoint を利用したマウントを行った場合、UNCパスはインスタンス毎に異なります。
アプリケーションからはUNCパスではなく、マウント先と指定したパス `C:\mounts\<path>` としてアクセスしましょう。

## ストレージアカウントの診断ログを有効にしているか

SMB クライアントとなる App Service がどのようなアクセスをおこなっているかはストレージアカウント側の診断ログから確認可能です。

ストレージアカウントの診断ログについては、下記ドキュメントをご参考にしてください。

https://learn.microsoft.com/ja-jp/azure/storage/files/storage-files-monitoring

