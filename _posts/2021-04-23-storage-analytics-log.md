---
title: "Storage Analytics ログについて"
author_name: "Eiji Mochida"
tags:
    - Storage
---

# 質問
自身のストレージアカウントに対するアクセスの情報を確認したい。

# 回答
Azure Storage におきましては Storage Analytics ログを有効にすることで、どのようなアクセスがあったかを確認することができます。(Azure Monitor の Azure Storage ログ が Public Preview ではありますが、本 Blog では Storage Analytics ログを紹介します。) 

まず、Storage Analytics ログは既定では有効になっておりませんので、アクセスの状況を確認したいストレージアカウントで有効にする必要がございます。参考ドキュメント [1.](https://docs.microsoft.com/ja-jp/azure/storage/common/storage-analytics-logging?tabs=dotnet) にあるとおり、Azure ポータルなどから有効にすることができます。なお、ログの形式として、1.0 と 2.0 がございますが、2.0 は 1.0 のフィールドすべてに加え、追加のフィールドの出力がございますので、新たに設定するなら 2.0 にしておくことをお勧めいたします。ログに何が出るのかという点については、参考ドキュメント [2.](https://docs.microsoft.com/ja-jp/rest/api/storageservices/storage-analytics-log-format) をご覧ください。

# よくあるご質問
**(Q1)<BR/>
 Storage Analytics ログからどのようなことが確認できるのか。<br/>**

(A1)<BR/>
どのような内容が記録されるかは参考ドキュメント [2.](https://docs.microsoft.com/ja-jp/rest/api/storageservices/storage-analytics-log-format) に記載がございますが、概略でお伝えすると、アクセスした時刻、操作内容、アクセスした URL、アクセス元 クライアント IP、User-Agent、アクセスの結果(正常応答だったのか、アクセス拒否などでアクセスできなかったのかなど) といったものが記録されるので、そこから、どのようなクライアントがどのパスにアクセスし、その結果アクセスがうまく行ったのか、などを確認するのに役立てていただけます。


**(Q2)<BR/>
 操作したユーザー名を確認することができるか。<BR/>**

(A2)<BR/>
こちらは、アクセスに Azure AD 認証を用いた場合で、ログのバージョンを 2.0 にしている場合は確認ができます。例えば、Azure PowerShell などからストレージアカウントにアクセスしたとき、Azure PowerShell で Connect-AzAccount を実行したユーザー名が xxxxx@yyyyy.com だったとすると、Storage Analytics ログのバージョン 2.0 のみに存在するフィールドである UserPrincipalName に xxxxx@yyyyy.com という記録が残ります。
一方、Shared Access Signature(SAS) や アクセスキーを用いた場合はユーザーに紐づくアクセスではないので、ログからアクセス元のユーザーを確認することはできません。

**(Q3)<BR/>
 コンテナのアクセスレベルがプライベートで匿名アクセスを受け付けていない時に、ブラウザの URLに Blob のパスを入力してアクセスを実施後、Storage Analytics ログを確認したが対応するログが出ていないのはなぜか。**

(A3)<BR/>
例えば、以下のようにアクセスレベルがプライベートのコンテナ配下にあるファイルにアクセスしようとすると、以下のように ResourceNotFound が返却されます。この場合、匿名アクセスで拒否された状況となりますので、Storage Analytics ログにはログが残りません。参考ドキュメント 参考ドキュメント [1.](https://docs.microsoft.com/ja-jp/azure/storage/common/storage-analytics-logging?tabs=dotnet) の先頭の、"匿名要求のログ記録" の部分にある、「その他の失敗した匿名要求は一切記録されません。」という部分にあたります。なお、アクセス拒否ですので、コンテンツの内容が参照されたのにログが残っていない、ということではございません。

![image.png]({{site.baseurl}}/media/2021/04/2021-04-23-storage-analytics-log-xml.png)

なお、匿名アクセスが許可されていないことによるアクセス拒否となりますが、エラーとして ResourceNotFound になるのは、リソースがある場合だけアクセス拒否を示すようなエラーを返すと、リソースの内容にはアクセスできなくても、リソースの存在有無が確認できてしまうことを避けるためです。(上記の例においては対象の URL に対応する Blob は存在しています。)

**(Q4)<BR/>
 Storage Analytics ログをみるのに便利な見方はあるか。<br/>**

(A4)<BR/>
Storage Analytics ログは ";" で区切られたテキストファイルですので、普段ご利用いただいているエディタなどでご覧いただけます。
また、参考ドキュメント [2.](https://docs.microsoft.com/ja-jp/rest/api/storageservices/storage-analytics-log-format) にはログのバージョン 1.0、2.0 ともに ; 区切りでどのような列が出るかを記載した部分がございますので、エディタでログを開いた後、参考ドキュメント [2.](https://docs.microsoft.com/ja-jp/rest/api/storageservices/storage-analytics-log-format)から ";" 区切りの列名をコピー＆ペーストして保存し、そのファイルを Excel で開いて ; 区切りのファイルとして読み込ませることもできます。
また、弊社社員が [AzureStorageLogReader](https://nunomo.github.io/AzureStorageLogReader/) というツールを公開しておりますので、このようなツールをご利用いただくことも可能です。

# 参考ドキュメント
1. Azure Storage Analytics のログ<br/>
https://docs.microsoft.com/ja-jp/azure/storage/common/storage-analytics-logging?tabs=dotnet

2. Storage Analytics ログの形式<br/>
https://docs.microsoft.com/ja-jp/rest/api/storageservices/storage-analytics-log-format

3. Storage Analytics によって記録される操作やステータス メッセージ<br/>
https://docs.microsoft.com/ja-jp/rest/api/storageservices/storage-analytics-logged-operations-and-status-messages

4. AzureStorageLogReader<br/>
https://nunomo.github.io/AzureStorageLogReader/

<br>
<br>

---

<br>
<br>

2021 年 4 月 23 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>