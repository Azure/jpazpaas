---
title: "API Management におけるバックエンド向けリージョン間負荷分散について"
author_name: "Koichiro Higashi"
tags:
    - API Management
---

# 質問
API Management ( APIM ) のバックエンドとして各リージョンに配置した API サーバーに対する API リクエストを負荷分散する際に選択すべき Azure サービスをご教示願います。

# 回答
 APIM のバックエンド向けにリージョン間負荷分散をご要望の場合は、以下の Azure 負荷分散サービスをバックエンドに配置することをご検討ください。
* Azure Front Door
* Traffic Manager
* Azure Load Balancer

![Backend load balancing.png]({{site.baseurl}}/media/2021/07/2021-07-09-backend-load-balancing.png)

それぞれの Azure 負荷分散サービスの特徴ならびに APIM と組み合わせて利用する上での注意点を以下に示します。

## Azure Front Door
Web アプリケーション向けのグローバル負荷分散およびサイト アクセラレーション サービスを提供するアプリケーション配信ネットワークであり、SSL オフロード、パスベースのルーティング、高速フェールオーバー、キャッシュなどの様々なレイヤー 7 機能を提供しているため、APIサーバーのパフォーマンスや可用性向上が期待されます。

現時点ではAzure Front Doorでは **WebSocket をサポートしていない** ため、 APIM にて WebSocket の利用を検討している場合は Traffic Manager もしくは Azure Load Balancer をご選択ください。

## Traffic Manager
世界中の Azure リージョン間でサービスへのトラフィックを最適に配分しつつ、高可用性と応答性を実現する DNS ベースのトラフィック ロード バランサーとなりますため、負荷分散先の切り替えタイミングは DNS キャッシュ時間に依存します。

APIM は Traffic Manager に対して **※約 2 分周期で DNS クエリを送出し、取得した IP アドレスは次周期までキャッシュします** ので、 Traffic Manager にて設定した DNS TTL は遵守せずにキャッシュしたIPアドレスに従って バックエンドの API サーバーに API リクエストを送信します。
従いまして、Azure Front Door や Azure Load Balancer ほど高速なフェールオーバーはできません。

※DNSサーバーへの負荷を考慮した実装とはなりますが、現時点での実装に基づいた動作となりますため、今後変更となる可能性がございます。

## Azure Load Balancer
UDP と TCP プロトコル向けの高パフォーマンス、超低待機時間のレイヤー 4 負荷分散サービスとなります。Azure Front Door と比較すると API リクエスト処理のオーバーヘッドが軽減されるため、 APIM から API サーバーへの転送時間の短縮が期待されます。

現状はAzure Load Balancerによるリージョン間負荷分散機能は **プレビュー** の段階でありますので、運用環境のワークロードでに使用するのはお勧めできません。

[リージョン間ロード バランサー (プレビュー)](https://docs.microsoft.com/ja-jp/azure/load-balancer/cross-region-overview)

## まとめ
上述いたしましたそれぞれの Azure 負荷分散サービスにおきまして特徴ならびに注意点を十分に理解した上で APIM との組み合わせをご検討いただけますと幸いでございます。

# 参考ドキュメント
* [Azure 負荷分散を理解する](https://docs.microsoft.com/ja-jp/azure/architecture/guide/technology-choices/load-balancing-overview)
* [Azure Front Door とは](https://docs.microsoft.com/ja-jp/azure/frontdoor/front-door-overview)
* [Traffic Manager について](https://docs.microsoft.com/ja-jp/azure/traffic-manager/traffic-manager-overview)
* [Azure Load Balancer の概要](https://docs.microsoft.com/ja-jp/azure/traffic-manager/traffic-manager-overview)

<br>
<br>

---

<br>
<br>

2021 年 7 月 9 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>