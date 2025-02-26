---
title: "Azure Functions トリガー・バインディング シリーズ - IoT Hub トリガー"
author_name: "Hayato Kuroda"
tags:
    - Function App
    - トリガー・バインディング シリーズ
---

# 概要と基本動作
[IoT Hub トリガー](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-event-iot?tabs=isolated-process%2Cextensionv5&pivots=programming-language-csharp)は、指定された IoT Hub からイベントを取得し実行される関数です。

IoT Hub トリガーは、IoT Hub の Event Hubs 互換エンドポイントを利用してトリガーされ、内部動作は Event Hubs トリガーと同一となります。そのため、詳細については[こちら](https://azure.github.io/jpazpaas/2024/02/22/Azure-functions-trigger-and-binding-series-eventhubs.html)を参照ください。

# 参考ドキュメント
[Azure Functions（関数アプリ）の内部アーキテクチャ概要や関連する用語について](https://azure.github.io/jpazpaas/2023/08/24/azure-functions-words-relative-management.html)

[Azure Functions の Azure IoT Hub バインド](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-event-iot?tabs=isolated-process%2Cextensionv5&pivots=programming-language-csharp)


<br>
<br>

---

<br>
<br>

2025 年 02 月 26 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>