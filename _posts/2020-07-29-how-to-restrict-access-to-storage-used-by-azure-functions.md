---
title: "Azure Functions が使用するストレージアカウントに対するアクセス制限の方法"
author_name: "Shogo Ohe"
tags:
    - Function App
---
***2022 年 5 月 20 日更新***：プライベートエンドポイントを使用したストレージアカウントがサポートされるようになりました。最新の情報は [こちらの記事]({{site.baseurl}}/2022/05/20/protect-functions-storage.html) を参照してください。

「Azure Functions が使用するストレージアカウントの接続制限を行いたい」というご質問を多くのお客様より頂戴しております。
Azure Functions は従量課金プランまたは Premium (Elastic Premium) プランでご利用頂く場合、アプリケーションコードはストレージアカウントに保存頂く必要がございます (App Service Plan 上でご利用頂く場合は不要)。

今回は、こちらについて、ご紹介します。

# 質問
Azure Functions を従量課金プランまたは Premium (Elastic Premium) プランで利用しており、ストレージアカウントを構成する際、下記の構成を取りたい。
- ストレージアカウントのファイアウォールを有効にして、接続元 IP を制限したい
- Functions が使用するストレージアカウントの接続元を Functions に限定したい

# 回答
ストレージアカウントのファイヤーウォールを有効にすることで、ストレージアカウントへの接続を特定の IP または仮想ネットワークからの接続に限定することが出来ます。
しかし、Functions と組み合わせて使用する際に以下の制約があるため、組み合わせには注意が必要です。

- Functions からストレージアカウントへ接続する際、同一リージョン内での通信はリージョン内で使用される、プライベート IP での通信となります。
- Azure Storage のファイアウォールでは、仮想ネットワークへの通信許可 IP としてパブリック IP は追加いただけますが、RFC 1918 で定義されるプライベート IP (10.* 、172.16.* - 172.31.* 、192.168.*) からの通信は許可リストに登録頂くことが出来ません。<br />
参考: [インターネットの IP 範囲からのアクセスを許可する](https://docs.microsoft.com/ja-jp/azure/storage/common/storage-network-security#grant-access-from-an-internet-ip-range) <br/>

参考：
[Azure Web Apps / Azure Functions から同じリージョンの Azure Storage Account へ接続する時の注意点](https://docs.microsoft.com/ja-jp/archive/blogs/jpcie/azure-web-apps-azure-functions-から同じリージョンの-azure-storage-account-へ接続する時#別リージョンであれば問題なく設定できる) <br />



# 回避策
2020 年 7 月時点でご案内可能な回避策は、以下の 2 点となります。

 1. ストレージアカウントにデータを保持しない、App Service プランを使う
 2. Functions とストレージアカウントの配置されるリージョンを別リージョンにする

回避策 1. に関して、従量課金プランおよび Premium プランでは、Azure で準備しているインスタンス (仮想マシン) が実行時またはスケールアウト時に動的に割り当てられる関係上、全てのデータはストレージアカウントに保存いただいておりました。
しかし、App Serivice プランでは、お客様のアプリケーションコードはお客様専用に確保されたリソースの上で実行されますため、通常の関数アプリ実行に関しては、ストレージアカウントには依存しなくなります (Durable Functions や拡張機能によってはストレージアカウントを使用しますため、こちらでの回避は出来ません)。

回避策 2. に関して、別リージョンのストレージアカウントへのアクセスはパブリック IP によるアクセスとなります。
ストレージアカウントを Functions とは別リージョンにご準備頂き、Functions が使用するストレージアカウントを変更頂けますと、ストレージアカウント側からのアクセス IP 制限がご利用いただけます。

仮想ネットワーク統合に関しては、リソースが別リージョンに作成される関係上、ご利用頂くことが出来ません。


Functions が使用するストレージアカウントに関する要件につきましては以下のドキュメントをご確認ください。<br />
[Azure Functions のストレージに関する考慮事項](https://docs.microsoft.com/ja-jp/azure/azure-functions/storage-considerations) <br />

# 具体的な構成方法
こちらで、異なるリージョンにストレージアカウントを配置された場合の構成方法をご案内いたします。
まず、使用するストレージアカウントを変更後に、ストレージアカウントのファイアウォール制限を行うという順番で設定します。

## Functions に異なるリージョンのストレージアカウントを設定する方法

Functions とストレージアカウントを異なるリージョンに作成される際、下記の操作をお客様自身にて行っていただく必要がございます。
  1.  ストレージアカウントをリージョン A に作成する。
  2.  Functions をリージョン B に作成する。
  3.  リージョン A に作成したストレージアカウントから、接続文字列を取得します。<br />
    [接続文字列を表示およびコピーする](https://docs.microsoft.com/ja-jp/azure/storage/common/storage-configure-connection-string#view-and-copy-a-connection-string) <br />
  4.  リージョン B に作成した Functions の [構成] より、アプリケーション設定のストレージアカウントを指定している箇所を変更します。 <br />
    具体的には "AzureWebJobsStorage" , "WEBSITE_CONTENTAZUREFILECONNECTIONSTRING" の値として指定されている接続文字列を (3) で取得した接続文字列と入れ替えます。 <br />
![functions-config1.jpg]({{site.baseurl}}/media/2020/07/2020-07-29-functions-config1.jpg) <br />
 <br />
![functions-config2.jpg]({{site.baseurl}}/media/2020/07/2020-07-29-functions-config2.jpg) <br />
    <br />
    参考ですが WEBSITE_CONTENTSHARE には Azure Files のファイル共有名が入り、こちらで指定したコンテナに Functions で使用するお客様のアプリケーションコードやログと行った情報が格納されます。 <br />
    ファイル共有名は既定で Functions の名前が使用されているため基本的には重複は発生しないためそのままで問題ございませんが、必要に応じて調整ください。
  5. リージョン B に作成した Functions にアプリケーションをデプロイします。
  6. (5) でデプロイした内容が、リージョン A に作成したストレージアカウントに作成されていることを確認します。<br/>
具体的には リージョン B のストレージアカウントの ファイル共有 ("WEBSITE_CONTENTSHARE" で指定した名前) を参照し、 "site/wwwroot/" 配下にデプロイした関数アプリがあるかご確認ください。


## Functions の送受信 IP アドレスの調べ方
ストレージアカウントへのアクセスに使用する (=ファイアウォールに登録が必要な) パブリック IP として、Functions の受信 IP と送信 IP があります。

受信 IP は Functions の [プロパティ] に記載がございます。 <br />
![functions-ip1..jpg]({{site.baseurl}}/media/2020/07/2020-07-29-functions-property.jpg)

送信 IP に関しまして、一番簡単な方法は Azure CLI により取得する方法です。
下記コマンドにより、取得いただくことができます。
カンマ区切りで複数の IP が列挙されますので、それぞれストレージアカウントの接続を許可する IP にご登録ください。

`az webapp show --resource-group <group_name> --name <app_name> --query possibleOutboundIpAddresses --output tsv`

実行例：
 ![functions-ip2.jpg]({{site.baseurl}}/media/2020/07/2020-07-29-functions-ip-cli.jpg)

上記で取得しました、パブリック IP アドレスを異なるリージョンに作成されたストレージアカウントのファイアウォールに許可する IP としてご登録いただくいただくことで、Functions からの接続を許可することができます。
 

送受信に使用します IP アドレスの取得方法につきましては、ドキュメントの下記に記載されておりますので、ご参考いただけると幸いです。

  [関数アプリの着信 IP アドレス](https://docs.microsoft.com/ja-jp/azure/azure-functions/ip-addresses#function-app-inbound-ip-address) <br />
  [関数アプリの送信 IP アドレス](https://docs.microsoft.com/ja-jp/azure/azure-functions/ip-addresses#function-app-outbound-ip-addresses) <br />


送信 IP アドレスに関しましては、将来的にリージョン内で提供する Functions 実行基盤の規模が拡大した際に追加される (お客様のアプリケーションも新しい IP から送信される) こともございます。
必要に応じて、各リージョンで使用している IP アドレス範囲での登録につきましてもご検討ください。<br />

[データ センターの送信 IP アドレス](https://docs.microsoft.com/ja-jp/azure/azure-functions/ip-addresses#data-center-outbound-ip-addresses)<br />




# 注意事項

## 異なるリージョンに配置する事について
上記の通り、手動で構成することで異なるリージョンのストレージアカウントをご利用いただくことができますが、Azure Functions は既定で同一リージョンのストレージアカウントを利用することが前提として構築されております。<br/>
ストレージアカウントを異なるリージョンに配置することは、推奨できる構成ではなく、リージョン間通信の遅延などに起因するパフォーマンスの影響が出る可能性がございますため、その点をご理解いただいた上でご利用いただけますと幸いです。

## IP 制限について
「業務上の機密を含むデータを保持するために、他のユーザーからアクセスを禁止を目的として、ストレージアカウントの接続元を制限したい」という背景の元、お問い合わせ頂いている場合がございます。
今回、ストレージアカウントのアクセスを許可として指定した IP アドレスは、各リージョンに配置された App Service / Functions の実行基盤 (スケールユニット) 全体となります。
そのため、パブリックなインターネットからの接続は出来ないものの、該当リージョンの別の App Service / Functions インスタンスからのアクセスは可能な設定となっております。
そのため、お客様側におきましても、ユーザー名やパスワードの管理と同様、データの暗号化や接続文字列の適切な管理などを心がけて頂く必要がございます。

---

<br>
<br>

2020 年 7 月 29 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>