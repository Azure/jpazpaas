---
title: "Azure Storageのアクセス ログの有効化手順について"
author_name: "Rui Hiraoka"
tags:
    - Azure Storage
---

# 質問
ストレージ アカウントの診断設定を有効化することで、データの書き込みや読み取り操作のアクセス ログを取得したいです。どのように設定をすればよいですか。

# 回答
書き込みや読み取りといったデータ プレーンのアクセス ログについては、診断設定から採取設定が必要となります。Azure Portalでの診断設定の設定手順を以下に記載します。<br><br>
(1) ブレード メニューの[監視]-> [診断設定]をクリックします。<br>


![image-dd2d9684-9cbf-4dc4-ac13-33d5814f0900.png]({{site.baseurl}}/media/2024/02/image-dd2d9684-9cbf-4dc4-ac13-33d5814f0900.png)<br>

ストレージ アカウント名のほかに、ホストされている4つのストレージ サービス、blob, queue, table, fileが表示されます※1。ストレージ アカウントでは、個々のストレージ　サービス単位で診断設定を構築し、アクセス ログの転送設定を行うことができます。<br>
![image-133ebc4f-4892-4a14-a086-73fb8e31f04b.png]({{site.baseurl}}/media/2024/02/image-133ebc4f-4892-4a14-a086-73fb8e31f04b.png)<br>
(2) 本記事では例としてblobを選択し、Blob Storageの診断設定を構築します。その他 3つqueue, file, tableのストレージ サービスにおいても設定手順は同様となります。<br>


![image-fed9036f-3d82-4e68-9c1c-4a1583c69a84.png]({{site.baseurl}}/media/2024/02/image-fed9036f-3d82-4e68-9c1c-4a1583c69a84.png)<br>
(3) [+診断設定を追加する]を押下します。<br>


![image-6c67924b-8fb0-408d-b919-b3a2c76416f9.png]({{site.baseurl}}/media/2024/02/image-6c67924b-8fb0-408d-b919-b3a2c76416f9.png)<br>

(4) [Azure Monitor の診断設定](https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/diagnostic-settings)のドキュメントに記載のある設定画面が表示されます。要件に沿うよう、ログのカテゴリと送付先のサービスを選択ください。カテゴリの詳細は[診断設定のカテゴリ グループ](https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/diagnostic-settings?WT.mc_id=Portal-Microsoft_Azure_Monitoring#resource-logs)をご参照ください。

サポートより採取を依頼した際は、allLogsをご選択ください。


![image-bb3f113e-f209-43d2-b75e-e08f1af090bc.png]({{site.baseurl}}/media/2024/02/image-bb3f113e-f209-43d2-b75e-e08f1af090bc.png)<br>

(5) 送付先としてストレージ アカウントを設定した場合、insights-logsのprefixを持つコンテナーにそれぞれログが格納されます。<br>
![image-e3f0c53f-465d-437d-84f5-b7afd6c34446.png]({{site.baseurl}}/media/2024/02/image-e3f0c53f-465d-437d-84f5-b7afd6c34446.png)<br>

ストレージ サービスの種類が異なるものや、別のストレージ アカウントのアクセス ログも同じコンテナーにフォルダ分けされて格納されます。対象のストレージ サービスとアカウント名の配下のアクセス ログをご参照ください。複数のファイルをGUIでダウンロードする場合は、当BLOGの[複数の BLOB をまとめてダウンロードする方法 ](https://azure.github.io/jpazpaas/2023/06/01/how-to-download-multiple-blobs.html)に方法を記載しておりますので、ご参照ください。<br>

# 補足
ストレージ アカウント スコープで診断設定を構築しても、アクセス ログの取得とはなりません。
下記画面の通り、メトリクスの取得のみとなります。blobなど対象のサービス レベルで診断設定を構築する必要があります。<br>


![image-6ec19a2d-011b-4b06-b602-e27818f54cb0.png]({{site.baseurl}}/media/2024/02/image-6ec19a2d-011b-4b06-b602-e27818f54cb0.png)<br>

また一部、記録されない操作もありますので、ご注意ください。詳細は[Azure Blob Storage の監視](https://learn.microsoft.com/ja-jp/azure/storage/blobs/monitor-blob-storage?tabs=azure-portal#log-authenticated-requests)でご確認いただけます。<br>
たとえば、別のストレージ アカウントに送付したアクセス ログは手順(5)の通り、insights-logsのprefixのコンテナーに格納されますが、このコンテナ一に対するアクセス ログは診断設定で、記録することが叶いませんので、[診断設定(クラシック)](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-analytics-logging?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&bc=%2Fazure%2Fstorage%2Fblobs%2Fbreadcrumb%2Ftoc.json)を用いてログを記録する必要があります。診断設定(クラシック)で、ログを有効化する手順については [ログの有効化](https://learn.microsoft.com/ja-jp/azure/storage/common/manage-storage-analytics-logs?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&bc=%2Fazure%2Fstorage%2Fblobs%2Fbreadcrumb%2Ftoc.json&tabs=azure-portal)をご参照ください。


# まとめ
本記事ではストレージ アカウントにホストされるストレージ サービスごとのアクセス ログの収集方法を記載しました。<br>ストレージ アカウント レベルではなく、blob, queue, table, fileの個別のサービスごとに設定を実施するものとなりますので、ご注意ください。

# 注釈
※1: Premium Storageなど、SKUによっては、サービスが4つホストされないことがあります。


