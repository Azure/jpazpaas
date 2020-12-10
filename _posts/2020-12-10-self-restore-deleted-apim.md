---
title: "削除された API Management をリストアする"
author_name: "Yusuke Yoneda"
tags:
    - API Management
---

本投稿では、[Self-restore Deleted APIM Service](https://techcommunity.microsoft.com/t5/azure-paas-blog/self-restore-deleted-apim-service/ba-p/1969746) を日本語翻訳し紹介します。

***
2020 年 11 月に、誤って削除された API Management をユーザーが REST API を使用して復元できる新機能がリリースされました。  
API Management が 2020-06-01-preview 以上のバージョンのREST API を経由して削除された場合のみ、ユーザーは、新しい API を使用して誤って削除されたサービスを復元することができます。  
Azure Portalで [すべてのリソース] から API Management を削除するか、API Management が存在するリソースグループを削除した場合、現在 Azure Resource Manager は API バージョン2020-06-01-preview を使用して API Management に DELETE リクエストを送信します。
この場合、サービスは "soft deleted" 状態になります。 
Azure ポータル全体で、API バージョン 2020-06-01-preview と一貫性がある "API Management の削除" の操作をできるよう計画しています。 

## "soft deleted"状態の API Management を Purge (完全削除) する
2020-06-01-previewより前のAPI バージョンで API Management を誤って削除した場合、削除された API Management のリソース名はすぐに解放され、同じ名前で新しいサービスを再作成することができます。もしこのようなケースでサービスを復元したい場合は、サポートにお問い合わせください。<br>
API Management が APIバージョン 2020-06-01-preview 以上で削除された場合、API Management  は "soft deleted" 状態のままになり、サービスを自身で復元することができます。しかしながら、削除直後に同じ名前の APIM サービスを作成することはできず、同じ名前で新しい APIM サービスを作成したい場合は、"soft deleted" 状態 のAPI Management  を Purge （完全削除）することができます。Purge されたAPI Management は元に戻すことができないことをご注意ください。
<br>
以下の例を参照してください。例として、APIM "jiaapim-testdelete"を使用します。

API を使用して APIM をPurge する | [Deleted Services - Purge](https://docs.microsoft.com/en-us/rest/api/apimanagement/2020-06-01-preview/deletedservices/purge)

![gujia93_0-1607484841237.png]({{site.baseurl}}/media/2020/12/2020-12-10-Self-restore-Deleted-APIM-gujia93_0-1607484841237.png)

レスポンスでは、削除されたサービス名と APIM サービスが削除された日時を返します。 

![gujia93_1-1607484884018.png]({{site.baseurl}}/media/2020/12/2020-12-10-Self-restore-Deleted-APIM-gujia93_1-1607484884018.png)

## APIMを復元するための前提条件
以下の例を参照してください。例では、APIM "jjiaapim-external"を使用します。<br>

削除された API Management  サービスは、次の条件にすべて合致する場合にのみ、自身で復元できます。
1. API Management は APIバージョン 2020-06-01-preview 以上で削除し、"soft deleted" 状態となっている必要があります。API Management  が削除されると、サービスは 2 日間「soft deleted」状態に保たされます。これは予告なしに変更される場合があります。
1. サブスクリプション上の削除されたサービスを一覧表示する REST API を使用して、削除された APIM サービスが "soft deleted" 状態であるかどうかを確認できます。この API は Purge (完全削除) になる日付も応答します。<br>
[Deleted Services - List By Subscription](https://docs.microsoft.com/en-us/rest/api/apimanagement/2020-06-01-preview/deletedservices/listbysubscription)
<br>
![gujia93_2-1607484967428.png]({{site.baseurl}}/media/2020/12/2020-12-10-Self-restore-Deleted-APIM-gujia93_2-1607484967428.png)
<br>
1. Check Name Availability REST API を使用して、soft deleted された API Management のホスト名が解放されているかどうかを確認できます。<br>
[Api Management Service - Check Name Availability](https://docs.microsoft.com/en-us/rest/api/apimanagement/2020-06-01-preview/apimanagementservice/checknameavailability)
<br>
![gujia93_3-1607485024622.png]({{site.baseurl}}/media/2020/12/2020-12-10-Self-restore-Deleted-APIM-gujia93_3-1607485024622.png)
<br>

1. 削除時に元のサービスが存在していたリソースグループが、サブスクリプションに存在している必要があります。リソースグループを削除した場合は、復元操作を正常に実行するために、同じ名前で再作成する必要があります。

以下のことはできない点に注意してください。
* "完全に削除された" サービスを復元する (サービスが "soft deleted" 状態ではなくなった)
* 特定のサービスの特定のエンティティのみを復元します。たとえば、ユーザーがサービスから API またはオペレーションを誤って削除した場合、それらのエンティティを復元することはできません。削除されたAPI Management サービスのみを復元できます。
* 復元時または復元中にサービスの名前を変更し、別のリソース グループまたはサブスクリプションに移動する。サービスは常に同じ名前で同じリソースグループと同じサブスクリプションに復元されます。
 

## APIM サービスを復元する
上記の APIM を復元するための前提条件を確認した後、「soft deleted」状態の API Management を復元する方法を確認してみましょう。この API (
[Api Management Service - Create Or Update](https://docs.microsoft.com/en-us/rest/api/apimanagement/2020-06-01-preview/apimanagementservice/createorupdate) )を使用してみてください。
リクエスト URL には、復元対象の  API Management のリソース名、リソースグループ名を含める必要があります。リクエストボディには、restore プロパティを含め、それを true に設定する必要があります。
 
リクエストボディの例は以下の通り:

```json
{
    "properties": {
        "publisherEmail": "gujia@microsoft.com",
        "restore": true
    },
    "sku": {
        "name": "Premium",
        "capacity": 1
    },
    "location": "australia east"
}
```

応答されたレスポンスペイロードは、API Managemnt サービスの復元が実行されていることを意味します。以下の例を参照してください。

![gujia93_0-1607483196504.png]({{site.baseurl}}/media/2020/12/2020-12-10-Self-restore-Deleted-APIM-gujia93_0-1607483196504.png)

## その他の考慮事項
サービスのパブリック VIP (IPアドレス) は、復元後に変更されます。バックエンドサービスが特定の IP アドレスからのトラフィックのみを許可するように構成されている場合は、新しい IP アドレスを許可するように構成を変更する必要があります (IP アドレスは Azure Portal の [概要] ブレードから取得できます)。

![gujia93_2-1607483963658.png]({{site.baseurl}}/media/2020/12/2020-12-10-Self-restore-Deleted-APIM-gujia93_2-1607483963658.png)

APIM サービスが外部/内部 VNET 内にあり、VNET/サブネット も削除されている場合、サービスは VNET に存在しない状態で復元されます(この操作は失敗する可能性があり、リトライにより解消する場合があります)。この復元された API Management サービスを後から VNET に追加する必要がある場合は、新しい VNET を作成し、復元された APIM サービスを VNET に追加する必要があります。

<br>
<br>

---

<br>
<br>

2020 年 12 月 10 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>