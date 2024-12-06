---
title: "Windows 版の App Service にインストールされている既定のルート証明書"
author_name: "Hidenori Yatsu"
tags:
     - App Service
---
    
# はじめに

App Serviceサポート担当の谷津です。
この投稿では、Windows 版の App Service にて既定でインストールされるルート証明書について解説いたします。

# インストールされているルート証明書の確認方法
以下の弊社公式ブログならびに記事内で引用されている公式ドキュメントにて言及があるように、Windows の場合は PowerShell にて `dir cert:\localmachine\root` を実行することで確認可能です。PowerShell コマンドは Kudu サイト (高度なツール) にて以下のように実行いただけます。

[Root CA on App Service - Azure App Service](https://azure.github.io/AppService/2021/06/22/Root-CA-on-App-Service-Guide.html)

![image-fadb6680-592a-4e53-8a37-3cab57632743.png]({{site.baseurl}}/media/2024/12/image-fadb6680-592a-4e53-8a37-3cab57632743.png)

# 既定でインストールされているルート証明書がどのように決定されるか

Windows のみならず、既定でインストールされている証明書については開発元のセキュリティポリシーや OS のセキュリティ要件に依存します。各 OS の開発元はそれぞれ異なる基準でルート証明書を評価し、各々のセキュリティポリシーに基づいて信頼するルート証明書を選定します。また、各 OS はそれぞれ異なるセキュリティ要件があるため、各 OS のセキュリティ要件（特定の業界標準や規制など）に応じて信頼するルート証明書が選定されます。弊社の信頼するルート証明書における具体的な要件については以下の公開情報にて詳細な解説がございますので、こちらをご参照ください。

[プログラム要件 - Microsoft の信頼できるルートプログラム - Microsoft Learn](https://learn.microsoft.com/ja-jp/security/trusted-root/program-requirements)

また、弊社にて信頼しているルート証明書の一覧は以下の公開情報にて確認が可能です。

[参加者リスト - Microsoft の信頼されたルート プログラム - Microsoft Learn](https://learn.microsoft.com/ja-jp/security/trusted-root/participants-list)

[ccadb.my.salesforce-sites.com/microsoft/IncludedCACertificateReportForMSFT](https://ccadb.my.salesforce-sites.com/microsoft/IncludedCACertificateReportForMSFT)

---

**重要**

ただし、上記全てのルート証明書が既定で Windows の証明書ストアにインストールされている訳ではございません。既定でインストールされているルート証明書は最小構成となっております。App Service 上から実行されたリクエストが信頼された証明機関からの証明書を持つリモートエンドポイントに到達すると、ルート証明書の自動更新コンポーネントは Microsoft Windows Update Web に接続してルート証明書をダウンロードし、信頼されたルート証明機関の証明書ストアに動的にインストールします。こちらの挙動は App Service 特有のものではなく、Windows OS の挙動に依存するものでございます。以下の公開情報の内容もご参考になりますと幸いです。

[Event ID 8 — Automatic Root Certificates Update Configuration | Microsoft Learn](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-server-2008-r2-and-2008/cc734054(v=ws.10))
> The Automatic Root Certificates Update component is designed to automatically check the list of trusted authorities on the Microsoft Windows Update Web site. Specifically, there is a list of trusted root certification authorities (CAs) stored on the local computer. When an application is presented with a certificate issued by a CA, it will check the local copy of the trusted root CA list. If the certificate is not in the list, the Automatic Root Certificates Update component will contact the Microsoft Windows Update Web site to see if an update is available. If the CA has been added to the Microsoft list of trusted CAs, its certificate will automatically be added to the trusted certificate store on the computer.

# App Service に任意のルート証明書をインストールしたい場合

前述のとおり、 Microsoft の信頼できるルートプログラムによって選定されたルート証明書であればユーザーが自身でインストールを行う必要はありません。もしそれ以外のルート証明書をご自身でインストールしたい場合はマルチテナントの App Service では対応していないため、専有環境である App Service Environment をご利用いただく必要があります。インストール手順の詳細は以下の公開情報にて解説がございますので、こちらのドキュメントがお役に立ちますと幸いでございます。

[証明書のバインド - Azure App Service Environment - Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/app-service/environment/certificates#private-client-certificate)

もしくは Windows コンテナを利用する Web App for Containers (カスタムコンテナ) をご利用いただくことでもユーザーが自由に Windows の証明書ストアをカスタマイズいただけますので、こちらの案もご検討くださいませ。

<br>
<br>
    
---
    
<br>
<br>
    
2024 年 12 月 06 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。
    
<br>
<br>