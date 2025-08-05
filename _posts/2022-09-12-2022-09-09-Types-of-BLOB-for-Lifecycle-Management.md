---
title: "ライフサイクル管理を設定する際の Blob の種類について"
author_name: "GaYeon Lee"
tags:
    - デプロイ履歴
---


#目次<br>
• はじめに<br>
• 自動でログを削除するライフサイクル管理ポリシーの作成方法<br>
• 作成したライフサイクル管理ポリシーの確認方法<br>
• まとめ<br>
• 3行要約<br>
<br><br>

#はじめに<br>
　
   Azure Monitor の出力するログ（以下の写真参照、[insights-logs-XXXX] ）を一定期間が経った後に消すために Azure Storage の Blob のライフサイクル管理の規則を入れたけれど、想定していた通りに消えないというお問い合わせをいただくことが時々あります。<br>

![image-f37e20bb-3374-45c9-9d2b-443fcb4cecb8.png]({{site.baseurl}}/media/2022/09/image-f37e20bb-3374-45c9-9d2b-443fcb4cecb8.png)

<br>

   このログは、診断設定で [他のストレージ アカウントに保存する] を選択したため出力される、対象アカウント（ここでは、「satest0829」）のログです。下記のスクリーンショットでは、「satest0829」 ストレージ アカウントの読み取り、上書き、削除のログを 「r40904mon」 ストレージ アカウントに出力させる診断設定になります。<br><br>
![image-4122d2d6-0cad-4b10-8e4a-41bb5d0cc9bc.png]({{site.baseurl}}/media/2022/09/image-4122d2d6-0cad-4b10-8e4a-41bb5d0cc9bc.png)

（診断設定の作成方法に関しては、以下の URL で確認出来ます。<br>
[Azure Monitor の診断設定｜診断設定の作成](https://docs.microsoft.com/ja-jp/azure/azure-monitor/essentials/diagnostic-settings?tabs=portal）))<br><br>

  ライフサイクル管理設定を行ったのにもかかわらず、このようなログが削除されない理由は、ライフサイクル管理ポリシーの対象がブロック BLOB のみになっていて、追加 BLOB を対象に入れていないことに起因します。<br><br>

   では、診断設定を変えずに、このようなログを自動で削除するためのライフサイクル管理ポリシーを実際に構成してみましょう。<br><br>
<br>
 
#自動でログを削除するライフサイクル管理ポリシーの作成方法 <br>

1. Azure Portal からログ出力対象のストレージ アカウントに移動します。<br>
2. [データ管理] の [ライフサイクル管理] を選択し、[規則を追加 (Add a rule)] をクリックします。<br><br>
![image-aa71c355-e66a-44ea-85c2-852809e1b965.png]({{site.baseurl}}/media/2022/09/image-aa71c355-e66a-44ea-85c2-852809e1b965.png)
	
3. 必要に応じて詳細を選択します。<br>
ここで、Azure Monitor が出力する insights-logs-XXXX と表記されるコンテナー配下のログは追加 BLOB になるので、ポリシー作成時に [追加 BLOB (Append blobs)] を対象にする必要があります。<br><br>
![image-9e96980e-1df0-4da2-af3d-faabe3eb7bdb.png]({{site.baseurl}}/media/2022/09/image-9e96980e-1df0-4da2-af3d-faabe3eb7bdb.png)
	
4. 条件（生成された後に実行するか、最後に修正された後に実行するか）を選択し、実行（削除）までの期間を設定し、[追加 (Add)] を選択します。<br><br>
![image-bf1d6cc5-f86a-4bfd-a1a3-bc7bac4e918f.png]({{site.baseurl}}/media/2022/09/image-bf1d6cc5-f86a-4bfd-a1a3-bc7bac4e918f.png)
<br>
	

   これで、ライフサイクル管理ポリシーの作成は完了です。<br><br>

  Azure Portal の他、PowerShell、Azure CLI 又は Azure Resource Manager テンプレートを使用してライフサイクル管理ポリシーを構成、編集する方法やその他の詳細については以下の URL で確認出来ます。<br>
[ライフサイクル管理ポリシーを構成する](https://docs.microsoft.com/ja-jp/azure/storage/blobs/lifecycle-management-policy-configure?tabs=azure-portal)
<br><br>

　＊ライフサイクル管理のポリシーを作成・更新してから有効になるまで最大 24 時間、さらに実行されるまでに 24 時間以上かかる場合がありますのでご注意ください。<br>
<br><br>


# 作成したライフサイクル管理ポリシーの確認方法<br>

　では、実際にログがライフサイクル管理ポリシーによって削除されたか確認する方法は何があるのでしょうか？<br>
<br>

###	• 件数だけの場合<br>
メトリックで Delete Blob 実行の件数を一目で見ることが可能です。<br><br>

Azure Monitor で Metrics に移動します。ストレージ アカウントの範囲を選択し適用すると、チャートが作成されます。その後、フィルターの [メトリック] 部分で [Transactions] を選択します。更に上の方にある [フィルターの追加 (Add Filter)] を選択し、[プロパティ (Property)] に [API name]、[値 (Value)] に [DeleteBlob] を入力すると、時間軸に沿った実行を確認することが出来ます。<br><br>
![image-5a702bf3-c8f9-47a2-a5a4-f907c1af0cec.png]({{site.baseurl}}/media/2022/09/image-5a702bf3-c8f9-47a2-a5a4-f907c1af0cec.png)
<br>
	
（その他のメトリック操作の詳細に関しては、以下の URL で確認出来ます。<br>
	[Azure Monitor のメトリックス エクスプローラーの高度な機能](https://docs.microsoft.com/ja-jp/azure/azure-monitor/essentials/metrics-charts) )<br>
	
<br>ただし、このチャートにはライフサイクル管理ポリシーだけでなく、Azure Portal からの手動削除など、他の操作による Delete Blob も含まれます。そのため、実行数が多い場合は以下の Log Analytics を利用し、ライフサイクル管理ポリシーの実行状況を詳細に確認することが出来ます。<br><br>
	
	
###	• 詳細に確認する場合<br>
<br>
$logs コンテナーに出力された診断ログから Delete Blob のオペレーションを確認することが可能です。この方法は、予め [診断ログ (クラシック)] から、ログの出力を有効化しておく必要があります。<br>

<br>[診断設定 (クラシック)] ログ出力設定例
![image-81d44b47-c906-4ff6-8615-7da8e25e5210.png]({{site.baseurl}}/media/2022/09/image-81d44b47-c906-4ff6-8615-7da8e25e5210.png)

<br>      
下記出力例のように、<user-agent-header> フィールドに、"ObjectLifeCycleScanner" が出力されたエントリを確認することで、
　　ライフサイクル管理によって行われたリクエストのログをご確認いただけます。
<br>
<br>凡例 (バージョン 2.0 の場合)：
		
	<version-number>;<request-start-time>;<operation-type>;<request-status>;<http-status-code>;<end-to-end-latency-in-ms>;<server-latency-in-ms>;<authentication-type>;<requester-account-name>;<owner-account-name>;<service-type>;<request-url>;<requested-object-key>;<request-id-header>;<operation-count>;<requester-ip-address>;<request-version-header>;<request-header-size>;<request-packet-size>;<response-header-size>;<response-packet-size>;<request-content-length>;<request-md5>;<server-md5>;<etag-identifier>;<last-modified-time>;<conditions-used>;<user-agent-header>;<referrer-header>;<client-request-id>;<user-object-id>;<tenant-id>;<application-id>;<audience>;<issuer>;<user-principal-name>;<reserved-field>;<authorization-detail>
	 
<br>出力例 (バージョン 2.0 の場合)：
	
	2.0;2025-07-18T19:17:16.5043171Z;DeleteBlob;TrustedAccessSasSuccess;202;13;13;TrustedAccessSas;;XXXXX;blob;"https://XXXXX.blob.core.windows.net:443/insights-logs-storageread/resourceId%3D/subscriptions/XXXXX/resourceGroups/XXXXX/providers/Microsoft.Storage/storageAccounts/XXXXX/blobServices/default/y%3D2024/m%3D10/d%3D18/h%3D12/m%3D00/PT1H.json?sv=2020-06-12&amp;ss=b&amp;srt=sco&amp;sp=rwdlacuptx&amp;se=2025-08-01T18%3A36%3A35.4416023Z&amp;spr=https&amp;sk=system-1&amp;sig=XXXXX&amp;timeout=10";"/XXXXX/insights-logs-storageread/resourceId=/subscriptions/XXXXX/resourceGroups/XXXXX/providers/Microsoft.Storage/storageAccounts/XXXXX/blobServices/default/y=2024/m=10/d=18/h=12/m=00/PT1H.json";4ec85451-801e-0041-1218-f86f9d000000;0;XXX.XXX.XXX.XXX:59234;2019-12-12;869;0;271;0;0;;;;;;"xAzure/1.0 AzureStorage/1.0 ObjectLifeCycleScanner/0.50";;"XXXXX_ms-tyo31prdstr02a_2025-07-18T18:34:57.4214940Z_2faabfc8-9051-4f24-ace5-1a95ceb43ee9_";;;;;;;;


<br>![image-2bf8a9cb-0c9f-49ee-83c5-c4c36cb5cde1.png]({{site.baseurl}}/media/2022/09/image-2bf8a9cb-0c9f-49ee-83c5-c4c36cb5cde1.png)

<br>出力されるログの項目内訳につきましては、下記をご参照ください。
<br>[Storage Analytics ログ形式 (REST API) - Azure Storage | Microsoft Learn](https://learn.microsoft.com/ja-jp/rest/api/storageservices/storage-analytics-log-format)

<br><br>	
	
# まとめ<br>

   ファイル ベースの方法で簡単に確認できるため、診断設定を利用しストレージ アカウントのログを出力するように設定した場合でも、そのログが多くなると確認しづらくなります。このようなジレンマを解決するため、ライフサイクル管理ポリシーを構成し一定期間が経過した後自動的にログが削除されるように設定することが出来ます。また、このポリシーの実行状況をメトリックや $logs コンテナーへ出力したログ などを利用し確認することが出来ます。<br>

![image-61c17501-e9f6-4112-9367-42dea6dedaf3.png]({{site.baseurl}}/media/2022/09/image-61c17501-e9f6-4112-9367-42dea6dedaf3.png)
<br>

# 3行要約<br>
1. 診断設定でログの出力が出来る<br>
2. ライフサイクル管理ポリシーで対象の Blob 種類を [追加 BLOB (Append blobs)] に設定することで、一定期間が経過した後ログを自動的に削除することが出来る<br>
3. メトリックや $logs コンテナーなどでログの自動削除を確認することが出来る<br>

<br><br><br>
2025 年 8 月 4 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

# 変更履歴  
2022 年 9 月 9 日：公開
<br>2025 年 8 月 4 日：「詳細に確認する場合」の確認方法を Log Analytics → [診断設定 (クラシック)] で出力したログ ($logs) から確認する方法に変更