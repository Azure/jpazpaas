---
title: "Azure App Service の計画メンテナンス通知の改善"
author_name: "Takeharu Oshida"
tags:
    - App Service
    - Function App
    - Web Apps
    - App Service Environment
---

# はじめに

本記事は 2025 年 4 月 29 日に公開されました [Routine Planned Maintenance Notifications Improvements for App Service](https://azure.github.io/AppService/2025/04/29/Azure-App-Service-Notifications-Improvements.html) の日本語訳です。

2025年4月現在、App Serviceの定期メンテナンス通知に関して大幅な改善を発表できることを嬉しく思います。

# 最近の改善点

App Serviceのお客様体験を向上させるため、メンテナンス通知システムに重要な改善を行いました。この更新は、2022年3月に発表された内容 [Routine Planned Maintenance Notifications for Azure App Service - Azure App Service](https://azure.github.io/AppService/2022/02/01/App-Service-Planned-Notification-Feature.html) (日本語訳 [Azure App Service の計画メンテナンスの事前通知 - Japan PaaS Support Team Blog](https://azure.github.io/jpazpaas/2023/02/16/Routine-Planned-Maintenance-Notifications-for-Azure-App-Service.html))をさらに拡張したものです。

* * *

# 影響を受けたリソース ブレード

主な改善点の1つとして、Azure Service Healthに「影響を受けたリソース」ブレードが導入されました。この新機能により、お客様は影響を受けるApp Serviceプランリソースを正確に確認することができるようになりました。
[影響を受けたリソース] ブレードでは、メンテナンスの開始時と終了時の正確なステータス タイムスタンプを提供することで、メンテナンスの進行状況を明確かつ詳細に表示できます。このセルフサービス機能により、お客様はリソースのステータスを個別に追跡できます。

Azureポータルから確認するには、**ホーム** > **モニター** > **サービス正常性** > **計画メンテナンス** > **問題名を選択** > **影響を受けたリソース** > **詳細情報**　へとアクセスします。

![MoreInfo1-90c0f7d3-9911-4011-b995-7e20b7b0d69d.png]({{site.baseurl}}/media/2025/05/MoreInfo1-90c0f7d3-9911-4011-b995-7e20b7b0d69d.png)

ここでは、App Serviceプラン内でアップグレードが行われているリソースとその現在の状態（保留中、開始済み、完了済み）をタイムスタンプ付きで確認できます。

![MoreInfo2-2c062b64-1a99-4471-b985-2aec9ac20cee.png]({{site.baseurl}}/media/2025/05/MoreInfo2-2c062b64-1a99-4471-b985-2aec9ac20cee.png)
    

# 自動化されたリリースノート

また、自動化されたリリースノートも実装されました。お客様は、メンテナンス通知内で、最も重要な情報のみを提供する App Service リリース ノートへの自動リンクを受け取るようになりました。これにより、基本的なリリースノートに対する高い需要に対応し、お客様が重要なアップデートにアクセスできるようにします。[App Service リリース ノート](https://github.com/Azure/AppService/releases)


# ビジネスアワーにおけるのアップグレード停止

もう 1 つの重要な機能強化は、、ビジネスアワーにおけるの App Service プラン リソースのアップグレードの一時停止です。メンテナンスオペレーションは、午前9時から午後5時までの標準営業時間外に開始するように最適化されています。特定のリージョンで午前 9 時までにリソースがまだアップグレード中である場合、アップグレードは安全な停止ポイントに達するまで続行され、次の重要なステップの前で一時停止し、営業時間が終了するまで待機します。このアプローチにより、ピーク時の顧客のワークロードの中断が最小限に抑えられ、より予測可能なメンテナンス スケジュールが提供されます。


<br>
<br>

2025 年 05 月 07 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>