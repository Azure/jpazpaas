---
title: "Request ID を基に Storage へのアクセス失敗を調べる方法"
author_name: "GaYeon Lee"
tags:
    - Storage
---

# 目次<br>
• はじめに<br>
• クライアント IP アドレスを確認する方法<br>
• ファイアウォールの設定<br>
• 3行要約<br>
<br><br>

# はじめに<br>
「Storage にアクセスしたら、403 のエラーでアクセスが拒否される」というお問い合わせをいただき、調べてみたら Storage Account のファイアウォールの設定で許可してない IP アドレスからのアクセスだったということが時々ございます。<br><br>
![image-a16eb5fd-2ced-4b6b-9663-428c0c479a75.png]({{site.baseurl}}/media/2022/09/image-a16eb5fd-2ced-4b6b-9663-428c0c479a75.png)<br><br>
この際、上記の写真のように要求 ID などが確認出来れば、プラットフォームのログから Storage Account にとってのクライアント IP アドレスが何になるか知ることが出来ます。以下では、このクライアント IP アドレスを確認する方法と、ファイアウォールの設定についてご紹介いたします。<br>

<br><br><br>

# クライアント IP アドレスを確認する方法<br>
診断設定を利用し、このアクセスが Storage Account にとってどんなアクセスになるかを確認することが出来ます。この方法は予め Log Analytics のワークスペースを作成するなど、ストレージへのアクセスのログを確認できるようにしておく必要があります。<br>
[Log Analytics ワークスペースを作成する](https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/quick-create-workspace?tabs=azure-portal)<br><br>

1. 対象の Storage Account にサインインし、[監視 (Monitoring)] から [診断設定 (Diagnostic settings)] に移動します。その後、[blob] を選択します。
<br><br>
![image-03f77c91-add3-4d23-a65f-bff8fb9afd53.png]({{site.baseurl}}/media/2022/09/image-03f77c91-add3-4d23-a65f-bff8fb9afd53.png)<br>
2. [診断設定を追加する] をクリックすると、写真のようなページが現れます。診断設定名を入力し、左側の [メトリック (Metrics)] では [Transaction] を選択します。右側の [宛先の詳細 (Destination Details)] では [Log Analytics ワークスペースへの送信 (Send to Log Analytics workspace)] を選択し、ワークスペースを指定します。最後に、左上の[保存 (Save)] をクリックすると、診断設定は完了です。
<br><br>
![image-a77de7b7-dfb0-4584-adea-15dcb2cd9475.png]({{site.baseurl}}/media/2022/09/image-a77de7b7-dfb0-4584-adea-15dcb2cd9475.png)<br>
3. 次に、エラーとなったアクセスを再現し、要求 ID を確認します。<br>
<br><br>
![image-7a4af3e9-545e-4ba1-ac22-fa05c31ceb35.png]({{site.baseurl}}/media/2022/09/image-7a4af3e9-545e-4ba1-ac22-fa05c31ceb35.png)<br>
4. 手順 1. で指定した Log Analytics Workspace で、[一般(General)] の [ログ(Logs)] に移動し以下のコマンドを入力します。<br>
	ー－－－－－－－－－－－－－－－－<br>
	StorageBlobLogs<br>
	| where CorrelationId == "(手順 3. でコピーした要求 ID)"<br>
	ー－－－－－－－－－－－－－－－－<br>
<br>
5. 写真のような結果が出力されます。
<br><br>
![image-717957ff-d877-4efa-8c15-66b71beb545c.png]({{site.baseurl}}/media/2022/09/image-717957ff-d877-4efa-8c15-66b71beb545c.png)<br><br>
該当のログを選択したら、以下の詳細情報が分かります。<br><br>

## • CallerIpAddress<br>
アクセス時の IP アドレスを表します。ファイアウォールで設定した IP アドレスで合ってるか、比較することが出来ます。<br>
## • ステータス メッセージ (StatusText) <br>
ログに記録された要求を表します。以下の URL で、全てのステータス メッセージの情報を確認することが出来ます。<br>
[ログに記録された操作と状態メッセージの Storage Analytics (REST API)](https://learn.microsoft.com/ja-jp/rest/api/storageservices/storage-analytics-logged-operations-and-status-messages) <br>
ファイアウォールによりアクセスが拒否される場合、ステータス メッセージは 「AuthorizationFailure」 となるため、「データへの不正アクセス、または承認の失敗」のログになります。
<br><br><br>

# ファイアウォールの設定方法
<br>
ファイアウォールとは、特定 IP アドレスや特定の仮想ネットワークからのアクセスのみを許可することで、ストレージのアクセス元を特定のクライアントに限定する設定となります。<br><br>

先述のようなログの確認の結果、想定とは異なる IP アドレスからアクセスされていた場合、以下のような対応方法が考えられます。<br>

1. 確認の結果、該当の IP アドレスがアクセス元として想定していない IP アドレスである場合は、アクセスの経路を確認し、ファイアウォールで許可した  IP アドレスからストレージにアクセスするようにクライアントの経路を修正する。<br>
2. 該当の IP アドレスがアクセス元として許可しても問題ないと判断できる場合、ファイアウォールの設定で、エラーとなった IP アドレスを追加する。<br>
<br><br>
![image-06fc7a47-f2bf-44d4-a16d-7a040aafb01c.png]({{site.baseurl}}/media/2022/09/image-06fc7a47-f2bf-44d4-a16d-7a040aafb01c.png)
<br><br>
ストレージのファイアウォールの構成方法については、以下の URL で確認することが可能です。
<br>
[Azure Storage ファイアウォールおよび仮想ネットワークを構成する](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-network-security?tabs=azure-portal)


<br><br><br>
# 3 行要約<br>
1. ファイアウォールはストレージアカウントへのアクセスの経路を特定の経路に限定する。<br>
2. ファイアウォールで設定していない IP アドレスからのアクセスは拒否される。<br>
3. 診断設定より、Storage Account にとってアクセス元のクライアント IP アドレスが何になるかを確認することができる。<br>

<br><br>

---

<br>
<br>

2022 年 9 月 22 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>

