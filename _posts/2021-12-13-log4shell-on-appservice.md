---
title: "App Service は CVE-2021-44228 (Apache Log4j 2) の脆弱性の影響を受けますか"
author_name: "Shogo Ohe"
tags:
    - App Servce
---

# 質問
App Service が提供する Java 実行環境は CVE-2021-44228 (Apache Log4j 2) の脆弱性の影響を受けます
か？

# 回答
いいえ。App Service 基盤は CVE-2021-44228 の影響を受けません。お客様のアプリケーションで Log4j 2 を使用している場合は別途ご確認ください。

CVE-2021-44228、Log4Shell と呼ばれる脆弱性は Log4j 2 (2.0-beta9 から 2.14.1 まで) を使用している場合に影響を受けます。
App Service では Java 実行基盤 (JDK, JRE, Tomcat) を提供しておりますが、Log4j を使用または提供しておりません。そのため、App Service 基盤自体は、この脆弱性の影響を受けません。

ただし、お客様のアプリケーションにて Log4j 2 を使用している場合、意図しないリモートコードを実行される可能性がございます。お客様のアプリケーションまたは関連するライブラリ、モジュールが log4j 2 に依存していないかご確認いただけますと幸いです。脆弱性の影響を受けるバージョンの log4j 2 をご利用されている場合は、以下の各参考ドキュメントにて案内されている回避策のご利用をご検討ください。

# 参考ドキュメント
- [Microsoft’s Response to CVE-2021-44228 Apache Log4j 2](https://msrc-blog.microsoft.com/2021/12/11/microsofts-response-to-cve-2021-44228-apache-log4j2/)
- 抄訳: [CVE-2021-44228 Apache Log4j 2 に対するマイクロソフトの対応](https://msrc-blog.microsoft.com/2021/12/12/microsofts-response-to-cve-2021-44228-apache-log4j2-jp/)
- [CVE-2021-44228](https://cve.mitre.org/cgi-bin/cvename.cgi?name=2021-44228)
- [Apache Log4j Security Vulnerabilities](https://logging.apache.org/log4j/2.x/security.html)

<br>
<br>

---

<br>
<br>

2021 年 12 月 13 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>