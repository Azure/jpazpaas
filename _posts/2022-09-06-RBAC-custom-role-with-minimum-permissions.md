---
title: "アクセス制御 (IAM) で App Service の操作に必要な最低限の権限を付与する"
author_name: "Hiroaki Machida"
tags:
    - App Service
---

# 質問

App Service でスケールイン/アウトやデプロイなどの操作に必要な最低限の権限をユーザーに付与したい。どのように設定すればよいでしょうか。

# 回答

## アクセス制御 (IAM) の概要

アクセス制御 (IAM) の機能では、ユーザー、グループ、サービス プリンシパル、またはマネージド ID に、特定のスコープでロールを割り当てることができます。ロールには、対象のリソースに対して読み取り、書き込み、削除などを実行するアクセス許可が設定されています。

![RBAC-custom-role-with-minimum-permissions4-28dece5e-04cd-4d76-a694-a073e3247d1c.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions4-28dece5e-04cd-4d76-a694-a073e3247d1c.png)

### スコープ

アクセス許可が適用されるリソースのセットを設定できます。これをスコープと呼びます。<br/>
Azure では、範囲の広いものから順に、管理グループ、サブスクリプション、リソース グループ、リソースの 4 つのレベルでスコープを指定できます。 

### ロール

読み取り、書き込み、削除などの実行できるアクセス許可のコレクションを設定することができます。これをロールと呼びます。

### 組み込みロール

Azure では、あらかじめ特定のアクセス許可が設定されたロールが用意されています。これを組み込みロールと呼びます。

[Azure 組み込みロール](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles)

### カスタム ロール

組み込みロールが組織の特定のニーズを満たさない場合は、独自のロールを作成することができます。これをカスタム ロールと呼びます。

## ロールを割り当てる手順

Azure ロールを割り当てる手順については以下の記事をご参照ください。

[Azure ロールを割り当てる手順](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/role-assignments-steps)

適切なロールを選択するときは、まず当該のロールに必要なアクセス許可を持つ組み込みロールがあるかどうかご確認ください。<br />
組み込みロールを利用する方が手順は簡単です。組み込みロールで要件を満たせない場合にのみ、カスタム ロールを検討してください。



![RBAC-custom-role-with-minimum-permissions-3a19a99d-7082-43a1-aa3d-0ed89cebb4d6.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions-3a19a99d-7082-43a1-aa3d-0ed89cebb4d6.png)

### 必要なアクセス許可を特定する方法

手順2-2: カスタム ロールを作成する で必要なアクセス許可を特定する方法については以下の記事をご参照ください。

[必要なアクセス許可を特定する方法](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/custom-roles#how-to-determine-the-permissions-you-need)

#### アクセス許可をひとつずつ追加して試す
上記の記事で最低限必要なアクセス許可を特定できない場合、以下の手順でアクセス許可を特定してください。

1. 権限を一切持たないカスタムロールを作成しユーザーへ割り当てる。
1. リソースに対する操作を実行する。リソースに対する操作が成功すればそれが必要最低限のアクセス許可が特定できている。
1. 画面または表示されるエラーメッセージを確認し、次に必要なアクセス許可を[アクセス許可一覧](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/resource-provider-operations#microsoftweb)から探してロールへ付与する。
1. 2.に戻り必要最低限のアクセス許可が特定できるまで繰り返す。



#### アクセス許可を削除しながら試す

あらかじめ対象の操作ができるように十分なアクセス許可をロールに付与して、そこから権限をひとつずつ削除していくことで必要最低限のアクセス許可を特定することもできます。


1. 十分な権限を持つカスタムロールを作成しユーザーへ割り当てる。
1. アクセス許可をひとつ削除して、リソースに対する操作を実行する。
1. リソースに対する操作が成功したら、2.で削除したアクセス許可は不要なので、削除したままにする。操作が失敗したら、当該のアクセス許可は必要なので、追加して維持する。
1. 必要最低限のアクセス許可が分かるまで2.-3.を繰り返す。

以下の手順で特定のリソースプロバイダーの権限をまとめて付与することができます。

[アクセス制御 (IAM)] > [役割] > 特定のロール > 右側の […] > [複製] > [アクセス許可] > [アクセス許可の追加] > 特定のリソースプロバイダー > [権限] のチェックボックスをクリック

![RBAC-custom-role-with-minimum-permissions2-f824bde2-5f46-426a-a22e-5e2bd8d64189.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions2-f824bde2-5f46-426a-a22e-5e2bd8d64189.png)

[JSON] タブで特定のリソースプロバイダーのアクセス許可がまとめて登録されていることが確認できます。

![RBAC-custom-role-with-minimum-permissions3-eb79f1c4-b777-4874-8c1a-cf5578cae4eb.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions3-eb79f1c4-b777-4874-8c1a-cf5578cae4eb.png)


## 例

## ポータルから Web App を再起動する

ポータルから Web App を再起動するために必要最小限のアクセス許可を探します。 App Service は作成済みとします。

### 手順2-1: 組み込みロールを検討する

[Azure ロールを割り当てる手順](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/role-assignments-steps)に記載の通り、[Azure 組み込みロール](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles)から適切なロールがないか探します。<br />
[Website Contributor](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles#website-contributor)が一番近いようですが、アクセス許可が多すぎるように見えます。カスタム ロールを検討します。

![RBAC-custom-role-with-minimum-permissions5-68e2fff9-64b5-4663-b7e1-20df4accaa02.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions5-68e2fff9-64b5-4663-b7e1-20df4accaa02.png)

### 手順2-2: カスタム ロールを作成する
[Azure ロールを割り当てる手順](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/role-assignments-steps)に記載の通り、[カスタム ロールの作成手順](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/custom-roles#steps-to-create-a-custom-role)を参考にアクセス権限を一切持たないカスタム ロールを作成し、ユーザーに割り当てます。

App Service の一覧を表示しても対象の App Service が表示されません。

![RBAC-custom-role-with-minimum-permissions6-05fdb220-5793-47d3-8630-c4af5568a78a.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions6-05fdb220-5793-47d3-8630-c4af5568a78a.png)

<br />

[Azure リソース プロバイダーの操作](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/resource-provider-operations)から対象の App Service を読み取るためのアクセス許可を探します。`Microsoft.Web/sites/Read`が該当します。


`Microsoft.Web/sites/Read`を対象のユーザーに付与して、再度 Portal にアクセスします。

<br />

今度は、 App Service が表示されました。

![RBAC-custom-role-with-minimum-permissions7-c7f6e426-6b78-4c37-8063-812ed125d1b5.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions7-c7f6e426-6b78-4c37-8063-812ed125d1b5.png)


<br />

ただ、再起動ボタンがグレーアウトされています。

![RBAC-custom-role-with-minimum-permissions8-28a2279d-c5f1-4a02-8a30-36edede5c3ef.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions8-28a2279d-c5f1-4a02-8a30-36edede5c3ef.png)

<br />

再度、[Azure リソース プロバイダーの操作](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/resource-provider-operations)から対象の App Service を再起動するためのアクセス許可を探します。`Microsoft.Web/sites/restart/Action`が該当します。

`Microsoft.Web/sites/restart/Action`を対象のユーザーに付与して、再度 Portal にアクセスします。

<br />

今度は再起動ボタンが有効化され、ボタンをクリックすることでアプリを再起動することができました。


![RBAC-custom-role-with-minimum-permissions10-84a253a6-2564-4d84-a94d-c1f7d7c1a4df.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions10-84a253a6-2564-4d84-a94d-c1f7d7c1a4df.png)

<br />

アプリを再起動するために必要なアクセス許可は以下の 2 つでした。<br />
`Microsoft.Web/sites/Read`<br />
`Microsoft.Web/sites/restart/Action`


## Visual Studio Code から Web App をデプロイする

Visual Studio Code から Web App をデプロイするために必要最小限のアクセス許可を探します。 
Web App は [Azure で Node.js Web アプリを作成する
](https://docs.microsoft.com/ja-jp/azure/app-service/quickstart-nodejs?tabs=windows&pivots=development-environment-vscode) の手順を元に一度作成済みであるとします。

### 手順2-1: 組み込みロールを検討する

[Azure ロールを割り当てる手順](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/role-assignments-steps)に記載の通り、[Azure 組み込みロール](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles)から適切なロールがないか探します。<br />
[Website Contributor](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles#website-contributor)が一番近いようですが、アクセス許可をもう少し減らしたいです。カスタム ロールを検討します。

![RBAC-custom-role-with-minimum-permissions5-68e2fff9-64b5-4663-b7e1-20df4accaa02.png]({{site.baseurl}}/media/2025/04/RBAC-custom-role-with-minimum-permissions5-68e2fff9-64b5-4663-b7e1-20df4accaa02.png)

### 手順2-2: カスタム ロールを作成する
[Azure ロールを割り当てる手順](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/role-assignments-steps)に記載の通り、[カスタム ロールの作成手順](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/custom-roles#steps-to-create-a-custom-role)を参考に Website Contributor をクローンしたカスタム ロールを作成し、ユーザーに割り当てます。

カスタム ロールのアクセス許可<br />
`Microsoft.Authorization/*/read`<br />
`Microsoft.Insights/alertRules/*`<br />
`Microsoft.Insights/components/*`<br />
`Microsoft.ResourceHealth/availabilityStatuses/read`<br />
`Microsoft.Resources/deployments/*`<br />
`Microsoft.Resources/subscriptions/resourceGroups/read`<br />
`Microsoft.Support/*`<br />
`Microsoft.Web/certificates/*`<br />
`Microsoft.Web/listSitesAssignedToHostName/read`<br />
`Microsoft.Web/serverFarms/join/action`<br />
`Microsoft.Web/serverFarms/read`<br />
`Microsoft.Web/sites/*`

<br />

デプロイが成功します。

![RBAC-custom-role-with-minimum-permissions3-1-cfcca984-8670-4b00-965a-772aa1ab27c5.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions3-1-cfcca984-8670-4b00-965a-772aa1ab27c5.png)

<br />

カスタム ロールのアクセス許可から `Microsoft.Authorization/*/read` を削除して、再度デプロイします。<br />

<br />

デプロイが成功します。

![RBAC-custom-role-with-minimum-permissions3-2-ebec12ec-01e6-4513-a89e-8249b05cdfb3.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions3-2-ebec12ec-01e6-4513-a89e-8249b05cdfb3.png)

<br />

カスタム ロールのアクセス許可から `Microsoft.Insights/alertRules/*` を削除して、再度デプロイします。<br />

<br />

デプロイが成功します。

![RBAC-custom-role-with-minimum-permissions3-3-11c5654f-92df-4b7b-829a-356892faba02.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions3-3-11c5654f-92df-4b7b-829a-356892faba02.png)

<br />

カスタム ロールのアクセス許可を以下のようにしてもデプロイは成功しました。

カスタム ロールのアクセス許可<br />
`Microsoft.Web/serverFarms/read`<br />
`Microsoft.Web/sites/*`

<br />

ここから、 `Microsoft.Web/serverFarms/read` を削除して、再度デプロイします。<br />
デプロイが失敗します。

![RBAC-custom-role-with-minimum-permissions3-4-97542982-d503-48ff-8591-3278c15a3226.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions3-4-97542982-d503-48ff-8591-3278c15a3226.png)

<br />

Visual Studio Code から Web App をデプロイするために必要なアクセス許可は以下の 2 つに絞られました。<br />
`Microsoft.Web/serverFarms/read`<br />
`Microsoft.Web/sites/*`

<br />



ここからさらに `Microsoft.Web/sites/*` 配下のアクセス許可も個別に指定したい場合は、以下の手順で `Microsoft.Web/sites/*` 配下のアクセス許可をまとめて登録して、同様の手順でアクセス許可を絞ることができます。

[アクセス制御 (IAM)] > [役割] > 特定のロール > 右側の […] > [複製] > [アクセス許可] > [アクセス許可の追加] > 特定のリソースプロバイダー > [アクセス許可を入力してください] テキストボックスに `Microsoft.Web/sites/` と入力 > [権限] のチェックボックスをクリック

![RBAC-custom-role-with-minimum-permissions3-5-d4a63f8c-7961-4f10-8241-4ce22ec63b4b.png]({{site.baseurl}}/media/2022/09/RBAC-custom-role-with-minimum-permissions3-5-d4a63f8c-7961-4f10-8241-4ce22ec63b4b.png)



<br>
<br>

---

<br>
<br>

2022 年 09 月 04 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>