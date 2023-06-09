---
title: "Shared Access Signature (SAS) トークンの失効方法"
author_name: "Haruka Matsumura"
tags:
    - Storage
---

<br>

# はじめに
本日はストレージ アカウントの Shared Access Signature (SAS) トークンを失効させる方法についてご紹介いたします。なお、SAS トークンは以下の 3 つの種類がございます。<br>

- ユーザー委任 SAS
- アカウント SAS
- サービス SAS

Shared Access Signatures (SAS) でデータの制限付きアクセスを付与する - Azure Storage | Microsoft Learn<br>
<https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview>

それぞれの SAS トークンの種類ごとに利用いただける SAS トークンの発行方法により失効方法が異なりますので、ご利用の SAS の発行方法に合わせて内容をご参照ください。

なお、SAS トークン自体は Azure 基盤側で生成の上で発行されるものではなく、お手元のプログラムなどで公開情報上の生成規則に従って発行することも可能であり、Azure 基盤側にて管理される情報ではございません。
そのため、以降のすべてのシナリオに該当いたしますが、特定の SAS のみ失効させることや、これまでに発行された SAS を Azure 基盤側の情報から網羅的に確認する方法はございません。本記事の内容を参考に、万が一 SAS トークンが流出した場合にどのような手順が必要になるか、SAS の失効が与える影響範囲を踏まえどの SAS トークンを利用すべきか、など、事前にセキュリティの観点からの慎重なご検討をいただければ幸いです。<br>
<br>

# シナリオ A : Azure AD (ユーザー委任キー) を利用して発行された場合
こちらは、SAS 発行時にユーザー委任キーを利用し、ユーザー委任 SAS を発行いただいた場合の失効方法です。

(Azure Portal でコンテナーを選択して SAS の生成を行う際の画面キャプチャ)<br>
![2023-06-06_19h28_16.png]({{site.baseurl}}/media/2023/06/2023-06-06_19h28_16.png)

本シナリオの場合、Azure PowerShell または Azure CLI からコマンドを実行することで、対象のストレージ アカウントに関連付けられているすべてのユーザー委任キーを取り消し、ユーザー委任 SAS の失効を行うことが可能です。

実行するコマンドは以下となります。(山かっこ内のプレースホルダーはお客様環境の値に置き換えてください。)

- Azure PowerShell: <br>
`Revoke-AzStorageAccountUserDelegationKeys -ResourceGroupName <resource-group> -StorageAccountName <storage-account>`<br>
<https://learn.microsoft.com/ja-jp/azure/storage/blobs/storage-blob-user-delegation-sas-create-powershell#revoke-a-user-delegation-sas>
- Azure CLI:<br>
`az storage account revoke-delegation-keys --name <storage-account>  --resource-group <resource-group>`<br>
<https://learn.microsoft.com/ja-jp/azure/storage/blobs/storage-blob-user-delegation-sas-create-cli#revoke-a-user-delegation-sas>

<br>
<b>なお、注意点として、特定のユーザー委任キーのみを取り消すことはできず、すべてのユーザー委任キーが取り消されるため、ユーザー委任キーを利用して発行された SAS がすべて失効となります。</b> ユーザー委任キーの取り消し後、必要に応じて、再度新たなユーザー委任 SAS を生成ください。<br>
<br>

# シナリオ B : アクセス ポリシーを設定してアカウント キーを利用して発行された場合
こちらは SAS 発行時に、以下のように “アクセス ポリシー” を指定して SAS を発行した場合のシナリオです。(アカウント キーは、アクセス キーや共有キーとも呼称される場合がございますが、いずれも同じものを指します。)

(Azure Portal でコンテナーを選択して SAS の生成を行う際の画面キャプチャ)<br>
![2023-06-06_19h28_50.png]({{site.baseurl}}/media/2023/06/2023-06-06_19h28_50.png)

アクセス ポリシーの利用により、SAS トークンの開始時刻、有効期限、およびアクセス許可をグループ化して管理いただくことが可能となります。アクセス ポリシーの詳細に関しましては、下記リンク先の参考情報をご高覧くださいませ。

格納されているアクセス ポリシーを定義する - Azure Storage | Microsoft Learn<br>
<https://learn.microsoft.com/ja-jp/rest/api/storageservices/define-stored-access-policy>

<br>
本シナリオの場合、参照されているアクセス ポリシーを

- 削除する
- ポリシーの名前を変更する
- 有効期限を過去の時間に変更し、すでに期限切れとする

のいずれかを実施いただくことで、対象のアクセス ポリシーを参照している SAS トークンを失効させることが可能です。
 
ただし、削除もしくは名前変更の場合、後日、<b>削除前もしくは名前変更前のポリシーとまったく同じ名前でアクセス ポリシーを再作成してしまった際に、新たに作成された同名のアクセス ポリシーのアクセス許可および有効期限に従って、対象の名前のアクセス ポリシーを参照していた既存のすべての SAS トークンが再び有効となってしまう恐れ</b>がございます。加えて、過去に削除したアクセス ポリシーの名前の一覧を、(別途お客様のデータベースなどで管理している場合を除き) Azure 基盤側からさかのぼって情報をご確認いただくことは叶いません。

別のアクセス ポリシーを作成される場合に名前変更や削除済みの同名のアクセス ポリシーを作成してしまうことを防ぐため、有効期限を過去の時間に変更いただき、期限切れのアクセスポリシーにする方法で失効いただくことをおすすめいたします。<br>
<br>

# シナリオ C : アクセス ポリシーを設定せずアカウント キーを利用して発行された場合
シナリオ A およびシナリオ B いずれのシナリオにも当てはまらず SAS の発行をいただいた場合、失効には署名文字列であるアカウント キーをローテーションし、新しいアカウント キーを再生成いただくしかございません。
しかしながら、アカウント キーの再生成では、<b>対象のアカウント キーを使用して署名されていたすべての SAS トークンだけではなく、対象のアカウント キーそのものの情報で認証されていた接続もすべて失効し、</b>対象のストレージ アカウントに対して行っている他の接続にも大きな影響を与える可能性がございます点、あらかじめご留意ください。

アカウント キーの再生成の手順に関しましては、下記リンク先に情報がございますのでこちらをご紹介いたします。

アカウントのアクセス キーを管理する - Azure Storage # アクセス キーを手動でローテーションする |  Microsoft Learn<br>
<https://learn.microsoft.com/ja-jp/azure/storage/common/storage-account-keys-manage?tabs=azure-portal#manually-rotate-access-keys><br>
<br>
<br>

---

<br>
<br>

2023 年 06 月 09 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>