---
title: "Application Gateway のリスナーに Key Vault に格納された App Service 証明書を表示させる方法"
author_name: "Kanta Kaneda"
tags:
    - "App Service 証明書"
---

App Service 証明書をエクスポートする手順が以下の記事で紹介されており、エクスポートされた証明書は Application Gateway などで利用することが可能となっております。

参考 : [App service 証明書をエクスポートしてパスワードを設定し、他の Azure サービスにインポートする方法 - Japan PaaS Support Team Blog](https://jpazpaas.github.io/blog/2020/09/23/how-to-export-appservice-certificate.html)

上記の記事とは別の方法として、 2022 年 9 月現在、Application Gateway SKU v2 をご利用いただくことで Key Vault に格納された App Service 証明書をエクスポートせずに Application Gateway に直接紐づけることが可能となっております。

Application Gateway と App Service 証明書を直接紐づけることにより、Application Gateway は Key Vault に格納された最新の App Service 証明書を定期的に取得することで、 Application Gateway において App Service 証明書の更新作業が不要となります。また、セキュリティの観点でも Application Gateway を直接紐づけることが推奨されます。

<br>

## 背景

App Service 証明書の有効期限は既定で 1 年のため、App Service 証明書を継続して使用していただくためには定期的に更新が必要となります。

参考 : [Azure App Service で TLS/SSL 証明書を追加する - Azure App Service &#124; Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate?tabs=apex%2Cportal#renew-app-service-certificate)


App Service 証明書をエクスポートする方法では、App Service 証明書の更新の度にお客さまにて App Service 証明書のエクスポートならびに Application Gateway へのインポートが必要となります。

一方で直接 Application Gateway に App Service 証明書を紐づける方法では、Application Gateway が Key Vault に格納された最新の App Service 証明書を自動で定期的に取得するため、 お客さまにて手動のエクスポートならびにインポート作業が不要となります。

<br>

## Application Gateway のリスナーに Key Vault に格納された App Service 証明書を表示させる手順

Application Gateway と Key Vault に格納された App Service 証明書を直接紐づけるためには、Application Gateway のリスナーにおいて証明書を選択する必要があります。
しかし、デフォルトでは下図のように "リスナーの追加" ページにおいて App Service 証明書を選択することができません。

![step0b-4dd7636f-21ac-46e6-ac1b-1caf47b3f264.png]({{site.baseurl}}/media/2022/09/step0b-4dd7636f-21ac-46e6-ac1b-1caf47b3f264.png)

そこで Application Gateway の "リスナーの追加" ページにおいて、Key Vault に格納された App Service 証明書を表示させる手順をご紹介いたします。

主な手順は以下の 5 ステップとなっています。

* [ステップ 1. ユーザーマネージド ID を作成する](#step1)
* [ステップ 2. ユーザーマネージド ID を Key Vault に委任する](#step2)
* [ステップ 3. Key Vault に対するファイアーウォールのアクセス許可を確認する](#step3)
* [ステップ 4. PowerShell を用いてキーコンテナーを参照する](#step4)
* [ステップ 5. Application Gateway のリスナーに App Service 証明書が表示されていことを確認する](#step5)

### <a id="step1"> ステップ 1. ユーザーマネージド ID を作成する </a>

新しいユーザーマネージド ID を作成します。既存のユーザーマネージド ID を再利用する場合は [ステップ 1-6](#step1-6) まで飛ばします。

ステップ 1-1. Azure Portal の検索ボックスに "マネージド ID" と入力し、サービス下の "マネージド ID" を選択します。

ステップ 1-2.  [+ 作成] を選択します。

![step1-2-88b7122e-9837-40e2-bcbb-cd5282305cb3.png]({{site.baseurl}}/media/2022/09/step1-2-88b7122e-9837-40e2-bcbb-cd5282305cb3.png)

<br>

<a id="step1-3"> ステップ 1-3. </a> "サブスクリプション" "リソースグループ" "リージョン" を選択します。 "名前" はユーザーマネージド ID に対する任意の名前をつけます。またこのユーザーマネージド ID 名は [ステップ 4-5](#step4-5) で使用します。

ステップ 1-4. [確認および作成] を選択し内容の確認を行います。

![step1-4-96656a58-a947-49c2-8e4e-dce83c220b6a.png]({{site.baseurl}}/media/2022/09/step1-4-96656a58-a947-49c2-8e4e-dce83c220b6a.png)

<br>

ステップ 1-5. 内容の確認後、[作成] を選択します。

<a id="step1-6"> ステップ 1-6. </a> 作成したユーザーマネージド ID の [概要] ブレードを選択し、"基本" セクションに表示される "オブジェクト (プリンシパル) ID" を確認します。この ID は [ステップ 2-5](#step2-5) で利用します。

![step1-6-f8c8e52c-6ce2-4e01-97d7-e7b93ef8b54a.png]({{site.baseurl}}/media/2022/09/step1-6-f8c8e52c-6ce2-4e01-97d7-e7b93ef8b54a.png)

<br>

### <a id="step2"> ステップ 2. ユーザーマネージド ID を Key Vault に委任する </a>

ステップ 2-1. Azure Portal の検索ボックスに "キー コンテナ―" と入力し、サービス下の "キー コンテナ―" を選択します。

ステップ 2-2. App Service 証明書が格納されたキーコンテナーを選択します。

ステップ 2-3. ページ左側の [アクセスポリシー] ブレードを選択した後に、[+ 作成] を選択します。

![step2-3-cded26fb-626c-45d6-ba9e-55d42e9fbd60.png]({{site.baseurl}}/media/2022/09/step2-3-cded26fb-626c-45d6-ba9e-55d42e9fbd60.png)

<br>

ステップ 2-4. "アクセス許可" において "シークレットのアクセス許可" の "取得" にチェックをつけ、[次へ] を選択します。

![step2-4-d08fa805-ac53-4b8f-bd3b-2b6b9171fc48.png]({{site.baseurl}}/media/2022/09/step2-4-d08fa805-ac53-4b8f-bd3b-2b6b9171fc48.png)

<br>

<a id="step2-5"> ステップ 2-5. </a> "プリンシパル" において [ステップ 1-6](#step1-6) で確認した "オブジェクト (プリンシパル) ID" を検索バーに入力し、表示されるユーザーマネージド ID を選択します。ページ下の "選択された項目" にマネージド ID が表示されていることを確認後、[次へ] を選択します。

![step2-5-f41849b6-d13c-4127-9baa-d9f05bdb67fc.png]({{site.baseurl}}/media/2022/09/step2-5-f41849b6-d13c-4127-9baa-d9f05bdb67fc.png)

<br>

ステップ 2-6. "アプリケーション (省略可能)" において [次へ] を選択します。

ステップ 2-7. "確認および作成" において内容を確認後、[作成] を選択します。

<a id="step2-8"> ステップ 2-8. </a> [シークレット] ブレードを選択し、紐づけたい App Service 証明書の "名前" を確認します。このシークレット名は [ステップ 4-6](#step4-6) で利用します。

![step2-8a-a5ae94ed-502f-43ec-803a-61750b7f46c2.png]({{site.baseurl}}/media/2022/09/step2-8a-a5ae94ed-502f-43ec-803a-61750b7f46c2.png)

* 注意 : ファイアーウォールによりクライアントの IP アドレスがキーコンテナーにアクセスする権限がない場合には、以下のようなエラーが発生する場合があります。この場合、キーコンテナーの [ネットワーク] ブレードから以下 2 点のいずれかの方法をご実施いただくことでエラーが解消することが見込まれます。

  ![step2-8b-2cb5e641-d90f-4c44-ab7b-021baf7221ae.png]({{site.baseurl}}/media/2022/09/step2-8b-2cb5e641-d90f-4c44-ab7b-021baf7221ae.png)

  * 方法 A. "Virtual networks" においてアクセスが許可された仮想ネットワーク内からアクセスする。

    ![step2-8d-64287b30-cf0b-4856-ac4a-7b5b2d63b82d.png]({{site.baseurl}}/media/2022/09/step2-8d-64287b30-cf0b-4856-ac4a-7b5b2d63b82d.png)

  * <a id="error"> 方法 B. </a> "ファイアーウォール" > [+ クライアントの IP アドレス ('10.0.0.0') を追加します] を選択し、コマンドを実行しているクライアント PC の IP アドレスをファイアーウォールで許可する。

    ![step2-8c-b81ba30c-061c-4f2b-ac93-c54328db8b8b.png]({{site.baseurl}}/media/2022/09/step2-8c-b81ba30c-061c-4f2b-ac93-c54328db8b8b.png)

<br>

### <a id="step3"> ステップ 3. Key Vault に対するファイアーウォールのアクセス許可を確認する </a>

ステップ 3-1. Azure Portal の検索ボックスに "キー コンテナ―" と入力し、サービス下の "キー コンテナ―" を選択します。

ステップ 3-2. App Service 証明書が格納されたキーコンテナーを選択します。

ステップ 3-3. ページ左側の [ネットワーク] ブレードを選択します。

ステップ 3-4. "許可するアクセス元" において "すべてのネットワークからのパブリックアクセスを許可する" にチェックが入っている場合、[ステップ 4](#step4) までスキップします。他方で "特定の仮想ネットワークと IP アドレスからのパブリックアクセスを許可する" にチェックが入っている場合、[+ 仮想ネットワークの追加] > [+ 既存の仮想ネットワークを追加] を選択します。

![step3-4a-2e66578e-57ce-4067-9ad5-40814edf4179.png]({{site.baseurl}}/media/2022/09/step3-4a-2e66578e-57ce-4067-9ad5-40814edf4179.png)

![step3-4b-49f4ca15-e68e-49da-a901-b8ad3ebd8b37.png]({{site.baseurl}}/media/2022/09/step3-4b-49f4ca15-e68e-49da-a901-b8ad3ebd8b37.png)

<br>

ステップ 3-5. "ネットワークを追加する" において "仮想ネットワーク" と "サブネット" では Application Gateway インスタンスの仮想ネットワークとサブネットを選択します。その後、[追加] を選択します。

ステップ 3-6. "信頼された Microsoft サービスがこのファイアウォールをバイパスすることを許可する" にチェックを入れます。

ステップ 3-7. 最後に [適用] を選択します。

![step3-6-d222cede-9061-4949-812f-85e71be68e9a.png]({{site.baseurl}}/media/2022/09/step3-6-d222cede-9061-4949-812f-85e71be68e9a.png)

<br>

### <a id="step4"> ステップ 4. PowerShell を用いてキーコンテナーを参照する </a>

本ステップは Azure Portal の GUI では行うことができず、ARM テンプレート、Bicep、Azure CLI、Azure PowerShell のいずれかで行う必要があります。ここでは Azure Cloud Shell 上で PowerShell を用いてキーコンテナーを参照する方法を紹介いたします。

ステップ 4-1. https://shell.azure.com にアクセスします。

ステップ 4-2. 以下のコマンドを実行し、Azure にログインします。指示に従いログインを行ってください。

```powershell
Connect-AzAccount -UseDeviceAuthentication
``` 

ステップ 4-3. 以下のコマンドを実行し、対象サブスクリプション ID を選択します。ここで `Select-AzSubscription -SubscriptionId "12345678-1234-1234-1234-123456789012"` のように サブスクリプション ID はダブルクォーテーションで囲みます。

```powershell
Select-AzSubscription -SubscriptionId "<対象サブスクリプション ID>"
``` 

ステップ 4-4. 以下のコマンドを実行し、Application Gateway を取得します。ここで <Application Gateway 名> と <リソースグループ名> は 対象の Application Gateway の [概要] ブレードから確認ができます。

```powershell
$appgw = Get-AzApplicationGateway -Name "<Application Gateway 名>" -ResourceGroupName "<リソースグループ名>"
``` 

![step4-4-a63a78e7-0d57-4d36-bd47-33c6321cad0d.png]({{site.baseurl}}/media/2022/09/step4-4-a63a78e7-0d57-4d36-bd47-33c6321cad0d.png)

<br>

<a id="step4-6"> ステップ 4-5. </a> 以下のコマンドを実行し、Application Gateway に マネージド ID を付与します。ここでコマンド中の {} で囲まれた文字列を以下の内容に置換してからコマンドを実行します。

* {対象サブスクリプション ID} : マネージド ID の対象サブスクリプション ID
* {リソースグループ名} : マネージド ID のリソースグループ名
* {マネージド ID 名} : [ステップ 1-3](#step1-3) で作成したユーザーマネージド ID 名

```powershell
Set-AzApplicationGatewayIdentity -ApplicationGateway $appgw -UserAssignedIdentityId "/subscriptions/{対象サブスクリプション ID}/resourceGroups/{リソースグループ名}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{マネージド ID 名}"
``` 

<a id="step4-6"> ステップ 4-6. </a> 以下のコマンドを実行し、Key Vault からシークレットを取得します。ここでコマンド中の <> で囲まれた文字列を以下の内容に置換してからコマンドを実行します。

* <キーコンテナー名> : App Service 証明書が格納されたキーコンテナー名
* <証明書のシークレット名> : [ステップ 2-8](#step2-8) で確認したシークレットの名前  

```powershell
$secret = Get-AzKeyVaultSecret -VaultName "<キーコンテナー名>" -Name "<証明書のシークレット名>"
``` 

* 注意 : `Get-AzKeyVaultSecret: Operation returned an invalid status code 'Forbidden'` というエラーが発生した場合、エラー文に表示される "Client Address" をキーコンテナーのファイアーウォールで許可することでエラーが解消することが見込まれます。 (ステップ 2-8 の [方法 B](#error) を参照)

![step4-6a-9b2ae63f-abf5-45b7-9d9f-422be5614e02.png]({{site.baseurl}}/media/2022/09/step4-6a-9b2ae63f-abf5-45b7-9d9f-422be5614e02.png)

<br>

ステップ 4-7. 以下のコマンドを順に実行し、Application Gateway に Key Vault に格納された App Service 証明書を追加します。

```powershell
$secretId = $secret.Id.Replace($secret.Version, "") 

Add-AzApplicationGatewaySslCertificate -KeyVaultSecretId $secretId -ApplicationGateway $appgw -Name $secret.Name
``` 

ステップ 4-8. 以下を実行し、Application Gateway を更新します。

```powershell
Set-AzApplicationGateway -ApplicationGateway $appgw
``` 

<br>

### <a id="step5"> ステップ 5. Application Gateway のリスナーに App Service 証明書が表示されていことを確認する </a>

ステップ 5-1. Azure Portal の検索ボックスに "アプリケーション ゲートウェイ" と入力し、サービス下の "アプリケーション ゲートウェイ" を選択します。

ステップ 5-2. App Service 証明書 を紐づけたい Application Gateway を選択します。

ステップ 5-3. ページ左側の [リスナー] ブレードを選択し、[+ リスナーの追加] を選択します。

ステップ 5-4. プロトコルにおいて "HTTPS" にチェックをいれます。また "証明書の選択" を "既存のものを選択" にチェックを入れます。すると "証明書" のプルダウンにおいて App Service 証明書を選択することが可能であることを確認できます。

![stepA-4-bede46d8-45d3-4222-80c1-d424427bda82.png]({{site.baseurl}}/media/2022/09/stepA-4-bede46d8-45d3-4222-80c1-d424427bda82.png)

<br>

##  Application Gateway において App Service 証明書の更新作業が不要となる理由

Application Gateway と App Service 証明書を直接紐づけることで、更新された App Service 証明書が Key Vault に格納されていた場合、Application Gateway に関連付けられている App Service 証明書も自動的に更新された証明書に置き換えられます。
これは Application Gateway が 4 時間ごとに Key Vault を参照し、Key Vault に格納された App Service 証明書の更新バージョンを取得するためです。
この際、Application Gateway が Key Vault を参照するためには、Key Vault のアクセスポリシーにおいて Application Gateway によるアクセスを許可する必要があります。
ここで、アクセスポリシーでは特定のユーザーマネージド ID が Key Vault に対して行うことができる操作 (取得や削除など) を設定します。

一方で Key Vault に格納された App Service 証明書をエクスポートした後に Application Gateway にインポートした場合には、Key Vault に格納された App Service 証明書が更新されたとしても、Application Gateway の App Service 証明書は自動で更新されません。
これは、Key Vault からエクスポートされた App Service 証明書は Key Vault に格納された App Service 証明書を直接参照しないためです。

<br>

## 参考ドキュメント

[Application Gateway の統合 &#124; Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/app-service/networking/app-gateway-with-service-endpoints)

[Application Gateway を使用した App Service の構成 &#124; Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/application-gateway/configure-web-app?tabs=customdomain%2Cazure-portal)

[App service 証明書をエクスポートしてパスワードを設定し、他の Azure サービスにインポートする方法 - Japan PaaS Support Team Blog](https://jpazpaas.github.io/blog/2020/09/23/how-to-export-appservice-certificate.html)  

[Azure Application Gateway の証明書を更新する &#124; Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/application-gateway/renew-certificates#certificates-on-azure-key-vault)

[Azure App Service で TLS/SSL 証明書を追加する  &#124; Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate?tabs=apex%2Cportal#renew-app-service-certificate)

[Key Vault 証明書を使用した TLS 終端 &#124; Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/application-gateway/key-vault-certs#configure-application-gateway-listener)

[ユーザー割り当てマネージド ID を作成する &#124; Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp#create-a-user-assigned-managed-identity)

<br>
<br>

---

<br>
<br>

2022 年 09 月 16 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
