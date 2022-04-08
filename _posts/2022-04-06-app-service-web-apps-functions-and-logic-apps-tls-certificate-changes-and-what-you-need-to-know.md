---
title: "App Service (Web Apps、 Functions) と Logic Apps (Standard) のデフォルトのホスト名に適用されているルート証明書が2022年4月1日より順次変更されます"
author_name: "Hidenori Yatsu"
tags:
    - App Service
    - Web Apps
    - Function App
---

# 概要

**App Service (Web Apps、Functions) と Logic Apps (Standard)** におけるリソースのデフォルトのホスト名である *.azurewebsites.net に適用されている TLS ルート証明書が、2022年4月より順次 Baltimore CyberTrust Root CA から DigiCert Global Root G2 CA に変更されます。また、現在の TLS ルート証明書 (Baltimore CyberTrust Root CA) は、2022 年 7 月 7 日に有効期限を迎えます。個々の Web Apps、Functions、およびロジック アプリ (Standard) における証明書の更新日については、恐縮ながら事前に把握することは叶いません。

# お客様への影響
 殆どのお客様には影響が発生しないことが想定されますが、*.azurewebsites.net に対してアクセスを行っているクライアント側のアプリケーション等において証明書のピン留め（※）を行っている場合は、TLS 証明書の移行後に *.azurewebsites.net に対する TLS 通信がブロックされます。

※証明書のピン留め・・・
クライアント側のアプリケーション等が受け入れ可能な証明機関 (CA) 、公開鍵、拇印などの特定のリストを指定するプラクティスです。

# 必要な対応
前述のとおり、殆どのお客様には影響が及ばないことが想定されるため、証明書のピン留めを行っている場合を除き、必要な対応はございません。証明書のピン留めを行っている場合ですが、*.azurewebsites.net の TLS 証明書は PaaS の特性上、Azure プラットフォーム側にて変更されることが想定されることから、証明書のピン留めを行うことは推奨されません。お客様の事情により証明書のピン留めを行うことがやむを得ず必要となる場合は、カスタムドメインをお客様のリソースに構成し、お客様自身にてご用意された TLS 証明書をカスタムドメインに適用する方法が推奨されます。こちらの対応により *.azurewebsites.net への依存関係が無くなり、TLS 証明書の変更に伴うダウンタイムのリスクが除外されることから、証明書のピン留めにおける信頼性を向上させることが可能でございます。

# 参考
 
App Service 開発チームによる Blog (英語)

[App Service Web Apps, Functions, and Logic Apps (Standard) *.azurewebsites.net TLS certificate changes and what you need to know](https://azure.github.io/AppService/2022/03/22/Default-Cert-Renew.html)

<BR>
上記、App Service 開発チーム Blog の日本語版抄訳

---
このブログでは、*.azurewebsites.net Web Apps、Functions、およびロジック アプリ (Standard) の TLS 証明書の変更について説明します。お客様はこの変更の影響を受けないようにする必要があります。影響を受けるサービスの範囲には、Web Apps、Functions、ロジック アプリ (Standard) が含まれます。ロジック アプリ (従量課金) は影響を受けません。この変更はパブリック Azure クラウドに限定されます。政府のクラウドは影響を受けません。

すべての Web アプリ、Functions、またはロジック アプリ (Standard) には、"<resource-name>.azurewebsites.net" で囲まれた独自の既定のホスト名があり、App Service はワイルドカード *.azurewebsites.net TLS 証明書で保護します。Baltimore CyberTrust Root CA によって発行された現在の TLS 証明書は、2022年7月7日に有効期限が切れるように設定されています。2022 年 4 月から、App Service はこれらの TLS 証明書の更新を開始し、代わりに DigiCert Global Root G2 CA によって発行された証明書を使用します。更新プロセスの分散非同期の性質上、個々の Web Apps、Functions、およびロジック アプリ (Standard) における証明書の更新日を事前に把握することは叶いません。

この変更はお客様に対して影響を与えないと予想されます。ただし、アプリケーションが誤って *.azurewebsites.net TLS 証明書への依存関係が適用されている場合 (たとえば、証明書のピン留め) は影響を受ける可能性があります。証明書のピン留めは、アプリケーションが受け入れ可能な証明機関 (CA)、公開キー、拇印などの特定のリストのみを許可するプラクティスです。アプリケーションは *.azurewebsites.net TLS 証明書にピン留めしないでください。証明書の安定性を必要とするアプリケーションでは、カスタム ドメインを、それらのドメインのカスタム TLS 証明書と組み合わせて使用する必要があります。詳細については、この記事の推奨されるベスト プラクティスのセクションを参照してください。

**推奨されるベスト プラクティス**

サービスとしてのプラットフォーム (PaaS) の性質上、*.azurewebsites.net TLS 証明書をいつでもローテーションできるため、*.azurewebsites.net に対する TLS 証明書のピン留めはお勧めできません。サービスが App Service の既定のワイルドカード TLS 証明書をローテーションする場合、証明書が固定されたアプリケーションは、特定の証明書属性のセットにハードコーディングされているアプリケーションの接続を妨げ、中断します。*.azurewebsites.net TLS 証明書のローテーションに使用する周期性においても、ローテーション頻度はいつでも変更される可能性があるため、保証されません。

アプリケーションが証明書のピン留め動作に依存する必要がある場合は、カスタム ドメインを Web Apps、Functions、またはロジック アプリ (Standard) に追加し、そのドメインにカスタム TLS 証明書を提供して、証明書のピン留めを信頼できるようにすることをお勧めします。

証明書のピン留めに依存するアプリケーションも、App Service マネージド証明書に大きく依存してはいけないことに注意してください。App Service マネージド証明書はいつでもローテーションできるため、安定した証明書プロパティに依存するアプリケーションでも同様の問題が発生します。ベスト プラクティスは、証明書のピン留めに依存するアプリケーションにカスタム TLS 証明書を提供することです。

---
※抄訳には細心の注意を払っておりますが、万が一原文との差異がございます場合は原文を正としてお取り扱いいただけますと幸いです。

<br>
<br>

2022 年 4 月 6 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
