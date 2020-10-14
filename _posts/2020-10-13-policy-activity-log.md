---
title: "Azure Policyの評価結果のログがアクティビティログに出力されなくなった。"
author_name: "Kentaro Katahira"
tags:
    - Policy
---

# 質問
日次でのAzure Policy 評価結果が、アクティビティログに出力されていましたが、2020年6月下旬以降出力されなくなりました。なぜでしょうか。
# 回答
2020年6月下旬以降日次でのAzure Policy 評価結果については、アクティビティログに出力されなくなりました。
ただし、リソースをアップデートする際（新規作成も含む）に、その操作がポリシーに準拠しない操作の場合だけ、アクティビティログに出力されます。
<br>

# 変更以前と同等の結果を取得するためには
代替策として、PowerShellなどのコマンドラインツールでAzure Plicyの評価結果を取得することが可能です。以下にGet-AzPolicyStateコマンドレットを使用し、対象のポリシー定義に対して非準拠のSubscriptionId、ResourceGroup、ResourceIdをCSVファイルに吐き出すサンプルコマンドを記載しました。参考情報としてご活用ください。
<br>
## サンプルコマンド ※1) ※2)
`Get-AzPolicyState  -Filter "PolicyDefinitionName eq 'ポリシー定義名' and ComplianceState eq 'NonCompliant'" |Select ResourceGroup, ComplianceState, ResourceId,SubscriptionId|Export-Csv -Path test.csv -Encoding UTF8 -NoTypeInformation`
<br>

※1 PolicyDefinitionNameは、Policyポータルのポリシー定義部分にある、定義IDの末尾の文字列になります。<br>
![図1.jpg]({{site.baseurl}}/media/2020/10/2020-10-13-item.png)
<br><br>
※2 複数のサブスクリプションの評価結果を取得する場合は。Select-AzSubscriptionコマンドレットを使用して、サブスクリプションを切り替えながらコマンド実行していただくことで取得可能です。以下にサンプルコマンドを記載しましたので、ご参考にしてください。<br>
`$test = Get-AzSubscription | Select-Object SubscriptionId` <br>
`$test |ForEach-Object{`<br>
`>>Select-AzSubscription -SubscriptionId $_.SubscriptionId`<br>
`>>Get-AzPolicyState  -Filter "PolicyDefinitionName eq 'ポリシー定義名' and ComplianceState eq 'NonCompliant'" |Select ResourceGroup, ComplianceState, ResourceId,SubscriptionId|Export-Csv -Path policystate_$_.SubscriptionId.csv -Encoding UTF8 -NoTypeInformation`<br>
`>>}`<br>

# 参考ドキュメント

[Get-AzPolicyStateコマンドレット](https://docs.microsoft.com/en-us/powershell/module/az.policyinsights/get-azpolicystate?view=azps-4.7.0)<br>
[Select-AzSubscriptionコマンドレット](https://docs.microsoft.com/en-us/powershell/module/servicemanagement/azure.service/select-azuresubscription?view=azuresmps-4.0.0)

<br>
<br>

---

<br>
<br>

2020 年 10 月 13 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>