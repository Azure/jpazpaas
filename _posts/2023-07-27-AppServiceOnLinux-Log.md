---
title: App Service on Linux におけるログ出力について
author_name: "Takeharu Oshida"
tags:
    - App Service
    - Kudu
---


# はじめに
お世話になっております。App Service サポート担当の押田です。

本記事では App Service on Linux において取得可能なログについて記載します。

# App Service on Linux におけるログの種類

App Service on Linux のファイルシステムにおいて、標準では大きく分けて以下のログが存在します。

1. コンテナ操作に関するプラットフォームのログ
1. アプリケーションコンテナからの標準出力/標準エラー
1. デプロイに関するログ
1. アプリケーションコンテナ内でファイルとして作成されるログ

## 1. アプリケーションコンテナ操作に関するプラットフォームのログ
イメージの Pull やコンテナの起動などに関するログを指します。コンテナ起動失敗などのトラブルシューティングにて主に利用するログとなります。

## 2.アプリケーションコンテナからの標準出力/標準エラー
コンテナ起動時のスクリプト(Docker EntryPoint で指定されたシェル)からの出力、スタートアップコマンドで実行するスクリプトからの出力、お客様のアプリケーションそのものや、利用するWebフレームワークからの出力、などが主に含まれます。コンテナの起動失敗やコンテナクラッシュなどのトラブルシューティングにて主に利用するログとなります。

## 3.デプロイに関するログ
ここでは Zip デプロイなどアプリケーションコードをデプロイしたときの Kudu のログを指します。デプロイエラーのトラブルシューティングにて主に利用するログとなります。Azure リソースとしてデプロイについてはこの記事では言及しません。

## 4.アプリケーションコンテナ内でファイルとして作成されるログ
`/var/log` 配下に出力されるような各種ログや、アプリケーションが独自のロガーを用いてファイルに出力しているログ等が含まれます。
出力先としては、`$home` 配下、BYOS でマウントしたボリューム、`/var/log` などコンテナ内に限られた領域があります。
`$home` 配下については、Kudu から確認することはできますが、それ以外については、App Service のプラットフォームの機能で参照することはできず、コンテナに SSH 接続して確認する必要があります。

アプリケーションが正常に動作していない場合などにおいては、前述 1, 2 のログをまずはご確認いただく必要がございます。
デプロイが失敗している場合などは 前述 3 のログを重点的にご確認ください。
上記いずれのログについても、お客様の資産であるため、弊社サポートから直接参照することはできません。
そのため、サポートにお問い合わせ頂く場合、上記のログについてお客様にて取得いただき提供をご依頼する場合がございます。

# App Service on Linux におけるログの取得方法

## 「1. コンテナ操作に関するプラットフォームのログ」および 「2. アプリケーションコンテナからの標準出力/標準エラー」

### ファイルとして取得する方法

ポータルより、[App Service ログ] にて有効にすることでファイルシステムに保存できるようになります。

![image-c34409e8-2283-44a0-8f62-0f9fb94f6fe8.png]({{site.baseurl}}/media/2023/07/image-c34409e8-2283-44a0-8f62-0f9fb94f6fe8.png)

出力されるファイルは

- `/home/LogFiles/<YYYY>_<MM>_<DD>_<COMPUTERNAME>_docker.log`
- `/home/LogFiles/<YYYY>_<MM>_<DD>_<COMPUTERNAME>_<ContainerName>_docker.log`

の 2 種類があります。
`<YYYY>_<MM>_<DD>`には日付(UTC)、`<COMPUTERNAME>` はワーカーインスタンスの環境変数 `COMPUTERNAME` が代入されます。
`<ContainerName>` は、アプリケーションコンテナの場合は `default` となります。
アプリケーションコンテナ以外には、`easyauth`(主に組み込み認証機能を提供) や `msi`(マネージドID機能を提供) といった機能に応じて起動されるサイドカーコンテナ毎のログが出力されます。

ファイルへのアクセスは Kudu より可能となっております。

#### Web UI でダウンロードする方法

[Kudulite newUI](https://azure.github.io/jpazpaas/2022/11/28/How-to-use-Kudu-site.html#tips-2-new-ui-kudulite) を利用してダウンロード可能となっています。

![image-9910b12a-80da-41a9-a530-2f1dbc5e2378.png]({{site.baseurl}}/media/2023/07/image-9910b12a-80da-41a9-a530-2f1dbc5e2378.png)

また、FileManagerから `/home/LogFiles` ディレクトリ、あるいは個別のファイルをダウンロードアイコンでダウンロード可能です。

![image-519d9608-bc51-4601-858a-f732252921d7.png]({{site.baseurl}}/media/2023/07/image-519d9608-bc51-4601-858a-f732252921d7.png)

#### Rest API でダウンロードする方法

- `$home/LogFiles` 以下全てを取得する場合

  `curl -u '<user>:<pass>' https://<appName>.scm.azurewebsites.net/api/dump --output dump.zip`

- docker関連ログを取得する場合

  `curl -u '<user>:<pass>' https://<appName>.scm.azurewebsites.net/api/logs/docker/zip --output dockerLog.zip`

`<user>` および `<pass>` は、ポータル上デプロイセンター > [ローカルGit または FTPSの資格情報] よりご確認いただけます。
通常、アプリケーションスコープのユーザー名は ”app名\$app名" といった表記となっており、その場合、curlコマンド等の基本認証のユーザ名として渡す値は、$app名 となります。
なお、いずれのzipもファイル数が多い場合は全てのファイルが含まれていない場合があります。その場合は個別のファイルをダウンロードしてください。


#### ログ例

- `/home/LogFiles/<YYYY>_<MM>_<DD>_<COMPUTERNAME>_docker.log`


    ```log
    2023-07-19T05:47:56.197Z INFO  - 960f88d01c6d Extracting 271B / 271B
    2023-07-19T05:47:56.681Z INFO  - 960f88d01c6d Pull complete
    2023-07-19T05:47:56.721Z INFO  - 5340db92048c Extracting 269B / 269B
    2023-07-19T05:47:56.730Z INFO  - 5340db92048c Extracting 269B / 269B
    2023-07-19T05:47:57.134Z INFO  - 5340db92048c Pull complete
    2023-07-19T05:47:57.173Z INFO  - 1de8884868db Extracting 125B / 125B
    2023-07-19T05:47:57.184Z INFO  - 1de8884868db Extracting 125B / 125B
    2023-07-19T05:47:57.618Z INFO  - 1de8884868db Pull complete
    2023-07-19T05:47:57.680Z INFO  - 13b92216e305 Extracting 418B / 418B
    2023-07-19T05:47:57.700Z INFO  - 13b92216e305 Extracting 418B / 418B
    2023-07-19T05:47:58.272Z INFO  - 13b92216e305 Pull complete
    2023-07-19T05:47:58.323Z INFO  - 2b61c8786332 Extracting 649B / 649B
    2023-07-19T05:47:58.325Z INFO  - 2b61c8786332 Extracting 649B / 649B
    2023-07-19T05:47:58.796Z INFO  - 2b61c8786332 Pull complete
    2023-07-19T05:47:59.329Z INFO  - df186a2aa8db Extracting 128KB / 877KB
    2023-07-19T05:48:00.053Z INFO  - df186a2aa8db Extracting 877KB / 877KB
    2023-07-19T05:48:00.105Z INFO  - df186a2aa8db Extracting 877KB / 877KB
    2023-07-19T05:48:00.276Z INFO  - df186a2aa8db Pull complete
    2023-07-19T05:48:00.324Z INFO  - 81174c23e08f Extracting 133B / 133B
    2023-07-19T05:48:00.326Z INFO  - 81174c23e08f Extracting 133B / 133B
    2023-07-19T05:48:00.688Z INFO  - 81174c23e08f Pull complete
    2023-07-19T05:48:00.718Z INFO  -  Digest: sha256:b1dc34abcaac6f00dd945840a5bc6367123b63b56465a5ec20c316eca98e569e
    2023-07-19T05:48:00.738Z INFO  -  Status: Downloaded newer image for mcr.microsoft.com/appsvc/node:18-lts_20230502.1.tuxprod
    2023-07-19T05:48:00.768Z INFO  - Pull Image successful, Time taken: 3 Minutes and 54 Seconds
    2023-07-19T05:48:01.039Z INFO  - Starting container for site
    2023-07-19T05:48:01.040Z INFO  - docker run -d --expose=8080 --name toshida-sandbox-linux_1_fbe8a298 -e WEBSITE_AUTH_ENABLED=False -e WEBSITE_SITE_NAME=toshida-sandbox-linux -e PORT=8080 -e WEBSITE_ROLE_INSTANCE_ID=0 -e WEBSITE_HOSTNAME=toshida-sandbox-linux.azurewebsites.net -e WEBSITE_INSTANCE_ID=302cf8973b5d900072f95accb5bb6ac5a41aef7e5771a06a1c43c6ab11398c27 -e HTTP_LOGGING_ENABLED=1 -e NODE_OPTIONS=--require /agents/nodejs/build/src/Loader.js -e WEBSITE_USE_DIAGNOSTIC_SERVER=True appsvc/node:18-lts_20230502.1.tuxprod  

    2023-07-19T05:48:16.173Z INFO  - Initiating warmup request to container toshida-sandbox-linux_1_fbe8a298 for site toshida-sandbox-linux
    2023-07-19T05:48:33.863Z INFO  - Waiting for response to warmup request for container toshida-sandbox-linux_1_fbe8a298. Elapsed time = 18.2996024 sec
    2023-07-19T05:48:52.040Z INFO  - Container toshida-sandbox-linux_1_fbe8a298 for site toshida-sandbox-linux initialized successfully and is ready to serve requests.
    ```

- `/home/LogFiles/<YYYY>_<MM>_<DD>_<COMPUTERNAME>_default_docker.log`

    ```log
    2023-07-19T05:48:05.285197980Z    _____                               
    2023-07-19T05:48:05.285232481Z   /  _  \ __________ _________   ____  
    2023-07-19T05:48:05.285237281Z  /  /_\  \\___   /  |  \_  __ \_/ __ \ 
    2023-07-19T05:48:05.285240781Z /    |    \/    /|  |  /|  | \/\  ___/ 
    2023-07-19T05:48:05.285244381Z \____|__  /_____ \____/ |__|    \___  >
    2023-07-19T05:48:05.285248581Z         \/      \/                  \/ 
    2023-07-19T05:48:05.285251881Z A P P   S E R V I C E   O N   L I N U X
    2023-07-19T05:48:05.285255181Z 
    2023-07-19T05:48:05.285258681Z Documentation: http://aka.ms/webapp-linux
    2023-07-19T05:48:05.285261881Z NodeJS quickstart: https://aka.ms/node-qs
    2023-07-19T05:48:05.285265081Z NodeJS Version : v18.16.0
    2023-07-19T05:48:05.285268081Z Note: Any data outside '/home' is not persisted
    2023-07-19T05:48:05.285271382Z 
    2023-07-19T05:48:09.901216305Z Starting OpenBSD Secure Shell server: sshd.
    2023-07-19T05:48:10.192789138Z Starting periodic command scheduler: cron.
    2023-07-19T05:48:10.341765656Z Cound not find build manifest file at '/home/site/wwwroot/oryx-manifest.toml'
    2023-07-19T05:48:10.341800257Z Could not find operation ID in manifest. Generating an operation id...
    2023-07-19T05:48:10.341805757Z Build Operation ID: 09181e63-0fde-47d7-b05c-41cf28d7abb9
    2023-07-19T05:48:12.091562959Z Environment Variables for Application Insight's IPA Codeless Configuration exists..
    2023-07-19T05:48:12.352149389Z Writing output script to '/opt/startup/startup.sh'
    2023-07-19T05:48:12.744738062Z Running #!/bin/sh
    2023-07-19T05:48:12.744754362Z 
    2023-07-19T05:48:12.744758862Z # Enter the source directory to make sure the script runs where the user expects
    2023-07-19T05:48:12.744762662Z cd "/home/site/wwwroot"
    2023-07-19T05:48:12.744774562Z 
    2023-07-19T05:48:12.744778063Z export NODE_PATH=/usr/local/lib/node_modules:$NODE_PATH
    2023-07-19T05:48:12.744781663Z if [ -z "$PORT" ]; then
    2023-07-19T05:48:12.744785363Z 		export PORT=8080
    2023-07-19T05:48:12.744789363Z fi
    2023-07-19T05:48:12.744792663Z 
    2023-07-19T05:48:12.744796163Z npm start
    2023-07-19T05:48:36.920990760Z npm info using npm@9.6.4
    2023-07-19T05:48:36.921418667Z npm info using node@v18.16.0
    2023-07-19T05:48:37.184253403Z 
    2023-07-19T05:48:37.184289204Z > my-app@0.1.0 start
    2023-07-19T05:48:37.184295004Z > node_modules/next/dist/bin/next start
    2023-07-19T05:48:37.184299504Z 
    2023-07-19T05:48:39.283556344Z npm http fetch GET 200 https://registry.npmjs.org/npm 2245ms (cache updated)
    2023-07-19T05:48:39.444670756Z npm http fetch GET 200 https://registry.npmjs.org/npm 2302ms (cache updated)
    2023-07-19T05:48:43.611854464Z ready - started server on 0.0.0.0:8080, url: http://localhost:8080
    ```

### 診断設定を利用する方法

「診断設定」から診断ログを有効にすることで、Azure Monitor にて利用可能なログもあります。

前述の `default_docker.log` に相当する内容は、`AppServiceConsoleLogs` として、`docker.log` に相当する内容は、`AppServicePlatformLogs` としてご確認いただけます。

診断設定では、前述のログ以外にも取得可能なログがあります。詳しくは下記をご参照ください。

[診断ログの有効化](https://learn.microsoft.com/ja-jp/azure/app-service/troubleshoot-diagnostic-logs#send-logs-to-azure-monitor)

![image-40e678a8-0777-4d02-b704-4ac965dd0504.png]({{site.baseurl}}/media/2023/07/image-40e678a8-0777-4d02-b704-4ac965dd0504.png)

例: AppServiceConsolLogs

![image-b895963b-e647-4aab-bdca-b7bd4e53a580.png]({{site.baseurl}}/media/2023/07/image-b895963b-e647-4aab-bdca-b7bd4e53a580.png)


### 問題の診断と解決を利用する方法

「問題の診断と解決」からは「Application Logs」メニューから前述の `default_docker.log` および `AppServiceConsoleLogs` に相当する内容を確認することが可能です。

![image-cba9d584-4f22-4a86-8d2a-4724ac103a95.png]({{site.baseurl}}/media/2023/07/image-cba9d584-4f22-4a86-8d2a-4724ac103a95.png)


## 「3. デプロイに関するログ」

### ファイルとして取得する方法

デプロイ処理は Kudu および Oryx によって行われ、そのログは `$home/LogFiles/kudu` 配下に出力されます。
取得方法としては、「コンテナ操作に関するプラットフォームのログ」および 「アプリケーションコンテナからの標準出力/標準エラー」と同様に Kudu newUI File Manager からダウンロード可能です。

### ポータルで確認する方法

デプロイセンターからご確認いただけます。

![image-b5c59c9a-2d40-4326-a0d9-0cbdff030855.png]({{site.baseurl}}/media/2023/07/image-b5c59c9a-2d40-4326-a0d9-0cbdff030855.png)

## 「4. アプリケーションコンテナ内でファイルとして作成されるログ」

前述のように `$home` 配下に出力されたファイルは Kudu newUI File Manager を利用することでダウンロード可能です。
`$home` 配下以外のアプリケーションコンテナに出力されたファイルは直接ダウンロードする事はできません。
 Kudu newUI Web SSH を利用してコンテナに接続後に操作ログとして取得することが可能となっています。

Web SSH で表示されたターミナル下部のメニューより、`Start Log`,`Stop Log`,`Download Log`　から操作ログを取得することができます。例えば `cat` コマンド等を利用して必要なファイルの確認を実施してください。

![image-28498e69-3e5f-4d29-a20e-75208040ede1.png]({{site.baseurl}}/media/2023/07/image-28498e69-3e5f-4d29-a20e-75208040ede1.png)


<br>
<br>

---

<br>
<br>

2023 年 07 月 27 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>