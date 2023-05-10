---
title: "App Service を構成する主要な要素とそれぞれの役割"
author_name: "Hiroki Nakajima"
tags:
    - App Service
    - Web Apps
---

# はじめに
App Service サポート担当の中島です。

今回は App Service をご利用いただいているお客様、またはご利用を検討いただいているお客様へ向け、 App Service のアーキテクチャについて解説いたします。

# 1. App Service を構成する主要な要素
ここでは、 App Service を構成する要素のうち、とりわけお客様のアプリに関連する要素について解説いたします。

## スケールユニット
App Service のサーバーはマイクロソフトのデータセンター内に配置されておりますが、そのデータセンター内に **"スケールユニット"** と呼ばれる単位のサーバー群が複数存在しており、各データセンターには複数のスケールユニットが存在しています。

それぞれのスケールユニット内には、複数の App Service Plan が存在し、それらとともに App Service を構成するいくつかのコンポーネントが配置されています。

つまり、 App Service のアプリ (Web App) はスケールユニット内の App Service Plan 内で実行され、 **1 つのスケールユニット内では複数のお客様の App Service Plan およびアプリ (Web App) が同時に実行** されています。

もちろん、それぞれのアプリは互いに干渉することなく分離されているため、別々のお客様のアプリが 1 つの App Service Plan 内に混在することはありません。

データセンターとスケールユニット、および App Service を構成するコンポーネントを図で表すと、下記のような形となります。<br>
![image-5d8ccbab-9e59-44d1-9e04-a1ab15debd74.png]({{site.baseurl}}/media/2023/05/image-5d8ccbab-9e59-44d1-9e04-a1ab15debd74.png)


また、上述の通り 1 つのスケールユニット内には複数のお客様の App Service Plan が配置されています。

その一例といたしまして、下記の図では、 A のお客様と B のお客様がそれぞれ 2 つの App Service Plan インスタンスをご利用いただいており、 C のお客様は 5 つのインスタンスをご利用いただいているイメージとなります。<br>
![image-20f45a89-939e-4fbf-905c-31d4084ad22a.png]({{site.baseurl}}/media/2023/05/image-20f45a89-939e-4fbf-905c-31d4084ad22a.png)


それでは、スケールユニットに存在する各コンポーネントのうち、よりお客様のアプリに関連することの多いコンポーネントについて詳しくみていきましょう。

## FrontEnd
FrontEnd は、ユーザーが App Service 内のアプリへアクセスする際の **入口の役割** を務めます。

1 つのスケールユニット内に FrontEnd は複数存在いたしますが、各 FrontEnd は独立して実行されており、各 App Service Plan と個別に紐づいてはおりません。

FrontEnd は受け取ったリクエストを、リクエスト内容に応じてそれぞれの App Service Plan に振り分けます。<br>
また、 SSL のオフロードなどもこの FrontEnd にて行われます。<br>
![image-16ef67b9-c2ea-416a-9811-1e057638c6e6.png]({{site.baseurl}}/media/2023/05/image-16ef67b9-c2ea-416a-9811-1e057638c6e6.png)

## File Server
File Server は App Service で使用する共有ストレージを管理しています。

App Service では、複数の Worker インスタンス間で共通的に使用できるストレージとして、ローカルストレージ内の **home** という名称でネットワークストレージが付帯します。


このネットワークストレージには各アプリの実行ファイルやアプリが使用する静的ファイル、出力されるログファイルなど幅広い用途で使用されますが、このネットワークストレージを App Service の各アプリでローカルストレージとして使用できるように管理しているのが File Server です。

各アプリで実行されるファイルに関する読み取り/書き込みは全て File Server を通して行われます。<br>
![image-f81f64b8-3850-479c-aeeb-8b0070819887.png]({{site.baseurl}}/media/2023/05/image-f81f64b8-3850-479c-aeeb-8b0070819887.png)

## Application
お客様にて構築されている各アプリは App Service Plan のインスタンス内で実行され、既定の構成であれば、スケールアウトもその単位で行われます。

また、 CPU やメモリなどのコンピューティングリソースは各 App Service Plan 内のインスタンスごとに割り当てられており、それらを App Service Plan のインスタンス内で実行されている **各アプリが共用して使用している** ものとなります。

つまり、 1 つのインスタンス内に複数のアプリが実行されており、いずれかのアプリが大量に CPU やメモリを使用した場合、 **同一インスタンス内の別のアプリも影響を受ける可能性がある** ということになります。

これらの関係性を図で表すと、下記のような形となります。


1. Azure ポータルからスケールアウトの操作を行う<br>
![image-6a1d2801-2d72-43cb-bd92-82b4dc460ea9.png]({{site.baseurl}}/media/2023/05/image-6a1d2801-2d72-43cb-bd92-82b4dc460ea9.png)

2. App Service Plan 内のインスタンスが複製され、 2 つに増える<br>
![image-8b139a8e-782c-497e-8930-8ab0e4ae63f9.png]({{site.baseurl}}/media/2023/05/image-8b139a8e-782c-497e-8930-8ab0e4ae63f9.png)



※上記の例は既定の構成の例であり、各アプリごとに実行されるインスタンス数を制御する方法についてはここでは割愛いたします。<br>
ご参考: [アプリごとのスケーリングを使って Azure App Service で高密度ホスティングを実現する](https://learn.microsoft.com/ja-jp/azure/app-service/manage-scale-per-app)

1 つの App Service Plan に最大いくつアプリを稼働させることができるかについては、下記に目安がございますのでご参照ください。

[アプリを新しいプランと既存のプランのどちらに入れる必要があるか](https://learn.microsoft.com/ja-jp/azure/app-service/overview-hosting-plans#should-i-put-an-app-in-a-new-plan-or-an-existing-plan)

<br>
<br>

App Service を構成する要素のうち、お客様のアプリに関連することの多い要素について解説いたしました。

上記でご紹介した内容は App Service を構成する要素の一部であり、全ての各要素についての詳細は [(日本語訳) Inside-the-Azure-App-Service-Architecture](https://jpazpaas.github.io/blog/2023/05/10/Inside-the-Azure-App-Service-Architecture.html) をご参照いただければ幸いです。

次に、 App Service で稼働するアプリケーションの内部構成について解説いたします。

# 2. App Service のアプリケーションの構成
App Service のアプリケーションは、大きく **Windows アプリ** , **Linux アプリ** , **コンテナーアプリ (Windows コンテナー、 Linux コンテナー)** に大別できます。

それぞれの特徴は下記のとおりです。

- Windows アプリ
  - Windows OS 上でアプリが起動します。 .NET 系のアプリと互換性が高く、 Web Jobs が利用可能などの特徴があります。<br>
![image-481d7091-4877-4ac2-b635-e0aaf8772612.png]({{site.baseurl}}/media/2023/05/image-481d7091-4877-4ac2-b635-e0aaf8772612.png)<br>
- Linux アプリ
  - Linux OS 上でアプリが起動します。 OSS 言語のアプリと互換性が高く、内部的にはコンテナーイメージの内部プロセスとしてアプリが動くなどの特徴があります。<br>下図の中に記載のある Blessed Image とは、 App Service がプラットフォームとして用意している組み込みのコンテナーイメージとなります。<br>Linux アプリでは、ベースのコンテナーイメージとして App Service であらかじめ用意しているイメージを使用して、そのイメージ内でお客様のアプリのコードが起動する形となります。<br>
![image-96482421-d030-4abf-a29a-4ef6bdaa2b47.png]({{site.baseurl}}/media/2023/05/image-96482421-d030-4abf-a29a-4ef6bdaa2b47.png)<br>
- コンテナーアプリ (Windows コンテナー、 Linux コンテナー)
  - お客様が指定したコンテナーイメージをベースとしてアプリが起動します。お客様にて独自のコンテナーイメージをご利用いただけるため、柔軟性が高いという特徴があります。<br>また、アプリの作成時にお客様のコンテナーイメージのベースとなっている OS を Windows または Linux からお選びいただくことが可能です。<br>Linux アプリとは異なり、アプリのコードをお客様がご用意したコンテナーイメージ内で起動することが可能です。<br>
![image-e0e505cc-140e-4cbe-ae5d-58afddf38962.png]({{site.baseurl}}/media/2023/05/image-e0e505cc-140e-4cbe-ae5d-58afddf38962.png)<br>
<br>

また、上図に記載のある Kestrel + YARP とはマイクロソフトにて開発している、 Apache や nginx などと同様、リクエストを中継するリバースプロキシです。

ご参考 :
[A Heavy Lift: Bringing Kestrel + YARP to Azure App Services](https://devblogs.microsoft.com/dotnet/bringing-kestrel-and-yarp-to-azure-app-services/)

上述の通り、 Windows アプリは Windows OS 上で、 Linux アプリとコンテナーアプリはそれぞれコンテナーイメージをベースとしてアプリが起動することが大きく異なります。

ご参考: [組み込みのイメージ](https://learn.microsoft.com/ja-jp/troubleshoot/azure/app-service/faqs-app-service-linux#--------)


また、 Windows, Linux アプリの両方で **Kudu** と呼ばれる管理ツールが自動的に付属します。

この Kudu は、アプリのデプロイや管理を行うためのサイドカーの役割を担い、 App Service でアプリを構築するにあたり必須のものとなります。

Kudu については下記の記事にて紹介しておりますため、ご参照いただけますと幸いでございます。

[Kudu サイトの使い方 (Tips 4 選)](https://jpazpaas.github.io/blog/2022/11/28/How-to-use-Kudu-site.html)

<br>
<br>

以上が App Service で稼働するアプリケーションの構成となります。

次に、 App Service の派生サービスである App Service Environment について解説いたします。


# 3. App Service Environment
App Service Environment (ASE または App Service 環境とも呼ばれる) とは、上述したスケールユニットに近しい環境を、お客様の占有環境として丸ごとご利用いただけるサービスとなります。

また、通常の App Service の場合は 1 つの App Service Plan のスケールアウト上限値が 30 インスタンスとなっておりますが、 App Service Environment では App Service Isolated v2 (Iv2) の App Service Plan をご利用いただくことで、最大 100 インスタンスまでスケールアウトすることが可能です。

そのため、 30 インスタンス以上を必要とするアプリや、複数のお客様の環境が 1 つのスケールユニット内に混在する通常の App Service ではコンプライアンス上の問題がありご利用いただけない場合に App Service Environment をご検討いただければ幸いでございます。


# 4. まとめ
App Service のアーキテクチャは上記でご紹介したような、複数の構成要素からなるアーキテクチャで稼働しております。

今回ご紹介した内容をご理解いただき App Service をお使いいただくことで、より最適なアーキテクチャ設計やトラブルシューティングなどにお役立ていただけますと幸いです。

また、より厳格な要件や多くのインスタンスを必要とするアプリ構築の際には、 App Service Environment のご利用をご検討いただければと存じます。


# 5. 参考ドキュメント
-  [(日本語訳) Inside-the-Azure-App-Service-Architecture](https://jpazpaas.github.io/blog/2023/05/10/Inside-the-Azure-App-Service-Architecture.html)
- [App Service の概要](https://learn.microsoft.com/ja-jp/azure/app-service/overview)
- [A Heavy Lift: Bringing Kestrel + YARP to Azure App Services](https://devblogs.microsoft.com/dotnet/bringing-kestrel-and-yarp-to-azure-app-services/)
- [組み込みのイメージ](https://learn.microsoft.com/ja-jp/troubleshoot/azure/app-service/faqs-app-service-linux#--------)
- [App Service Environment の概要](https://learn.microsoft.com/ja-jp/azure/app-service/environment/overview)
<br>
<br>
---
<br>
<br>
2023 年 05 月 10 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。
<br>
<br>