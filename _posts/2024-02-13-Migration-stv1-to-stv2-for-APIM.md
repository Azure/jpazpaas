---
title: "API Management における stv1 から stv2 への移行について"
author_name: "Koichiro Higashi"
tags:
    - API Management
---

# はじめに
API Management ( 以降 APIM と表記 ) の stv1 プラットフォームが 2024 年 8 月 31 日をもって廃止されます。
期日以降、 stv1 プラットフォームで ホストされている APIM インスタンスはすべてシャットダウンされ、API 要求に応答しません。
既に [移行ガイド](https://learn.microsoft.com/ja-jp/azure/api-management/migrate-stv1-to-stv2) を公開しておりますので、計画的な移行にご協力の程よろしくお願いいたします。

また、当該移行ガイドはお客様から寄せられるお問い合わせやご要望によっては、よりスムーズに移行ができるよう内容が適宜変更される可能性があります。
移行作業を検討ならびに実行される際は日本語版と併せて [英語版](https://learn.microsoft.com/en-us/azure/api-management/migrate-stv1-to-stv2) にて最新情報をキャッチアップしてくださいますようお願いいたします。

# 比較
移行対象の APIM が VNET インジェクションされているかどうか、また、VNET インジェクションされている場合、外部モードか内部モードかによって移行プロセスにいくつか差分がございますので、以下の表にまとめました。
[移行ガイド](https://learn.microsoft.com/ja-jp/azure/api-management/migrate-stv1-to-stv2) と併せてご参照していただけますと幸いです。


| 項目       | VNET インジェクション なし                                                                                                                                      | VNET インジェクション（外部モード）                                                                              | VNET インジェクション（内部モード）                                                   |
|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| [VIP](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-ip-addresses#ip-addresses-of-api-management-service-in-vnet) 更新   | VIP (パブリック仮想 IP アドレス) を維持するオプションあり                                                                                                                                      | 必要                                                                                                | 必要                                                                     |
| 移行ダウンタイム | VIP を維持する場合： 約 15 分<br/>新しい VIP を使用(推奨)： なし *1                                                                                                         | なし *1                                                                                             | なし *1                                                                  |
| 所要時間     | 約 45 ～ 60 分 *2                                                                                                                                             | 約 45 ～ 60 分 *2                                                                                         | 約 45 ～ 60 分 *2                                                              |
| 前提条件  | なし                                                                                                                                                    | 以下 2 つを事前に用意 *3<br/>　- 現在の仮想ネットワーク内の新しいサブネット<br/>　- 新しいパブリック IPv4 アドレス                            | 以下 2 つを事前に用意 *3<br/>　- 現在の仮想ネットワーク内の新しいサブネット<br/>　- 新しいパブリック IPv4 アドレス |                                             |
| stv1 保持時間   | 約 15 分                                                                                                                                                 | 約 15 分（最大 48 時間まで延長可）*4                                                                                             | 約 15 分（最大 48 時間まで延長可）*4                                                                 |
| DNS 変更    | VIP を維持する場合：不要<br/>新しい VIP を使用 ( 推奨 ) ： 必要<br/>　- カスタムドメインを使用していない（ 既定ドメインを使用している ）場合：不要 *5<br/>　- カスタムドメインを既定ドメインに CNAME で紐づけている場合：不要 *5<br/>　- カスタムドメインを直接 VIP に紐づけている場合：必要 | カスタムドメインを使用していない（ 既定ドメインを使用している ）場合：不要 *5<br/>カスタムドメインを既定ドメインに CNAME で紐づけている場合：不要 *5<br/>カスタムドメインを直接 VIP  (パブリック仮想 IP アドレス) に紐づけている場合：必要 | 新しい VIP ( プライベート仮想IPアドレス ) への更新が必要                                                          |


*1 : 対クライアント/バックエンドとの既存コネクションは一旦切断されますが、クライアントから即座に再接続を実行することで新規コネクションを確立できます<br>
*2 : あくまで目安のため 2 時間以上 APIM の状態が 「 Updating（更新中）」など Online 状態にならない場合には [サポートリクエスト](https://learn.microsoft.com/ja-jp/azure/api-management/migrate-stv1-to-stv2?tabs=cli#help-and-support) にてお問い合わせください<br>
*3 : stv2 移行後に元のサブネットに変更する際にも新しいパブリック IPv4 アドレスが必要となります<br>
*4 : 新旧 APIM の共存は既定で約 15 分となっております。これは移行プロセス中に問題が発生した際に自動ロールバックするための時間であり、もしお客様自身にて新旧 APIM を使用して検証したい場合には最大 48 時間まで Azure ポータルからの [サポートリクエスト](https://learn.microsoft.com/ja-jp/azure/api-management/migrate-stv1-to-stv2?tabs=cli#help-and-support) 発行により事前に延長申請可能です。また実際にロールバックを要求する場合におきましても [サポートリクエスト](https://learn.microsoft.com/ja-jp/azure/api-management/migrate-stv1-to-stv2?tabs=cli#help-and-support) にてお問い合わせください<br>
なお、当該延長申請はお問い合わせいただいたサブスクリプション配下の全ての APIM に適用されます<br>
*5 : 既定ドメインは新しい VIP に自動更新されます<br>

# FAQs

* **SKU の種類 ( Developer, Basic, Standard, Premium など ) で移行時における挙動の差分はありますか？**
    * 一部 [【ネットワーク構成の適用】を実行する手順](https://learn.microsoft.com/ja-jp/azure/api-management/migrate-stv1-to-stv2?tabs=portal#update-vnet-configuration-1) がありますが、こちらを実行すると Developer SKU の場合はダウンタイムが発生します。その他は特筆すべき差分はありません。
<br><br>

* **stv1/subnet A -> stv2/subnet B -> stv2/subnet A と移行した場合、stv2/subnet A から 元のサブネット stv1/subnet A へロールバックできますか？ ※stv1/subnet A はサブネット A に属する stv1 の APIM を意味しています**
    * いいえ、stv2 に subnet A を設定するためには、stv1 から subnet A を解放する必要があり、当該 subnet A を解放すると、stv1 は削除されることになるため、結果的に stv2/subnet A -> stv1/subnet A へのロールバックはできません。
<br><br>

* **stv2 へ移行後、すぐに stv1 がデプロイされていた元の Vnet と サブネット に戻せますか？**
    * いいえ、stv2 移行後、既定で約 15 ～ 45 分 （ 最大 48 時間まで Azure ポータルからの [サポートリクエスト](https://learn.microsoft.com/ja-jp/azure/api-management/migrate-stv1-to-stv2?tabs=cli#help-and-support) 発行により事前に延長申請可能 ）は、 元の Vnet と サブネットには戻すことはできません。<br>
もし元の Vnet とサブネットに変更して更新しようとすると以下の[Resource Navigation Link が存在することによるエラー](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet?tabs=stv2#resource-navigation-links)が発生しますので、時間を空けてから再度実行するようにしてください。<br><br>
>エラーメッセージ：<br>
API Management service stv2 platform deployment into /subscriptions/<subscription id>/resourcegroups/<resource group name>/providers/microsoft.network/virtualnetworks/<vnet name>/subnets/<subnet name> is not supported, as it already contains API Management service(s) with stv1 platform in it as suggested by presence of "/subscriptions/<subscription id>/resourceGroups/<resource group name>/providers/Microsoft.Network/virtualNetworks/<vnet name>/subnets/<subnet name>/resourceNavigationLinks/<Navigation Link Name>". Please select a different subnet and try again. Refer to https://aka.ms/apim-infrastructure for definition of stv1 vs stv2.

<br><br>
元のサブネットにて Resource Navigation Link が削除されたかどうかは以下の REST API を実行することで確認できます。<br>
https://learn.microsoft.com/en-us/rest/api/virtualnetwork/resource-navigation-links/list
<br><br>
該当サブネットに Resource Navigation Link が存在する場合は以下の内容が返却されます。
```
    {
      "name": "Navigation Link Name",
      "id": "/subscriptions/<subscription id>/resourceGroups/<resource group name>/providers/Microsoft.Network/virtualNetworks/<vnet name>/subnets/<subnet name>/resourceNavigationLinks/<navigation link name>",
      "etag": "W/\"c905e4c5-7a4d-4d3c-8adf-8b83773c782b\"",
      "type": "Microsoft.Network/virtualNetworks/subnets/resourceNavigationLinks",
      "properties": {
        "provisioningState": "Succeeded",
        "linkedResourceType": "Microsoft.ApiManagement/service",
        "link": "/subscriptions/<subscription id>/providers/Microsoft.ApiManagement/service?vnetResourceGuid=<resource gu id>&subnet=<subnet name>&api-version=2016-07-07"
      }
    }
```
* **移行時に一時的には新旧２つのインスタンスが存在することになるが、stv2にアップグレードすることも含め追加コストは発生しますか？**
    * いいえ、移行中ならびに移行後において追加コストは発生しません。
<br><br>

* **stv1 での APIM サービスや API 構成（ ポリシー、バックエンド、カスタムドメイン、証明書、開発者ポータルコンテンツなど ）は　stv2 へ移行後も引き継がれますか？**
    * はい、引き継がれます。今回 stv2 への移行を行う上で変更する Vnet 構成以外の APIM の構成においてはそのまま引き継がれ、特に stv2 移行後に追加で設定変更を行う必要はありません。
<br><br>


* **48 時間を超過して stv1 と stv2 を共存させて検証したい場合はどうすればよいでしょうか？**
    * お客様にて APIM インスタンスを新規作成し、[バックアップ・リストア機能](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-disaster-recovery-backup-restore) や [開発者ポータルコンテンツ移行スクリプト](https://learn.microsoft.com/ja-jp/azure/api-management/automate-portal-deployments) などを用いて、手動にて stv1 と同じ構成に復元することになります。バックアップ・リストア機能 には [バックアップされないもの](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-disaster-recovery-backup-restore?tabs=powershell#what-is-not-backed-up) もありますのでご注意ください。
    * stv2 を新規作成する際には stv1 の API Management インスタンス名とは異なる名前（併せて既定ドメインも異なる）を使用する必要がありますので、APIM へのアクセス先を変更したくない場合には [カスタムドメイン](https://learn.microsoft.com/ja-jp/azure/api-management/configure-custom-domain) を設定するようご検討ください。
<br><br>


# 参考ドキュメント
* stv1 プラットフォームの廃止 (2024 年 8 月) <br>
https://learn.microsoft.com/ja-jp/azure/api-management/breaking-changes/stv1-platform-retirement-august-2024
<br><br>

* stv1 プラットフォームでホストされている API Management インスタンスを stv2 に移行する<br>
https://learn.microsoft.com/ja-jp/azure/api-management/migrate-stv1-to-stv2
<br> <br>

* Migrating API Management platform version from stv1 to stv2<br>
https://techcommunity.microsoft.com/t5/fasttrack-for-azure/migrating-api-management-platform-version-from-stv1-to-stv2/ba-p/3951108
<br> 


<br>
<br>

---

<br>
<br>

2024 年 2 月 8 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>