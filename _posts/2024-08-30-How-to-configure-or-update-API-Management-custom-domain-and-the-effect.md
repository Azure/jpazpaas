---
title: "API Management カスタム ドメインの設定及び証明書の更新方法とその影響"
author_name: "a-mnanami / yukiarai"
tags:
    - "API Management"
---

---
# はじめに
Azure Cloud で Azure API Management サービス インスタンスを作成すると、Azure によってそれに `azure-api.net` サブドメイン (例: `apim-service-name.azure-api.net`) が割り当てられます。
また、独自のカスタム ドメイン名 (例: `contoso.com`) を使用して API Management エンドポイントを公開することもできます。   
本記事では、Azure Portal を使った Azure API Management のカスタム ドメイン名の設定および証明書の更新方法とその影響について説明します。

ご参考：[Azure API Management インスタンスのカスタム ドメイン名を構成する](https://learn.microsoft.com/ja-jp/azure/api-management/configure-custom-domain?tabs=custom)

---

(目次)  
[1. 前提](#premise)  
[2. カスタム ドメイン設定の作成](#creation)  
[3. 証明書の更新について](#update)  
[4. カスタム ドメイン設定の作成及び証明書の更新時のご注意点](#attention)  
[5. API Management 側でのカスタム ドメインの作成及び証明書の手動更新による影響](#affect)   
[6. よくいただくご質問](#faq)

---
<a id="premise"></a>
# 1. 前提

## 準備
カスタム ドメインを設定する前に、以下をご準備いただく必要がございます。

- API Management インスタンス
- カスタム ドメイン名
- ドメイン証明書
- カスタム ドメインの CNAME レコードの設定 （ご参考： [CNAME レコード](https://learn.microsoft.com/ja-jp/azure/api-management/configure-custom-domain?tabs=custom#cname-record)）

## API Management にて使用できるドメイン証明書の適用方法について
API Management にて使用できるドメイン証明書の適用方法には、以下の2種類がございます。
- サード パーティー プロバイダーから購入したプライベート証明書 (カスタム TLS 証明書) のアップロードによって適用する
- Azure Key Vault で管理される証明書を参照 (インポート) する


### ◆ キー コンテナーへのアクセス許可設定
カスタムドメイン名の割り当ての前に、予め Azure Key Vault のキー コンテナーに対するアクセス許可を API Management のシステム割り当てマネージド ID または ユーザー割り当てマネージド ID に権限を付与する設定が必要となります。    
● ご参考： [Azure API Management でマネージド ID を使用する](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-use-managed-service-identity)  
● ご参考： [キー コンテナーへのアクセスを構成する](https://learn.microsoft.com/ja-jp/azure/api-management/configure-custom-domain?tabs=key-vault#domain-certificate-options)

手順は以下の項目にてご案内しております。  
→ [4. カスタム ドメイン設定の作成及び証明書の更新時のご注意点](#attention)

※※ 注意点 ※※  
Azure Key Vault ファイアウォールが有効化されている場合は、ユーザー割り当てマネージド ID を利用しての キー コンテナー へのアクセスはできませんのでご注意ください。  
 → 代替案といたしまして、システム割り当てマネージド ID をご利用いただき、キーコンテナーのファイアウォール設定内の [ 信頼された Microsoft サービスがこのファイアウォールをバイパスすることを許可する ] ( [ Allow Trusted Microsoft Services to bypass this firewall in KeyVault firewall ] ) を「 有効 」にご設定いただくことにより、キー コンテナー内の証明書へのアクセスが可能となります。
![image-a1db4e53-2023-46ce-9e47-52b81c1f90e4.png]({{site.baseurl}}/media/2024/08/image-a1db4e53-2023-46ce-9e47-52b81c1f90e4.png)

### ◆ Azure Key Vault のキー コンテナー証明書のご使用をお勧めする理由
API Management のセキュリティ向上に役立ちますため、Azure Key Vault のキー コンテナー証明書をご使用いただくことをお勧めいたします。

- キー コンテナーに格納されている証明書は、サービス間で再利用できます
- キー コンテナーに格納されている証明書には、きめ細かいアクセス ポリシーを適用できます
- キー コンテナーで更新された証明書は、設定によりAPI Management で自動的にローテーションされることが可能です。この場合、キー コンテナー内で更新が行われると、4 時間以内に API Management 内の証明書が更新されます。 また、Azure portal または管理 REST API を使用して、証明書を手動で更新することもできます。

● ご参考： [証明書のオプション](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-mutual-certificates-for-clients#certificate-options)  


## API Management エンドポイントについて
カスタム ドメイン名を割り当てることができる API Management エンドポイントについて、現時点では以下をご利用いただけます。
- ゲートウェイ ・・・ 既定値: `<apim-service-name>.azure-api.net`
- 開発者ポータル ・・・ 既定値: `<apim-service-name>.developer.azure-api.net`
- 管理 ・・・ 既定値: `<apim-service-name>.management.azure-api.net`
- 構成 API (v2)・・・規定値: `<apim-service-name>.configuration.azure-api.net`
- SCM ・・・ 既定値: `<apim-service-name>.scm.azure-api.net`

本記事では**ゲートウェイ**を利用した例をご案内いたします。  
● ご参考： [カスタム ドメインのエンドポイント](https://learn.microsoft.com/ja-jp/azure/api-management/configure-custom-domain?tabs=custom#endpoints-for-custom-domains)

## 前提まとめ
- 本記事ではゲートウェイに対し、カスタム ドメイン名の割り当てと、Azure Key Vault の証明書をシステム割り当てマネージド ID を使用して API Management インスタンスから参照 (インポート) して使用する例をご紹介いたします。
- 合わせて、本設定における API Management での影響についてご案内いたします。
- また、本記事では、ドメイン証明書として、自己署名証明書を使用いたします。

---
<a id="creation"></a>
# 2. カスタム ドメイン設定の作成
カスタム ドメイン設定を作成し、 **Azure Key Vault ( キーコンテナー )** 内の **自己署名証明書** を、**システム割り当てマネージド ID** を使ってAPI Management インスタンスから参照 ( インポート ) して使用する設定手順を以下にご案内いたします。

## 手順

( 1 ) Azure portal で API Management インスタンスに移動します。
 
( 2 ) 左側のナビゲーションで [カスタム ドメイン] を選択し、「＋追加」を選択します。
 
( 3 ) 「ゲートウェイ」ブレードで、以下を入力します。

- 型： ゲートウェイ
- ホスト名： ご用意いただいたカスタム ドメイン名
- 証明書： Azure Key Vault で生成した自己署名証明書
- クライアント ID： ご利用のマネージド ID の種類

![image-58a37649-8b10-4e50-8951-32a8a46ae295.png]({{site.baseurl}}/media/2024/08/image-58a37649-8b10-4e50-8951-32a8a46ae295.png)

「証明書の Key Vault ID」項目は「選択」をクリックして以下を選択します。

- サブスクリプション：ご使用のサブスクリプション
- キー コンテナー：Azure Key Vault 内のキー コンテナー名
- 証明書：参照する証明書名

![image-18839b0d-b9e7-4f33-a4d2-da1e59945e9c.png]({{site.baseurl}}/media/2024/08/image-18839b0d-b9e7-4f33-a4d2-da1e59945e9c.png)

( 4 ) 画面下部の「選択」を押下し、「ゲートウェイ」ウィンドウ下部の「追加」をクリックします。

( 5 ) カスタムドメイン内に新しいエンドポイントが作成されたことを確認後、「保存」をクリックすると更新が開始されます。   
※初回の設定時には以下のメッセージが表示されます。「はい」を選択することによってマネージド ID と Azure Key Vault のアクセスポリシーの設定が自動的に行われます。  
![image-f86c9a0a-aba8-4b5c-a937-df77be93ffa9.png]({{site.baseurl}}/media/2024/08/image-f86c9a0a-aba8-4b5c-a937-df77be93ffa9.png) 

( 6 ) 「概要」ブレードからゲートウェイのURLが更新されていることが確認できます。
![image-cc48c396-17ae-4de9-8735-1e5e80e15d16.png]({{site.baseurl}}/media/2024/08/image-cc48c396-17ae-4de9-8735-1e5e80e15d16.png)

● ご参考： [カスタム ドメイン名を設定する - ポータル](https://learn.microsoft.com/ja-jp/azure/api-management/configure-custom-domain?tabs=key-vault#set-a-custom-domain-name---portal)

---
<a id="update"></a>
# 3. 証明書の更新について

## **ドメイン証明書の有効期限は最長 1 年**のため、**1 年に 1 回証明書の更新が必要**です。  
カスタム ドメイン設定の作成時と同様に API Management にてカスタム TLS 証明書をアップロードした場合には、 API Management のカスタム ドメイン設定にて、Azure Key Vault からインポートした場合には Azure Key Vualt にて、それぞれ設定の更新が必要です。

### ◆ カスタム TLS 証明書を登録
「カスタム ドメイン」ブレードから新しい証明書をアップロードし保存します。(カスタム ドメイン設定時と同じ手順を行うことにより、証明書の更新が行われます。)

### ◆ Azure Key Vault で管理
Azure Key Vault で証明書を更新ください。Azure Key Vault では自動更新にも対応しておりますので、自動更新を設定されている方はそのまま設定が反映されるまでお待ちください。
Azure Key Vault での証明書の更新設定後、4 時間程度で API Management 側にも自動的に反映されます。

## Azure Key Vault のキー コンテナー証明書の更新
更新方法は以下のドキュメントをご参照ください。  
● ご参考： [Azure Key Vault の証明書の更新](https://learn.microsoft.com/ja-jp/azure/key-vault/certificates/overview-renew-certificate?tabs=azure-powershell#renew-a-nonintegrated-ca-certificate)

※ Azure Key Vault は、自己署名証明書の自動更新に対応しています。  
● ご参考：  [チュートリアル - Key Vault における証明書の自動ローテーション頻度を更新する](https://learn.microsoft.com/ja-jp/azure/key-vault/certificates/tutorial-rotate-certificates)

### ◆ API Management 側での証明書更新の反映
- Key Vault での証明書の更新設定後、4 時間程度で API Management 側にも自動的に反映されます。
- より短時間での更新の反映が必要な場合は、API Management での証明書の手動更新設定を行うことにより可能となります。(カスタム ドメイン設定時と同じ手順を行うことにより、証明書の更新が行われます。)


※**Azure Key Vault を使用して証明書を管理しそれらを自動更新に設定することにより、API Management のレベルに SLA がある場合 ( つまり Developer レベルを除くすべてのレベル ) は、API Management によってサービスにダウンタイムを発生させることなく、最新バージョンが自動的に取得されます。**


## 更新されていることの確認方法
証明書が更新されているかを確認する方法を以下にご案内いたします。
### ◆ Azure portal で確認する
( 1 ) Azure portal で API Management インスタンス内の左側のナビゲーションで [カスタム ドメイン] を選択します。  
( 2 ) 「カスタム ドメイン」ブレードで、設定されているカスタム ドメインのリストが表示されます。  
( 3 ) 確認したいカスタム ドメインの行の「証明書」欄でこのカスタム ドメインに関連付けられている SSL 証明書の有効期限、拇印 ( フィンガープリント ）を確認できます。  
( 4 ) 証明書の情報が更新された証明書の情報を反映 (有効期限が新しい証明書に基づいて延長されている) していることを確認します。

![image-2b1bb5d8-387a-4903-8984-1df62005a646.png]({{site.baseurl}}/media/2024/08/image-2b1bb5d8-387a-4903-8984-1df62005a646.png)

### ◆ OpenSSL コマンドで確認する
OpenSSL の環境をお持ちの場合は以下のコマンドにて、現在適用中の証明書の発行日及び有効期限をご確認いただけます。  

● 実行する OpenSSL コマンド  
**openssl s_client -showcerts -connect <ホスト名>:443**   
※ コマンド末尾の「443」は HTTPS のポート番号です。  

● 実行結果の抜粋  
※赤枠 → コマンド、黄緑色枠 → 証明書発行日、水色枠 → 証明書有効期限  
![image-17006fab-c5fc-44db-b299-06d183b36f2c.png]({{site.baseurl}}/media/2024/08/image-17006fab-c5fc-44db-b299-06d183b36f2c.png)

---
<a id="attention"></a>
# 4. カスタム ドメイン設定の作成及び証明書の更新時のご注意点
## ( 1 ) システム割り当てマネージド ID / ユーザー割り当てマネージド ID が有効化されていることをご確認ください。

API Management インスタンスは、Microsoft Entra ID によって生成されたマネージド ID によって、Azure Key Vault などの Microsoft Entra で保護された他のリソースに簡単かつ安全にアクセスすることができます。  
**カスタム ドメイン設定の作成や証明書の更新を行う前に、システム割り当てマネージド ID / ユーザー割り当てマネージド ID を有効化しておくことが必要です。**  

● ご参考： [Azure API Management でマネージド ID を使用する](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-use-managed-service-identity)  
● ご参考： [Azure リソースのマネージド ID とは](https://learn.microsoft.com/ja-jp/entra/identity/managed-identities-azure-resources/overview)

● ご参考： (例) システム割り当てマネージド ID の有効化  

![image-442857f6-d5c8-47c6-8d62-be4e770353cf.png]({{site.baseurl}}/media/2024/08/image-442857f6-d5c8-47c6-8d62-be4e770353cf.png)


## ( 2 ) 権限「キー コンテナー証明書ユーザー」を付与する
カスタムドメインを作成及び証明書の更新をする場合は、使用するユーザー割り当てマネージド ID や API Management のリソース の システム割り当てマネージド ID に対して「****キー コンテナー証明書ユーザー****」の権限(ロール)を予め付与します。

● ご参考： (例) システム割り当てマネージド ID に 「キー コンテナー証明書ユーザー」 のロールを付与する手順  

![image-c5efdaee-bead-41fd-b69d-2c45d0991a8d.png]({{site.baseurl}}/media/2024/08/image-c5efdaee-bead-41fd-b69d-2c45d0991a8d.png)

※ 役割の選択項目から「キー コンテナー証明書ユーザー」または「キー コンテナー シークレット ユーザー」を選択してください。  
![image-2e79173d-a611-4d8e-8a40-b1884afad8d0.png]({{site.baseurl}}/media/2024/08/image-2e79173d-a611-4d8e-8a40-b1884afad8d0.png)

### ◆ ユーザー割り当てマネージド ID の設定手順は以下のドキュメントをご覧ください。
● ご参考： [ユーザー割り当てマネージド ID の管理](
https://learn.microsoft.com/ja-jp/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp)

---
<a id="affect"></a>
# 5. API Management 側でのカスタム ドメイン設定の作成及び証明書の手動更新による影響
- 作成・更新には 15 分から 45 分かかります。
- 作成・更新中は管理操作ができず、ロックがかかります。
- 価格レベルが Developer の場合はダウンタイムが発生する可能性があります。

## 補足
価格レベルが Basic 以上の場合はインスタンスを 1 度に 1 つずつ設定を反映するので、有効期限が異なるものが並走することはありません。

---
<a id="faq"></a>
# 6. よくいただくご質問
Azure Key Vault の証明書を使って API Management サービスに対するカスタム ドメインを追加しようとすると、次のようなエラー メッセージが表示されることがあります。  
以下のエラーに対して、必要な設定をご案内します。


## エラーメッセージ 1
`API Management サービスのホスト名を更新できませんでした  
Failed to access KeyVault Secret https://xxxxxxxxxxxx using Managed Service Identity (http://aka.ms/apimmsi) of Api Management service. **Check if Managed Identity of Type: SystemAssigned,** ClientId: xxxxxxxxxx and ObjectId: xxxxxxxxxx has GET permissions on secrets in the KeyVault Access Policies.`

- このエラーが発生する原因： Azure Key Vault のキー コンテナーに対するアクセス許可を API Managementの **システム割り当てマネージドID** に権限を付与する設定がされていない場合にこのエラーが発生します。
- 対応策：本ブログ記事の [4. カスタム ドメイン設定の作成及び証明書の更新時のご注意点](#attention) をご覧いただき、システム割り当てマネージド ID の設定と、ロール ( キー コンテナー証明書ユーザー ) の付与を行ってください。

## エラーメッセージ 2
`API Management サービスのホスト名を更新できませんでした  
Failed to access KeyVault Secret https://xxxxxxxxxxxx using Managed Service Identity ( http://aka.ms/apimmsi) of Api Management service. **Check if Managed Identity of Type: UserAssigned,** ClientId: xxxxxxxxxx and ObjectId: xxxxxxxxxx has GET permissions on secrets in the KeyVault Access Policies. `

- このエラーが発生する原因： Azure Key Vault のキー コンテナーに対するアクセス許可を API Managementの **ユーザー割り当てマネージド ID** に権限を付与する設定がされていない場合にこのエラーが発生します。
- 対応策：以下のドキュメントをご覧いただき、ユーザー割り当てマネージド IDの設定と、ロール (キー コンテナー証明書ユーザー) の付与を行ってください。  
● ご参考： [ユーザー割り当てマネージド ID の管理](
https://learn.microsoft.com/ja-jp/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp)

---

<br>
<br>

2024 年 11 月 05 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>