---
title: "複数の BLOB をまとめてダウンロードする方法"
author_name: "Eiji Mochida"
tags:
    - Storage
---

# 質問
ストレージアカウント配下にコンテナーがあり、そのコンテナー配下に複数の BLOB がある。特定のパス以下の BLOB をまとめてダウンロードしたいが、Azure ポータルからだと単一の BLOB を選択してダウンロードすることしかできない。複数の BLOB をまとめてダウンロードする方法を確認したい。

# 回答
Azure Storage でコンテナー配下に BLOB を格納するというシナリオは、ご自身で手動で配置したり、何かしらのクライアントプログラムで配置したりするほか、Azure の各サービスで診断設定を利用する際に "ストレージアカウントへのアーカイブ" を選択した場合の出力先になる、など、様々なパターンで使用されます。  
  このようなシナリオの中で、複数の BLOB を出力した後、何かしらの方法で出力した BLOB の内容を確認したい、というような状況になることがございます。その際、まずは、Azure ポータルからダウンロードを試みるものの、Azure ポータルからですと単一の BLOB を選択してダウンロードする、という方法しか提供をしておらず、どうすれば一括でダウンロードできるか、というご質問を受けることがあります。こちらについてはいくつか方法がありますが、本ブログでは Storage Explorer を用いた方法を紹介いたします。


## Storage Explorer のインストール
こちらについては、以下からインストーラーがダウンロードできますので、起動してインストールを実施いただきます。

Azure Storage Explorer  (ダウンロードのリンクがあります)  
[https://azure.microsoft.com/ja-jp/products/storage/storage-explorer/](https://azure.microsoft.com/ja-jp/products/storage/storage-explorer/)

以降、Windows 11 に本ブログ作成時点での最新バージョンである Version 1.29.2 を使用した手順となります。


## ストレージアカウントへの接続
接続方法もいくつか選択肢がございますが、ストレージアカウントのサブスクリプションを参照する方法を紹介いたします。

1) Azure Storage Explorer を起動します。
2) 以下の画面のコンセントのマークを選択します。

![image-27104483-b32d-40d8-9074-e255bbd655f3.png]({{site.baseurl}}/media/2023/06/image-27104483-b32d-40d8-9074-e255bbd655f3.png)

3) サブスクリプションを選択します

![image-95955c44-5b77-4691-9cd0-3d5dfc46791e.png]({{site.baseurl}}/media/2023/06/image-95955c44-5b77-4691-9cd0-3d5dfc46791e.png)

4) どの Azure 環境を使用してサインインしますか? については、Azure を選択します

![image-13275b9c-275b-4b53-bf08-206acdb7cec4.png]({{site.baseurl}}/media/2023/06/image-13275b9c-275b-4b53-bf08-206acdb7cec4.png)

5) ブラウザでサインインを実施するよう促されますので、ストレージアカウントの参照が可能なアカウントでサインインします。サインインが完了しますとそのアカウントで参照可能なサブスクリプションの一覧と、ストレージアカウントの一覧が表示されます。

![image-8bd0d0fc-bbba-4b47-bfa9-7d264c7f2ad1.png]({{site.baseurl}}/media/2023/06/image-8bd0d0fc-bbba-4b47-bfa9-7d264c7f2ad1.png)

なお、ここまでの作業は、インストール後一度実施すれば、一度 Storage Explorer を終了してから起動しても設定は維持されますので、基本的には起動のたびに実施が必要となる作業ではございません。

## 実際のコンテナーの中身を参照してダウンロードする

上記の手順でストレージアカウントの一覧が出るようになりましたので、今度は該当のストレージアカウントの対象のコンテナーを参照し、まとめてダウンロードする方法について記載します。  
なお、今回のダウンロードの例としては、以下の資料にあるように、アクティビティログをストレージアカウントにアーカイブして、そのログがたまったという前提として記載します。ただし、アクティビティログに限定した手順ではございませんので、適宜ご自身のシナリオでコンテナー名やパス名は読み替えてください。

Azure Monitor アクティビティ ログ  
[https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/activity-log?tabs=powershell#send-to-azure-storage](https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/activity-log?tabs=powershell#send-to-azure-storage)


そして、この前提ですと、アクティビティログは insights-activity-logs というコンテナー配下に以下のようなパスで出力されます。

```
resourceId=/SUBSCRIPTIONS/{subscription ID}/y={four-digit numeric year}/m={two-digit numeric month}/d={two-digit numeric day}/h={two-digit 24-hour clock hour}/m=00/PT1H.json
```

実際にストレージアカウントにアーカイブされたアクティビティログについては、 Storage Explorer でたどると、以下のような見え方をします。赤く囲った部分がパスの情報となります。

![image-f001d662-247a-497b-927a-746bf5290698.png]({{site.baseurl}}/media/2023/06/image-f001d662-247a-497b-927a-746bf5290698.png)

この前提でそれぞれのダウンロード手順を紹介します。

### コンテナー配下をすべてダウンロード

1) Storage Explorer の左ペインで該当のストレージアカウントを選択後、"insights-activity-logs" コンテナーを選択します。
2) 右ペインの上部で "ダウンロード" - ”すべてダウンロード" を選びます。

![image-0787bb9f-a5a9-48b7-9877-e9252ca45c24.png]({{site.baseurl}}/media/2023/06/image-0787bb9f-a5a9-48b7-9877-e9252ca45c24.png)

3) どのフォルダー配下にダウンロードするかを選択することになりますので、任意のフォルダーを選択します。
4) ダウンロードを実施すると、該当のフォルダー配下に "insights-activity-logs" というフォルダーができ、その配下の BLOB がすべてローカルにダウンロードされます。以下 logs というローカルのフォルダーにダウンロードした例となります。

![image-f661bb20-8741-43e2-8e79-4c0c1b3c62a6.png]({{site.baseurl}}/media/2023/06/image-f661bb20-8741-43e2-8e79-4c0c1b3c62a6.png)


### 特定のパス配下のみをダウンロード
例えば、特定の日付のアクティビティログをダウンロードしたいという場合には、"resourceId=/SUBSCRIPTIONS/<サブスクリプションID>/y=2023/m=05/d=29/" というようなパスを指定することになります。その場合、コンテナー配下のダウンロードとほぼ同様の手順ですが、対象のパスまでたどってから実施という点だけ異なります。以下に手順を記載します。

1) Storage Explorer の左ペインで該当のストレージアカウントを選択後、"insights-activity-logs" コンテナーを選択します。
2) 右ペインでフォルダーが出ますので、順に "resourceId=/SUBSCRIPTIONS/<サブスクリプションID>/y=2023/m=05/d=29/" と辿ります。画面上は d=29 配下に h=18,h=19,h=21,h=22 という 4 つのフォルダーがあるような見え方となります。(あくまで今回の例ですので、フォルダー数が違うなどがあってもその点は気にしないでください。)
3) 右ペインの上部で "ダウンロード" - ”すべてダウンロード" を選びます。

![image-a57b44ed-d9b2-4fbd-840f-0cc820bca57b.png]({{site.baseurl}}/media/2023/06/image-a57b44ed-d9b2-4fbd-840f-0cc820bca57b.png)

4) どのフォルダー配下にダウンロードするかを選択することになりますので、任意のフォルダーを選択します。
5) ダウンロードを実施すると、該当のフォルダー配下に "d=29" というフォルダーができ、その配下の BLOB がすべてローカルにダウンロードされます。以下 logs2 というローカルのフォルダーにダウンロードした例となります。

![image-e9643730-351a-419f-8bef-e6247f2121e6.png]({{site.baseurl}}/media/2023/06/image-e9643730-351a-419f-8bef-e6247f2121e6.png)


## うまくダウンロードできない場合には
必要な権限があり、アクセスの経路も問題なければ上記の手順が実施できますが、うまくダウンロードできない、という場合もあるかもしれません。
トラブルシューティングの話について細かく書くとボリュームが多くなるので、本ブログでは概略にとどめますが、よくあるパターンとしては、

- サインインしたユーザーに BLOB にアクセスする適切な権限がない
- ストレージアカウントのファイアウォールなどでアクセスを許可するクライアントを制限していて、Storage Explorer からのアクセスの経路が許可されていない

というものが挙げられます。うまくアクセスができない、という場合にはまずは以下の資料をご覧いただき、何か思い当たる点がないかをご確認いただくことをお勧めいたします。

Azure portal で BLOB データへのアクセスの承認方法を選択する  
[https://learn.microsoft.com/ja-jp/azure/storage/blobs/authorize-data-operations-portal](https://learn.microsoft.com/ja-jp/azure/storage/blobs/authorize-data-operations-portal)


Azure Storage ファイアウォールおよび仮想ネットワークを構成する  
[https://learn.microsoft.com/ja-jp/azure/storage/common/storage-network-security?tabs=azure-portal](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-network-security?tabs=azure-portal)


# 参考ドキュメント

Azure Storage Explorer  (ダウンロードのリンクがあります)  
[https://azure.microsoft.com/ja-jp/products/storage/storage-explorer/](https://azure.microsoft.com/ja-jp/products/storage/storage-explorer/)

クイック スタート:Azure Storage Explorer を使用して BLOB を作成する  
[https://learn.microsoft.com/ja-jp/azure/storage/blobs/quickstart-storage-explorer](https://learn.microsoft.com/ja-jp/azure/storage/blobs/quickstart-storage-explorer)



<br>
<br>

---

<br>
<br>

2023 年 06 月 01 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
