---
title: "App Service (Windows/Java) で httpPlatformHandler をカスタマイズする際の代替案 "
author_name: "Hidenori Yatsu"
tags:
    - App Service, Web Apps, Java
---

# はじめに

お世話になっております。App Service サポート担当の谷津です。

Windows の Java スタックにおける App Service にて Tomcat の設定変更を目的として IIS モジュールである HttpPlatformHandler の定義を web.config に設定しているリソースに関するお問い合わせを頂戴することがございます。しかしながら、PaaS である App Service が提供している HttpPlatformHandler ではなく独自で web.config に HttpPlatformHandler の定義を追加されますと、アプリケーションの起動処理に失敗するような予期せぬ影響を及ぼす可能性がございます。このように PaaS 基盤のモジュールである HttpPlatformHandler を差し替えるようなカスタマイズは恐縮ながら推奨されませんため、Tomcat の設定変更を行いたい場合の代替案と併せて当記事にて詳細を解説いたします。

# HttpPlatformHandler を独自定義している NG 例
以下のように web.config にて **<httpPlatform>** 要素により httpPlatformHandler の定義を追加し、追加したハンドラーの定義にて独自で用意した server.xml の指定や CATALINA_OPTS の値を環境変数として指定することは推奨されておりません。


```
<?xml version="1.0" encoding="utf-8"?> 
<configuration> 
  <system.webServer>
    <handlers> 
      <remove name="httpPlatformHandlerMain" /> 
      <add name="httpPlatformHandlerMain" path="*" verb="*" modules="httpPlatformHandler" resourceType="Unspecified"/> 
    </handlers> 
    <httpPlatform processPath="D:\Program Files\apache-tomcat-9.0.37\bin\startup.bat" requestTimeout="00:04:00" arguments="-config D:\home\site\wwwroot\conf\server.xml start" startupTimeLimit="60" startupRetryCount="3" stdoutLogEnabled="true"> 
        <environmentVariables> 
            <environmentVariable name="CATALINA_OPTS" value="-Xms1024m -Xmx1024m -Dport.http=%HTTP_PLATFORM_PORT% -Dsite.logdir=d:/home/LogFiles/ -Dsite.tempdir=d:/home/site" /> 
        </environmentVariables> 
    </httpPlatform> 
  </system.webServer> 
</configuration>
```
 
# HttpPlatformHandler を独自定義するとアプリの起動処理時に問題が発生する可能性がある理由

 App Service の基盤では、 Java プロセスを呼び出す際に JavaBootStrapper というプロセスが IIS の w3wp.exe と Java.exe の間に入り、Java の起動処理を安定的に完了させる役割や基盤のログ出力を司る役割を担っております。お客様が web.config 内にて独自で httpPlatformHandler を定義してしまうと JavaBootStrapper のプロセスが生成されないため、起動処理時に予期しない問題が発生することがございます。


# 代替案
HttpPlatformHandler の独自定義は、カスタマイズしたTomcat の設定を目的としたものが多いものと考えられます。
以下に、App Service 上で Tomcat の設定を行う代替案をご紹介します。

## その１) カスタマイズした server.xml を適用する方法
CATALINA_BASE のパスを指定することで home フォルダ配下の任意の場所に配置した server.xml を使用することが可能でございます。外部サイトのご紹介とはなりますが、弊社のエンジニアが執筆した記事である[App Service にて Tomcat の設定ファイルを変更してカスタムエラーページを構成する方法](https://qiita.com/hyatsu/items/61ae74d0c308f9ec9048) の内容がご参考になりますと幸いです。

## その２) CATALINA_OPTS の値を指定する方法

Azure ポータルの該当 App Service のページより [構成] > [アプリケーションの設定] の画面にて「新しいアプリケーション設定」より CATALINA_OPTS を追加いただくことが可能です。

![image-abb93fc4-2bb6-4ade-9b21-c45953c997a3.png]({{site.baseurl}}/media/2022/12/image-abb93fc4-2bb6-4ade-9b21-c45953c997a3.png)


より詳細な解説については弊社のエンジニアが執筆した外部ブログ[Web Apps (java) への java コマンドラインオプションの指定方法](https://qiita.com/YusukeTobo/items/e0519bbe2d5e84d742e0) にも記載がございますので、こちらの内容も併せてご参照ください。


<br>
<br>

---

<br>
<br>

2022 年 12 月 29 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>