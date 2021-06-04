---
title: "Service Bus および Event Hubs の UserError、ServerError について"
author_name: "Kentaro Katahira"
tags:
    - Service Bus,Event Hubs
---

# よくある質問
Service Bus や Event Hubs のサーバエラー、ユーザエラーともに、どういった場合に発生しますか？

# 回答

## ・サーバエラーについて

基本的にはプラットフォーム側で何かあった場合に Event Hubs のサービス側 (Azure プラットフォーム側) で何らかのエラーが発生した場合に生じます。
具体的には、以下になります。<br>
- メンテナンスによる一過性の状況
- 障害による緊急的な状況

## ・ユーザーエラーについて

ユーザーエラーはクライアントサイド側で何らかの処理が適切に行われない状況に発生する可能性がございます。<br>
具体的には、以下になります。<br>
- Service bus や、Event Hubs に接続するアプリケーションで設定する接続文字列が間違っているなど、アプリケーションの構成に問題がある状況
- Service bus や、Event Hubs に接続するアプリケーション内で、例外が発生している状況（メンテナンスの影響も含む）
<br>

## ・対処について

・サーバエラーについて<br>
基本的に一過性のものについては、リトライを実施いただくことにより、処理が完了することが見込まれます。<br>

・ユーザーエラーについて<br>
状況によって対応が分かれます。<br>
極端な例となってしまいますが、URL のパスが間違っていたり、メソッドで指定した引数が間違っている場合などは<br>
何度リトライしても失敗で終わってしまいます。<br>
そのため、まずは Service Bus や、Event Hubs に接続するアプリケーションの構成をご確認いただいたり、<br>SDK、.NET Framework APIを使用している場合は、例外時のアクションが公式ドキュメントにございますので、ご参考にしていただけると幸いです。<br>
<br>
Service Bus や、Event Hubs に接続するアプリケーションとして、Funstion App の Service Bus トリガーなどをご利用いただいている構成でユーザーエラーが発生する事例がございます。Function App では定期的に AMQP プロトコルを使用して、Event Hubs や Service Bus へメッセージがないか確認する処理を実施しています。そのタイミングでメンテナンスが発生、またはネットワークの瞬断が発生した場合には、ユーザーエラーが記録される可能性がございます。これは、関数自体が実行されていないときに発生する可能性がございますので、業務影響がないものであれば無視していただいてもよいものもございます。<br>

## ・メトリック監視について

上記のように、ユーザーエラー・サーバーエラーのメトリックは、様々な要因でカウントされうるメトリックとなります。<br>
そのため、メトリックカウントを 0 以上などの閾値で監視している場合、リトライ処理でカバーされるものや、一過性の事象の場合も<br>
検知される可能性がございます。監視要件については、お客様環境に依存するため、お客様の運用方針に基づき決めていただく必要が御座いますが、<br>
例えば、継続して、エラーが発生している場合に warning として発砲し、そのタイミングで Service Bus、Event Hubs に接続しているアプリケーションに影響が出ていた場合に初めてエラーとして扱う、または、アプリケーション側のエラーを検知したうえで、メトリックを確認し、継続的に発生しているか確認するなど、複合的に判断していただくことをお薦めいたします。<br>


数分以上継続して発生しており、リトライ処理でカバーできない状況が発生している場合は、サーバー側での障害の可能性もございますため、<br>
弊社にお問合せください。

# 参考ドキュメント

再試行～一般的なガイドライン<br>
https://docs.microsoft.com/ja-jp/azure/architecture/best-practices/transient-faults
<br><br>
Event Hub ~ 再試行メカニズム<br>
https://docs.microsoft.com/ja-jp/azure/architecture/best-practices/retry-service-specific#event-hubs
<br><br>
Service Bus ~ 再試行メカニズム<br>
https://docs.microsoft.com/ja-jp/azure/architecture/best-practices/retry-service-specific#service-bus
<br><br>
Event Hub ~ メッセージングの例外<br>
https://docs.microsoft.com/ja-jp/azure/event-hubs/event-hubs-messaging-exceptions
<br><br>
Service Bus ~ メッセージングの例外<br>
https://docs.microsoft.com/ja-jp/azure/service-bus-messaging/service-bus-messaging-exceptions
<br><br>
<br><br>


---

<br>
<br>

2020 年 6 月 4 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>

