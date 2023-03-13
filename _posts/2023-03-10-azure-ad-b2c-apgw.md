---
title: "Azur AD B2C 認証を設定した App Service と Application Gateway の統合"
author_name: "syukumoto"
tags:
    - App Service
    - Function App
---

# はじめに
お世話になっております。App Service サポート担当の行本です。

App Service に [Azure AD B2C](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/overview) というサービスを連携する事で、好みのソーシャル、エンタープライズ、またはローカル アカウント ID を使用して、アプリケーションに SSO (シングル サインオン) でアクセスする事ができます。<br>
[Azure AD B2C を使って Azure Web アプリで認証を構成する](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/configure-authentication-in-azure-web-app) 方法については既にドキュメントがございますが、この構成の前段に Application Gateway を配置した際の環境構築手順について、お問い合わせをいただく事もございますので本記事で順を追って説明させていただきます。
<br>
なお、本記事ではバックエンドに App Service を用いた手順を記載しておりますが、Azure Functions を用いた場合も同様に環境構築可能です。

## 構成イメージ
![image-9252386f-25d9-4bff-b66b-09481fb5ac88.png]({{site.baseurl}}/media/2023/03/image-9252386f-25d9-4bff-b66b-09481fb5ac88.png)


# 事前準備
事前に、Application Gateway の環境構築に推奨されるカスタムドメインを用意しておきます。
1. カスタムドメイン設定済みの App Service
2. カスタムドメインにバインドした証明書 (本記事では Key Vault に格納した App Service 証明書を用います)

## 参考資料
・[Azure App Service のカスタム ドメイン名を購入する](https://learn.microsoft.com/ja-jp/azure/app-service/manage-custom-dns-buy-domain)<br>
・[App Service の TLS/SSL 証明書の更新手順 - App Service の TLS/SSL 証明書の更新手順](https://jpazpaas.github.io/blog/2023/03/10/AppService-How-to-bind-AppService-Certificate-with-NewUI.html#app-service-%E8%A8%BC%E6%98%8E%E6%9B%B8%E3%81%AE%E3%83%90%E3%82%A4%E3%83%B3%E3%83%89%E6%89%8B%E9%A0%86)

**※注意点**<br>
・「**無料の App Service マネージド証明書**」は、Application Gateway に紐づける事が出来ません。<br>
<br>

# Azure AD B2C 認証の設定
まずは、App Service 単体で Azure AD B2C 認証を設定し問題なく認証が行える事を確認します。Application Gateway と App Service を統合後に Azure AD B2C 認証を設定しようとすると、上手く動作しなかった場合に問題箇所の切り分けが難しくなるため先に App Service 単体で確認する事を推奨いたします。<br>
手順は大きく以下の段階に分かれます。各チュートリアルに従い設定を行ってください。<br>

1. Azure AD B2C テナントの作成<br>
・[チュートリアル:Azure Active Directory B2C テナントの作成](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/tutorial-create-tenant)<br>
2. ユーザーフローの作成<br>
・[チュートリアル: Azure Active Directory B2C でユーザー フローとカスタム ポリシーを作成する](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/tutorial-create-user-flows?pivots=b2c-user-flow)
3. App Service に Azure AD B2C を設定<br>
・[Azure AD B2C を使ってサンプル web アプリケーションで認証を構成する](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/configure-authentication-sample-web-app?tabs=visual-studio)<br><br>
**※注意点**<br>
・上記 2.、3. のユーザーフロー作成や Web アプリケーション登録は Azure AD B2C テナント上で行う必要がありますが、連携する App Service や Application Gateway は Azure AD B2C テナント以外に作成して構いません。<br>
・上記 3. で Web アプリケーションの登録時に指定するリダイレクト URI には、カスタムドメインを指定します。<br>
![image-a69d12f9-03f4-40e7-a3bf-f41d4868f989.png]({{site.baseurl}}/media/2023/03/image-a69d12f9-03f4-40e7-a3bf-f41d4868f989.png)<br><br>
また、App Service 認証を構成する際の ID プロバイダーに指定する「クライアント ID」、「シークレット」もそれぞれお間違えの無い様ご注意ください。
![image-f9b2e1aa-ec68-494f-a79e-820ace58573b.png]({{site.baseurl}}/media/2023/03/image-f9b2e1aa-ec68-494f-a79e-820ace58573b.png)

Azure AD B2C 認証設定後、カスタムドメインにアクセスし想定通り認証が通る事を確認します。確認出来たら、次のステップに進みます。

<br>

# Application Gateway の環境構築
いよいよ Application Gateway の環境構築に進みます。Application Gateway の環境構築は大きく以下の段階に分かれます。

1. Application Gateway の作成
2. Application Gateway へのカスタムドメイン設定
3. Application Gateway への証明書設定
4. バックエンドの設定

設定項目の詳細については[Application Gateway を使用した App Service の構成](https://learn.microsoft.com/ja-jp/azure/application-gateway/configure-web-app?tabs=customdomain%2Cazure-portal)を参照いただくのがよいですが、本記事では必要な手順をかい摘んで説明させていただきます。

## 1. Application Gateway の作成
1. [クイック スタート:Azure Application Gateway による Web トラフィックのルーティング - Azure portal](https://learn.microsoft.com/ja-jp/azure/application-gateway/quick-create-portal)に従い、バックエンドターゲットを持たない Application Gateway を作成します。リンク先の手順末尾にある「**バックエンド ターゲットの追加**」以降の手順は実施不要です。<br>
<br>
**※注意点**<br>
・App Service と Application Gateway に関連付けるカスタムドメインの証明書として Key Vault に格納した App Service 証明書を使う場合、Application Gateway の作成時点では証明書を参照する事が出来ません。これによりリスナーを HTTPS で作成時に証明書が指定出来ないため、一旦 **HTTP** でリスナーを作成します。<br>
<IMG  src="https://learn.microsoft.com/ja-jp/azure/application-gateway/media/application-gateway-create-gateway-portal/application-gateway-create-rule-listener.png"  alt="新しいアプリケーション ゲートウェイの作成: リスナー"/>
<br><br>
・ネットワーク セキュリティ グループ (NSG) を使用する場合、Application Gateway v2 SKU では以下のセキュリティ規則は許可する必要があります。詳しくは [Application Gateway インフラストラクチャの構成 - ネットワーク セキュリティ グループ](https://learn.microsoft.com/ja-jp/azure/application-gateway/configuration-infrastructure#network-security-groups) を参照ください。<br><br>
　**受信セキュリティ規則:**<br>
　・[ソース] **サービスタグ GatewayManager**、[宛先] **Any**、[サービス] **Custom**、[宛先ポート範囲] **65200-65535**<br>
　・[ソース] **サービスタグ AzureLoadBalancer**、[宛先] **Any**、[サービス] **Custom**、[宛先ポート範囲] *<br>
<br>
　**送信セキュリティ規則:**<br>
　・既定のルールを削除しないようにします。また、アウトバウンド接続を拒否する他のアウトバウンド規則は作成しないようにします。<br>


## 2. Application Gateway へのカスタムドメイン設定
事前準備の段階では、用意したカスタムドメインは App Service に関連付けられています。同じカスタムドメインにアクセスした際に Application Gateway を参照する様に設定を行います。<br>
1. Application Gateway の [概要] ブレードを開き、**[フロントエンド パブリック IP アドレス]** のリンクを選択します。
![image-fd9158e6-94e0-4afc-9c93-7e0535670f01.png]({{site.baseurl}}/media/2023/03/image-fd9158e6-94e0-4afc-9c93-7e0535670f01.png)

2. パブリック IP アドレスの [概要] ブレードから **[IP アドレス]**、**[DNS 名]** の値を確認します。
![image-0789983a-0e58-43d9-901b-8f786ff60f78.png]({{site.baseurl}}/media/2023/03/image-0789983a-0e58-43d9-901b-8f786ff60f78.png)
3. カスタムドメインの DNS ゾーンを開き、A レコードの値を手順 2. の **[IP アドレス]**、CNAME レコードの値を手順 2. の **[DNS 名]** に変更します。
![image-24ed504f-f4de-4a41-ab7f-6ad5ddaac0aa.png]({{site.baseurl}}/media/2023/03/image-24ed504f-f4de-4a41-ab7f-6ad5ddaac0aa.png)

## 3. Application Gateway への証明書設定
設定したカスタムドメインへアクセス出来るよう、証明書を設定します。
1. [Application Gateway のリスナーに Key Vault に格納された App Service 証明書を表示させる方法](https://jpazpaas.github.io/blog/2022/09/16/How-to-import-ASC-to-AppGW.html)に従い Key Vault に格納した App Service 証明書を設定します。<br>

## 4. バックエンドの設定
最後に、Application Gateway からバックエンドまでの通信を HTTPS で行うよう設定します。
1. Application Gateway の [正常性プローブ] ブレードを開き、[+追加] ボタンを選択します。
![image-b06b9c16-5293-4402-9b13-d78d9169ebd6.png]({{site.baseurl}}/media/2023/03/image-b06b9c16-5293-4402-9b13-d78d9169ebd6.png)
<br><br>
[プロトコル] に **HTTPS**、[ホスト名] にカスタムドメインを指定します。<br>
[パス] にはリクエストに対し HTTP Status 200-399 が返るパスを指定します。Azure AD B2C の認証を設定すると通常 401 などの応答が返り、正常応答と見做されません。<br>
この対策としては**認証対象外のパス (excludedPaths)** を設定いただくか、**Azure AD B2C のリダイレクト URI として設定したパスを指定する**方法がございます。(リダイレクト URI へのリクエストに対しては 200 応答が返るため)<br>
**認証対象外のパス (excludedPaths)** を設定する方法については、弊社エンジニアが執筆した記事 [Azure App Service の Authentication/Authorization(EasyAuth)で認証しないURLを設定する](https://qiita.com/takashiuesaka/items/57db6c1600240408cd03) をご参照ください。<br>

2. [テスト] ボタンを選択し、バックエンドの正常性が確認できたら保存します。<br>
![image-0a39d219-1b2c-47f5-bbc8-1ee3e68443db.png]({{site.baseurl}}/media/2023/03/image-0a39d219-1b2c-47f5-bbc8-1ee3e68443db.png)
3. [バックエンド設定] ブレードを選択し、作成済みバックエンド設定を選択します。<br>
[バックエンド プロトコル] を **HTTPS**、[既知の CA 証明書を使用する] を **はい**、[新しいホスト名でオーバーライドする] を **いいえ**、に指定します。[カスタム プローブ] に先ほど作成した正常性プローブを指定して保存します。
![image-a13bb800-f720-4930-af83-516852cfa479.png]({{site.baseurl}}/media/2023/03/image-a13bb800-f720-4930-af83-516852cfa479.png)

以上で Azur AD B2C 認証を設定した App Service と Application Gateway の統合は完了です。<br>
<br>

# Appendix
バックエンド正常性に関する問題が発生した場合は、以下のトラブルシューティングガイドを参照していただけますと幸いです。<br>

[Application Gateway のバックエンドの正常性に関する問題のトラブルシューティング](https://learn.microsoft.com/ja-jp/azure/application-gateway/application-gateway-backend-health-troubleshooting)

<br>
<br>

---

<br>
<br>

2023 年 03 月 13 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>