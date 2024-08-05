---
title: "BLOB やコンテナーのリース状態を、[解約済み] から [利用可能] にする方法"
author_name: "Eiji Mochida"
tags:
    - Storage
---


# 質問
Azure ポータルで BLOB やコンテナーのリース状態を操作していたら、[解約済み] というステータスになってしまいました。これを他のコンテナーなどと同じように [利用可能] に戻すにはどうすればいいですか。

![image-fb39932f-7feb-41c5-a2f5-1579dbf20f82.png]({{site.baseurl}}/media/2024/08/image-fb39932f-7feb-41c5-a2f5-1579dbf20f82.png)
# 回答

## 手順
手順自体は以下の方法を取ります。ここではコンテナーを対象に作業を行います。  
実施している概要としては、一度リースを取得しなおし、リース ID を指定してリースを解放するという作業をしています。
実行環境としては Azure ポータルで Cloud Shell を利用し、ストレージへの認証方法はアクセスキーを利用し、Azure CLI を用いて実施しています。あくまで一例ですので、実行環境や認証方法などはお手元の状況に応じて調整ください。

```
account=<ストレージアカウント名>
container=<コンテナー名>
key="ストレージのアクセスキー"

az storage container lease acquire --container-name $container --account-name $account --auth-mode key --account-key $key
※ 上記コマンドの結果としてリース ID が返ります。次のコマンドの --lease-id にその値を指定します。

az storage container lease release  --container-name $container --account-name $account --auth-mode key --account-key $key --lease-id "xxx"
```

以下は Cloud Shell で実際に実行した例です。この例ですと、リース取得時に a0c15742-0135-4fc5-9e08-994c2fce4898 というリース ID を取得しているので、その内容を az storage container lease release で指定しています。実行しているコマンドのパラメータなどについては参考ドキュメント(1) をご覧ください。

![image-04381a85-e6c3-4c29-9e00-a8dbaea15976.png]({{site.baseurl}}/media/2024/08/image-04381a85-e6c3-4c29-9e00-a8dbaea15976.png)

## 補足

ポータルの操作で[解約済み] となる理由ですが、リース中のコンテナーに対して実施できる作業として、リースID を指定した release、リース ID を指定しない break の処理があり、ポータルで該当のコンテナーを選択し、「リースの解約」を選ぶと、break の処理を実施することになります。そして break されたリースの状態は、[解約済み] になります。  
一方先述のように CLI で az storage container lease release を用いてリースID を指定した場合はコマンドにあるように release の処理となるため、リースの状態は [利用可能] になります。リースの状態遷移については後述の参考ドキュメント (2) をご覧ください。


![image-4f880fa6-41a3-4082-bb40-a76ab89e8b69.png]({{site.baseurl}}/media/2024/08/image-4f880fa6-41a3-4082-bb40-a76ab89e8b69.png)



# 参考ドキュメント
(1) [az storage container lease](https://learn.microsoft.com/ja-jp/cli/azure/storage/container/lease?view=azure-cli-latest#az-storage-container-lease-acquire)

(2) [Lease Blob - リースの状態](https://learn.microsoft.com/ja-jp/rest/api/storageservices/lease-blob?source=recommendations&tabs=microsoft-entra-id#lease-states)


<br>
<br>

---

<br>
<br>

2024 年 07 月 31 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>