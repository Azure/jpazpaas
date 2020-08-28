---
title: "DeploymentQuotaExceeded のエラーについて"
author_name: "Toshinao Onodera"
tags:
    - ARM
    - デプロイ履歴
---

# 質問

サービスをデプロイしようとしたときに以下のエラーに遭遇した。対処方法を教えてほしい。<br><br>
`Creating the deployment 'cra3000_website-20190131-150417-7d5f' would exceed the quota of '800'. The current deployment count is '800', please delete some deployments before creating a new one. Please see https://aka.ms/arm-deploy for usage details.`

# 原因
各リソース グループには、そのデプロイ履歴が 800 までという上限があります。 このエラーメッセージは、デプロイを行うときに、デプロイ履歴が許可されている 800 を超える場合に表示されます。<br><br>

ただし、現時点でのプラットフォーム側の実装では、デプロイ履歴の自動削除機能が実装されています。そのため、以下のようなシナリオにおいて、本事象が発生する可能性があります。<br><br>

1. リソースグループに対して削除ロックを付与している場合
2. デプロイ履歴の自動削除をオプトアウトしている場合

# 対処方法
本制限値についてはプラットフォーム側で上限の緩和などの対応ができません。そのため、デプロイ履歴を削除する必要があります。前述の原因 1, 2 それぞれのアプローチについて以下説明いたします。なお、デプロイ履歴を削除しても配置済みのリソースへの影響はございません。しかしながら、デプロイ履歴からのテンプレートのエクスポートやデプロイ履歴からの再デプロイなど、デプロイ履歴が削除されることにより実施ができなくなりますのでご承知おきください。
<br>
<br>

- リソースグループに対して削除ロックを付与している場合<br>
リソースグループの削除ロックの解除を行う必要があります。<br>
  <br> 1. [Azure ポータル](https://portal.azure.com/) にアクセスします。
  <br> 2. 対象のリソースグループを表示します。
  <br> 3. 左側メニュー [ロック] を選択すると、右側にロックの一覧が表示されます。
![image.png]({{site.baseurl}}/media/2020/08/2020-08-28-delete-lock.png)
　<br> 4. 削除するロックの右側の [削除] を押してください。 
  <br>　→ 確認は表示されずすぐにロックが削除されます。
  <br>
　<br> 5. デプロイ履歴の自動削除がオプトアウトされていない場合には、自動でデプロイ履歴は削除が行われます。もし、オプトアウトしている場合には、後述の [デプロイ履歴の自動削除をオプトアウトしている場合] に進んでください。
  <br><br>
- デプロイ履歴の自動削除をオプトアウトしている場合
　<br> 手動での履歴の削除が必要です。Powershell や CLI での削除も可能ですが、ここでは Azure ポータルからの削除について案内します。
　<br><br>
  <br> 1. [Azure ポータル](https://portal.azure.com/) にアクセスします。
  <br> 2. 対象のリソースグループを表示します。
　<br> 3. [デプロイ] メニューを開きます。
　<br> 4. デプロイ履歴の一覧が表示されますので、削除する履歴にチェックを入れ、[削除] を押下します。
<br> ![image.png]({{site.baseurl}}/media/2020/08/2020-08-28-delete-deployment-history1.png)
<br>
  <br> 5. 確認の画面が表示されますので、 [はい] と入力後、[削除] を押下します。
<br> ![image.png]({{site.baseurl}}/media/2020/08/2020-08-28-delete-deployment-history2.png)

# 参考リンク

- [Azure Resource Manager でのデプロイ履歴の表示](https://docs.microsoft.com/ja-jp/azure/azure-resource-manager/templates/deployment-history?tabs=azure-portal)
- [デプロイ履歴からの自動削除](https://docs.microsoft.com/ja-jp/azure/azure-resource-manager/templates/deployment-history-deletions?tabs=azure-powershell)
- [デプロイ数が 800 を超えたときのエラーを解決する](https://docs.microsoft.com/ja-jp/azure/azure-resource-manager/templates/deployment-quota-exceeded)

<br>
<br>

----

<br>
<br>

2020 年 8 月 28 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>