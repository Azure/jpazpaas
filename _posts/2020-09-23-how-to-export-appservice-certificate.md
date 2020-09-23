---
title: "App service 証明書をエクスポートしてパスワードを設定し、他の Azure サービスにインポートする方法"
author_name: "Kosuke Uo"
tags:
    - App Service 証明書
    - Application Gateway
---

# はじめに
[App Service 証明書](https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate ) は、Azure によって管理される証明書です。

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

![1-1.gif]({{site.baseurl}}/media/2020/09/2020-09-23_1-1-export-cert.gif)

<br>

1-2. エクスポート対象として、現在のバージョンを選択します。

![1-2.gif]({{site.baseurl}}/media/2020/09/2020-09-23_1-2-export-cert.gif)

<br>

1-3. 下部の [証明書としてダウンロード] を選択します。

![1-3.gif]({{site.baseurl}}/media/2020/09/2020-09-23_1-3-export-cert.gif)

<br>

手順２） エクスポートした証明書をローカル環境にインポートする

2-1. 証明書を右クリックし、 [PFX のインストール(I)] をクリックします。

![2-1.gif]({{site.baseurl}}/media/2020/09/2020-09-23_2-1-import-cert.gif)

<br>

2-2. [証明書のインポート ウィザード] が開始されますので、順を追って進みます。保存場所は [現在のユーザー(C)] を指定します。

![2-2.gif]({{site.baseurl}}/media/2020/09/2020-09-23_2-2-import-cert.gif)

<br>

2-3. インポートする証明書ファイルは 1-3 でエクスポートした証明書を指定します。

![2-3.gif]({{site.baseurl}}/media/2020/09/2020-09-23_2-3-import-cert.gif)

<br>

2-4. インポートオプションで、 [このキーをエクスポート可能にする(M)] にチェックを入れます。

![2-4.gif]({{site.baseurl}}/media/2020/09/2020-09-23_2-4-import-cert.gif)

<br>

2-5. [証明書の種類に基づいて、自動的に証明書ストアを選択する (U)] を選択します。

![2-5.gif]({{site.baseurl}}/media/2020/09/2020-09-23_2-5-import-cert.gif)

<br>

2-6. インポートの確認が表示されますので、[完了] をクリックします。以上でインポートは完了です。

![2-6.gif]({{site.baseurl}}/media/2020/09/2020-09-23_2-6-import-cert.gif)

<br>

手順３） ローカル環境でパスワードを設定する。

先ほどインポートした証明書にパスワードを設定して保存 (エクスポート) します。

3-1. [ユーザー証明書の管理] を実行します。スタートメニューより cert で検索しますと、表示されます。

<br>

![3-1.gif]({{site.baseurl}}/media/2020/09/2020-09-23_3-1-export-cert.gif)

<br>

3-2. ローカルの Windows に格納されている証明書の一覧が表示されます。先ほどインポートした証明書は、左側のツリーの [個人 > 証明書] に格納されています。

![3-2.gif]({{site.baseurl}}/media/2020/09/2020-09-23_3-2-export-cert.gif)

<br>

3-3. 対象の証明書を右クリックして、 [すべてのタスク(K)] > [エクスポート(E)] をクリックします。

3-4. 証明書のエクスポート ウィザードが開始されます。

![3-4.gif]({{site.baseurl}}/media/2020/09/2020-09-23_3-4-export-cert.gif)

<br>

3-5. [はい、秘密鍵をエクスポートします(Y)] を選択します。

![3-5.gif]({{site.baseurl}}/media/2020/09/2020-09-23_3-5-export-cert.gif)

<br>

3-6. エクスポートするファイル形式として、[Personal Information Exchange - PKCS#12] を選択します。

![3-6.gif]({{site.baseurl}}/media/2020/09/2020-09-23_3-6-export-cert.gif)

<br>

3-7. パスワードを設定します。任意の文字列で問題ございませんが、ある程度長い文字列で推測が難しいものをお勧めします。

![3-7.gif]({{site.baseurl}}/media/2020/09/2020-09-23_3-7-export-cert.gif)

<br>

3-8. パスワードを付けて、エクスポートした証明書を保存先を指定します。

![3-8.gif]({{site.baseurl}}/media/2020/09/2020-09-23_3-8-export-cert.gif)

<br>

3-9. エクスポートの確認が表示されますので、[完了] をクリックします。

![3-9.gif]({{site.baseurl}}/media/2020/09/2020-09-23_3-9-export-cert.gif)

<br>

以上の手順によって、GUIでパスワード付きの証明書をエクスポートすることができます。

## CLI ( Azure PowerShell )を用いる方法

本操作を行うにあたり、Azure PowerShell のインストール が必要となります。
お客様の環境に、Azure PowerShell がインストールされていない場合、以下のリンクを参考に、インストールをお願いします。

https://docs.microsoft.com/ja-jp/powershell/azure/install-az-ps?view=azps-4.7.0

1. ローカルの環境で、PowerShell を開きます。
2. 以下のスクリプト全体をコピーし、PowerShellにペーストした後、Enterを押します。<br>
以下のスクリプトにより、Export-AppServiceCertificate という新しいコマンドが作成されます。

```
Function Export-AppServiceCertificate
{
Param(
[Parameter(Mandatory=$true,Position=1,HelpMessage="ARM Login Url")]
[string]$loginId,

[Parameter(Mandatory=$true,HelpMessage="Subscription Id")]
[string]$subscriptionId,

[Parameter(Mandatory=$true,HelpMessage="Resource Group Name")]
[string]$resourceGroupName,

[Parameter(Mandatory=$true,HelpMessage="Name of the App Service Certificate Resource")]
[string]$name
)

Login-AzAccount
Set-AzContext -SubscriptionId $subscriptionId

$ascResource= Get-AzResource -ResourceId "/subscriptions/$subscriptionId/resourceGroups/$resourceGroupName/providers/Microsoft.CertificateRegistration/certificateOrders/$name"
$certProps = Get-Member -InputObject $ascResource.Properties.certificates[0] -MemberType NoteProperty
$certificateName = $certProps[0].Name
$keyVaultId = $ascResource.Properties.certificates[0].$certificateName.KeyVaultId
$keyVaultSecretName = $ascResource.Properties.certificates[0].$certificateName.KeyVaultSecretName

$keyVaultIdParts = $keyVaultId.Split("/")
$keyVaultName = $keyVaultIdParts[$keyVaultIdParts.Length - 1]
$keyVaultResourceGroupName = $keyVaultIdParts[$keyVaultIdParts.Length - 5]

Set-AzKeyVaultAccessPolicy -ResourceGroupName $keyVaultResourceGroupName -VaultName $keyVaultName -UserPrincipalName $loginId -PermissionsToSecrets get
Write-Host "Get Secret Access to account $loginId has been granted from the KeyVault, please check and remove the policy after exporting the certificate"

$secret = Get-AzKeyVaultSecret -VaultName $keyVaultName -Name $keyVaultSecretName
$pfxCertObject= New-Object System.Security.Cryptography.X509Certificates.X509Certificate2 -ArgumentList @([Convert]::FromBase64String($secret.SecretValueText),"",[System.Security.Cryptography.X509Certificates.X509KeyStorageFlags]::Exportable)
$pfxPassword = -join ((65..90) + (97..122) + (48..57) | Get-Random -Count 50 | % {[char]$_})
$currentDirectory = (Get-Location -PSProvider FileSystem).ProviderPath
[Environment]::CurrentDirectory = (Get-Location -PSProvider FileSystem).ProviderPath
[io.file]::WriteAllBytes(".\appservicecertificate.pfx",$pfxCertObject.Export([System.Security.Cryptography.X509Certificates.X509ContentType]::Pkcs12,$pfxPassword))

Write-Host "Created an App Service Certificate copy at: $currentDirectory\appservicecertificate.pfx"
Write-Warning "For security reasons, do not store the PFX password. Use it directly from the console as required."
Write-Host "PFX password: $pfxPassword"
}
```
<br>3. 次に、作成した Export-AppServiceCertificate コマンドを実行します。

```powershell
Export-AppServiceCertificate 
-loginId yourarmemail@domain.com 
-subscriptionId yoursubid 
-resourceGroupName resourceGroupNameOfYourAppServiceCertificate 
-name appServiceCertificateName
```
上記コマンドは、以下のようにご変更ください。
- yourarmemail@domain.com：ログインid
- yoursubid：サブスクリプションid
- resourceGroupNameOfYourAppServiceCertificate： App Service 証明書が存在するリソースグループ名
- appServiceCertificateName： App Service 証明書名

コマンドが実行されると、'appservicecertificate.pfx' という名前の新たな証明書がカレントディレクトリにエクスポートされます。

これはパスワードで保護されたPFX形式の証明書で、**PowerShellコンソールには対応するパスワードが生成され、表示されます。**

以上の手順によって、CLI でパスワード付きの PFX 形式証明書をエクスポートすることができます。

## Application Gateway に App Service 証明書をインポートする方法
上記の方法で手に入れたパスワード付きのPFX形式証明書を、Application Gatewayにインポートする方法を紹介します。

<br>1. Azure Portalから、証明書をインポートする Application gateway のページに移動します。
<br>2. 左側のメニューから、リスナーを選択します。
<br>
![4-2.png]({{site.baseurl}}/media/2020/09/2020-09-23_4-2-appgw.png)

<br>3. 証明書をインポートする HTTPS リスナーを選択します。
<br>
![4-3.png]({{site.baseurl}}/media/2020/09/2020-09-23_4-3-appgw.png)

<br>4.  証明書をインポートします
- 「証明書の選択」から、「新規作成」を選択します。
- 「 HTTPS 設定」>「証明書の選択」から、「証明書のアップロード」を選択します。
- 「 PFX 証明書ファイル」に、エクスポートした PFX 形式の証明書をアップロードします。
- 「証明書名」に、任意の名前を記入します。
- 「パスワード」に、設定したパスワードを記入します。
<br>
 ![4-4.png]({{site.baseurl}}/media/2020/09/2020-09-23_4-4-appgw.png)

<br>5. 「保存」を選択します。

以上の手順によって、Application Gateway に証明書をインポートできます。

## 参考リンク
[Creating a local PFX copy of App Service Certificate](https://azure.github.io/AppService/2017/02/24/Creating-a-local-PFX-copy-of-App-Service-Certificate.html)

[ポータルで Application Gateway を使用してエンド ツー エンド TLS を構成する](https://docs.microsoft.com/ja-jp/azure/application-gateway/end-to-end-ssl-portal)

[Application Gateway の証明書関連のトラブルシューティング](https://jpaztech.github.io/blog/network/appgw-troubleshooting-cert)

<br>
<br>

----

<br>
<br>

2020 年 9 月 23 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>