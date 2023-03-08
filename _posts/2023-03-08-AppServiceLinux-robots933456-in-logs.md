---
title: App Service on Linux における robots933456.txt について
author_name: "Takeharu Oshida"
tags:
    - App Service
    - App Service on Linux
    - Node.js
---


# はじめに
お世話になっております。App Service サポート担当の押田です。

本記事では App Service on Linux において 組み込みイメージおよびカスタム Linux コンテナー(Web App for Containers) がどのように起動確認が行われるのかについて解説します。

# 起動確認に用いられるリクエスト (GET /robots933456.txt)

App Service on Linux において、組み込みイメージおよびカスタムイメージはプラットフォーム上で、docker コマンドによって起動されます。App Service のプラットフォームは、docker コマンドで起動したコンテナが動作可能な状態であるかの確認のため、アプリケーションに対して、`GET /robots933456.txt` リクエストを実行します。
このリクエストに応答ができない場合、コンテナ起動失敗とみなされ、コンテナのリサイクルが行われます。
なお、`GET /robots933456.txt` に対しては、必ずしも 200 応答を返す必要はありません。

>  /robots933456.txt は、コンテナーが要求を処理できるかどうかを調べるために App Service が使用するダミーの URL パスです。 404 応答は、パスが存在しないことを示すだけですが、コンテナーが正常であり、要求に応答できる状態であることを App Service に知らせます。

これは、組み込みイメージの言語や、カスタム Linux コンテナーに共通の仕組みとなっています。公式ドキュメントにおいても、下記の記事が各構成向けの資料にインクルードされているものとなります。

共通記事

[azure\-docs/app\-service\-web\-configure\-robots933456\.md at main · MicrosoftDocs/azure\-docs](https://github.com/MicrosoftDocs/azure-docs/blob/main/includes/app-service-web-configure-robots933456.md)

ドキュメント例

- [Node\.js アプリの構成 \- Azure App Service \| Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/app-service/configure-language-nodejs?pivots=platform-linux#robots933456-in-logs)

- [ASP\.NET Core アプリの構成 \- Azure App Service \| Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/app-service/configure-language-dotnetcore?pivots=platform-linux#robots933456-in-logs)

- [カスタム コンテナーを構成する \- Azure App Service \| Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/app-service/configure-custom-container?tabs=debian&pivots=container-linux#robots933456-in-logs)

## 起動確認リクエストの詳細

本記事では、組み込みイメージとして Node 18 LTS を利用して動作を確認してみます。

### 動作確認環境

- LinuxFxVersion: NODE|18-lts"
- App Service Plan: S1

### 検証コード

`/home/site/wwwroot/package.json` では、以下のように `start` スクリプトに `index.js` を指定します。

```json
{
  "name": "jpazpaas-blog-examples-linuxnodeapp-20230307",
  "version": "1.0.0",
  "scripts": {
    "start": "node index.js"
  }
}

```

`/home/site/wwwroot/index.js` ではリクエストを受信したらリクエスト内容をログに出力し、200 OK を返却するようにします。

```javascript
const http = require("http");
const port = process.env.PORT || 3000;
console.log(`index.js started with process.env.PORT=${process.env.PORT}`)

const logRequest = (request) => {
  const msg = `Request received
  [time]: ${new Date().toISOString()}
  [request.url]: ${request.url}
  [request.headers]: ${JSON.stringify(request.headers,null,4)}`;
  console.log(msg);
};

// https://nodejs.org/docs/latest-v18.x/api/http.html#httpcreateserveroptions-requestlistener
const server = http.createServer((req, res) => {
    logRequest(req);
    res.writeHead(200, {"Content-Type": "text/plain"});
    res.end("OK");
})
server.listen(port);
console.log(`Server started`);
```

クライアント側はリクエスト開始日時がわかるように以下のようなコードを用意します。
```javascript
const clientReqest = () => {
    const clientTime = new Date().toISOString();
    console.log(`client request started ${clientTime}`);
    fetch(`https://jpazpaas-blog-examples-linuxnodeapp-20230307.azurewebsites.net/fromClient/${clientTime}`)
    .then((res) => {
        console.log(`response received ${new Date().toISOString()},${res.status}`)
    })
    .catch((e) => {
        console.log(`request failed ${new Date().toISOString()},${e}`)
    })
}
```

### 検証

まず、アプリケーションリソースが停止状態の動作を確認します。

App Service リソースとして起動されていない場合、以下のように `403 (Site Disabled)` エラーがすぐに返却されます。
![image-ad410849-ee97-4675-a919-f51212297f26.png]({{site.baseurl}}/media/2023/03/image-ad410849-ee97-4675-a919-f51212297f26.png) 

これは、アプリケーションコードではなく、App Service のプラットフォームから返却されているものとなります。

ポータルからアプリケーションを起動し、すぐにクライアント側からリクエストを送信してみます。

![image-a93144a2-87fc-4238-af5c-d008f53809ea.png]({{site.baseurl}}/media/2023/03/image-a93144a2-87fc-4238-af5c-d008f53809ea.png)

おおよそ 2分後に 200 レスポンスを受信しました。

このときのログを確認してみます。

プラットフォーム側のログとして、診断設定の `AppServicePlatformLogs` や、ファイル 
 `/home/LogFiles/<YYY_MM_DD>_<HostName>_docker.log` には、以下のような出力が見られます。

```log
2023-03-07T02:49:57.774Z INFO  - 18-lts_20221208.1.tuxprod Pulling from appsvc/node
2023-03-07T02:49:57.776Z INFO  -  Digest: sha256:0213c7471d5f274a1ecd012920a26e9f5ed76dc3008981c3c46e0fc98a124e79
2023-03-07T02:49:57.777Z INFO  -  Status: Image is up to date for mcr.microsoft.com/appsvc/node:18-lts_20221208.1.tuxprod
2023-03-07T02:49:57.894Z INFO  - Pull Image successful, Time taken: 0 Minutes and 0 Seconds
2023-03-07T02:49:58.150Z INFO  - Starting container for site
2023-03-07T02:49:58.151Z INFO  - docker run -d -p 8020:8080 --name jpazpaas-blog-examples-linuxnodeapp-20230307_0_3ab3e0bf -e WEBSITE_SITE_NAME=jpazpaas-blog-examples-linuxnodeapp-20230307 -e WEBSITE_AUTH_ENABLED=False -e WEBSITE_ROLE_INSTANCE_ID=0 -e WEBSITE_HOSTNAME=jpazpaas-blog-examples-linuxnodeapp-20230307.azurewebsites.net -e WEBSITE_INSTANCE_ID=60c176d165cff1190180ffa3f443f64d8269bd82832141c75f0f4d8dee36c586 -e HTTP_LOGGING_ENABLED=1 -e NODE_OPTIONS=--require /agents/node/build/src/Loader.js -e WEBSITE_USE_DIAGNOSTIC_SERVER=True appsvc/node:18-lts_20221208.1.tuxprod  

2023-03-07T02:50:12.665Z INFO  - Initiating warmup request to container jpazpaas-blog-examples-linuxnodeapp-20230307_0_3ab3e0bf for site jpazpaas-blog-examples-linuxnodeapp-20230307
2023-03-07T02:50:35.733Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_3ab3e0bf. Elapsed time = 23.0750703 sec
2023-03-07T02:50:52.895Z INFO  - Container jpazpaas-blog-examples-linuxnodeapp-20230307_0_3ab3e0bf for site jpazpaas-blog-examples-linuxnodeapp-20230307 initialized successfully and is ready to serve requests.
```

今回注目すべき内容は以下となります。
- 02:49:58.151 に `docker run -d -p 8020:8080 <~省略~>`にてイメージを起動開始処理実行
- 02:50:12.665 に warmup requestを開始
- 02:50:35.733 に Waiting for response to warmup にて、23秒経過
- 02:50:52.895 に ready to serve requests

アプリケーション側のログとして、診断設定の `AppServiceConsoleLogs` や、ファイル 
`/home/LogFiles/<YYY_MM_DD>_<HostName>_default_docker.log` には、以下のような出力が見られます。
※一部マスクしています。

```
2023-03-07T02:50:13.040241471Z    _____                               
2023-03-07T02:50:13.040290971Z   /  _  \ __________ _________   ____  
2023-03-07T02:50:13.040312772Z  /  /_\  \\___   /  |  \_  __ \_/ __ \ 
2023-03-07T02:50:13.040317372Z /    |    \/    /|  |  /|  | \/\  ___/ 
2023-03-07T02:50:13.040321272Z \____|__  /_____ \____/ |__|    \___  >
2023-03-07T02:50:13.040325572Z         \/      \/                  \/ 
2023-03-07T02:50:13.040329072Z A P P   S E R V I C E   O N   L I N U X
2023-03-07T02:50:13.040332472Z 
2023-03-07T02:50:13.040335772Z Documentation: http://aka.ms/webapp-linux
2023-03-07T02:50:13.040339272Z NodeJS quickstart: https://aka.ms/node-qs
2023-03-07T02:50:13.040342872Z NodeJS Version : v18.12.1
2023-03-07T02:50:13.040346172Z Note: Any data outside '/home' is not persisted
2023-03-07T02:50:13.040349772Z 
2023-03-07T02:50:13.613486570Z Starting OpenBSD Secure Shell server: sshd.
2023-03-07T02:50:13.973876656Z Starting periodic command scheduler: cron.
2023-03-07T02:50:14.227141627Z Found build manifest file at '/home/site/wwwroot/oryx-manifest.toml'. Deserializing it...
2023-03-07T02:50:14.238570157Z Build Operation ID: |UC/E2l5dpjI=.2e033bab_
2023-03-07T02:50:16.626325323Z Environment Variables for Application Insight's IPA Codeless Configuration exists..
2023-03-07T02:50:16.747346429Z Writing output script to '/opt/startup/startup.sh'
2023-03-07T02:50:16.896343282Z Running #!/bin/sh
2023-03-07T02:50:16.896375582Z 
2023-03-07T02:50:16.896381082Z # Enter the source directory to make sure the script runs where the user expects
2023-03-07T02:50:16.896385182Z cd "/home/site/wwwroot"
2023-03-07T02:50:16.896388782Z 
2023-03-07T02:50:16.896392182Z export NODE_PATH=/usr/local/lib/node_modules:$NODE_PATH
2023-03-07T02:50:16.896395882Z if [ -z "$PORT" ]; then
2023-03-07T02:50:16.896399582Z 		export PORT=8080
2023-03-07T02:50:16.896403482Z fi
2023-03-07T02:50:16.896412183Z 
2023-03-07T02:50:16.896416083Z echo Found tar.gz based node_modules.
2023-03-07T02:50:16.896419583Z extractionCommand="tar -xzf node_modules.tar.gz -C /node_modules"
2023-03-07T02:50:16.896423283Z echo "Removing existing modules directory from root..."
2023-03-07T02:50:16.896426883Z rm -fr /node_modules
2023-03-07T02:50:16.896430283Z mkdir -p /node_modules
2023-03-07T02:50:16.896433683Z echo Extracting modules...
2023-03-07T02:50:16.896437083Z $extractionCommand
2023-03-07T02:50:16.896440383Z export NODE_PATH="/node_modules":$NODE_PATH
2023-03-07T02:50:16.896444083Z export PATH=/node_modules/.bin:$PATH
2023-03-07T02:50:16.896447483Z if [ -d node_modules ]; then
2023-03-07T02:50:16.896461683Z     mv -f node_modules _del_node_modules || true
2023-03-07T02:50:16.896465483Z fi
2023-03-07T02:50:16.896468883Z 
2023-03-07T02:50:16.896472083Z if [ -d /node_modules ]; then
2023-03-07T02:50:16.896475483Z     ln -sfn /node_modules ./node_modules 
2023-03-07T02:50:16.896478883Z fi
2023-03-07T02:50:16.896482383Z 
2023-03-07T02:50:16.896485683Z echo "Done."
2023-03-07T02:50:16.906654610Z npm start
2023-03-07T02:50:16.907561321Z Found tar.gz based node_modules.
2023-03-07T02:50:16.946023600Z Removing existing modules directory from root...
2023-03-07T02:50:16.957690445Z Extracting modules...
2023-03-07T02:50:16.987503315Z tar (child): node_modules.tar.gz: Cannot open: No such file or directory
2023-03-07T02:50:16.988244925Z tar (child): Error is not recoverable: exiting now
2023-03-07T02:50:16.994973408Z tar: Child returned status 2
2023-03-07T02:50:16.995494915Z tar: Error is not recoverable: exiting now
2023-03-07T02:50:17.136177265Z Done.
2023-03-07T02:50:36.519401247Z npm info it worked if it ends with ok
2023-03-07T02:50:36.520006255Z npm info using npm@6.14.15
2023-03-07T02:50:36.520016155Z npm info using node@v18.12.1
2023-03-07T02:50:40.754968999Z npm info lifecycle jpazpaas-blog-examples-linuxnodeapp-20230307@1.0.0~prestart: jpazpaas-blog-examples-linuxnodeapp-20230307@1.0.0
2023-03-07T02:50:40.815105848Z npm info lifecycle jpazpaas-blog-examples-linuxnodeapp-20230307@1.0.0~start: jpazpaas-blog-examples-linuxnodeapp-20230307@1.0.0
2023-03-07T02:50:40.975282743Z 
2023-03-07T02:50:40.975346543Z > jpazpaas-blog-examples-linuxnodeapp-20230307@1.0.0 start /home/site/wwwroot
2023-03-07T02:50:40.975357044Z > node index.js
2023-03-07T02:50:40.975361044Z 
2023-03-07T02:50:52.056046116Z index.js started with process.env.PORT=8080
2023-03-07T02:50:52.056090917Z Server started
2023-03-07T02:50:52.689402008Z Request received
2023-03-07T02:50:52.696386693Z  [time]: 2023-03-07T02:50:52.659Z
2023-03-07T02:50:52.696393193Z  [request.url]:/robots933456.txt
2023-03-07T02:50:52.696397093Z  [request.headers]:{
2023-03-07T02:50:52.696408293Z     "host": "127.0.0.1:8020",
2023-03-07T02:50:52.696412693Z     "user-agent": "HealthCheck/1.0"
2023-03-07T02:50:52.696416493Z }
2023-03-07T02:50:53.311775366Z Request received
2023-03-07T02:50:53.318595749Z  [time]: 2023-03-07T02:50:53.272Z
2023-03-07T02:50:53.318631150Z  [request.url]:/fromClient/2023-03-07T02:48:56.457Z
2023-03-07T02:50:53.318635850Z  [request.headers]:{
2023-03-07T02:50:53.318639050Z     "accept": "*/*",
2023-03-07T02:50:53.318671950Z     "host": "jpazpaas-blog-examples-linuxnodeapp-20230307.azurewebsites.net",
2023-03-07T02:50:53.318675850Z     "user-agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.0.0 Safari/537.36",
2023-03-07T02:50:53.318679250Z     "accept-encoding": "gzip, deflate, br",
2023-03-07T02:50:53.318682550Z     "accept-language": "ja-JP,ja;q=0.9",
2023-03-07T02:50:53.318685650Z     "cache-control": "no-cache",
2023-03-07T02:50:53.318688950Z     "max-forwards": "10",
2023-03-07T02:50:53.318692050Z     "pragma": "no-cache",
2023-03-07T02:50:53.318695150Z     "referer": "https://jpazpaas-blog-examples-linuxnodeapp-20230307.azurewebsites.net/",
2023-03-07T02:50:53.318698450Z     "sec-ch-ua": "\"Chromium\";v=\"110\", \"Not A(Brand\";v=\"24\", \"Google Chrome\";v=\"110\"",
2023-03-07T02:50:53.318702050Z     "sec-ch-ua-mobile": "?0",
2023-03-07T02:50:53.318705151Z     "sec-ch-ua-platform": "\"Windows\"",
2023-03-07T02:50:53.318708351Z     "sec-fetch-site": "same-origin",
2023-03-07T02:50:53.318711351Z     "sec-fetch-mode": "cors",
2023-03-07T02:50:53.318714551Z     "sec-fetch-dest": "empty",
2023-03-07T02:50:53.318717751Z     "x-arr-log-id": "*******",
2023-03-07T02:50:53.318720851Z     "client-ip": "*******",
2023-03-07T02:50:53.318724051Z     "x-client-ip": "*******",
2023-03-07T02:50:53.318727051Z     "disguised-host": "jpazpaas-blog-examples-linuxnodeapp-20230307.azurewebsites.net",
2023-03-07T02:50:53.318730151Z     "x-site-deployment-id": "jpazpaas-blog-examples-linuxnodeapp-20230307",
2023-03-07T02:50:53.318733351Z     "was-default-hostname": "jpazpaas-blog-examples-linuxnodeapp-20230307.azurewebsites.net",
2023-03-07T02:50:53.318736451Z     "x-forwarded-proto": "https",
2023-03-07T02:50:53.318739651Z     "x-appservice-proto": "https",
2023-03-07T02:50:53.318742751Z     "x-arr-ssl": "2048|256|CN=Microsoft Azure TLS Issuing CA 05, O=Microsoft Corporation, C=US|CN=*.azurewebsites.net, O=Microsoft Corporation, L=Redmond, S=WA, C=US",
2023-03-07T02:50:53.318746151Z     "x-forwarded-tlsversion": "1.2",
2023-03-07T02:50:53.318749251Z     "x-forwarded-for": "*******",
2023-03-07T02:50:53.318752251Z     "x-original-url": "/fromClient/2023-03-07T02:48:56.457Z",
2023-03-07T02:50:53.318755351Z     "x-waws-unencoded-url": "/fromClient/2023-03-07T02:48:56.457Z",
2023-03-07T02:50:53.318758451Z     "x-client-port": "49890"
2023-03-07T02:50:53.318761451Z }
```

実際に index.js が実行された時間は
ここで注目すべき内容は以下となります。
- 02:50:40.975 に `node index.js` 実行
- 02:50:52.056 に index.js 内の最初のログ出力
- 02:50:52.689 に /robots933456.txt へのリクエスト受信
- 02:50:53.311 に /fromClient/2023-03-07T02:48:56.457Z へのリクエスト受信

/robots933456.txt は hostヘッダからわかるように、`127.0.0.1:8020` 宛にリクエストが送られています。
これは、`docker -p 8020:8080` コマンドを実行している インスタンスの Host OS から送信されているということになります。そのため、App Service の各種アクセス制限の範囲外となります。

02:48:56にクライアント側から実行したリクエストは、02:50:53にはじめてアプリケーションに到達しています。

2つのログを合わせて確認すると以下のような順序となります。
なお各処理間にかかる時間は、インスタンスの性能であったり、アプリケーションの初期化ロジックに影響するものとなります。ここでは前後関係にのみ着目します。


|Time|Platform|ApplicationContainer|
|--|--|--|
|02:49:58.151|`docker run -d -p 7636:8080 <~省略~>`にてイメージを起動開始処理実行|  |
|02:50:12.665|warmup requestを開始|  |
|02:50:35.733|Waiting for response to warmup にて、23秒経過|  |
|02:50:40.975|  |`node index.js` 実行|
|02:50:52.056|  |index.js 内の最初のログ出力|
|02:50:52.689|  |/robots933456.txt へのリクエスト受信|
|02:50:52.895|ready to serve requests|  |
|02:50:53.311|  |/fromClient/2023-03-07T02:48:56.457Z へのリクエスト受信 へのリクエスト受信|

すなわち、App Service のプラットフォームとしては、docker コマンドで起動したアプリケーションコンテナが、`/robots933456.txt` に対して応答を返すことではじめて、アプリケーションが応答可能な状態(`ready to server requests`)となり、実際のユーザーからのリクエストがルーティングされることになります。

## トラブルシューティング

プラットフォーム側では、アプリケーションコンテナが稼働したかどうかは、一定時間 warmup リクエスト(`GET /robots933456.txt`)をリトライします。
最終的にエラーとなった場合は、以下のようなログが出力されます。

 `AppServicePlatformLogs` または `/home/LogFiles/<YYY_MM_DD>_<HostName>_docker.log`
```
2023-03-07T03:15:19.987Z INFO  - Initiating warmup request to container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a for site jpazpaas-blog-examples-linuxnodeapp-20230307
2023-03-07T03:15:39.033Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 18.2989576 sec
2023-03-07T03:16:09.103Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 49.1164098 sec
2023-03-07T03:16:26.012Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 66.0247886 sec
2023-03-07T03:16:42.828Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 82.8346583 sec
2023-03-07T03:17:00.479Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 100.4923859 sec
2023-03-07T03:17:18.697Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 118.710235 sec
2023-03-07T03:17:34.987Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 135.0003789 sec
2023-03-07T03:17:51.299Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 151.3122307 sec
2023-03-07T03:18:07.801Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 167.8145076 sec
2023-03-07T03:18:23.440Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 183.4531577 sec
2023-03-07T03:18:39.558Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 199.5713445 sec
2023-03-07T03:18:56.582Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a. Elapsed time = 216.5952289 sec
2023-03-07T03:19:10.099Z ERROR - Container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a for site jpazpaas-blog-examples-linuxnodeapp-20230307 did not start within expected time limit. Elapsed time = 230.1068051 sec
2023-03-07T03:19:10.206Z ERROR - Container jpazpaas-blog-examples-linuxnodeapp-20230307_0_e642bb9a didn't respond to HTTP pings on port: 8080, failing site start. See container logs for debugging.
2023-03-07T03:19:10.240Z INFO  - Stopping site jpazpaas-blog-examples-linuxnodeapp-20230307 because it failed during startup.
```

問題の診断と解決では、`Web アプリの再起動`　や `コンテナの問題` 、`コンテナクラッシュ` に参考情報が表示されます。
![image-a93245d5-f130-4169-913a-6dc2b9a8fd2f.png]({{site.baseurl}}/media/2023/03/image-a93245d5-f130-4169-913a-6dc2b9a8fd2f.png)


### エラー例1: アプリケーションコード内でのエラー

まずは、 `AppServiceConsoleLogs` や `/home/LogFiles/<YYY_MM_DD>_<HostName>_default_docker.log`などから、アプリケーションコードが正常に Web サーバーを起動できているか確認をしてください。

参考
 - [HTTP 502 と HTTP 503 エラーを修正する \- Azure App Service \| Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/app-service/troubleshoot-http-502-http-503)

 - [Troubleshooting Node\.js deployments on App Service Linux \-](https://azureossd.github.io/2023/02/09/troubleshooting-nodejs-deployments-on-appservice-linux/index.html#post-deployment-scenarios)



### エラー例2: アプリケーションが指定されたポートを利用して Web サーバーを起動していないケース

プラットフォーム側はアプリケーションが特定のポートを listen していることを想定してリクエストを実行します。
そのため、アプリケーション側がそのポートを利用して Web サーバーを起動していない場合 `GET /robots933456.txt` に疎通することができず、コンテナ起動に失敗と判定されます。
組み込みイメージの場合は、基本的には予め設定されている環境変数の `PORT` を使用する必要があります。カスタムイメージの場合はアプリケーション設定 `WEBSITES_PORT` を利用して変更することも可能です。

[カスタム コンテナーを構成する \- Azure App Service \| Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/app-service/configure-custom-container?tabs=debian&pivots=container-linux#configure-port-number)

PORTの利用方法については、下記記事も参考にしてください。

[Whats the difference between PORT and WEBSITES\_PORT \-](https://azureossd.github.io/2023/02/15/Whats-the-difference-between-PORT-and-WEBSITES_PORT/index.html#port-blessed-images)

### エラー例3: アプリケーションの起動に時間がかかる場合

デフォルトでは起動確認のタイムアウトは 230 秒となっています。
アプリケーション設定値 `WEBSITES_CONTAINER_START_TIME_LIMIT` を利用することで、最大 1800 秒まで延長することが可能です。

[WEBSITES\_CONTAINER\_START\_TIME\_LIMIT](https://learn.microsoft.com/ja-jp/azure/app-service/reference-app-settings?tabs=kudu%2Cpython#custom-containers)


例えば、`WEBSITES_CONTAINER_START_TIME_LIMIT` に 500 をセットした状態で、アプリケーション側にてWebサーバー起動まで 235秒遅延するようにしてみます。
```
setTimeout(() => {
  server.listen(port);
  console.log(`Server started`);
},230*1000 + 5000);
```

アプリケーション側では　Server startedが出力されるまで 235 秒かかっています。
```
2023-03-07T03:36:39.677370883Z > jpazpaas-blog-examples-linuxnodeapp-20230307@1.0.0 start /home/site/wwwroot
2023-03-07T03:36:39.677377183Z > node index.js
2023-03-07T03:36:39.677381183Z 
2023-03-07T03:36:49.323864062Z index.js started with process.env.PORT=8080
2023-03-07T03:40:44.477611509Z Server started
```

プラットフォーム側では以下のように warmup に失敗し続けますが、03:40:44にアプリ側が正常起動した以降の確認にて正常と判定されています。

```
2023-03-07T03:36:26.055Z INFO  - Initiating warmup request to container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304 for site jpazpaas-blog-examples-linuxnodeapp-20230307
2023-03-07T03:36:44.797Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 18.7491015 sec
2023-03-07T03:37:01.130Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 35.078963 sec
2023-03-07T03:37:16.867Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 50.8190484 sec
2023-03-07T03:37:33.695Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 67.6477506 sec
2023-03-07T03:37:49.107Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 83.0596245 sec
2023-03-07T03:38:09.202Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 103.1545329 sec
2023-03-07T03:38:25.324Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 119.276316 sec
2023-03-07T03:38:40.633Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 134.5835292 sec
2023-03-07T03:38:56.333Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 150.2849426 sec
2023-03-07T03:39:11.863Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 165.8133384 sec
2023-03-07T03:39:27.593Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 181.5454235 sec
2023-03-07T03:39:43.395Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 197.3478286 sec
2023-03-07T03:39:58.961Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 212.9132567 sec
2023-03-07T03:40:20.483Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 234.4350637 sec
2023-03-07T03:40:36.400Z INFO  - Waiting for response to warmup request for container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304. Elapsed time = 250.3523063 sec
2023-03-07T03:40:45.837Z INFO  - Container jpazpaas-blog-examples-linuxnodeapp-20230307_0_c320e304 for site jpazpaas-blog-examples-linuxnodeapp-20230307 initialized successfully and is ready to serve requests.
```

### エラー例4: アプリケーションは起動しているが、`GET /robots933456.txt` をハンドリングしていない

`GET /robots933456.txt` は インスタンス内部から送信されているため、各種アクセス制限は適用されません。
そのため、アプリケーションコードにて、正しいポートで正常に起動しているにも関わらず、App Service として正しく開始されていない場合は、アプリケーションコードや利用しているフレームワークにて、`GET /robots933456.txt` へのルーティングに対して何も応答を返さないような実装となっている可能性が考えられます。ローカル環境で動作させた場合に、`GET /robots933456.txt` から応答が返るかご確認ください。

また、Node.js 特有の事例とはなりますが、
`server.listen` において、`host` を指定している場合においても、正しく起動確認リクエストを受け付けられないことになります。

```javascript
server.listen(port, "localhost");
```

[Net \| Node\.js v18\.14\.2 Documentation](https://nodejs.org/docs/latest-v18.x/api/net.html#serverlisten)


# 参考記事

[App Service on Linux の組み込みコンテナイメージについて \- Qiita](https://qiita.com/georgeOsdDev@github/items/abdadd35248b98c4ef88)


2023 年 03 月 08 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
