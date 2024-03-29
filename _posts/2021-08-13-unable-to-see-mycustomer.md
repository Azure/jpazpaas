---
title: "Azure Lighthouse で顧客サブスクリプションをオンボードした際に Azure Portal の [マイ カスタマー] に表示されない"
author_name: "Haruka Matsumura"
tags:
    - Lighthouse
---

# 質問
Azure Lighthouse を用いて、顧客サブスクリプションの委任を行いました。<br>
顧客サブスクリプションのある委任元のテナントでは、[サービス プロバイダー] ページから委任が確認できるのですが、委任先のテナントの [マイ カスタマー] ページに委任されている情報が表示されません。<br>
設定等に問題があるのでしょうか。<br>

# 前提動作
Azure Lighthouse を利用した顧客リソースのオンボード (委任) 作業に関しましては、下記公開情報にて手順の紹介がございますのでこちらをご参考いただければと存じます。<br>

Azure Lighthouse への顧客のオンボード<br>
[https://docs.microsoft.com/ja-jp/azure/lighthouse/how-to/onboard-customer](https://docs.microsoft.com/ja-jp/azure/lighthouse/how-to/onboard-customer)

# トラブルシューティング
Azure Portal にて [マイ カスタマー | 顧客] 内に委任元の情報が確認できない場合、以下の点をご確認ください。<br>

## 委任されたディレクトリ (テナント) が選択されているか
[マイカスタマー | 顧客] 情報の確認ページでは、既定ではグローバル サブスクリプション フィルターがオンとなっています。<br>
初回オンボード時、または初回以降で既定の設定をオフに設定されていない場合、Azure Portal 全体の設定の [既定のサブスクリプション フィルター] で指定されたサブスクリプションで選択されていない情報をご確認いただくことができません。<br>

グローバル サブスクリプション フィルターは Azure Portal 右上の \[歯車アイコン\] - \[ディレクトリとサブスクリプション\] よりご確認いただけますので、こちらで委任されたディレクトリならびに委任されたサブスクリプションをご選択の上、[マイカスタマー | 顧客] をご確認ください。<br>
※ [既定のサブスクリプション フィルター] が更新されていない場合、後述の項目をご確認の上 [既定のサブスクリプション フィルター] が更新されるかご確認ください。<br>
<br>

![Default_Subscription_Filter.jpg]({{site.baseurl}}/media/2021/08/2021-08-03_17h59_42.jpg)<br>

## 権限を持ったユーザーであるか
委任情報をご確認いただくには、オンボード対象に対する承認が必要です。ログインしているユーザーが、オンボードの設定で権限を付与されたユーザー (または権限を付与された Azure AD グループに属するユーザー) であるかご確認ください。<br>

権限が付与されているユーザー (あるいはグループ) の一覧は、オンボード時に使用した ARM テンプレート内の "authorizations" -> "value" -> "principalId" よりご確認いただけます。
また、委任元のテナントで、[サービス プロバイダー | 委任] ページを開き、対象の委任情報の [ロールの割り当て] ページを開いていただくことでも、同様の情報をご確認いただくことが可能です。
<br>

![Service_Privider_onboarding_roledefinition.jpg]({{site.baseurl}}/media/2021/08/2021-08-10_18h53_49_a.jpg)<br>

## ブラウザのキャッシュなどが残存していないか
ブラウザによっては、キャッシュにより情報が確認いただけない場合が考えられます。<br>
プライベート モード (たとえば Microsoft Edge の場合、\[Ctrl\] + \[Shift\] + \[N\] キー) で開いたブラウザで Azure Portal の [マイ カスタマー | 顧客] ページを開いていただき、情報が更新されるかご確認ください。<br>
<br>

# 解決しない場合は
上記トラブルシューティングの項目をご確認いただいても解決いただけなかった場合、我々サポート チームよりご支援ができればと思いますので、以下情報とともに弊社までお問い合わせくださいませ。<br>

- 委任先 / 委任元のテナント ID ならびに委任リソースの情報
- 上記トラブルシューティングの各項目を確認した際の情報
- 事象発生のブラウザ トレース情報
>トラブルシューティングのためにブラウザー トレースをキャプチャする<br>
[https://docs.microsoft.com/ja-jp/azure/azure-portal/capture-browser-trace](https://docs.microsoft.com/ja-jp/azure/azure-portal/capture-browser-trace)

<br>
本内容が、少しでも皆様のお役に立てましたら幸いです。
<br>
<br>

---

<br>
<br>

2021 年 8 月 13 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>