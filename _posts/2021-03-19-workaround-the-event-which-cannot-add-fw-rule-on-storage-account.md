---
title: "複数のストレージアカウントから同一のサブネットをファイアウォール設定している場合、サブネットが再作成されるとストレージアカウントのファイアウォール設定で対象のサブネットを追加できなくなる事象の回避策"
author_name: "Kentaro Katahira"
tags:
    - Storage
---

# 質問
複数のストレージアカウントから同一のサブネットを参照している場合、サブネットが再作成されるとストレージアカウント側で対象のサブネットを追加できなくなる事象や、<br>
対象のサブネットの FW 設定が有効にならないような挙動が発生する。<br>

## 具体的な例
１．以下のようなストレージアカウントを 2 つ用意します。（ここでは、test1kk、test2kk とします。）<br>
２．同様のサブネットをファイアウォール設定に追加します。（ここでは、windows というサブネットとします）
<br>
<br>

![image.png]({{site.baseurl}}/media/2021/03/2021-03-19-storage-account-test1kk.png)<br><br>


![image.png]({{site.baseurl}}/media/2021/03/2021-03-19-storage-account-test2kk.png)<br><br><br>
３．対象のサブネットを削除し、同様の名前で再作成します。<br>

![image.png]({{site.baseurl}}/media/2021/03/2021-03-19-delete-subnet.png)<br><br><br>
![image.png]({{site.baseurl}}/media/2021/03/2021-03-19-add-subnet.png)<br><br><br>
４．ストレージアカウントのファイアウォール設定で、対象のサブネットを再設定すると以下のようなエラーが発生します。

![image.png]({{site.baseurl}}/media/2021/03/2021-03-19-unable-save-settings.png)<br><br>

５．この状態で対象のサブネット内のリソースから対象のストレージアカウントにアクセスすると、エラーが発生します。
<br>（例えば StorageExplorer からアクセスした場合は、以下のように Unauthorized エラーが発生します。）<br>
![image.png]({{site.baseurl}}/media/2021/03/2021-03-19-error-details.png)<br><br>


# 回答
本事象は Azure の既知の挙動となります。<br>
このような事象が発生した場合は、複数のストレージアカウントのファイアウォール設定で対象のサブネットを削除し、再設定していただくことで事象を回避していただくことが可能です。



<br>
<br>

---

<br>
<br>

2021 年 3 月 19 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
