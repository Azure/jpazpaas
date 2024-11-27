---
title: "App service 証明書をエクスポートしてパスワードを設定し、他の Azure サービスにインポートする方法"
author_name: "Kosuke Uo"
tags:
    - App Service 証明書
    - Application Gateway
---

# 2023-02 追記

2020 年 9 月に投稿された、本記事では、App Service 証明書をエクスポートし、Application Gateway で利用する手順を紹介しておりますが、
Application Gateway SKU v2 をご利用いただくことで Key Vault に格納された App Service 証明書をエクスポートせずに Application Gateway に直接紐づけることが可能となっております。

Application Gateway と App Service 証明書を直接紐づけることにより、Application Gateway は Key Vault に格納された最新の App Service 証明書を定期的に取得することで、 Application Gateway において証明書の更新作業が不要となります。また、セキュリティの観点でも Application Gateway を直接紐づけることが推奨されます。

詳しくは以下の記事をご確認ください。

[Azure Key Vault 証明書を使用した TLS 終端 \| Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/application-gateway/key-vault-certs)  
[Application Gateway のリスナーに Key Vault に格納された App Service 証明書を表示させる方法 \- Japan PaaS Support Team Blog](https://azure.github.io/jpazpaas/2022/09/16/How-to-import-ASC-to-AppGW.html)

---

# はじめに

[App Service 証明書](https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate) は、Azure によって管理される証明書です。

App Service 証明書は Azure Portal より注文いただけます。
購入された App Service 証明書は Azure Key Vault に格納され、App Service にインポートして使用いただくことができます。

また、App Service 証明書は、App Service 以外のサービスに証明書をご利用いただけるように、PFX 形式でのエクスポートをサポートしております。

しかしながら、エクスポートした PFX 形式の証明書にはパスワードが設定されておりません。

Azure Application Gateway などの Azure サービスに証明書をインポートする際には、証明書のパスワード入力が必要とされるため、エクスポート後のパスワード設定が必要となります。

本記事では、App Service 証明書を PFX 形式でエクスポートし、パスワードを設定する方法をご紹介します。

また、今回は App Service 以外のサービスとしてApplication Gateway を用い、実際に App Service 証明書をインポートする方法も紹介します。

本記事の構成は以下の通りです。

- App Service 証明書をエクスポートし、パスワードを設定する方法
   - GUI を用いる方法
   - CLI (Azure PowerShell) を用いる方法
- Application Gateway に App Service 証明書をインポートする方法
- 参考リンク

#  App Service 証明書をエクスポートし、パスワードを設定する方法

## GUI を用いる方法

手順は以下の３つです。

手順１）App Service 証明書をエクスポートする

手順２） エクスポートした証明書をローカル環境にインポートする (Windows 10 を使用した場合の例)

手順３） ローカル環境でパスワードを設定する

***

手順１）App Service 証明書をエクスポートする。

1-1. App Service 証明書のページより、[証明書のエクスポート] を選択し、Key Vault シークレットを開きます。

![1-1-6668bbe3-b039-45a1-ac00-fec8d21486e0.gif]({{site.baseurl}}/media/2024/11/1-1-6668bbe3-b039-45a1-ac00-fec8d21486e0.gif)

1-2. エクスポート対象として、現在のバージョンを選択します。

![1-2-f50f0c39-291d-434b-a70d-993fbfca0392.gif]({{site.baseurl}}/media/2024/11/1-2-f50f0c39-291d-434b-a70d-993fbfca0392.gif)

1-3. 下部の [証明書としてダウンロード] を選択します。

![1-3-7529019f-9c0b-43de-ad8c-fe5556c332ee.gif]({{site.baseurl}}/media/2024/11/1-3-7529019f-9c0b-43de-ad8c-fe5556c332ee.gif)

手順２） エクスポートした証明書をローカル環境にインポートする

2-1. 証明書を右クリックし、 [PFX のインストール(I)] をクリックします。

![2-1-77705061-5e84-48bd-9249-fbfd59474de6.gif]({{site.baseurl}}/media/2024/11/2-1-77705061-5e84-48bd-9249-fbfd59474de6.gif)

2-2. [証明書のインポート ウィザード] が開始されますので、順を追って進みます。保存場所は [現在のユーザー(C)] を指定します。

![2-2-b2f1cfba-f240-4a1b-b064-a41a86d7c2a1.gif]({{site.baseurl}}/media/2024/11/2-2-b2f1cfba-f240-4a1b-b064-a41a86d7c2a1.gif)

2-3. インポートする証明書ファイルは 1-3 でエクスポートした証明書を指定します。

![2-3-455d6592-d39e-46c5-9590-cd30b2df59ae.gif]({{site.baseurl}}/media/2024/11/2-3-455d6592-d39e-46c5-9590-cd30b2df59ae.gif) 

2-4. インポートオプションで、 [このキーをエクスポート可能にする(M)] にチェックを入れます。

![2-4-5331fe4e-7836-4db5-9e46-96fb5e7c843b.gif]({{site.baseurl}}/media/2024/11/2-4-5331fe4e-7836-4db5-9e46-96fb5e7c843b.gif)

2-5. [証明書の種類に基づいて、自動的に証明書ストアを選択する (U)] を選択します。

![2-5-e0c46e5d-6255-489c-a1b1-32ca7e895fcd.gif]({{site.baseurl}}/media/2024/11/2-5-e0c46e5d-6255-489c-a1b1-32ca7e895fcd.gif)

2-6. インポートの確認が表示されますので、[完了] をクリックします。以上でインポートは完了です。

![2-6-cdad02c5-d0a3-4473-a3ae-abe98835d830.gif]({{site.baseurl}}/media/2024/11/2-6-cdad02c5-d0a3-4473-a3ae-abe98835d830.gif)

手順３） ローカル環境でパスワードを設定する。

先ほどインポートした証明書にパスワードを設定して保存 (エクスポート) します。

3-1. [ユーザー証明書の管理] を実行します。スタートメニューより cert で検索しますと、表示されます。

![3-1-4bcfa759-54a0-4bf4-a978-7f41f91d5599.gif]({{site.baseurl}}/media/2024/11/3-1-4bcfa759-54a0-4bf4-a978-7f41f91d5599.gif)

3-2. ローカルの Windows に格納されている証明書の一覧が表示されます。先ほどインポートした証明書は、左側のツリーの [個人 > 証明書] に格納されています。

![3-2-13f09546-e483-43e1-a753-7378212ff27b.gif]({{site.baseurl}}/media/2024/11/3-2-13f09546-e483-43e1-a753-7378212ff27b.gif)

3-3. 対象の証明書を右クリックして、 [すべてのタスク(K)] > [エクスポート(E)] をクリックします。

3-4. 証明書のエクスポート ウィザードが開始されます。

![3-4-a41ec8e4-ba88-4b84-b997-687a1b591cb1.gif]({{site.baseurl}}/media/2024/11/3-4-a41ec8e4-ba88-4b84-b997-687a1b591cb1.gif)

3-5. [はい、秘密鍵をエクスポートします(Y)] を選択します。

![3-5-1eaf074c-aaa6-4e7f-91af-d2eed1bf8262.gif]({{site.baseurl}}/media/2024/11/3-5-1eaf074c-aaa6-4e7f-91af-d2eed1bf8262.gif)

3-6. エクスポートするファイル形式として、[Personal Information Exchange - PKCS#12] を選択します。

![3-6-c42d64d9-ed23-477e-8aee-833662fdc96b.gif]({{site.baseurl}}/media/2024/11/3-6-c42d64d9-ed23-477e-8aee-833662fdc96b.gif)

3-7. パスワードを設定します。任意の文字列で問題ございませんが、ある程度長い文字列で推測が難しいものをお勧めします。

![3-7-1e8a4507-8986-44ae-954e-34eb14dda066.gif]({{site.baseurl}}/media/2024/11/3-7-1e8a4507-8986-44ae-954e-34eb14dda066.gif)

3-8. パスワードを付けて、エクスポートした証明書を保存先を指定します。

![3-8-0fa78f15-24e6-49ca-add0-0a7dd2fbcb7e.gif]({{site.baseurl}}/media/2024/11/3-8-0fa78f15-24e6-49ca-add0-0a7dd2fbcb7e.gif)

3-9. エクスポートの確認が表示されますので、[完了] をクリックします。

![3-9-1b11a24a-db20-483e-b692-3391f162c7f9.gif]({{site.baseurl}}/media/2024/11/3-9-1b11a24a-db20-483e-b692-3391f162c7f9.gif)

以上の手順によって、GUIでパスワード付きの証明書をエクスポートすることができます。

## CLI を用いる方法

以下の Azure CLI コマンドを使って、CLI から PFX 形式証明書をエクスポートすることができます。<br>
[Azure CLI のコマンド | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/app-service/configure-ssl-app-service-certificate?tabs=cli#tabpanel_1_cli)  

エクスポート後、前述の「**GUI を用いる方法**」の手順２）３）を実施し、証明書へのパスワードを設定してください。


**注意**

上記、[Azure CLI のコマンド | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/app-service/configure-ssl-app-service-certificate?tabs=cli#tabpanel_1_cli) のリンク先には App Service 証明書を PowerShell でエクスポートするコマンドの記載もありますが、Application Gateway で利用する場合は、PowerShell によるエクスポートは推奨されません。<br>
PowerShell でエクスポートした証明書は中間証明書が含まれていない構成となり、Application Gateway で利用する場合は別途、中間証明書を含む形で構成し直す必要があります。詳細については[Application Gateway の証明書関連のトラブルシューティング - 中間証明書の構成方法](https://jpaztech.github.io/blog/network/appgw-troubleshooting-cert/#%E4%B8%AD%E9%96%93%E8%A8%BC%E6%98%8E%E6%9B%B8%E3%81%AE%E6%A7%8B%E6%88%90%E6%96%B9%E6%B3%95)でご紹介しておりますので、必要に応じて参照ください。


## Application Gateway に App Service 証明書をインポートする方法

上記の方法で手に入れたパスワード付きの PFX 形式証明書を、 Application Gateway にインポートする方法を紹介します。

1. Azure Portalから、証明書をインポートする Application gateway のページに移動します。

2. 左側のメニューから、リスナーを選択します。

![3-2-adf0d8bd-2400-4e8a-ace1-174e2d680a42.png]({{site.baseurl}}/media/2024/11/3-2-adf0d8bd-2400-4e8a-ace1-174e2d680a42.png)

3. 証明書をインポートする HTTPS リスナーを選択します。

![3-3-b7ff820b-9ed4-41f3-b70a-65275767fc78.PNG]({{site.baseurl}}/media/2024/11/3-3-b7ff820b-9ed4-41f3-b70a-65275767fc78.PNG)

4.  証明書をインポートします

- 「証明書の選択」から、「新規作成」を選択します。
- 「 HTTPS 設定」>「証明書の選択」から、「証明書のアップロード」を選択します。
- 「 PFX 証明書ファイル」に、エクスポートした PFX 形式の証明書をアップロードします。
- 「証明書名」に、任意の名前を記入します。
- 「パスワード」に、設定したパスワードを記入します。

![3-4-b4ee5f78-8599-4967-99a1-d8cb720f5588.PNG]({{site.baseurl}}/media/2024/11/3-4-b4ee5f78-8599-4967-99a1-d8cb720f5588.PNG)

5. 「保存」を選択します。

以上の手順によって、Application Gateway に証明書をインポートできます。

## 参考リンク

[Creating a local PFX copy of App Service Certificate](https://azure.github.io/AppService/2017/02/24/Creating-a-local-PFX-copy-of-App-Service-Certificate.html)  
[ポータルで Application Gateway を使用してエンド ツー エンド TLS を構成する](https://learn.microsoft.com/ja-jp/azure/application-gateway/end-to-end-ssl-portal)  
[Web アプリの App Service 証明書を作成して管理する](https://learn.microsoft.com/ja-jp/azure/app-service/configure-ssl-app-service-certificate?tabs=portal)  
[Application Gateway の証明書関連のトラブルシューティング](https://jpaztech.github.io/blog/network/appgw-troubleshooting-cert)

<br>
<br>

----

<br>
<br>

2024 年 11 月 27 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>