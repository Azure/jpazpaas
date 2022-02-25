---
title: "www サブドメインの App Service 証明書発行時のドメイン検証に関する新たな制限"
author_name: "Tomomi Hirotane"
tags:
    - App Service 証明書
---
# 概要
2021 年 12 月 1 日より、App Service 証明書の新規発行または更新、および App Service 証明書のキー更新の際に、以下の検証方法にて発行される新規 App Service 証明書は www サブドメイン を保護しなくなります。
<br>

- App Service
- 手動（HTML Web ページによるドメイン検証）

<br>
これは、App Service 証明書を発行する証明書発行機関 GoDaddy が、証明書発行プロセスにおいて HTML Web ページを利用したドメイン検証がされた際には、www サブドメインを保護する証明書を発行しないとの決定をしたことに伴うものです。
<br>

[https://jp.godaddy.com/help/verify-domain-ownership-dns-or-html-for-my-ssl-certificate-7452](https://jp.godaddy.com/help/verify-domain-ownership-dns-or-html-for-my-ssl-certificate-7452)
<br>

> 警告:HTML の方法を使うと、HTML ファイルが保存されているディレクトリに対してのみ証明書が検証されます。たとえば、HTML ファイルを Webサイトのルートディレクトリに保存する場合、証明書は coolexample.com に対しては検証されますが、www.coolexample.com に対しては検証されません。ルートドメインに対しても www. に対しても証明書を検証したい場合は、DNS の方法を使ってドメイン制御を証明してください。
<br>
<br>

# 対処法
上記の制限は `App Service` または `手動` の HTML Web ページによるドメイン検証に対するものです。
<br>
そのため、`ドメイン` または `メール` によるドメイン検証を代わりに実施する必要があります。
<br>
それぞれの手順の詳細については下記資料をご覧ください。
<br>
[https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate#store-in-azure-key-vault](https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate#store-in-azure-key-vault)
<br>
<br>

# 背景
## App Service 証明書が保護するドメイン
<br>
App Service 証明書は、ルート ドメインと www サブドメイン両方を保護することができる証明書です。
<br>

[https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate#start-certificate-order](https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate#start-certificate-order)
<br>

> ドメインはここで指定します。 発行された証明書によって、ルート ドメインと www サブドメインの "両方" が保護されます。 発行された証明書の [共通名] フィールドにはルート ドメインが含まれ、[サブジェクトの別名] フィールドには www ドメインが含まれています。 任意のサブドメインのみをセキュリティで保護するには、ここでサブドメインの完全修飾ドメイン名 を指定します (例: mysubdomain.contoso.com)。
<br>
<br>

## 証明書発行機関 GoDaddy がサポートするドメイン検証
<br>
証明書発行機関 GoDaddy は証明書発行時のドメイン検証の方法として `DNS による方法` および `HTML Web ページによる方法` の 2 種類をサポートしています。
<br>
詳細については下記の、GoDaddy の公式 Web サイトをご参照ください。
<br>

[https://jp.godaddy.com/help/verify-domain-ownership-dns-or-html-for-my-ssl-certificate-7452](https://jp.godaddy.com/help/verify-domain-ownership-dns-or-html-for-my-ssl-certificate-7452)

<br>
App Service 証明書のドメイン検証では以下の 4 種類の方法がサポートされていますが、`App Service` または `手動` の HTML Web ページによるドメイン検証が GoDaddy の HTML Web ページを利用した検証に該当します。
<br>

[https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate#verify-domain-ownership](https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate#verify-domain-ownership)
<br>

> 4 種類のドメイン検証方法がサポートされています。
<br>
> - __App Service__ - ドメインが同一のサブスクリプション内で既に App Service アプリにマップされている場合に最も便利なオプションです。 この方法は、App Service アプリがドメインの所有権を既に確認済みである事実を利用しています。
<br>
<br>
> - __ドメイン__ - Azure から購入した App Service ドメインを確認します。 Azure は確認 TXT レコードを自動的に追加し、プロセスを完了します。
<br>
<br>
> - __メール__ - ドメイン管理者に電子メールを送信することによってドメインを確認します。 手順は、オプションを選択したときに提供されます。
<br>
<br>
> - __手動__ - HTML ページ (標準 証明書のみ) または DNS TXT レコードを使用してドメインを確認します。 手順は、オプションを選択したときに提供されます。 HTML ページ オプションは、[HTTPS のみ] が有効になっている Web アプリでは機能しません。

<br>
結果として、www サブドメインに対する App Service 証明書を発行する場合には、上記のドメイン検証方法のうち、`ドメイン` もしくは `メール` によるドメイン検証をする必要があります。
<br>
<br>

# 想定される事象
上記の制限は、ドメイン検証を必要とするすべての作業 (App Service 証明書の新規発行、更新、キー更新) で該当するため注意が必要です。
<br>
例えば、ルート ドメインと www サブドメインを一つの App Service 証明書を保護しているとします。
<br>
この時、App Service 証明書更新の際に、'App Service' または '手動' の HTML Web ページによるドメイン検証を実施すると、ルート ドメインは新しい App Service 証明書で保護されますが、上記の制限により、www サブドメインに対しては証明書の更新が行われず、古い App Service 証明書が適応されることになります。
<br>
<br>

## **<検証 1>**  HTML Web ページによるドメイン検証の場合 www サブドメインへのドメイン検証が正常に行われないことを確認
<br>
App Service 証明書を新規発行する際に、`手動` の HTML Web ページによるドメイン検証を実施し、新規発行された App Service 証明書が www サブドメインに適用できるか確認しました。
<br>
手順は下記の通りです。

1. App Service に `<ドメイン名>.com` および `www.<ドメイン名>.com` というカスタム ドメインを構成
2. ルート ドメインに `<ドメイン名>.com` を設定した App Service 証明書を購入
3. `手動` の HTML Web ページによる確認を実施<br><br>
![2022-02-25-html-val-flow]({{site.baseurl}}/media/2022/02/2022-02-25-html-val-flow.png)<br><br>
    3-1. godaddy.html というファイル名のファイルを作成し、ドメイン確認トークンを記載します。<br>
    3-2. http://<ドメイン名>/.well-known/pki-validation/godaddy.html で手順 3-1 で作成した html ファイルの内容が表示されるように、godaddy.html を配置します。
4. 証明書のバインド
カスタムドメイン `<ドメイン名>.com` に対しては証明書のバインドが可能でしたが、www サブドメインである `www.<ドメイン名>.com` に対しては「選択したカスタム ドメインに一致する証明書はありません。」との表示がされ、バインドができませんでした。<br><br>

![2022-02-25-html-val-binding-alert]({{site.baseurl}}/media/2022/02/2022-02-25-html-val-binding-alert.png)

![2022-02-25-html-val-cert]({{site.baseurl}}/media/2022/02/2022-02-25-html-val-cert.png)<br><br>
これは、HTML Web ページによるドメイン検証の場合、www サブドメインに対してはドメイン検証が行われず、対応する App Service 証明書も発行されなかったためです。

<br>

## **<検証 2>** DNS によるドメイン検証の場合 www サブドメインへのドメイン検証が正常に完了することを確認
<br>
App Service 証明書を新規発行する際に、`手動` の DNS によるドメイン検証を実施し、新規発行された App Service 証明書が www サブドメインに適用できるか確認しました。
<br>
手順は下記の通りです。

1. App Service に `<ドメイン名>.com` および `www.<ドメイン名>.com` というカスタム ドメインを構成
2. ルート ドメインに `<ドメイン名>.com` を設定した App Service 証明書を購入
3. `手動` の手動による確認を実施 (この方法は DNS によるドメイン検証です。)<br><br>
![2022-02-25-manual-val-flow]({{site.baseurl}}/media/2022/02/2022-02-25-manual-val-flow.png)<br><br>
    3-1. 手順 1 で作成したカスタム ドメインの DNS ゾーンを開きます。<br>
    3-2. ルート ドメインに、TXT レコードを追加します。値にはドメイン確認トークンを設定します。<br><br>
    ![2022-02-25-manual-val-add-txt]({{site.baseurl}}/media/2022/02/2022-02-25-manual-val-add-txt.png)<br><br>
4. 証明書のバインド
カスタムドメイン `<ドメイン名>.com` および www サブドメインである `www.<ドメイン名>.com` に対してバインドができました。<br><br>

![2022-02-25-manual_val_binding]({{site.baseurl}}/media/2022/02/2022-02-25-manual-val-binding.png)

![2022-02-25-manual_val_cert]({{site.baseurl}}/media/2022/02/2022-02-25-manual-val-cert.png)<br><br>
これは、DNS によるドメイン検証の場合、www サブドメインに対してもドメイン検証が行われ、対応する App Service 証明書が発行されたためです。

<br>
上記の検証結果から、HTML Web ページによるドメイン検証の場合、発行された証明書は www サブドメインの保護しないことが確認されました。
<br>
<br>

# 参考ドキュメント

- 本変更についてアナウンスがされています。
<br>
[https://azure.github.io/AppService/2021/11/22/ASC-1130-Change.html](https://azure.github.io/AppService/2021/11/22/ASC-1130-Change.html)
<br>
<br>
- App Service 証明書の発行手順の詳細について記載されています。
<br>
[https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate](https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate)
<br>
<br>
- 証明書発行機関 GoDaddy がサポートするドメイン検証の方法について記載されています。
<br>
[https://jp.godaddy.com/help/verify-domain-ownership-dns-or-html-for-my-ssl-certificate-7452](https://jp.godaddy.com/help/verify-domain-ownership-dns-or-html-for-my-ssl-certificate-7452)

---

<br>
<br>

2022 年 2 月 25 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>