---
title: "Event Hubs に格納されたイベントの中身を確認する方法"
author_name: "Takumi Nagaya"
tags:
    - Event Hubs
---

# 質問
Event Hubs に格納されたイベントの中身を確認する方法を教えてください。
# 回答
まず、Event Hubs 自体には、イベントを中身を確認する機能はございません。<br/>
基本的には、以下のように別のサービスにイベントを出力したり、別のサービスが Event Hubs のデータを取得して閲覧できるようにしたり、
お客様自身で受信用のアプリケーションをご用意いただく必要がございます。

- Stream Analytics ジョブを使用して、イベントデータをプレビューする
- お客様にてイベント受信のアプリケーションを構築する
- Event Hubs のキャプチャ機能を用いて、Azure Storage / Azure Data Lake Storage に格納されたファイルを読み取る
- コミュニティが所有するオープンソースの Service Bus Explorer の Event Hubs Listener 機能を利用する

## Stream Analytics ジョブを使用して、イベントデータをプレビューする
イベントデータが CSV / Json / Avro 形式であれば、Stream Analytics ジョブを使用することで、<br/>
Azure Portal 上で簡単にイベントの中身を確認することができます。
以下のドキュメントに記載されています。

>- データのプレビュー – Azure portal 内でイベント ハブからの受信データをプレビューできます。<br/>
[Azure Stream Analytics を使用してイベント ハブからのデータを処理する](https://learn.microsoft.com/ja-JP/azure/event-hubs/process-data-azure-stream-analytics)

例えば Blob Storage の診断ログを Event Hubs に送信するようにした場合、
以下のように診断ログを確認することができます。<br/>
![image-032fdbe2-e8ee-4baf-8f59-db1936549b96.png]({{site.baseurl}}/media/2023/05/image-032fdbe2-e8ee-4baf-8f59-db1936549b96.png)

詳細な手順については、ドキュメント [エンド ツー エンドのフロー](https://learn.microsoft.com/ja-JP/azure/event-hubs/process-data-azure-stream-analytics#end-to-end-flow) の手順 7 まで実施いただければと存じます。

また、Stream Analytics を利用することで、追加の料金が発生を懸念される場合もあると存じます。<br/>
Stream Analytics ジョブを作成せず、上記のプレビュー機能をご利用いただく限りでは、追加の料金は発生しませんので、ご安心いただければと存じます。<br/>
ただし、Stream Analytics ジョブを作成する (ドキュメント [エンド ツー エンドのフロー](https://learn.microsoft.com/ja-JP/azure/event-hubs/process-data-azure-stream-analytics#end-to-end-flow) の手順 10 を実施) する場合は、[Stream Analytics 料金](https://azure.microsoft.com/ja-jp/pricing/details/stream-analytics/) が発生いたしますのでご注意ください。

## お客様にてイベント受信のアプリケーションを構築する
Event Hubs からイベントを受信するためのアプリケーションを構築いただき、受信したイベントの中身を確認することが可能でございます。<br/>
以下の .NET のみならず、Java、Python などで受信アプリケーション構築のクイックスタートがございますので参考になれば幸いです。

>[クイック スタート: .NET を使用して Azure Event Hubs との間でイベントを送受信する イベント ハブからイベントを受信する](https://learn.microsoft.com/ja-jp/azure/event-hubs/event-hubs-dotnet-standard-getstarted-send?tabs=connection-string%2Croles-azure-portal#receive-events-from-the-event-hub)

上記クイックスタートを元にアプリケーションを作成し実行すると、例えば Blob Storage の診断ログを Event Hubs に送信するようにした場合、以下のようにイベントの中身を確認することができます。<br/>
![image-2a31653e-1139-4255-9ce3-bc18e722e99a.png]({{site.baseurl}}/media/2023/05/image-2a31653e-1139-4255-9ce3-bc18e722e99a.png)

## Event Hubs のキャプチャ機能を用いて、Azure Storage / Azure Data Lake Storage に格納されたファイルを読み取る
Event Hubs のキャプチャ機能を用いて、Azure Storage / Azure Data Lake Storage に Avro 形式でイベントデータを格納することが可能でございます。<br/>
Avro 形式で保存されるため、Avro ファイルを読み取るアプリケーションを用意した上でイベントの中身を確認する必要がございますが、長期的にイベントデータを保持する要件がある場合は、ご利用を検討いただけますと幸いです。

>[Azure Event Hubs で Azure Blob Storage または Azure Data Lake Storage にイベントをキャプチャする](https://learn.microsoft.com/ja-jp/azure/event-hubs/event-hubs-capture-overview)

キャプチャ機能を有効にする方法につきましては、以下のドキュメントをご確認ください。<br/>
>[Azure Event Hubs からストリーム配信されるイベントのキャプチャを有効にする](https://learn.microsoft.com/ja-jp/azure/event-hubs/event-hubs-capture-enable-through-portal)

Avro ファイルの操作につきましては、以下のドキュメントが参考になれば幸いです。
>[Azure Event Hubs でキャプチャされた Avro ファイルの操作](https://learn.microsoft.com/ja-jp/azure/event-hubs/explore-captured-avro-files)<br/>
[クイックスタート: Event Hubs データを Azure Storage にキャプチャし、Python を使用してそれを読み取る (azure-eventhub)](https://learn.microsoft.com/ja-jp/azure/event-hubs/event-hubs-capture-python)

Event Hubs のキャプチャ機能は Event Hubs 名前空間が Premium プランの場合は追加の料金は発生しませんが、<br/>
Standard レベルの場合はご利用のスループットユニット数に比例して料金が発生するため、ご注意ください。<br/>
詳細は [Event Hubs Capture に対する課金方法](https://learn.microsoft.com/ja-jp/azure/event-hubs/event-hubs-capture-overview#how-event-hubs-capture-is-charged) をご確認ください。<br/>
また、キャプチャ機能では Azure Storage / Azure Data Laka Storage にイベントデータが格納されるため、Azure Storage / Azure Data Laka Storage でも料金が発生する旨ご留意いただけますと幸いです。

## コミュニティが所有するオープンソースの Service Bus Explorer の Event Hubs Listener 機能を利用する
Microsoft としてサポートしている製品ではございませんが、[Service Bus Explorer](https://github.com/paolosalvatori/ServiceBusExplorer/tree/main) と呼ばれるコミュニティが所有するオープンソースのスタンドアロンアプリケーションがあり、こちらを利用することで Event Hubs のイベントを取得し閲覧することができます。

Service Bus Explorer をインストールの上、以下の手順でイベントを確認することができます。

1. Service Bus Explorer を起動し、「Create Event Hub Listener」を選択します。

![image-018a8cfe-18f1-40d5-8281-298974205ff4.png]({{site.baseurl}}/media/2023/05/image-018a8cfe-18f1-40d5-8281-298974205ff4.png)

2. 「Event Hub Connection String」に Event Hubs の接続文字列を、「Event Hub Path」に Event Hub 名を、「Consumer Group」 にコンシューマグループ名 (既定では $Default) を記載します。

![image-516ff741-8e81-4ac8-932a-2f5b25cfba08.png]({{site.baseurl}}/media/2023/05/image-516ff741-8e81-4ac8-932a-2f5b25cfba08.png)

3. 「Events」タブに移動し、「Start」をクリックするとイベントの読み取りを開始します。「Stop」をクリックするまで継続してイベントを取得します。以下のようにイベントの中身を確認することができます。

![image-bdc65576-b579-42bd-b6d4-9fa671e2d442.png]({{site.baseurl}}/media/2023/05/image-bdc65576-b579-42bd-b6d4-9fa671e2d442.png)

上記手順とほぼ同様のものが、以下の記事にも記載がございますので、参考になれば幸いです。<br/>
Event Hubs への接続方法については「Connect to a namespace」を、<br/>
実際にイベントを取得する方法については「Event Hub - Receiving event messages through a consumer group」の項目をご確認ください。<br/>
[How to send messages to or receive from Service Bus/Event Hub with Service Bus Explorer?](https://techcommunity.microsoft.com/t5/azure-paas-blog/how-to-send-messages-to-or-receive-from-service-bus-event-hub/ba-p/2136244)
# 参考ドキュメント

- [Azure Stream Analytics を使用してイベント ハブからのデータを処理する](https://learn.microsoft.com/ja-JP/azure/event-hubs/process-data-azure-stream-analytics)
- [クイック スタート: .NET を使用して Azure Event Hubs との間でイベントを送受信する | イベント ハブからイベントを受信する](https://learn.microsoft.com/ja-jp/azure/event-hubs/event-hubs-dotnet-standard-getstarted-send?tabs=connection-string%2Croles-azure-portal#receive-events-from-the-event-hub)
- [Azure Event Hubs で Azure Blob Storage または Azure Data Lake Storage にイベントをキャプチャする](https://learn.microsoft.com/ja-jp/azure/event-hubs/event-hubs-capture-overview)
- [Azure Event Hubs からストリーム配信されるイベントのキャプチャを有効にする](https://learn.microsoft.com/ja-jp/azure/event-hubs/event-hubs-capture-enable-through-portal)
- [Azure Event Hubs でキャプチャされた Avro ファイルの操作](https://learn.microsoft.com/ja-jp/azure/event-hubs/explore-captured-avro-files)
- [クイックスタート: Event Hubs データを Azure Storage にキャプチャし、Python を使用してそれを読み取る (azure-eventhub)](https://learn.microsoft.com/ja-jp/azure/event-hubs/event-hubs-capture-python)
- [How to send messages to or receive from Service Bus/Event Hub with Service Bus Explorer?](https://techcommunity.microsoft.com/t5/azure-paas-blog/how-to-send-messages-to-or-receive-from-service-bus-event-hub/ba-p/2136244)

<br>
<br>

---

<br>
<br>

2023 年 5 月 12 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>