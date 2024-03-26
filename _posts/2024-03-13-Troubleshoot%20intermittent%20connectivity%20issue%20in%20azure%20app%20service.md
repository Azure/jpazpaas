---
title: "[日本語訳]App Service における断続的な接続問題のトラブルシューティング"
author_name: "Tsutomu Mori"
tags:
    - App Service
---

# はじめに
お世話になっております。App Service サポート担当の森です。


本記事は [Apps on Azure Blog](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/bg-p/AppsonAzureBlog) にて2023 年 11 月に公開されている [Troubleshoot intermittent connectivity issue in azure app service](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/troubleshoot-intermittent-connectivity-issue-in-azure-app/ba-p/3969841#user-content-tcpping) の日本語抄訳としてご案内いたします。

# App Service における断続的な接続問題のトラブルシューティング

Azure App Service 上で動作しているアプリケーションでは、断続的な接続問題によりターゲットサービスへの接続時にエラーが発生することがあります。その問題を特定するためにコードを変更してデプロイするのは手間がかかります。App Service Kudu サイトを使用すれば迅速にテストを行うことができ、その場合コードを変更する必要はありません。

## Linux

Kudu サイト（例 : `https://<appname>.scm.azurewebsites.net`）にログインし、SSH メニューへ移動すると SSH Shell を確認できます。カスタムイメージを使用している場合は、次のドキュメントに従ってSSHを有効にする必要があります：[Azure App Service のカスタム コンテナーを構成する](https://learn.microsoft.com/ja-jp/azure/app-service/configure-custom-container?tabs=debian&pivots=container-linux#enable-ssh)

### nslookup

`nslookup` は通常、Linux Kudu イメージに既にインストールされており、FQDN が正しい IP アドレスに解決されているかを確認することができます。
以下では、`login.microsoftonline.com` に DNS 解決を行っていますが、任意のアドレスに変更することができます。

```bash
# nslookup command to test DNS resolution for login.microsoftonline.com, you could change it to other destination
nslookup login.microsoftonline.com
Server:         127.0.0.11
Address:        127.0.0.11#53

Non-authoritative answer:
login.microsoftonline.com       canonical name = login.mso.msidentity.com.
login.mso.msidentity.com        canonical name = ak.privatelink.msidentity.com.
ak.privatelink.msidentity.com   canonical name = www.tm.ak.prd.aadg.trafficmanager.net.
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 20.190.166.67
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 40.126.38.22
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 20.190.166.133
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 20.190.166.68
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 40.126.38.21
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 20.190.166.132
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 20.190.166.66
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 40.126.38.19
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 2603:1046:2000:148::4
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 2603:1046:2000:158::3
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 2603:1046:2000:148::3
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 2603:1047:1:150::1
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 2603:1046:2000:148::5
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 2603:1046:2000:148::2
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 2603:1047:1:150::3
Name:   www.tm.ak.prd.aadg.trafficmanager.net
Address: 2603:1047:1:150::2
```

nslookup がインストールされていない場合は、以下のコマンドを実行してください。

```bash
apt update
apt install dnsutils
```

`nslookup` のテストを繰り返し実行する場合は、以下のスクリプトをファイル（例 : `dnstest.sh`）に保存して利用することができます。パラメータは必要に応じて調整することができます。

```bash
fqdn=$1 # get the first parameter
max=$2  # get the second parameter
for ((i = 0 ; i < max ; i++ )); 
do 
   now=$(date +"%Y-%m-%d %T")
   echo "$now Progress $i / $max" # show the progress
   nslookup $fqdn            # execute the command
   sleep 1                   # sleep 1 second
done
```

以下のコマンドで実行します。

```bash
bash dnstest.sh login.microsoftonline.com 10
```


以下のコマンドで実行結果をログファイルに出力します。

```bash
bash dnstest.sh login.microsoftonline.com 10 > output.log
```

### dig

`dig` もデフォルトでインストールされています。DNS 解決の詳細な情報を確認することができます。

```bash
dig login.microsoftonline.com
```

`dig` のテストを繰り返し実行する場合は、以下のスクリプトをファイル（例 : `digtest.sh`）に保存して利用することができます。


```bash
fqdn=$1 # get the first parameter
max=$2  # get the second parameter
for ((i = 0 ; i < max ; i++ )); 
do 
   now=$(date +"%Y-%m-%d %T")
   echo "$now Progress $i / $max" # show the progress
   dig $fqdn                 # execute the command
   sleep 1                   # sleep 1 second
done
```

 
以下のコマンドで実行します。

```bash
bash digtest.sh login.microsoftonline.com 10
bash digtest.sh login.microsoftonline.com 10 > output.log
```

### NodeJS

NodeJS ランタイムを使用している場合、以下のコマンドを使用して DNS 解決に要した時間を取得できます。

```bash
node -e "const dns = require('dns'); console.time('lookup_time'); dns.lookup('login.microsoftonline.com', (err, out) => { console.timeEnd('lookup_time'); console.log(err, out)});"
```

### Python

以下のスクリプトを `dnstest.py` に保存します。 
 
```python
import sys
import socket
import time
import datetime

fqdn = sys.argv[1]
total = int(sys.argv[2])

def get_ipv4_by_hostname(hostname):
    # see `man getent` `/ hosts `
    # see `man getaddrinfo`

    return list(
        i        # raw socket structure
            [4]  # internet protocol info
            [0]  # address
        for i in
        socket.getaddrinfo(
            hostname,
            0  # port, required
        )
        if i[0] is socket.AddressFamily.AF_INET  # ipv4

        # ignore duplicate addresses with other socket types
        and i[1] is socket.SocketKind.SOCK_RAW
    )
for i in range(0, total):
    now = datetime.datetime.now()
    print(str(now) + " testing " + fqdn)
    print(get_ipv4_by_hostname(fqdn))
    time.sleep(1)
```


以下のコマンドで実行します。

```bash
python dnstest.py www.google.com 10
python dnstest.py www.google.com 10 > output.log
```

### tcpping

`tcpping` はインストールされていない可能性があります。その場合は以下に従ってインストールしてください。

```bash
apt-get update
apt-get install bc
apt-get install tcptraceroute  
cd /usr/bin  
wget http://www.vdberg.org/~richard/tcpping
chmod 755 tcpping  
```

以下のコマンドで接続をテストします。
 
```bash
tcpping login.microsoftonline.com 443
# 以下のようなレスポンスが確認できます。
seq 0: tcp response from 20.190.144.161 [open]  67.911 ms
seq 1: tcp response from 20.190.163.20 [open]  2.130 ms
seq 2: tcp response from 20.190.144.161 [open]  70.361 ms
seq 3: tcp response from 20.190.144.137 [open]  68.137 ms
seq 4: tcp response from 20.190.148.163 [open]  69.158 ms
seq 5: tcp response from 40.126.35.19 [open]  1.443 ms
seq 6: tcp response from 20.190.148.162 [open]  69.235 ms
seq 7: tcp response from 20.190.144.139 [open]  69.703 ms
```

デフォルトでは `tcpping` は無限に繰り返されます。`-x` パラメーターを使用してリピートする回数を指定してください。
 
```bash
tcpping -x 100 www.google.com 443 
```
 
### curl

`curl` コマンドでは、クライアントをシミュレートしてターゲットサービスからデータを取得することができます。 例えば、API などのターゲットアプリケーションをテストすることが可能です。postman-echo サービスを使用して `curl` がターゲットサービスに正しいデータを送信するかを確認し、その後サービスをターゲット API サービスに変更できます。

```bash
curl -v https://login.microsoftonline.com

curl  https://postman-echo.com/get?name=value
# 応答結果例
{
  "args": {
    "name": "value"
  },
  "headers": {
    "x-forwarded-proto": "https",
    "x-forwarded-port": "443",
    "host": "postman-echo.com",
    "x-amzn-trace-id": "Root=1-6508142c-47c6ba1a402278df3911d701",
    "user-agent": "curl/7.74.0",
    "accept": "*/*"
  },
  "url": "https://postman-echo.com/get?name=value"
}

curl -X POST -H "Content-Type: application/json" -d '{"name": "name1", "email": "test@example.com"}' https://postman-echo.com/post
# 応答結果例
{
  "args": {},
  "data": {
    "name": "name1",
    "email": "test@example.com"
  },
  "files": {},
  "form": {},
  "headers": {
    "x-forwarded-proto": "https",
    "x-forwarded-port": "443",
    "host": "postman-echo.com",
    "x-amzn-trace-id": "Root=1-650814f6-5bfb1beb60fbc2e47a4a81ce",
    "content-length": "46",
    "user-agent": "curl/7.74.0",
    "accept": "*/*",
    "content-type": "application/json"
  },
  "json": {
    "name": "name1",
    "email": "test@example.com"
  },
  "url": "https://postman-echo.com/post"
}
```

`curl` のテストを繰り返し実行する場合は、以下のスクリプトをファイル（例 : `curltest.sh`）に保存して利用することができます。パラメータは必要に応じて調整することができます。

```bash
fqdn=$1 # get the fqdn parameter
port=$2 # get the port parameter
max=$3  # get the loop count

for ((i = 0 ; i < max ; i++ )); 
do 
   now=$(date +"%Y-%m-%d %T")
   echo "$now Progress $i / $max" # show the progress
   curl -v $fqdn $port            # execute the command
   sleep 1                   # sleep 1 second
done
```


以下のコマンドで実行します。

```
bash curltest.sh login.microsoftonline.com 443 10
bash curltest.sh login.microsoftonline.com 443 10 > output.log
```

### tcpdump

接続が失敗し、その原因が不明な場合、`tcpdump` を使用してネットワークトレースをキャプチャすることができます。

```bash
# Ubuntu/Jessie ベースのイメージへのインストール方法
apt-get update
apt install tcpdump

# Alpine ベースのイメージへのインストール方法
apk update
apk add tcpdump

# 実行方法
tcpdump -w /home/site/wwwroot/traffic.pcap
```
 
これで `tcpdump` はネットワークパケットをキャプチャするようになりました。別のタブで Kudu サイトを開き、問題を再現するためのコマンドを実行してください。そして、`tcpdump` を実行しているタブに戻り、`ctrl+C` を押してキャプチャを停止します。 

`https://<appname>.scm.azurewebsites.com/newui/fileManager` にアクセスして、`traffic.pcap` ファイルをダウンロードします。

![image-3eeef4b2-0366-4bf1-8ba9-80d9a3369a3e.png]({{site.baseurl}}/media/2024/03/image-3eeef4b2-0366-4bf1-8ba9-80d9a3369a3e.png)
	
ダウンロードが成功したら、[wireshark](https://www.wireshark.org/) を使用してネットワークトレースファイルを分析することができます。

##　Windows

Kuduサイト `https://<appname>.scm.azurewebsites.net/` にログインし、「Debug console」->「PowerShell」メニューに移動すると、PowerShell ウィンドウが表示されます。

### nameresolver.exe

`nameresolver.exe` は `nslookup` と同様に DNS サーバーに対して DNS lookup を行います。

```ps
nameresolver.exe login.microsoftonline.com 
# 応答例
Server: 168.63.129.16

Non-authoritative answer:
Name: www.tm.ak.prd.aadg.trafficmanager.net
Addresses: 
	40.126.35.87
	20.190.163.18
	40.126.35.128
	40.126.35.144
	40.126.35.129
	40.126.35.134
	40.126.35.19
	40.126.35.21
Aliases: 
	login.mso.msidentity.com
	ak.privatelink.msidentity.com
	www.tm.ak.prd.aadg.trafficmanager.net

# DNSサーバーを指定することもできます
nameresolver.exe login.microsoftonline.com 8.8.8.8
```


nameresolver のテストを繰り返し行うために、ターゲットドメインを継続的に解決するループを記述し、断続的な問題を確認することができます。 例えば、以下のファイルを `nameresolvertest.ps1` に保存します。

```ps
param (
    [string]$fqdn, 
    [int]$max = 10
)

for ($i = 0; $i -lt $max; $i++) {
    $now = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")
    Write-Host "$now Progress $i / $max"
    nameresolver.exe $fqdn 
    Start-Sleep 1
}
```
	 
以下のコマンドで実行します。

```ps
powershell ./nameresolvertest.ps1 www.google.com 10
powershell ./nameresolvertest.ps1 www.google.com 10 > output.log
```

### tcpping.exe
		
`tcpping.exe` は、特定のポートでターゲットサービスへの接続が機能しているかをテストできます。	
サーバーとポートの間には ":" の記号があることに注意してください。これは Linux 環境の `tcpping` とは異なります。 Linux では空白 " " が使われます。
		
```ps
tcpping login.microsoftonline.com:443

# 応答例
tcpping login.microsoftonline.com:443
Connected to login.microsoftonline.com:443, time taken: 139ms
Connected to login.microsoftonline.com:443, time taken: 74ms
Connected to login.microsoftonline.com:443, time taken: 62ms
Connected to login.microsoftonline.com:443, time taken: 78ms
Complete: 4/4 successful attempts (100%). Average success time: 88.25ms

# 実行回数を指定
tcpping login.microsoftonline.com:443 -n 10

# 失敗応答例1
tcpping xxxxxxxxxx.redis.cache.windows.net:6380
Connection attempt failed: No such host is known
Connection attempt failed: No such host is known
Connection attempt failed: No such host is known
Connection attempt failed: No such host is known
Complete: 0/4 successful attempts (0%). Average success time: 0ms

# 失敗応答例2
tcpping xxxxxxxxxx.redis.cache.windows.net:6380
Connection attempt failed: Connection timed out.
Connection attempt failed: Connection timed out.
Connection attempt failed: Connection timed out.
Connection attempt failed: Connection timed out.
Complete: 0/4 successful attempts (0%). Average success time: 0ms
```		
					
`tcpping` を使用して、各サービスが利用するポート毎にテストすることができます。
		
```ps
tcpping <sqlservername>.database.azure.com:1433
tcpping <mysqlname>.database.azure.com:3389
tcpping <postgresqlname>.postgres.database.azure.com:5432
tcpping <redisname>.redis.cache.windows.net:6380
```	

Windows の `tcpping` では、デフォルトの ping 回数は 4 回ですが、-n パラメータを指定してターゲットを複数回 ping することができます。例えば、以下のスクリプトは `tcpping` を使ってターゲットを 100 回呼び出します。
			
```ps
tcpping www.google.com:443 -n 100
```

### curl.exe

Windows Kudu サイトでは、`curl` コマンドは PowerShell の `Invoke-WebRequest` コマンドレットを呼び出しますが、PowerShell のコマンドレットの代わりにスクリプト内で  "curl.exe" を使用することができます。
	
```ps
curl.exe -v https://login.microsoftonline.com
curl.exe  https://postman-echo.com/get?name=value
```

`curl.exe` は 応答で赤文字を返すことがありますが、それ自体に問題はありません。正しい内容が返ってきていれば、それは接続が良好であることを示しています。


![image-1733edee-44e6-4c5f-b8a3-0c9ddcc1152a.png]({{site.baseurl}}/media/2024/03/image-1733edee-44e6-4c5f-b8a3-0c9ddcc1152a.png)

### Webjob

バックエンドで実行するための PowerShell スクリプトの Webjob を作成することもできます。これは、Kudu コンソールから実行するよりも安定しています。

次の例をローカルファイル （例 : `pswebjob.ps1`）に保存してください。

```ps
while($true){
    Write-Output (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")
    Write-Output "Hello World from PowerShell WebJob!"
    curl.exe -v https://www.google.com
    Start-Sleep -Seconds 1
}
```

PowerShell スクリプト内で、`Write-Host` を `Write-Output` に変更する必要があることに注意してください。変更しない場合、ハンドルエラーが発生します。

継続的な Webjob を作成する方法については、以下のドキュメントを参照してください。

[Web ジョブでバックグラウンド タスクを実行する - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/webjobs-create#CreateContinuous)

Webjob が作成されると、以下のようなログが確認できます。

![image-d33b9201-5168-4481-9235-5d59e574cceb.png]({{site.baseurl}}/media/2024/03/image-d33b9201-5168-4481-9235-5d59e574cceb.png)

## 参考資料
- [Azure App Service仮想ネットワーク統合のトラブルシューティング ガイド - Azure ](https://learn.microsoft.com/ja-jp/troubleshoot/azure/app-service/troubleshoot-vnet-integration-apps#troubleshoot-outbound-connectivity-on-windows-apps)
- [Quickly test connectivity from Azure Website to SQL DB - Microsoft Community Hub](https://techcommunity.microsoft.com/t5/azure-database-support-blog/quickly-test-connectivity-from-azure-website-to-sql-db/ba-p/368868)
- [Networking Related Commands for Azure App Services - Microsoft Community Hub](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/networking-related-commands-for-azure-app-services/ba-p/392410)
- [Installing TcpPing on Azure App Service Linux](https://azureossd.github.io/2021/06/17/installing-tcpping-linux/)
- [https://gist.github.com/cnDelbert/5fb06ccf10c19dbce3a7](https://gist.github.com/cnDelbert/5fb06ccf10c19dbce3a7 ) 


---

<br>
<br>

2024 年 03 月 26 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>