---
title: "PHP 8.X で Nginx の設定をカスタマイズする方法"
author_name: "Hiroki Yoshii"
tags:
    - App Service
---

# はじめに
お世話になっております。App Service サポート担当の吉井です。

PHP 7.X までは、App Service on Linux の PHP イメージ内で Apache がウェブ サーバーとして採用されておりましたが、8.X からは Nginx が使用されるようになりました。アプリの要件やトラブルシューティングで Nginx の設定を調整したい場合があるかと思いますので、Nginx 設定値を調整する方法をご紹介します。

今回は例として、Nginx の設定ファイルより、レスポンスにカスタムの HTTP ヘッダーを追加します。

# 手順
## 1. 既定の設定値を編集
まずは、ベースとなる Nginx の設定ファイルが必要なので、既定の ```/etc/nginx/sites-available/default``` を取得します。

ポータル上で該当の Web App を開き、左メニューの ```SSH``` を選択します。```SSH``` を選択すると、アプリが動作しているコンテナに直接アクセスできます。アプリ コンテナのターミナルが表示されましたら、以下コマンドで設定ファイルを ```/home/site``` にコピーします。

```sh
 /etc/nginx/sites-available/default /home/site/default
 ```

<sub> ※ ```/home``` 配下以外は再起動時に初期化されるため、ユーザー独自のものは ```/home``` 配下に配置しておくことが重要です。 </sub>
<br>
<sub> ※ 今回、コピー先を ```/home/site``` に指定してますが、```/home``` 配下であれば、他の任意のパスでも問題ございません。 </sub>
<br>
<br>

次に、コピーした```default``` ファイルを編集します。編集方法は幾つかありますが、今回はブラウザ内で完結するやり方を紹介します。まずは、以下 URL にアクセスします。
```
https://<アプリ名>.scm.azurewebsites.net/newui/FileManager
```
すると、GUI ベースのファイル エクスプローラーが表示されます。
```site``` を選択し、```default``` の左に表示されている鉛筆ボタンを選択します。
<br>
![image-b6a96ed6-d695-4165-abdb-bdf693955f46.png]({{site.baseurl}}/media/2023/04/image-b6a96ed6-d695-4165-abdb-bdf693955f46.png)

エディターが開かれます。PHP 8.2 の ```default``` は、以下のような設定値になっています。

```nginx
server {
    #proxy_cache cache;
	#proxy_cache_valid 200 1s;
    listen 8080;
    listen [::]:8080;
    root /home/site/wwwroot;
    index  index.php index.html index.htm;
    server_name  example.com www.example.com; 
    port_in_redirect off;

    location / {            
        index  index.php index.html index.htm hostingstart.html;
    }

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /html/;
    }
    
    # Disable .git directory
    location ~ /\.git {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Add locations of phpmyadmin here.
    location ~ [^/]\.php(/|$) {
        fastcgi_split_path_info ^(.+?\.php)(|/.*)$;
        fastcgi_pass 127.0.0.1:9000;
        include fastcgi_params;
        fastcgi_param HTTP_PROXY "";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param QUERY_STRING $query_string;
        fastcgi_intercept_errors on;
        fastcgi_connect_timeout         300; 
        fastcgi_send_timeout           3600; 
        fastcgi_read_timeout           3600;
        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 256k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;
    }
}
```

今回の例では、カスタムのヘッダーを追加したいので、以下のように add_header を使って任意のヘッダーを追加して保存します。

```nginx
server {
    #proxy_cache cache;
	#proxy_cache_valid 200 1s;
    listen 8080;
    listen [::]:8080;
    root /home/site/wwwroot;
    index  index.php index.html index.htm;
    server_name  example.com www.example.com; 
    port_in_redirect off;
    
    add_header X-My-Custom-Header "My Custom Header";

    location / {            
        index  index.php index.html index.htm hostingstart.html;
    }
    ...
```

## 2. スタートアップ コマンドの追加
カスタムの設定ファイルが作成できたので、最後にこの設定ファイルを反映させる必要があります。これは、App Service の ```スタートアップ コマンド``` 機能で実現できます。

```スタートアップ コマンド``` 機能を使用することで、コンテナの起動処理に任意のコマンドを実行することができます。```構成``` > ```全般設定``` > ```スタートアップ コマンド``` より設定できます。
<br>
![image-52c80ab0-7758-4635-b913-c8f3ca0e89f5.png]({{site.baseurl}}/media/2023/04/image-52c80ab0-7758-4635-b913-c8f3ca0e89f5.png)

今回は、起動時に既定の ```default``` ファイルをカスタムのもので上書きしたいので、```スタートアップ コマンド```に以下を追加します。
```
cp /home/site/default /etc/nginx/sites-available/default && service nginx reload 
```
<sub> ※ App Service の PHP イメージの動作上、```スタートアップ コマンド``` が nginx の起動処理後に実行されるため、 カスタムの設定を反映させるために ```service nginx reload``` が必要です。 </sub>

もし、```スタートアップ コマンド``` が複雑化する場合は、処理内容をスクリプト ファイルに記載し、```スタートアップ コマンド```では、そのスクリプト ファイルだけを実行する方法もございます。例えば、```/home/site/startup.sh``` を作成し、```startup.sh``` に以下のように記載します。

```bash
#!/bin/bash
<その他処理>

cp /home/site/default /etc/nginx/sites-available/default
service nginx reload

<その他処理>
```
そして、```スタートアップ コマンド``` を以下のように設定します。
```bash
/home/site/startup.sh
```

# 確認
ブラウザのネットワーク トレース機能を使って、レスポンスのヘッダーを確認すると、以下のようにカスタムのヘッダーが追加されています。
<br>
![image-ec7042d0-f82c-4d2c-8cda-2f342fb9e087.png]({{site.baseurl}}/media/2023/04/image-ec7042d0-f82c-4d2c-8cda-2f342fb9e087.png)

# 補足
今回は、```スタートアップ コマンド``` を活用して Nginx の設定値を変更しましたが、実際にはこのユースケースに限らず、様々なことができます。例えば、```apt install``` を使って任意のパッケージをインストールすることも可能です。ただし、```スタートアップ コマンド``` が複雑化すればするほど起動処理も遅くなるため、再起動時のダウンタイムが延びてしまう可能性もあります。既定の App Service イメージを大幅にカスタマイズしたい場合は、Web App for Container をご検討ください。

<br>
<br>

---

<br>
<br>

2023 年 4 月 12 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>