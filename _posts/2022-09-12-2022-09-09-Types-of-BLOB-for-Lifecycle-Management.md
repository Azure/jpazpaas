---
title: "ライフサイクル管理を設定する際の Blob の種類について"
author_name: "GaYeon Lee"
tags:
    - デプロイ履歴
---


# 目次<br>
• はじめに<br>
• 自動でログを削除するライフサイクル管理ポリシーの作成方法<br>
• 作成したライフサイクル管理ポリシーの確認方法<br>
• まとめ<br>
• 3行要約<br>
<br><br>

# はじめに<br>
　
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
 
# 自動でログを削除するライフサイクル管理ポリシーの作成方法 <br>

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

　＊ライフサイクル ポリシーは、プラットフォームによって 1 日に 1 回実行されます。 新規ポリシーを作成した後、アクションによっては、初回実行時に最大 24 時間かかる場合がありますのでご注意ください。<br>
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
Log Analytics を使用し Delete Blob のオペレーションを列挙することが可能です。この方法は予めログの出力先として設定したストレージ アカウントの Log Analytics を作成する必要があります。<br>
	
<br>先ず、対象 Log Analytics ワーク スペース（写真では、「workspacefordeleteblob」）に移動します。[全般 (General)] の項目より[ログ (Logs)] を選択し、以下のクエリを実行した結果が BLOB の削除が成功したログとなります。
	
	   StorageBlobLogs
	   | where OperationName == "DeleteBlob"
	   | where StatusText == "Success"
	   | where UserAgentHeader contains "ObjectLifeCycleScanner"

	
	 
<br>![image-7f3e9845-e834-44ed-94e6-8978b4ac4d5a.png]({{site.baseurl}}/media/2022/09/image-7f3e9845-e834-44ed-94e6-8978b4ac4d5a.png)
<br>各項目をクリックすると、テナント ID や URI などの詳細情報が更に確認出来ます。

<br><br>	
	
# まとめ<br>

   ファイル ベースの方法で簡単に確認できるため、診断設定を利用しストレージ アカウントのログを出力するように設定した場合でも、そのログが多くなると確認しづらくなります。このようなジレンマを解決するため、ライフサイクル管理ポリシーを構成し一定期間が経過した後自動的にログが削除されるように設定することが出来ます。また、このポリシーの実行状況をメトリックや Log Analytics などを利用し確認することが出来ます。<br>

![image-61c17501-e9f6-4112-9367-42dea6dedaf3.png]({{site.baseurl}}/media/2022/09/image-61c17501-e9f6-4112-9367-42dea6dedaf3.png)
<br>

# 3行要約<br>
1. 診断設定でログの出力が出来る<br>
2. ライフサイクル管理ポリシーで対象の Blob 種類を [追加 BLOB (Append blobs)] に設定することで、一定期間が経過した後ログを自動的に削除することが出来る<br>
3. メトリックや Log Analytics などでログの自動削除を確認することが出来る<br>

<br><br><br>
2022 年 9 月 9 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。
