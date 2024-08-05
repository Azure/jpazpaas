---
title: "API Management で Azure AD B2C を利用した認証の構成手順について"
author_name: "yiwenchen"
tags:
    - "API Management"
---
こんにちは、PaaS Developer チームの陳です。

API Management（以下 APIM）の資格情報マネージャーを構成して、バックエンドを認証する方法について、Entra ID をプロバイダとして構成する方法を以下のドキュメントに掲載されていますが、

[資格情報マネージャーの構成 - Microsoft Graph API](https://learn.microsoft.com/ja-jp/azure/api-management/credentials-how-to-azure-ad)

Azure AD B2C を使用して構成する方法はないため、今回は APIM で Azure AD B2C を用いた バックエンド認証の構成手順についてまとめます。


# 質問
APIM での バックエンド認証には、Entra ID ではなく、Azure AD B2C を使用する予定ですが、Credential Manager とポリシーの構成方法について知りたい。

# 回答
本ブログは以下の流れで説明していきます。

- [Azure AD B2C の構成手順](#azureAD)
- [APIM Credential Manager の構成手順](#APIM)
- [ポリシーの構成手順](#policy)

## Azure AD B2C の構成手順 <a id="azureAD"></a>

### 1. Azure AD B2C テナントを作成し、左側メニューの「アプリの登録」でアプリを作成します <a id="step1"></a>

Azure AD B2C テナントの作成方法　<br>
https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/tutorial-create-tenant?WT.mc_id=Portal-Microsoft_AAD_B2CAdmin

アプリの作成
![image-68adc031-108b-4f6d-8770-563ab7f9dac9.png]({{site.baseurl}}/media/2024/08/image-68adc031-108b-4f6d-8770-563ab7f9dac9.png)

![image-f7c18ab3-6f8d-4912-ba84-7b83408677b2.png]({{site.baseurl}}/media/2024/08/image-f7c18ab3-6f8d-4912-ba84-7b83408677b2.png)

リダイレクト URL は一旦空のままで大丈夫ですが、のちほど APIM での設定において認証のためのリダイレクト URL が指定されるため、その際に入力します。

### 2. 作成したアプリで各種設定をします <a id="step2"></a>


「 証明書とシークレット 」でシークレットを作成し、値をメモします。（作成後には再び参照できないため、作成時に値のメモを忘れなく）

![image-41ef0856-7343-44ba-83f5-ca9288c48541.png]({{site.baseurl}}/media/2024/08/image-41ef0856-7343-44ba-83f5-ca9288c48541.png)

「 API の公開 」でアプリケーション ID URL を作成し、スコープの追加でスコープを追加します。

![image-5b3ea4e6-7b23-4d75-bbd6-7b37f440c981.png]({{site.baseurl}}/media/2024/08/image-5b3ea4e6-7b23-4d75-bbd6-7b37f440c981.png)

### 3. ユーザーフローの定義 <a id="step3"></a>

ここでは具体的にどのような認証方法を採用するかを設定できます。複数のユーザーフローを定義できるため、下記 4 の Authorization URL に適用したいユーザーフローポリシー名を含めることで、どのポリシーを適用するかを設定できます。後ほど述べる APIM での Credential Manager の構成において、最後に構成した接続を認証する際にユーザーフローの動作を確認できます。
ここでは、例として MFA を使用してメールか電話番号の追加認証を導入するユーザーフローを定義してみます。

![image-b6da5cc2-dcf1-4613-bd9b-57e102bf969e.png]({{site.baseurl}}/media/2024/08/image-b6da5cc2-dcf1-4613-bd9b-57e102bf969e.png)

ほかにもトークン内に含める情報（ issuer 、sub など）やトークンの有効期間などを編集できるため、実際の運用要件に応じて設定していただければと思います。

### 4. Azure AD B2C で認証するためのユーザーを作成します <a id="step4"></a>


後述する Credential Manager の接続認証において、ユーザーとして一回ログインする必要があるため、ここで予めユーザーを作成しておきます。

添付のスクリーンショットのように、Azure AD B2C の 「 ユーザー 」から新しいユーザーをクリックし、ユーザープリンシパルとパスワードを設定します。パスワードは作成後に再び参照できないため、事前にメモしてください。

![image-1a32bc41-538a-4d4b-9716-483869ad9656.png]({{site.baseurl}}/media/2024/08/image-1a32bc41-538a-4d4b-9716-483869ad9656.png)

![image-6aa46481-134f-47f5-a7a8-18bc139ed8b0.png]({{site.baseurl}}/media/2024/08/image-6aa46481-134f-47f5-a7a8-18bc139ed8b0.png)

### 5. APIM の Credential Manager を構成するための情報を整理します <a id="step5"></a>

| 名前           | 説明                                                                 |
|----------------|----------------------------------------------------------------------|
| Authorization URL | https://<tenant name>.b2clogin.com/<tenant name>.onmicrosoft.com/<userflow-policy-name>/oauth2/v2.0/authorize |
| Client ID      | アプリの概要から確認できます                                        |
| Client Secret  | 作成したシークレット値                                               |
| Token URL      | https://<tenant name>.b2clogin.com/<tenant name>.onmicrosoft.com/<userflow-policy-name>/oauth2/v2.0/token |
| Scopes         | 作成したスコープのURL                                               |



1. userflow-policy-name は [手順3](#step3) で作成したユーザーフローの名前を入力。
2. Client ID は [手順1](#step1) で作成したアプリの概要で確認できます。
3. Client Screct は [手順2](#step2) でメモしたシークレット値を入力。
4. Scopes は [手順2](#step2) で作成したスコープをそのままコピペする。

## APIM Credential Manager の構成手順 <a id="APIM"></a>

APIM の Credential Manager で作成を押したら以下の画面となり、さきほど整理した情報を各項目に入力します。

今回は Entra ID ではなく、Azure AD B2C での認証のため、ID プロバイダを OAuth2.0 にします。
![image-fdb89b48-dae7-4924-a0f8-55c7a78490b0.png]({{site.baseurl}}/media/2024/08/image-fdb89b48-dae7-4924-a0f8-55c7a78490b0.png)

作成したら、APIM のリダイレクト URL を教えてくれますので、その URL をアプリのリダイレクト URL に設定します。
![image-0f9ef8e2-086c-4f56-bb62-daaca5e19c02.png]({{site.baseurl}}/media/2024/08/image-0f9ef8e2-086c-4f56-bb62-daaca5e19c02.png)

設定が終わりましたら、作成した Credential Manager の Connections メニューで Connections を作成します。
作成後の画面の OAuth2.0でのログイン、もしくは下の画面でログインボタンをクリックし、定義した認証方法で状態が Connected になったかをご確認ください。

![image-6d985baa-8ac8-42b3-ae92-28f16ece6240.png]({{site.baseurl}}/media/2024/08/image-6d985baa-8ac8-42b3-ae92-28f16ece6240.png)

今回は例として MFA の追加認証を設定したため、さきほど作成したユーザーでログインし、追加認証があるかどうかを確認します。

![image-27a44738-9e70-40b2-bf3f-097b5330ce9f.png]({{site.baseurl}}/media/2024/08/image-27a44738-9e70-40b2-bf3f-097b5330ce9f.png)

ログイン出来たら、下記の図で確認できるように電話番号は求められているため、ユーザーフローポリシーは正しく動作していると分かります。

電話番号で認証できたら、Credential Manager の接続状態が Connected となり、認証が成功となります。

![image-98912c52-0c0b-4750-b69e-0ebb16836ab5.png]({{site.baseurl}}/media/2024/08/image-98912c52-0c0b-4750-b69e-0ebb16836ab5.png)

以上で Credential Manager の構成は完了ですので、最後は APIM ポリシーの構成となります。

## ポリシーの構成手順<a id="policy"></a>
今回使用するポリシーは「 get-authorization-context 」となります。
認証の流れとしては、まずポリシー内で指定した Credential Manager に構成した authentication URL を通じてクライアント認証情報で承認コードをもらいます。

それから、承認コードを使用して Token URL でアクセストークンをもらいます。それ以降は要件に応じてトークンをヘッダーに添付して認証するなどのプロセスを各自で構築できます。

ポリシーの構文例を以下に示します。

```
    <inbound>
        <base />
        <get-authorization-context provider-id="b2ctest" authorization-id="b2ctest" context-variable-name="auth-context" identity-type="managed" ignore-error="false" />
        <!-- Return the token -->
        <return-response>
            <set-status code="200" />
            <set-body template="none">@(((Authorization)context.Variables.GetValueOrDefault("auth-context"))?.AccessToken)</set-body>
        </return-response>
    </inbound>
```

ここで、注意点として identity-type を jwt ではなく、managed にすることです。

理由としては、Azure AD B2C を使用して取得したトークンには tenant id が含まれていないため、jwt トークンとして認証してしまうと tenant id error となります。

https://learn.microsoft.com/ja-jp/azure/api-management/get-authorization-context-policy





# 参考ドキュメント
https://learn.microsoft.com/ja-jp/azure/api-management/credentials-how-to-azure-ad

https://qiita.com/makotsur/items/cc42fda61d048d94384b

<br>
<br>

---

<br>
<br>

2024 年 3 月 29 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>