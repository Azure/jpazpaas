---
title: "Azure Portal から Function App 作成時にネットワーク制限を設定した既存のストレージアカウントを選択すると作成に失敗する"
author_name: "Kohei Hosono"
tags:
    - Function App
---

# 質問

Azure Portal から既存のストレージアカウントを選択して Function App を作成すると以下のエラーメッセージが表示され作成に失敗する。

```txt
Creation of storage file share failed with: 'The remote server returned an error: (403) Forbidden.'. Please check if the storage account is accessible.
```

# 回答

選択されているストレージアカウントでファイアウォールが有効になっている可能性が考えられます。 Azure Portal において、 ファイアウォールを有効にした既存のストレージアカウントを使用して Function App を作成するシナリオは [サポートされておりません](https://learn.microsoft.com/ja-jp/azure/azure-functions/storage-considerations?tabs=azure-cli#storage-account-requirements) 。

回避策としては、以下のような方法をご検討いただけますと幸いです。

- Function App の作成時、一時的にストレージアカウントのファイアウォール (パブリックネットワークアクセス) を `すべてのネットワークから有効` に変更する
- Function App の作成時には一旦、新規にストレージアカウントを作成し、リソースの作成後に [ストレージアカウントを入れ替える](https://learn.microsoft.com/ja-jp/azure/azure-functions/configure-networking-how-to?tabs=portal#existing-function-app)

# エラーが発生する原因

[従量課金プランならびに Elastic Premium プランの Function App は、既定で Azure Files を使用し](https://learn.microsoft.com/ja-jp/azure/azure-functions/storage-considerations?tabs=azure-cli#storage-account-requirements) 、ファイル共有をマウントしてお客様にてデプロイされたアプリケーションコードなどを格納します。当該エラーは、ストレージアカウントでファイル共有の作成に失敗したことに起因しております。

> あくまでファイル共有の作成に失敗していることが原因となりますので、 Dedicated プラン (App Service プラン) の場合には Azure Files を必要としないため、 Azure Portal から Function App の作成時にファイアウォールを有効にしたストレージアカウントを選択してもリソースの作成に成功する場合がございます。

## ストレージアカウントのファイアウォールで特定の IP アドレスを許可すれば作成可能ですか？

いいえ、 Function App 作成時に選択するストレージアカウントのファイアウォールにおいて、 「Function App の VNet 統合を有効にしてサービスエンドポイントを有効にする」 ことや 「Azure Portal を操作するクライアント PC のグローバル IP アドレスをファイアウォールで許可する」 ことで事象を回避することは叶いません。

これは、ファイル共有を Function App 自身や Portal を操作するクライアント PC ではなく [Functions (App Service) プラットフォーム内部のコンポーネント](https://jpazpaas.github.io/blog/2023/05/10/Inside-the-Azure-App-Service-Architecture.html#api-controllers) が作成しているためです。

上記コンポーネントが使用する IP アドレスの範囲は公開されておらず、また仮想ネットワークを経由してストレージアカウントに接続することも叶いませんので、ストレージアカウントでファイアウォールを有効にしていると Function App リソースの作成に失敗いたします。

なお、 [ストレージアカウントで診断設定](https://learn.microsoft.com/ja-jp/azure/storage/files/storage-files-monitoring#collection-and-routing) を有効にしていると Function App 作成失敗時刻周辺で 403 エラーが記録されていることが確認できます。

![image-7a2dd95c-3b14-449a-8ce8-dc096576891a.png]({{site.baseurl}}/media/2024/01/image-7a2dd95c-3b14-449a-8ce8-dc096576891a.png)

403 エラーログの CallerIpAddress がプライベート IP アドレスになっている場合があり、これは [同一リージョン内でのストレージアカウントへの通信にはプライベート IP アドレスが使用される](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-network-security?toc=%2Fazure%2Fstorage%2Ffiles%2Ftoc.json&tabs=azure-portal#restrictions-for-ip-network-rules) 想定された動作となりますが、ストレージアカウントのファイアウォールではプライベート IP アドレスに基づいてルールを構成できないため、仮に Functions プラットフォームが使用する IP アドレスの範囲が判明してもファイアウォール規則で許可することは叶いません。

## 補足
本記事は Function App で [デプロイスロットアプリを作成するシナリオ](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-deployment-slots?tabs=azure-portal#create-a-deployment-setting) にも該当します。


お客様にて従量課金プラン並びに Elastic Premium プランの Function App でデプロイスロットアプリを追加しようとする際に、Function プラットフォームの内部のコンポーネントはストレージアカウントに対してプライベート IP アドレスで接続してファイル共有を作成する動作があります。


ストレージアカウントでファイアウォールを有効にする場合、Function プラットフォームの内部のコンポーネントからストレージアカウントへのアクセスが拒否されるため、ファイル共有の作成が失敗した結果、デプロイスロットアプリの作成が出来ません。

そのため、ストレージアカウントでファイアウォールを有効にする場合、Function App でデプロイスロットアプリを追加するには、一時的にストレージアカウントのファイアウォール (パブリックネットワークアクセス) を すべてのネットワークから有効 に変更する必要がございます。

ご不便をおかけしますが、記事冒頭でご紹介した回避策などをご検討いただけますと幸いです。

<br>
<br>

---

<br>
<br>

2024 年 05 月 01 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>