---
title: "API Management で Developer SKU や Basic SKU を利用する際の注意点"
author_name: "Hirotaka Shionoiri"
tags:
    - API Management
---

# 質問
Developer SKU や Basic SKU で利用している API Management において Capacity が急上昇したり、接続が不安定になりました。<br>
また、過去にも発生しており、稀ではあるもののある程度定期的に発生している気がします。<br>
これはなぜですか？

# 回答
Developer SKU はテスト用の価格のため、1 台のインスタンスで稼働していることから、<br>
アプリケーションの更新や OS の更新に伴いご利用不可能な状態が発生する場合があります。<br>
また、Basic SKU も限られたリソースで稼働するため、高負荷な状態が発生する場合があります。

# 解説
API Management (以下 APIM) は PaaS サービスであるため、普段、ほとんど基盤を意識することなくご利用頂けます。<br>
そんな APIM がどのように稼働しているかというと、<br>
Cloud Service または Virtual Machine Scale Set 上でアプリケーションが稼働しています。<br>
これは下記の資料に記載があります。<br>
<br>
**Azure API Management のコンピューティング プラットフォーム**<br>
[https://docs.microsoft.com/ja-jp/azure/api-management/compute-infrastructure](https://docs.microsoft.com/ja-jp/azure/api-management/compute-infrastructure)<br>
<br>
簡潔には、VM 上で APIM というアプリケーションが稼働していると捉えて頂ければ問題ありません。<br>
<br>
VM 上で稼働しているということは、<br>
APIM のアプリケーションの更新だけでなく、APIM を動かしている VM の OS の更新も発生するということになります。<br>
更に、普段ご利用頂いているコンピュータ同様、APIM においてもアプリケーションの更新や OS の更新時には、<br>
再起動を伴うケースがあります。<br>
<br>
Developer SKU の場合には上述の通り開発テスト用 SKU なので、1 台のインスタンスで稼働 (冗長構成が行われていない) しており、<br>
更新時には再起動等によって稼働中のインスタンスが一時的に 0 台となってしまいます。<br>
このことから、接続不可能などの症状が発生することが予想されます。<br>
また、再起動の前後も高負荷な状態（Capacity が高い値となる状態）となり、パフォーマンスが安定しない状態が予想されます。<br>
<br>
Basic SKU 以上の場合、1 ユニットごとに 2 台のインスタンスが稼働しています。<br>
よって、Basic SKU 以上であれば、更新時も稼働中のインスタンスが 1 台は維持されるため、完全なサービス停止は発生しません。<br>
しかし、稼働中のインスタンスが 1 台は維持されるとはいえ、更新中は 1 台のインスタンスで 2 台分の働きをする必要がありますので、<br>
Basic SKU を 1 ユニットでご利用中の場合に、更新に伴い高負荷な状態を検知する例が見受けられます。<br>
<br>
OS の更新は定期的に実行されます。よって定期的に本記事で紹介した事象は発生し得るものです。<br>
ただし、ご利用のない時間帯に更新作業が実施されたために、<br>
「今回初めて（更新中の高負荷状態に）気づいた」「極めて稀にしか体感していない」ということもあるかと思います。<br>
とはいえ、Developer SKU には SLA も適用されないことから、常時安定稼働が必要な場合には Basic 以上の SKU をご利用下さい。<br>
<br>
恐縮ながら、これらの更新についてはユーザー側でタイミングを指定できるものではありません。<br>
また、リージョンの時間帯や各サービスの利用状況から自動的に判断して更新が実行されるため、<br>
具体的なタイミングを事前に通知することはできません。<br>
<br>
事後であればアクティビティログ等からご確認頂ける場合があります。<br>
下記の画像はあくまで一例とはなりますが、<br>
Developer SKU の APIM が更新により再起動していた際のアクティビティログです。<br>

![image.png]({{site.baseurl}}/media/2022/04/2022-04-01-apim-activitylog.png)

# 参考ドキュメント
* Developer SKU をご利用頂いていると各種構成変更の際にもダウンタイムがありますが、<br>
これも上述のインスタンスの台数に起因します。Developer SKU をご利用の際には併せてご注意下さい。<br>
<br>
    **Azure API Management インスタンスのアップグレードとスケーリングを行う**<br>
    [https://docs.microsoft.com/ja-jp/azure/api-management/upgrade-and-scale#downtime-during-scaling-up-and-down](https://docs.microsoft.com/ja-jp/azure/api-management/upgrade-and-scale#downtime-during-scaling-up-and-down) <br>
    >Developer レベルへのスケールダウンまたは Developer レベルからのスケールアップを行うとき、<br>
    >ダウンタイムが発生します。 それ以外の場合、ダウンタイムはありません。<br>

    **Azure API Management インスタンスのカスタム ドメイン名を構成する**<br>
    [https://docs.microsoft.com/ja-jp/azure/api-management/configure-custom-domain?tabs=custom#set-a-custom-domain-name---portal](https://docs.microsoft.com/ja-jp/azure/api-management/configure-custom-domain?tabs=custom#set-a-custom-domain-name---portal)　<br>
    >証明書を割り当てる処理は、デプロイのサイズによっては、15 分以上かかる場合があります。<br>
    >Developer レベルにはダウンタイムがありますが、Basic 以上のレベルにはありません。<br>

* Windows系 OS の更新は毎月発生し得るものです。よって月に1度程度、本記事で紹介した事象が発生する可能性があります。<br>
<br>
**毎月の品質更新プログラム**<br>
[https://docs.microsoft.com/ja-jp/windows/deployment/update/quality-updates](https://docs.microsoft.com/ja-jp/windows/deployment/update/quality-updates)

<br>
<br>

---

<br>
<br>

2022 年 4 月 1 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>