---
title: "App Service にて Basic 認証を検証する"
author_name: "Kohei Mayama"
tags:
    - App Service
    - Web Apps
---

# はじめに
いつもお世話になっております。App Service サポートの間山です。
App Service にて Azure AD を用いた認証方法ではなく簡単な認証を設定したいことから Basic 認証は設定することが出来ないかというお問い合わせを頂戴しております。本記事では App Service を利用して Basic 認証を設定する方法をご案内して参ります。


# App Service の Basic 認証について
App Service プラットフォームからのご案内となりますが、Web Apps を使用することで Basic 認証機能の実現を行うことができます。
しなしながら、大変心苦しい限りではございますが、App Service プラットフォームそのものには組込みの Basic 認証モジュールのご用意がございません。これは、Basic 認証はセキュリティ上のリスクがあり、パブリッククラウドである App Service とは適合しないと考えているためです。昨今の情勢を踏まえましても Basic 認証は推奨いたしかねますが、App Service で Basic 認証を実現する方法として、下記 2 種類の方法がございます。

1. 第三者モジュールなどをご利用いただき、お客様独自に実装いただく
2. web.config や Nginx の default ファイルにて疑似的に Basic 認証と同等の動作を制御いただく

1.に関しましては、独自でモジュールを選定いただくものとなりますので、本記事では、2.の方法についてご案内差し上げます。


## web.config ファイルを利用した Basic 認証（Windows 版）
App Service の Windows 版を使用した際には Web サーバが IIS が使用されます。IIS は web.config ファイルを読み込みことができ、制御する方法として rewrite をご利用いただくことで実現いただけます。
今回検証環境にて Basic 認証を実施するパスと OS、ランタイムスタックは以下の条件といたします。
```
例) 特定のURL配下の認証
OS : Windows
ランタイム スタック : .NET 6 (Webサーバー : IIS)
```

1. 高度なツール（Kudu）よりディレクトリを下記の通り構成する

* 一般ユーザーのアクセスを想定（認証しない）
  * ファイル : /home/site/wwwroot/ hostingstart.html
  * アクセスする際の URL : https:// <App Service 名>.azurewebsites.net （デフォルトのURL）
  * <App Service 名>.azurewebsites.net はカスタムドメインであっても問題ございません。
 
* 管理ユーザーのアクセスを想定（認証する）
  * ファイル : /home/site/wwwroot/admin/admin.html
  * アクセスする際の URL : https:// <App Service 名>.azurewebsites.net/admin/admin.html
  * <App Service 名>.azurewebsites.net はカスタムドメインであっても問題ございません。

2. `/home/site/wwwroot/` 配下の web.config へ下記を追記する

web.config 内に記載の <Base64 でエンコードした[ユーザ名:パスワード]> は、あらかじめユーザ名とパスワードを Base64 へエンコードした値として記載していただくことが必要となります。例えば、ユーザ名を admin としてパスワードを admin とした場合には、admin:admin の Base 64 エンコードとなるため、YWRtaW46YWRtaW4= となります。

### web.config 追記例:
```
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <system.webServer>
        <rewrite>
            <rules>
                <rule name="BasicAuthOK" stopProcessing="true">
                    <match url="^admin/.*$" />
                    <conditions>
                        <add input="{HTTP_Authorization}" pattern="Basic <Base64 でエンコードした[ユーザ名:パスワード]>" />
                    </conditions>
                    <action type="None" />
                </rule>
                <rule name="BasicAuthNG" stopProcessing="true">
                    <match url="^admin/.*$" />
                    <serverVariables>
                        <set name="RESPONSE_WWW-Authenticate" value="Basic Realm=SecretZone" replace="true"/>
                    </serverVariables>
                    <action type="CustomResponse" statusCode="401" />
                </rule>   
            </rules>    
        </rewrite>
  </system.webServer>
</configuration>
```

3. アクセスを試みる

`https:// <App Service 名>.azurewebsites.net` へのアクセスでは Basic 認証は要求されませんが、`https:// <App Service 名>.azurewebsites.net/admin/admin.html` へアクセスが行われると、Basic 認証が要求され、`web.config` にて設定した ID とパスワードを入力する事でアクセスができます。
尚、`match url` の値を `<match url=".*" />` と記載する事により、`https://<App Service 名>.azurewebsites.net/` のサイト全体に認証が適用させることも可能でございます。

**https:// <App Service 名>.azurewebsites.net へのアクセス**
![image-96af1430-b386-4094-ab58-85fc61ab5c43.png]({{site.baseurl}}/media/2023/04/image-96af1430-b386-4094-ab58-85fc61ab5c43.png)

** https:// <App Service 名>.azurewebsites.net/admin/admin.html へのアクセス**
![image-dcf3ec40-40fc-409e-b891-4f210ca5a87b.png]({{site.baseurl}}/media/2023/04/image-dcf3ec40-40fc-409e-b891-4f210ca5a87b.png)

---

## Nginx の default ファイルを利用した Basic 認証（Linux 版）
App Service on Linux にて Basic 認証を実施するには Nginx が事前に入っているランタイムスタックで設定が可能となります。Nginx があらかじめ入っているランタイムスタックは `PHP 8.x` からとなります。Nginx のカスタマイズ方法につきまして以下の弊社サポートブログの関連記事をご覧ください。
[PHP 8.X で Nginx の設定をカスタマイズする方法](https://jpazpaas.github.io/blog/2023/04/13/Customize-Nginx-On-PHP-8.X.html)

今回検証環境にて Basic 認証を実施するパスと OS、ランタイムスタックは以下の条件といたします。
```
例) ルートのURL配下の認証
OS : Linux
ランタイム スタック : PHP 8.2 (Webサーバー : Nginx)
```

1. Basic認証用の「. htpasswd」 コマンドを利用するため、下記のパッケージをインストールする
```
apt-get install apache2-utils
```

**apache2-utils のインストール状況**

![image-2188323c-c83c-433c-9778-7ec2235ea8e4.png]({{site.baseurl}}/media/2023/04/image-2188323c-c83c-433c-9778-7ec2235ea8e4.png)


2. 以下のコマンドで、/.htpasswdファイルのパスを/home/.htpasswd に指定し、ユーザー名とパスワードを追加する

<任意のユーザー名> では Basic 認証に必要なユーザ名として入力を行い、その後新しいパスワードを対話型での登録となります。
```
htpasswd -c /home/.htpasswd <任意のユーザー名>
```

**ユーザ名とパスワードの設定**

![image-dba7530a-ce01-4fc9-aa6e-6127a43dde32.png]({{site.baseurl}}/media/2023/04/image-dba7530a-ce01-4fc9-aa6e-6127a43dde32.png)

3. Nginx の default ファイルを /home/defaultファイルをコピー

cp コマンドを用いて nginx で動作している default ファイルを /home ディレクトリへコピーします。
```
cp /etc/nginx/sites-enabled/default /home/default
```

4. /home/default ファイルを編集する

vi /home/default コマンドを実行し、i キーを入力してファイルを編集します。
以下のように　Basic 認証を実施する auth_basic_user_file のパスを /home/.htpasswd にします。
```
auth_basic "Restricted";
auth_basic_user_file /home/.htpasswd;
```

**/home/default ファイルの内容**

![image-f28a2266-2b4f-4dab-82e6-c819db105692.png]({{site.baseurl}}/media/2023/04/image-f28a2266-2b4f-4dab-82e6-c819db105692.png)

5. スタートアップコマンドより再起動後も Basic 認証 が行われるように設定する

以下のコマンドを用いて Basic 認証を設定した default ファイルを nginx 側へコピーを行い nginx の再起動コマンドをスタートアップコマンドへ記載します。
```
cp /home/default /etc/nginx/sites-enabled/default; service nginx restart
```

**スタートアップコマンドの設定**

![image-ea5cc1ab-094e-4009-9dad-d15f5ff1ccc9.png]({{site.baseurl}}/media/2023/04/image-ea5cc1ab-094e-4009-9dad-d15f5ff1ccc9.png)


6. 対象 App Serviceの「概要」メニューよりアプリケーションの再起動とアクセスを試みる

App Service を再起動後に `https:// <App Service 名>.azurewebsites.net` へのアクセスを試みると Basic 認証が要求されます。2. で設定したユーザ名とパスワードを入力することでアプリケーション内のコンテンツを閲覧することができます。

**https:// <App Service 名>.azurewebsites.net へのアクセス**
![image-38cb67df-db80-46a9-b95c-4adf5057eb32.png]({{site.baseurl}}/media/2023/04/image-38cb67df-db80-46a9-b95c-4adf5057eb32.png)

<br>
<br>

---

<br>
<br>

2023 年 04 月 25 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>