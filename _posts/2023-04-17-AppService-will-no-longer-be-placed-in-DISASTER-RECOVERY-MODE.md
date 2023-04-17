---
title: "2025 年 3 月 31 日以降、Azure App Service は Disaster Recovery Mode にならなくなります"
author_name: "Masafumi Kokui"
tags:
    - App Service
    - Function App
    - Web Apps
---

# はじめに
2023 年 3 月 31 日に [Action recommended: Implement disaster recovery strategies for your Azure App Service web apps by 31 March 2025](https://azure.microsoft.com/ja-jp/updates/action-recommended-implement-disaster-recovery-strategies-for-your-azure-app-service-web-apps-by-31-march-2025/) が更新情報として公開されました。
こちらの変更は下記の [App Service アプリを別のリージョンに移動する](https://learn.microsoft.com/ja-jp/azure/app-service/manage-disaster-recovery) でも確認することができます。
 ![image-32127351-0351-4db0-855f-4a8223d9285a.png]({{site.baseurl}}/media/2023/04/image-32127351-0351-4db0-855f-4a8223d9285a.png) 

本記事では、Disaster Recovery Mode がどのようなものであるかを説明し、取りうる対応について説明します。


# Disaster Recovery Mode
2025 年 3 月 31 日までは障害によって Azure リージョン全体がオフラインになると、そのリージョンでホストされているすべての App Service アプリ (マルチテナントの App Service プランに限定されます。ASE、または、Azure Functions の従量課金プラン、Premium プランは含まれません。) は Disaster Recovery Mode になります。Disaster Recovery Mode になると、影響を受けたリージョンの App Servie は SKU に関わらず [自動バックアップの復元](https://learn.microsoft.com/azure/app-service/manage-backup?tabs=portal#restore-a-backup) を行うことができる状態になります。制限なく「自動バックアップの復元」をご利用いただくためには、通常 Standard または Premium レベルの App Service プランである必要があります。Disaster Recovery Mode は、Free、Shared、Basic レベルの App Service プランのための機能であると言えます。また、Disaster Recovery Mode は自動的に他のリージョンに App Service アプリを復元させるものではございませんのでご注意ください。

Azure リージョン全体がオフラインになりました際には、他リージョンに "新規に App Service" を作成いただき、Disaster Recovery Mode　となった App Service の自動バックアップより App Service アプリを復元することができます。具体的な手順は [アプリを別のリージョンに復元する](https://learn.microsoft.com/azure/app-service/manage-disaster-recovery#restore-app-to-a-different-region) よりご確認ください。


# 変更による影響
2025 年 3 月 31 日後より Disaster Recovery Mode にならなくなることで Free, Shared, Basic レベルの App Service プランで動作している App Service アプリで「自動バックアップの復元」を使用することができなくなります。(Basic レベルでは、運用スロットのみをバックアップおよび復元できます。)

# 2025 年 3 月 31 日後に必要な対応
運用環境でワークロードを実行されます場合には Standard レベル以上の App Service プランをお勧めしています。何らかの事象で Free、Shared、Basic レベルの App Service プランを利用されます場合には Standard レベルの App Service プランへアップグレードいただくか、災害発生時の復旧手順をご検討ください。

Disaster Recovery Mode は災害発生後に Azure プラットフォームで取得しているバックアップをご提供するものです。Azure リージョンまたはデータセンター全体がオフラインになった場合でも、ミッション クリティカルなワークロードは別のリージョンで処理が継続する必要があります。Azure App Services、および Azure Functions について、以下より詳細をご確認ください。
- [Azure Functions geo ディザスター リカバリー](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-geo-disaster-recovery)
- [Azure App Service での事業継続とディザスター リカバリーの戦略](https://learn.microsoft.com/ja-jp/azure/app-service/overview-disaster-recovery) 


# 参考ドキュメント
- [Action recommended: Implement disaster recovery strategies for your Azure App Service web apps by 31 March 2025](https://azure.microsoft.com/ja-jp/updates/action-recommended-implement-disaster-recovery-strategies-for-your-azure-app-service-web-apps-by-31-march-2025/) 
- [App Service アプリを別のリージョンに移動する](https://learn.microsoft.com/ja-jp/azure/app-service/manage-disaster-recovery) 
- [自動バックアップの復元](https://learn.microsoft.com/azure/app-service/manage-backup?tabs=portal#restore-a-backup)
- [Azure Functions geo ディザスター リカバリー](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-geo-disaster-recovery)
- [Azure App Service での事業継続とディザスター リカバリーの戦略](https://learn.microsoft.com/ja-jp/azure/app-service/overview-disaster-recovery) 
<br>
<br>

---

<br>
<br>

2023 年 04 月 17 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>