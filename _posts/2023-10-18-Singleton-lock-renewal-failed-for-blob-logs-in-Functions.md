---
title: "Functions の 「Singleton lock renewal failed for blob '***/host' with error code 409 …」 ログについて"
author_name: "koheihosono, hakuroda"
tags:
    - Function App
---

# 質問

Azure Functions を利用しています。 Application Insights ログ等に 「Singleton lock renewal failed for blob '***/host' with error code 409 …」 が記録されています。関数自体は正常に実行されているようですが、影響の有無を懸念しています。

```log
Singleton lock renewal failed for blob '***/host' with error code 409: LeaseIdMismatchWithLeaseOperation. The last successful renewal completed at 2023-00-00T00:00:00.000Z (*** milliseconds ago) with a duration of * milliseconds. The lease period was 15000 milliseconds.
```

# 回答

対象のログは Azure Functions の仕組みに関連した内部的かつ一時的なエラーであり、お客様の関数の実行への影響は想定されません。ログ メッセージを無視していただいて問題ございません。

## 動作原理

Azure Functions の動作において、各インスタンスには Azure Functions Host と呼ばれるプロセスが 1 つ動作しております。このプロセスの中で、プライマリーとされる Azure Functions Host を 1 つ選択する動作が存在します（特段ご利用者様で意識いただくことは不要でございます）。このプライマリーの Azure Functions Host は、アプリケーション設定 [`AzureWebJobsStorage`](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#azurewebjobsstorage) で指定されたストレージ アカウントの `azure-webjobs-hosts/locks/<Azure Functoins 名>/host` のファイルをロックすることでプライマリーとして動作いたします。必ずいずれかの Azure Functions Host がプライマリーとなり、Azure Functions の動作中は、`azure-webjobs-hosts/locks/<Azure Functoins 名>/host` のファイルをロックがされる状態となります。

※Azure Functions Host については[こちら](https://azure.github.io/jpazpaas/2023/08/24/azure-functions-words-relative-management.html)のブログ記事を参照ください。

![image-087b4036-9e14-4e88-820b-4a74f488f09f.png]({{site.baseurl}}/media/2023/10/image-087b4036-9e14-4e88-820b-4a74f488f09f.png)

また、host ファイルは空(0バイト)ですが、メタデータを確認すると該当の Azure Functions に割り当てされているインスタンスの ID の一部分が `FunctionInstance` の値として記録されています。

※任意の Azure Functions に割り当てられているインスタンスの ID はこちらの [List Instance Identifiers](https://learn.microsoft.com/ja-jp/rest/api/appservice/web-apps/list-instance-identifiers) からご確認いただけます。

![image-4a885582-19df-4132-83a2-be7ecfbfe871.png]({{site.baseurl}}/media/2023/10/image-4a885582-19df-4132-83a2-be7ecfbfe871.png)

上記の host ファイルのロックが競合する場合に、「Singleton lock renewal failed for blob '***/host' with error code 409 …」が発生いたします。


さて、ロックの競合とはどういう場合でしょうか。
[こちら](https://github.com/Azure/azure-functions-host/issues/1864#issue-255397275) の github issue でも紹介されておりますが、前述の通りインスタンスの ID が host ロック時に利用されております。通常一つのインスタンスに一つの Azure Functions Host となりますが、プロセスの再起動のタイミングには瞬間的ではあるものの二つの Azure Functions Host が存在する場合がございます。
しかしながら、該当 Azure Functions Host の新旧間で host ファイルのロック取得更新のタイミングが入れ違った場合には「Singleton lock renewal failed for blob '***/host' with error code 409 …」が発生いたします。

[本メッセージ出力時のフロー]
1. Azure Functions Host(旧) が host ファイルのロックを取得し、動作の過程でロックの更新を続けます。
   この時インスタンス A の ID `xxx` が host ファイルのメタデータ `FunctionInstance` に設定されています。
2. プロセスの再起動を契機に Azure Functions Host(新) が host ファイルのロックを取得します。同一のインスタンス A 上で動作しているためロックが取得できます。
3. プロセスの再起動の過程で Azure Functions Host(旧) が host ファイルのロックを開放します。
4. Azure Functions Host(新) が host ファイルのロックを更新しますが、この時項番 3 ですでに `FunctionInstance: xxx` のロックは解放されているために本メッセージが出力します。

![image-40b851e5-ecda-49f6-9dba-97111f463ed5.png]({{site.baseurl}}/media/2023/10/image-40b851e5-ecda-49f6-9dba-97111f463ed5.png)


また、本メッセージが出力された場合であっても、いずれかの Azure Functions Host が host ファイルのロックを取得するように動作する(host ファイルのロックが誰からも取得されいない状態は発生しません)ため Azure Functions の動作への影響はございません。

## 対応策
大変恐縮ながら Azure Functions の動作が原因となっており、具体的に発生頻度を減らすなどの軽減策が現時点ではございません。
そこで、Application Insights に出力されるログ メッセージをフィルタ(構成方法は[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/configure-monitoring?tabs=v2#configure-categories))することで本メッセージが Application Insights に出力されないように制御いただくことができます。

本メッセージは、Application Insights の `Host.Singleton`(ログカテゴリ) の `Error` レベルで出力され、例えば下記のクエリより発生有無をご確認いただけます。

![image-4d9f7423-ffc9-4b5c-94a8-83dafd366bdd.png]({{site.baseurl}}/media/2023/10/image-4d9f7423-ffc9-4b5c-94a8-83dafd366bdd.png)


```
traces
| where timestamp between (datetime("yyyy-mm-dd HH:MI") .. datetime("yyyy-mm-dd HH:MI"))
| where message startswith "Singleton lock renewal failed"
| where customDimensions.Category == "Host.Singleton"
| order by timestamp asc 
```

下記のように host.json を構成いただくことで `Host.Singleton` カテゴリのログを `None` とし、出力されないように制御できます。
```
{
  "version": "2.0",
  "logging": {
    "logLevel": {
      "Host.Singleton": "None"
    }
}
```

<br>
<br>

---

<br>
<br>

2023 年 10 月 18 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
