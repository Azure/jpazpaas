---
title: "Kudu サイトの使い方 (Tips 4 選)"
author_name: "Gakushi Ishii"
tags:
    - Azure App Service
    - Azure Functions
---

# はじめに <!-- omit in toc -->

Kudu は App Service や Functions のさまざまな機能の背後にあるエンジンです。本稿では、 App Service や Functions をご利用いただく上で活用いただけるであろう Kudu が提供する機能を Tips として紹介します。

# 目次 <!-- omit in toc -->

- [Kudu へのアクセス方法](#kudu-へのアクセス方法)
- [Tips 1. Kudu サイトへのログイン](#tips-1.-kudu-サイトへのログイン)
  - [Azure Active Directory (AAD) 認証を利用して Kudu サイトへログインする方法](#azure-active-directory-(AAD)-認証を利用して-kudu-サイトへログインする方法)
  - [Basic 認証を利用して Kudu サイトへログインする方法](#basic-認証を利用して-kudu-サイトへログインする方法)
- [Tips 2. New UI (KuduLite)](#tips-2.-new-ui-(kudulite))
- [Tips 3. Docker ログのダウンロード](#tips-3.-docker-ログのダウンロード)
  - [Docker ログの取得方法](#docker-ログの取得方法)
- [Tips 4. TCP Ping を用いた疎通確認](#tips-4.-tcp-ping-を用いた疎通確認)
  - [TCP Ping の実行方法](#tcp-ping-の実行方法)

# Kudu へのアクセス方法

Kudu サイトへのアクセス方法は以下の 2 種類があります。

- `https://<app-name>.scm.azurewebsites.net` へアクセスする
- Azure Portal から対象の App Service を選び、 [開発者ツール] > [高度なツール] から [移動] を選択する

    ![kudu-access-ace8156b-48e9-4df0-8fa8-8bc7a0b5cd5b.png]({{site.baseurl}}/media/2022/11/kudu-access-ace8156b-48e9-4df0-8fa8-8bc7a0b5cd5b.png)

# Tips 1. Kudu サイトへのログイン

## Azure Active Directory (AAD) 認証を利用して Kudu サイトへログインする方法

AAD 認証を使用してブラウザーから Kudu にアクセスするには、ログインするユーザーにリソースプロバイダー（権限の様なものとお考え下さい）`Microsoft.Web/sites/publish/Action` が必要となります。  
このため、`Microsoft.Web/sites/publish/Action` を予め含む組み込みロールまたはカスタムロールを作成してユーザーに割り当てます。

**組み込みロールを使用して Kudu へのアクセスを許可する場合**

Kudu サイトへアクセスするためには "Webサイト共同作成者", "共同作成者", "所有者" のいずれかのロールを割り当ててください。

- [組み込みロールの割り当て方](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/role-assignments-steps)
- [組み込みロールの一覧](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/custom-roles)

**カスタムロールを使用して Kudu へのアクセスを許可する場合**

次のリソースプロバイダーが必要です : `Microsoft.Web/sites/publish/Action`

- [カスタムロールの作成手順](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/custom-roles )

AAD 認証を使用した Kudu サイトへのアクセスは上記の RBAC を割り当てることで可能になります。

## Basic 認証を利用して Kudu サイトへログインする方法

他方、 App Service では Basic 認証を使用して Kudu サイトにアクセスすることも可能です。  
Kudu へのアクセスを行いたいものの、AAD のアカウントがない場合などのシナリオの対処としてご利用いただけるものと存じます。

1. Azure Portal から対象の App Service を選び、 [発行プロファイルのダウンロード] を選択します。発行プロファイルがローカル環境にダウンロードされますので、適当なエディタでダウンロードされたファイルを開きます。

    ![1_1-51c1c48e-c670-44d6-a9c8-7944ebc916c1.png]({{site.baseurl}}/media/2022/11/1_1-51c1c48e-c670-44d6-a9c8-7944ebc916c1.png)

2. 発行プロファイル内の、 "userName" の値がユーザー名、 "userPWD" の値がパスワードとなりますので、コピーします。

3. Kudu サイトの既定リンク `https://<app-name>.scm.azurewebsites.net` に `/basicauth` を追加した `https://<app-name>.scm.azurewebsites.net/basicauth` へアクセスしてください。下図のようなポップアップが表示されますので、手順 2 でコピーした "userName" と "userPWD" をユーザー名とパスワードの欄にペーストし、サインインします。

    ![1_3-47e3c9f0-ded9-45bc-ae94-92e37890e18e.png]({{site.baseurl}}/media/2022/11/1_3-47e3c9f0-ded9-45bc-ae94-92e37890e18e.png)

4. ログインが完了すると Kudu サイトが表示されます。

    ![1_4-c38d616e-37f4-41fc-8325-6caa6927f1c5.png]({{site.baseurl}}/media/2022/11/1_4-c38d616e-37f4-41fc-8325-6caa6927f1c5.png)

# Tips 2. New UI (KuduLite)

**Linux OS** の App Service には [KuduLite](https://github.com/Azure-App-Service/KuduLite) という 新しい UI が提供されており、様々な機能が活用できます。KuduLite は、 `/newui` を追加することでアクセスが可能です。

New UI (KuduLite) のリンク : `https://<app-name>.scm.azurewebsites.net/newui`

![2_0-2bc8d3f3-b09f-4473-aa6b-835c58028fa5.png]({{site.baseurl}}/media/2022/11/2_0-2bc8d3f3-b09f-4473-aa6b-835c58028fa5.png)

**New UI の主なメリット**

- ファイルマネージャーによって、ファイルの GUI 操作ができます。

    ![2_1-5f5d203e-ad2d-4b2c-98c8-067f2359b66d.png]({{site.baseurl}}/media/2022/11/2_1-5f5d203e-ad2d-4b2c-98c8-067f2359b66d.png)

- App Service が複数のインスタンスにスケールアウトされている場合、 Kudu を実行しているインスタンスを選択することができます。

    ![2_2-ae715125-6427-4632-afbe-988e42b7e284.png]({{site.baseurl}}/media/2022/11/2_2-ae715125-6427-4632-afbe-988e42b7e284.png)

# Tips 3. Docker ログのダウンロード

Docker ログは、クライアントから Linux OS の App Service 上で動くアプリケーションに接続できない事象が発生した場合などの調査に必要な基本的な情報の 1 つです。皆様のトラブルシューティング時の材料の 1 つとして、 Docker ログをダウンロードする手順を紹介致します。

## Docker ログの取得方法

以下の 2 つの方法で Kudu サイトからで Docker ログをダウンロードすることが可能です。

- Kudu サイトへアクセスして、REST API から [Download as zip] をクリックします。

    ![3_1-e3c8d9c1-5bc2-4f02-bb91-40f79ed0b417.png]({{site.baseurl}}/media/2022/11/3_1-e3c8d9c1-5bc2-4f02-bb91-40f79ed0b417.png)

    ※ Kudu の REST API には他にも様々な機能があります。詳しくは [Kudu wiki REST API](https://github.com/projectkudu/kudu/wiki/REST-API) をご参照ください。

- KuduLite へアクセスして、 Web App Container から [Download All Container Logs] をクリックします。

    ![3_2-d21bcd95-2126-4948-b076-235e70109c37.png]({{site.baseurl}}/media/2022/11/3_2-d21bcd95-2126-4948-b076-235e70109c37.png)

# Tips 4. TCP Ping を用いた疎通確認

App Service や Azure Functions から SQL Database など別サービスのエンドポイントに対して疎通確認をしたい場合があるかと存じます。しかしながら App Service では ICMP を使用した ping コマンドを実行することは叶いません。そこで Kudu のコンソールで扱えるコマンド [TCP Ping](https://github.com/cicorias/webjobtcpping) が役に立ちます。
Kudu には CUI ツールを使える機能 (CMD や PowerShell, Bash シェルなど) があり、 TCP Ping を使用することでサーバーとの接続をテストすることが可能です。

## TCP Ping の実行方法

**windows OS の場合**

1. Kudu サイトから [Debug console]-> [CMD] を選択して、コンソールを開きます。

    ![4_1-683c7bb4-66dc-412e-99cb-01204897996f.png]({{site.baseurl}}/media/2022/11/4_1-683c7bb4-66dc-412e-99cb-01204897996f.png)

2. 画面下のコンソールで tcpping コマンドをお試しください。

    **（例） SQL Server への疎通が成功**

    ![4_tcpping_ok_windows-32af426d-ba41-48a0-b1e8-62a89650a103.png]({{site.baseurl}}/media/2022/11/4_tcpping_ok_windows-32af426d-ba41-48a0-b1e8-62a89650a103.png)

    **（例） SQL Server への疎通が失敗**

    ![4_tcpping_ng_windows-d17a5b4b-8c66-4695-9cad-e2b2f1b9ac93.png]({{site.baseurl}}/media/2022/11/4_tcpping_ng_windows-d17a5b4b-8c66-4695-9cad-e2b2f1b9ac93.png)   

**Linux OS の場合**

1. Kudu サイトから [SSH] or [Bash] へアクセスをして、コンソールを開きます。

    ![4_3-2d3383e0-b806-4b9c-94fb-c8bc652e6965.png]({{site.baseurl}}/media/2022/11/4_3-2d3383e0-b806-4b9c-94fb-c8bc652e6965.png)

2. コンソールで `apt install bc` を実行してください。

3. tcpping コマンドをお試しください。コマンドが見つからない場合は、 [こちらのブログ](https://azureossd.github.io/2021/06/17/installing-tcpping-linux/)を参考にしてください。

    **（例） SQL Server への疎通が成功**

    ![4_tcpping_ok_linux-427f2e01-9835-4cab-9aba-15614c2ef72e.png]({{site.baseurl}}/media/2022/11/4_tcpping_ok_linux-427f2e01-9835-4cab-9aba-15614c2ef72e.png)

    **（例） SQL Server への疎通が失敗**

    ![4_tcpping_ng_linux-7038e2e6-2777-474d-8343-cde6a91f92e8.png]({{site.baseurl}}/media/2022/11/4_tcpping_ng_linux-7038e2e6-2777-474d-8343-cde6a91f92e8.png)

※ OS によってポートの指定方法や使用可能なオプションが異なります。詳しくは上記のコンソールで `tcpping` を実行してください。