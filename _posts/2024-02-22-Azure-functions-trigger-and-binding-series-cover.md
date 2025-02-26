---
title: "Azure Functions トリガー・バインディング シリーズ"
author_name: "Hayato Kuroda"
tags:
    - Function App
    - トリガー・バインディング シリーズ
---

# 概要
Azure Functions はイベントドリブンな処理を実装する手段を提供します。つまり、Azure Functions では何かしらの契機無くアプリケーションが動作するというシナリオは存在しておらず、何かしらのイベントによってアプリケーション コードが動作する仕組みとなっています。Azure Functions の動作契機として接続可能な製品は多く、[トリガーとバインディング](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-triggers-bindings?tabs=csharp#supported-bindings)の2種類がそれぞれ定義されています。
トリガーやバインディングは Azure Functions の拡張機能を利用して各リソースと接続を行っています。サポートされているトリガーとバインディングの使用例は公式ドキュメントや GitHub のリポジトリにも記載しておりますが、トリガーの動作詳細やチューニング方法について多数お問い合わせをいただいております。本ブログでは、Azure Functions に関していただいておりましたお問合せの内容や、弊社エンジニアにて検証した内容をトリガーごとに整理した各記事へのリンク記事となっております。本ブログで紹介いたします情報が Azure Functions ご利用の際のお役に立ちますと幸いです。

# 記事一覧
※リンクが無い種類については今後投稿されますため、お待ちください。

| 項番 | 種類 |
|--|--|
| 1 | [ Blob Storage ](https://azure.github.io/jpazpaas/2025/02/26/Azure-functions-trigger-and-binding-series-blob.html) |
| 2 | Azure Cosmos DB |
| 3 | Azure Data Explorer |
| 4 | Azure SQL |
| 5 | Dapr |
| 6 | Event Grid |
| 7 | [Event Hubs](https://azure.github.io/jpazpaas/2024/02/22/Azure-functions-trigger-and-binding-series-eventhubs.html) |
| 8 | HTTP と Webhook |
| 9 | [IoT Hub](https://azure.github.io/jpazpaas/2025/02/26/Azure-functions-trigger-and-binding-series-iothub.html) |
| 10 | Kafka |
| 11 | Mobile Apps |
| 12 | Notification Hubs |
| 13 | [ Queue Storage ](https://azure.github.io/jpazpaas/2025/02/26/Azure-functions-trigger-and-binding-series-queue.html) |
| 14 | Redis |
| 15 | RabbitMQ |
| 16 | SendGrid |
| 17 | Service Bus |
| 18 | SignalR |
| 19 | Table Storage |
| 20 | [Timer](https://azure.github.io/jpazpaas/2024/02/22/Azure-functions-trigger-and-binding-series-timer.html) |
| 21 | Twilio |


# 関連リンク
[Azure Functions（関数アプリ）の内部アーキテクチャ概要や関連する用語について](https://azure.github.io/jpazpaas/2023/08/24/azure-functions-words-relative-management.html)



<br>
<br>

---

<br>
<br>

2025 年 02 月 26 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>