---
title: "App Service の TLS/SSL 証明書の更新手順"
author_name: "Masafumi Kokui"
tags:
    - App Service
---

# はじめに
これまで App Service に存在していた "Custom domains (Classic)", "TLS/SSL settings (Classic)" ブレードが 2023 年 2 月半ば頃に非表示となりました。これに伴い "Custom domains (Classic)" は "Custom domains" ブレードに、"TLS/SSL settings (Classic)" は "Certificates" ブレードに移行しています。公式ドキュメントの更新が遅れておりますため、特に変更点の多い "TLS/SSL Settings (Classic)" から "Certificats" ブレードの変更点を中心として、新しい UI で App Service にバインドされた TLS/SSL 証明書の更新方法をご案内します。
 ![image-fd4f4dbf-15af-4bae-9887-350a5aa3fe28.png]({{site.baseurl}}/media/2023/03/image-fd4f4dbf-15af-4bae-9887-350a5aa3fe28.png)  

# "TLS/SSL settings (Classic)" から "Certificates" ブレードへの変更点
以下は新旧の画面を比較したものです。

### 以前の画面 - TLS/SSL Settings(Classic)
 ![image-7a1f247c-628c-4758-a9ff-a11bf1f12e09.png]({{site.baseurl}}/media/2023/03/image-7a1f247c-628c-4758-a9ff-a11bf1f12e09.png) 

### 新しい画面 - Certificates
 ![image-c00c3f8b-7dbc-427f-ae17-073a1987eebf.png]({{site.baseurl}}/media/2023/03/image-c00c3f8b-7dbc-427f-ae17-073a1987eebf.png) 

### 変更点
1. 以前の画面にありました "Private Key Certificates (.pfx)" が 新しい画面の "Bring your own certificates (.pfx)"に変わりました。(新旧画面の図中赤枠)
   - 以前の画面にありました "Import App Service Certificate", "Upload Certificate", "Import Key Vault Certificate" は、新しい画面の "Add" ボタンをクリックすることで表示されるように変更されました。(新旧画面の図中青枠)
   - 以前の画面にありました "Create App Service Managed Certificate" は、以前の画面の "Bindings" メニュー (図中黒枠) が存在していた位置に移動しました。(新旧画面の図中紫枠)
2. 以前の画面に存在していた "Binding" メニュー (図中黒枠) は廃止され、新しい画面の "Custom domains" ブレードに機能が引き継がれました。
   - 証明書のバインドは、後述の [App Service 証明書のバインド手順](#app-service-証明書のバインド手順) で説明しています。


# App Service 証明書のバインド手順

## a) 新規にカスタムドメインを登録し、TLS/SSL証明書をバインドするとき
1. "Custom domains" ブレードを開き、"Add custom domain" よりカスタムドメインを追加します。
 ![image-cc3cb6e6-8734-4708-8985-ea00220607dd.png]({{site.baseurl}}/media/2023/03/image-cc3cb6e6-8734-4708-8985-ea00220607dd.png) 
2. カスタムドメインの追加が完了後、"Add binding" をクリックします。
 ![image-4e562f56-09e5-4877-866e-7293f2d35772.png]({{site.baseurl}}/media/2023/03/image-4e562f56-09e5-4877-866e-7293f2d35772.png) 
3. "Certificate" に "Add new certificate" を指定し、"Source" 以下で証明書をインポートする方法を指定します。
4. "Add" をクリックすることで、Source 以下に指定された方法で証明書がインポートされ、App Service にバインドされます。

## b) 既存の TLS/SSL 証明書 (あるいは自己証明書) で更新するとき
1. "Certificates" ブレードの "Bring your own certificates (.pfx)" より自己証明書をアップロードします。
2. "Custom domains" ブレードを開き、該当ドメインの "..." より "Update binding" を開きます。
3. "Certificate" に新しい証明書を指定し、"Update" をクリックすることで証明書が更新されます。
 ![image-85b509d3-2484-49c6-baa5-00485ff4b90d.png]({{site.baseurl}}/media/2023/03/image-85b509d3-2484-49c6-baa5-00485ff4b90d.png) 


# App Service にバインドされた証明書の確認方法
"Custom domains" の "Update Binding" もしくは "Diagnose and solve problems" より確認することができます。
## a) Custom domains より確認する方法
1. "Custom domains" ブレードを開き、該当ドメインの "..." より "Update binding" を開きます。
 ![image-85b509d3-2484-49c6-baa5-00485ff4b90d.png]({{site.baseurl}}/media/2023/03/image-85b509d3-2484-49c6-baa5-00485ff4b90d.png) 
2. "TLS/SSL type" の "Current certificates used" より現在バインドされている証明書を確認できます。
 ![image-38fbeea4-4631-425f-b908-5abde8f1ba91.png]({{site.baseurl}}/media/2023/03/image-38fbeea4-4631-425f-b908-5abde8f1ba91.png) 

## b) Diagnose and solve problems より確認する方法
1. 該当 App Service の "Diagnose and solve problems" を開きます。
2. "Binding & SSL Configuration" を開きます。
 ![image-f429429b-8e64-4985-850f-1f415859d1e7.png]({{site.baseurl}}/media/2023/03/image-f429429b-8e64-4985-850f-1f415859d1e7.png) 
3. "Findings for 'xxxx' を開き、Insight より現在バインドされている証明書を確認できます。
 ![image-125229e3-0538-4188-8626-cea91093f2c5.png]({{site.baseurl}}/media/2023/03/image-125229e3-0538-4188-8626-cea91093f2c5.png) 

# 参考ドキュメント
- [証明書を App Service にアップロードする](https://learn.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate?tabs=apex%2Cportal#upload-certificate-to-app-service)
- [Updates to App Service Overview Blade for Custom Domains](https://azure.github.io/AppService/2023/02/03/Custom-domain-ux-updates.html)
<br>
<br>

---

<br>
<br>

2023 年 03 月 10 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>