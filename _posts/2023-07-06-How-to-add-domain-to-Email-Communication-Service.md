---
title: "メール通信サービスにドメインを追加する際の DNS 設定について"
author_name: "Kohei Hosono"
tags:
    - Azure Communication Services
---

# 質問

メール通信サービスにカスタムドメインを追加したいのですが、検証が完了しません。

# 回答

検証に必要ないくつかの DNS レコードがパブリック DNS サーバーに設定されている必要があります。まずはご利用のドメインにて下記 DNS レコードが正しく設定されているかご確認ください。

カスタムドメインを Azure DNS でホストされている場合には [DNS レコードの設定方法](https://learn.microsoft.com/ja-jp/azure/dns/tutorial-alias-rr) を、 Azure 以外の DNS プロバイダーをご利用の場合には対象のドメインプロバイダーにご確認ください。

| Record | Type | Name | Value |
|--|--|--|--|
| Domain Verify | TXT | \<custom-domain\> | ms-domain-verification=\<token\> |
| SPF | TXT | \<custom-domain\> | v=spf1 include:spf.protection.outlook.com -all |
| DKIM | CNAME | selector1-azurecomm-prod-net._domainkey.\<custom-domain\> | selector1-azurecomm-prod-net._domainkey.azurecomm.net |
| DKIM2 | CNAME | selector2-azurecomm-prod-net._domainkey.\<custom-domain\> | selector2-azurecomm-prod-net._domainkey.azurecomm.net |

## メール通信サービスにカスタムドメインを追加する手順

1. Azure Portal でメール通信サービスを開き [ドメインをプロビジョニングする] ブレードに移動し、 [ドメインの追加] > [カスタムドメイン] から使用したいカスタムドメインを追加すると、 "Domain status", "SPF status", "DKIM status", "DKIM2 status" の欄にそれぞれ "構成" と表示されます。なお本稿では、例としてカスタムドメイン (`mail.koheihosono.com`) を追加しています。

    ![image-969564bc-ef7a-46a7-984f-9f65d64c7c49.png]({{site.baseurl}}/media/2023/07/image-969564bc-ef7a-46a7-984f-9f65d64c7c49.png)

2. "Domain status" の [構成] を選択し表示される情報について、当該 DNS レコードを管理するプロバイダー (Azure DNS ゾーン等) に追加します。

    ![image-553a382f-f6ba-4709-aace-c6aba8dfb534.png]({{site.baseurl}}/media/2023/07/image-553a382f-f6ba-4709-aace-c6aba8dfb534.png)

    [digwebinterface](https://digwebinterface.com) などで TXT レコードが追加されていることを確認します。

    ![image-8c160ab2-3c87-4d5d-9cc9-a4d160b71e3c.png]({{site.baseurl}}/media/2023/07/image-8c160ab2-3c87-4d5d-9cc9-a4d160b71e3c.png)

    Azure Portal に戻り、 [次へ] > [完了] > [閉じる] を選択して [更新] を何度か行い、 Domain status が "検証済み" となれば完了です。 TXT レコードは削除して問題ありません。

    ![image-42a47747-b979-43b8-b0ff-44dfff3f7562.png]({{site.baseurl}}/media/2023/07/image-42a47747-b979-43b8-b0ff-44dfff3f7562.png)

3. "SPF status" の [構成] を選択すると "SPF", "DKIM", "DKIM2" が表示されますので、 SPF 用の TXT レコードと DKIM/DKIM2 用の CNAME レコードを追加します。

    **<SPF (TXT レコード)>**

    ![image-df63708a-84dc-4659-8802-56a2b6d42aec.png]({{site.baseurl}}/media/2023/07/image-df63708a-84dc-4659-8802-56a2b6d42aec.png)

    ![image-6b60fff3-f69b-4b19-920b-3390d3cf14c5.png]({{site.baseurl}}/media/2023/07/image-6b60fff3-f69b-4b19-920b-3390d3cf14c5.png)

    **<DKIM (CNAME レコード)>**

    ![image-2cf15561-08be-4efa-9273-57b8f5c3f7fc.png]({{site.baseurl}}/media/2023/07/image-2cf15561-08be-4efa-9273-57b8f5c3f7fc.png)

    ![image-0e3ea3ae-0204-4d50-858b-c6d821e35a37.png]({{site.baseurl}}/media/2023/07/image-0e3ea3ae-0204-4d50-858b-c6d821e35a37.png)

    **<DKIM2 (CNAME レコード)>**

    ![image-1dfdd496-a6d4-4952-8d3b-111ba19fdc2e.png]({{site.baseurl}}/media/2023/07/image-1dfdd496-a6d4-4952-8d3b-111ba19fdc2e.png)

    ![image-8f4e3514-6ac3-425a-af9d-eccc401b43de.png]({{site.baseurl}}/media/2023/07/image-8f4e3514-6ac3-425a-af9d-eccc401b43de.png)

    上記 3 つの DNS レコードを追加できましたら、 Azure Portal で [Next] > [Done] > [Close] と進み、 [更新] を何度か行い、すべての status が "検証済み" となれば完了です。

    ![image-f6b51368-8c86-4cce-a2e9-0a42ff84088c.png]({{site.baseurl}}/media/2023/07/image-f6b51368-8c86-4cce-a2e9-0a42ff84088c.png)

メール通信サービスでカスタムドメインの検証が完了しない場合、 DNS レコードが正しく構成されていない可能性があります。

特に DKIM/DKIM2 に関しましては `selector1-azurecomm-prod-net._domainkey.<custom-domain>` に CNAME レコードを追加する必要がございます。まずは [digwebinterface](https://digwebinterface.com) などで DNS レコードをご確認いただけますと幸いです。

## SPF (TXT) レコードについて

SPF レコードに関しましては、以前 [TXT レコードに複数の SPF 値が含まれていると検証が成功しない](https://learn.microsoft.com/en-us/answers/questions/929882/spf-record-for-domain-wont-verify-for-custom-email) 事象が報告されておりましたが、 2024 年 1 月 29 日時点では既に改善されており、 TXT レコードに複数の SPF 値が含まれていても検証は成功することが期待されます。

一方で、 SPF レコード (例 : `v=spf1 include:spf.protection.outlook.com -all`) の末尾に記載する `all` につきましては、 `~all (チルダ)` ではなく `-all (ハイフン)` としてレコードを設定いただく必要がございますのでご留意ください。

SPF (送信者ポリシーフレームワーク) において、 `~all` は [Soft fail](http://www.open-spf.org/SPF_Record_Syntax/) と呼ばれ、送信サーバーがドメインを使用して送信することを許可されているかを明確に示すものではなく、 SPF の検証に失敗したメッセージを完全に拒否しません。そのため悪意のある何者かによって悪用された場合、お客様のドメインから送信されたように見えるメールが送信されるといった可能性も考えられ、 Azure メール通信サービスのカスタムドメインでは `~all` でドメイン検証に通過できない動作となっております。

SPF レコードを DNS サーバーに追加したにもかかわらず検証が成功しないといった場合、 SPF レコードが Hard fail (`-all`) として構成されているかご確認ください。

なお、対象のカスタムドメインを他のメールサービスなどで使用しており、既に SPF レコードの末尾を `~all (チルダ)` として DNS サーバーに登録している場合など、 SPF レコードを `-all (ハイフン)` として登録することが難しいご事情がある可能性も考えられます。
その場合には、それら 「SPF レコードを Hard fail (`-all`) にて登録することが難しいご事情」 を添えて、実際にカスタムドメインを登録されたいメール通信サービスリソースが属するサブスクリプションより [Azure サポートまでご相談いただければと存じます](https://learn.microsoft.com/ja-jp/azure/azure-portal/supportability/how-to-create-azure-support-request) (数日程度のお時間を要することが見込まれますので、予めご了承ください)。

# 参考ドキュメント

<https://learn.microsoft.com/azure/communication-services/quickstarts/email/add-custom-verified-domains>

<br>
<br>

---

<br>
<br>

2024 年 01 月 29 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>