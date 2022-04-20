---
title: "Azureのサーバー証明書更新に関する通知が来たので、ユーザーへの影響を知りたい"
author_name: "Hirotaka Shionoiri"
tags:
    - etc
---

# 質問
ある Azure のサービスにおいて、Azure のサーバー証明書更新をする旨の通知が来ました。<br>
私に影響はあるのでしょうか？ユーザー側への影響を教えてください。

# 回答
殆どのお客様は影響を受けないものと思われます。<br>
ただし、クライアント アプリケーションのユーザー実装部分において、通信に許可される証明書を明示的に指定している場合<br>
（いわゆる「証明書のピン留め」を利用している場合）<br>
証明書が変更となるのでアプリケーションの通信に影響が出る場合がございます。<br>
<br>
対象の Azure サービスと通信をするアプリケーションの実装部分をご確認頂き、<br>
証明書のピン留めにあたるような証明書の指定をしているかどうかご確認下さい。<br>
証明書のピン留めを実施されている場合には許可する証明書を更新する対応が必要となります。

# 解説
2022 年 4 月 1 日時点では、直近 Storage Account や Azure Cache for Redis などのサービスについて<br>
サーバー証明書の更新に関する通知が送られました。<br>
下記の記事は英文とはなりますが、本記事同様 Azure の証明書更新の内容を解説した記事となります。<br>
具体的な証明書の情報も記載されておりますので、証明書の Thumbprint などの情報が必要な場合には下記記事をご参照下さい。<br>
<br>
**Azure Storage TLS: Critical changes are almost here! (…and why you should care)**<br>
[https://techcommunity.microsoft.com/t5/azure-storage-blog/azure-storage-tls-critical-changes-are-almost-here-and-why-you/ba-p/2741581](https://techcommunity.microsoft.com/t5/azure-storage-blog/azure-storage-tls-critical-changes-are-almost-here-and-why-you/ba-p/2741581)

### そもそも証明書とは？
今回更新されるサーバー証明書とは、HTTPS 通信を実施する際にサーバー側がクライアント側に提示するものでございます。<br>
今回の例では、サーバー側が Azure Storage や Redis となり、クライアント側がそれらと通信するアプリケーションです。<br>
HTTPS を利用してブラウザから Web ページなどにアクセスした際にも、実は証明書のやり取りが実施されています。<br>
サーバー側から提示されている証明書は、各種ブラウザからもご確認頂くことが可能です。<br>
下記の例は本ブログの証明書を Web ブラウザである Edge (Chromium 版) より確認した際の例となります。<br>
<br>
![image.png]({{site.baseurl}}/media/2022/04/2022-04-20-cert1.png)<br>
<br>
これらの証明書は「認証局」と呼ばれる機関から発行されており、<br>
有効期限付きでサーバーの身分を証明する証明書となります。<br>
また、クライアント側の OS やブラウザなどには「信頼されたルート証明機関」等のリストが存在しており、<br>
証明書を発行した認証局が信頼できる機関かどうかは、このリストに含まれているかどうかで確認されます。<br>
大雑把な概要とはなりますが、これが今回の話題のサーバー証明書というものです。

### 証明書の更新と影響について
先ほど少し触れたように、証明書には有効期限が存在します。<br>
よって、定期的に更新する必要があることはご理解頂けるかと思います。<br>
より詳細な状況はその時々にお受け取りになられた証明書更新の通知をご確認頂きたいのですが、<br>
基本的には有効期限などに伴い更新を実施します。<br>
<br>
![image.png]({{site.baseurl}}/media/2022/04/2022-04-20-cert2.png)<br>
<br>
また、先ほど証明書の解説において、<br>
「証明書を発行している機関が信頼できるかどうかは、クライアント側の持つリストによって判断される」<br>
という旨の説明をしました。<br>
よって、証明書が更新されたとしても、信頼された機関から発行された証明書であれば、<br>
たとえ発行機関が前の証明書とは異なっていても基本的には通信に影響を与えません。<br>
影響を与える例外的なパターンとしては、実際に通信をするクライアント アプリケーション内で<br>
信頼する証明書を指定（証明書のピン留め）している場合です。

### 証明書の指定（ピン留め）について
影響を受ける例外的なパターンとは、上述の様に、特定の証明書のみを信頼するような処理を入れている場合であり、<br>
具体的には<br>
「サーバーから提示された証明書のデータを確認し、特定の証明書でなければ通信を拒否する」<br>
といった条件文を入れている場合です。このような処理が証明書のピン留めや証明書の指定と呼ばれています。<br>
例えば<br>
「xx から発行された証明書でなければ、通信を中止する」<br>
「証明書の thumbprint（証明書が持つ固有の値）が xxxxx でなければ、通信を中止する」<br>
といった条件が入っている場合、通信に影響が発生する可能性があります。<br>
これは証明書の更新によって発行元が変更されたり、証明書の thumbprint が変更されるためです。<br>
<br>
つきましては、ユーザーが実装したクライアント側のアプリケーション内に<br>
上述のような証明書のピン留めに該当する処理が含まれていないかをご確認頂き、<br>
影響の有無をご判断頂く必要がございます。<br>
<br>
Azure のサービス内においては基本的に影響が無い様調整済みであるため、ご確認頂く必要はございません。<br>
ユーザーのクライアント アプリケーションや、 Azure サービス内で稼働するアプリケーションのユーザー実装部分に<br>
証明書のピン留めに該当する処理がないかご確認ください。<br>
もし、ピン留めを実施されている場合には新しく適用される予定の証明書の情報に変更してください。<br>
これは 2021 年 - 2022 年にかけて実施された証明書更新の一例となりますが、<br>
下記記事の "Certificate Renewal Summary"に記載された内容が証明書の具体的な情報となります。<br>
<br>
**Azure Storage TLS: Critical changes are almost here! (…and why you should care)**<br>
[https://techcommunity.microsoft.com/t5/azure-storage-blog/azure-storage-tls-critical-changes-are-almost-here-and-why-you/ba-p/2741581](https://techcommunity.microsoft.com/t5/azure-storage-blog/azure-storage-tls-critical-changes-are-almost-here-and-why-you/ba-p/2741581)


# 参考ドキュメント
- **マイクロソフトが提供する信頼できる証明書利用基盤**<br>
[https://msrc-blog.microsoft.com/2013/01/16/65374/](https://msrc-blog.microsoft.com/2013/01/16/65374/)<br>
<br>
少々古い記事とはなりますが、弊社セキュリティチームより公開されている証明書に関する記事となります。<br>
証明書の役割や Windows 上の証明書管理に関してより詳しいご説明が確認頂けます。

- **Microsoft 信頼されたルート証明書プログラム**<br>
[https://docs.microsoft.com/ja-jp/security/trusted-root/release-notes](https://docs.microsoft.com/ja-jp/security/trusted-root/release-notes)<br>
<br>
Windows では、信頼されている証明書のリストを毎月（12 月を除く）更新されています。<br>
本記事で紹介した「クライアント側の持つリスト」の具体的なイメージとして参考資料になれば幸いです。

- **Azure Storage TLS: Critical changes are almost here! (…and why you should care)**<br>
[https://techcommunity.microsoft.com/t5/azure-storage-blog/azure-storage-tls-critical-changes-are-almost-here-and-why-you/ba-p/2741581](https://techcommunity.microsoft.com/t5/azure-storage-blog/azure-storage-tls-critical-changes-are-almost-here-and-why-you/ba-p/2741581)<br>
<br>
繰り返しのご紹介となり恐縮ですが、2021 年 - 2022 年にかけて実施された Azure の証明書更新の内容を解説した記事となります。<br>
英文ではありますが、具体的な証明書の情報も記載されているので、必要に応じてご参照下さい。

<br>
<br>

---

<br>
<br>

2022 年 4 月 20 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>