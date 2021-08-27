---
title: "Notification Hubs のデバイス登録情報を取得する方法"
author_name: "Yo Motoya"
tags:
    - Notification Hubs
---

# 概要

Notification Hubs を利用する際、[デバイスの登録](https://docs.microsoft.com/ja-jp/azure/notification-hubs/notification-hubs-push-notification-registration-management) を実施します。登録情報に基づいて通知の送信が行われるため、通知に失敗した場合に、現在の登録状況を確認したい場合があります。

本ブログでは、Service Bus Explore と REST API を使った方法について説明します。登録情報が取得できないとしてご質問いただくことが多いので、以下情報が少しでもお役に立ちましたら幸いです。なお、登録情報は Visual Studio を利用して取得することも可能ですが、[ドキュメント](https://docs.microsoft.com/ja-jp/azure/notification-hubs/notification-hubs-push-notification-fixer#visual-studio) に説明が記載されていますので、そちらをご確認ください。

# Service Bus Explore を使った登録情報の取得

## インストール

Service Bus Explore は [こちら](https://github.com/paolosalvatori/ServiceBusExplorer) のリポジトリに公開されています。README の手順に従って、インストールします。

## Notification Hubs への接続

Service Bus Explore を立ち上げ、[File]->[Connect] をクリックします。

![image.png]({{site.baseurl}}/media/2021/08/2021-08-27-sbx1.png)

[Service Bus Namespaces] で [Enter connection string...] を選択します。

![image.png]({{site.baseurl}}/media/2021/08/2021-08-27-sbx2.png)

[Filter Expressions] の [Selected Entities] にて [Notification Hubs] **のみ** を選択します。

![image.png]({{site.baseurl}}/media/2021/08/2021-08-27-sbx3.png)

Azure ポータルにて、対象の名前空間のページの [アクセス ポリシー] に移動し、接続文字列を取得します。対象ポリシーの [アクセス許可] に `Manage` が含まれていることを確認してください。`Manage` が含まれないポリシーの接続文字列では、対象 Notification Hubs に接続できません。

Azure ポータルから取得した接続文字列を [Connection Settings] の [Connection String:] に入力し、[OK] をクリックします。

![image.png]({{site.baseurl}}/media/2021/08/2021-08-27-sbx4.png)

## 登録情報の取得

対象の Notification Hub を右クリックし、[Get Registrations] を選択します。

![image.png]({{site.baseurl}}/media/2021/08/2021-08-27-sbx5.png)

以下の設定項目が表示されますが、確認したい情報に合わせて設定した後、[OK] を選択します。

![image.png]({{site.baseurl}}/media/2021/08/2021-08-27-sbx6.png)

# REST API を使った登録情報の取得

登録情報を取得するための REST API は以下の通り、複数用意されています。

* [登録を読み取る](https://docs.microsoft.com/ja-jp/rest/api/notificationhubs/read-registration)
* [チャネルのすべての登録を読み取ります。](https://docs.microsoft.com/ja-jp/rest/api/notificationhubs/read-all-registrations-channel)
* [タグを持つすべての登録を読み取ります](https://docs.microsoft.com/ja-jp/rest/api/notificationhubs/read-all-registrations-tag)
* [すべての登録を読み取る](https://docs.microsoft.com/ja-jp/rest/api/notificationhubs/read-all-registrations)

REST API を利用する際、`Authorization` ヘッダーに SAS トークンを付けてリクエストする必要があります。指定した SAS トークンに問題があり 401 エラーとなる場合が多いため、本ブログでは SAS トークンの作成手順について説明します。

[こちら](https://docs.microsoft.com/ja-jp/rest/api/eventhub/generate-sas-token) のドキュメントにて、SAS トークンを作成するためのサンプルプログラムが複数の言語で提供されています。今回は、PowerShell を利用します。

`$URI`、`$Access_Policy_Name`、`$Access_Policy_Key` を環境に合わせて変更する必要があります。その他の変数はご状況に合わせて変更をご検討ください。
* `$URI` は `<namespace_name>.servicebus.windows.net/<notification_hub_name>` とご指定ください。
* `$Access_Policy_Name` にはアクセスポリシー名を指定します。
* `$Access_Policy_Key` は、`$Access_Policy_Name` で指定したポリシーの接続文字列内の `SharedAccessKey=xxx` の `xxx` の部分を指定します。

```
[Reflection.Assembly]::LoadWithPartialName("System.Web")| out-null
$URI="yomotoyanhubnamespace.servicebus.windows.net/yomotoyanhub"
$Access_Policy_Name="RootManageSharedAccessKey"
$Access_Policy_Key="xxx"
#Token expires now+300
$Expires=([DateTimeOffset]::Now.ToUnixTimeSeconds())+300
$SignatureString=[System.Web.HttpUtility]::UrlEncode($URI)+ "`n" + [string]$Expires
$HMAC = New-Object System.Security.Cryptography.HMACSHA256
$HMAC.key = [Text.Encoding]::ASCII.GetBytes($Access_Policy_Key)
$Signature = $HMAC.ComputeHash([Text.Encoding]::ASCII.GetBytes($SignatureString))
$Signature = [Convert]::ToBase64String($Signature)
$SASToken = "SharedAccessSignature sr=" + [System.Web.HttpUtility]::UrlEncode($URI) + "&sig=" + [System.Web.HttpUtility]::UrlEncode($Signature) + "&se=" + $Expires + "&skn=" + $Access_Policy_Name
$SASToken
```

出力された SAS トークンを `Authorization` ヘッダーに指定いただくことで、REST API を使って登録情報を取得いただけます。もちろん、登録情報を取得するための REST API 以外の Notification Hubs の REST API でも、上記方法で作成した SAS トークンをご利用いただけます。

<br>
<br>

---

<br>
<br>

2021 年 8 月 30 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>