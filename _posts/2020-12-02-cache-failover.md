---
title: "Azure Cache for Redis のフェールオーバーについて"
author_name: "Eiji Mochida"
tags:
    - Azure Cache for Redis
---

# 質問
Azure Cache for Redis を使っていますが、メトリックを見ていたら、フェールオーバーが発生したように見受けられますが、障害等発生しておりますでしょうか。

# 回答
フェールオーバーというと障害のイメージがあるかと思いますが、Azure Cache for Redis におきましては、プラットフォーム側の各種更新の範囲内でフェールオーバーが発生することは想定される動作となります。そのため、フェールオーバーが発生したことがすべて障害を示すものではございません。

こちらが、フェールオーバーについてまとめたドキュメントとなります。フェールオーバーの概要やプラットフォーム側の各種更新の範囲内でもフェールオーバーが発生することを踏まえてどのようなことを考慮しておくべきかということが記載されております。

[Azure Cache for Redis のフェールオーバーと修正プログラムの適用](https://docs.microsoft.com/ja-jp/azure/azure-cache-for-redis/cache-failover)

上記資料の "フェールオーバーはクライアント アプリケーションにどのような影響を与えますか?" という節にある通り、タイムアウト例外、接続例外、またはソケット例外など、さまざまな種類の例外が発生する可能性がございます。その場合、クライアントアプリケーション側からの再接続が必要になることがございます。このように Redis にアクセスするクライアントアプリケーション側でも考慮していただく点については以下の資料に記載がございます。

[Azure Cache for Redis のベスト プラクティス](https://docs.microsoft.com/ja-jp/azure/azure-cache-for-redis/cache-best-practices#client-library-specific-guidance)

こちらの資料内の "クライアント ライブラリ固有のガイダンス" という節にライブラリごとのベストプラクティスへのリンクがございます。例えば、.NET Framework でご利用いただく StackExchange.Redis や Java でご利用いただく Jedis や Lettuce についての言及がご確認いただけます。

Azure Cache for Redis をご利用いただく際に、少しでも参考になれば幸いです。

---

<br>

2020 年 12 月 02 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>