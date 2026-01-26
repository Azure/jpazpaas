---
title: "Azure Communication Services Email クォータ緩和について"
author_name: "Takeharu Oshida"
tags:
    - Azure Communication Services
    - Email
---

# はじめに
お世話になっております。Azure Communication Services サポート担当の押田です。

Azure Communication Services で Email を送信には規定でクォータが設けられております。

[Azure Communication Services のサービス制限 - An Azure Communication Services article | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/service-limits#email)

> 送信できる電子メール メッセージの数には制限があります。 サブスクリプションに応じた[メールのレート制限](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/service-limits#rate-limits-for-email)を超えた場合、要求が拒否されます。 再試行までの時間が経過した後に、これらの要求をもう一度試すことができるようになります。 必要に応じて、送信ボリュームの制限を引き上げるリクエストを行って、制限に達する前に対処してください。

クォータの緩和はサポートリクエストを起票いただき申請いただく必要がございます。

[メール ドメインのクォータの引き上げ - An Azure Communication Services concept document | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/email/email-quota-increase)

クォータの引き上げは申請いただいた内容ならびに、当該リソースの状況に応じて審査されます。

[Azure Communication Services メール機能でよく頂くご質問 - Japan PaaS Support Team Blog](https://azure.github.io/jpazpaas/2025/09/08/ACS-Email-Onboarding-FAQ.html#%E9%80%81%E4%BF%A1%E6%95%B0%E3%81%AE%E5%88%B6%E9%99%90%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6) にて、クォータ引き上げに関する FAQ を記載させていただいております。

この記事ではクォータの引き上げの際に弊社よりお伺いする情報や、申請が却下される要因について補足します。
なお、具体的な審査内容は公開されておりません。この記事で記載する項目がすべてではございませんことご了承くださいませ。

> メール クォータの引き上げ要求は自動的に承認されません。 レビュー チームは、承認の状態を決定するときに、送信者の全体的な評判を考慮します。 送信者の評判には、メール配信の失敗率、ドメインの評判、スパムや不正使用のレポートなどの要因が含まれます。

# クォータ緩和審査の対策

## ドメインの信頼性の維持

### カスタムドメインの管理

クォータの緩和対象はカスタムドメインのみとなります。申請にあったってはカスタムドメインを構成ください。

[カスタム検証済みメール ドメインを追加する - An Azure Communication Services article | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/add-custom-verified-domains?pivots=platform-azp)

- [Azure Communication Services メール機能でよく頂くご質問 - Japan PaaS Support Team Blog](https://azure.github.io/jpazpaas/2025/09/08/ACS-Email-Onboarding-FAQ.html) 
  > * 送信数の制限は引き上げられますか？

### カスタムドメインがブラックリストへの登録されていないこと

メール送信に用いるドメインが、RBLに登録されていないことが求められます。
下記は外部サービスとなりますが、指定したドメインがブラックリスト登録されているかどうかを確認できるサイトとなります。

[MultiRBL.valli.org - Results of the query 0410mail1.t-oshida.org](https://multirbl.valli.org/lookup/0410mail1.t-oshida.org.html)

もし、いずれかのリストにお客様の指定したドメインが登録されている場合、当該のリストを管理されているベンダーにお問い合わせいただき、解除の対応を進めていただく必要がございます。

以下の例では `0410mail1.t-oshida.org` はいずれのブラックリストにも登録されていないことが確認できます。

![image-c4655fff-b877-49b2-ba98-b2e8c942946f.png]({{site.baseurl}}/media/2026/01/image-c4655fff-b877-49b2-ba98-b2e8c942946f.png)

### MX レコードの登録

MX レコードを登録することで送信者ドメインの評判を向上に寄与します。

Azure Communication Serviceにおいて、カスタムドメイン利用時([メール通信サービスにドメインを追加する際の DNS 設定について - Japan PaaS Support Team Blog](https://azure.github.io/jpazpaas/2023/07/06/How-to-add-domain-to-Email-Communication-Service.html))に MX レコードの登録は必須ではございませんが、
Azure Communication Service から送信されたメールを受信するサーバーによっては、受信時に MX レコードの有無を判定に用いられる場合がございますため、
`<お客様のカスタムドメイン>` を管理するDNSにて、 `<お客様のカスタムドメイン>` 宛てのメールを振り分けるべき宛先をご確認、ご設定くださいませ。
なお、Azure Communication Service ならびに、Email Serviceについては、Email の送信を行うサービスとなり、Emailの受信機能はございません。
そのため、Azure Communication Serviceとして、 MX レコードに登録可能な正当なメールサーバーの IP や FQDN について提供できるものはございません。
`<お客様のカスタムドメイン>` 宛てのメールをどのサーバーへ転送すべきかについては、お客様次第となるものとなり、弊社側から具体的なレコードをお出しするものではございません。

- [メール ドメインのクォータの引き上げ - An Azure Communication Services concept document | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/email/email-quota-increase#3-configure-a-mail-exchange-record-for-your-custom-domain)

  > メール交換 (MX) レコードは、ドメイン名に代わって電子メール メッセージを受信する電子メール サーバーを指定します。 MX レコードは、ドメイン ネーム システム (DNS) のリソース レコードです。 基本的に、MX レコードは、ドメインが電子メールを受信できることを示します。
  >
  > Azure Communication Service では送信メールのみがサポートされますが、送信者ドメインの評判を向上させるために MX レコードを設定することをお勧めします。 MX レコードがないカスタム ドメインからのメールは、受信者の電子メール サービス プロバイダーによってスパムとしてラベル付けされる可能性があります。 これにより、ドメインの評判が低下する可能性があります。


### DMARC レコードの登録

MX レコードと同様にカスタムドメイン利用時に DMARC レコードは必須ではございませんが、DMARC レコードを設定いただくことで送信者ドメインの評判を向上に寄与します。

- [送信者認証のサポートに関するベスト プラクティス - An Azure Communication Services article | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/email/email-authentication-best-practice#implementing-dmarc)

- [DMARC を使用してメールを検証する、セットアップ手順 - Microsoft Defender for Office 365 | Microsoft Learn](https://learn.microsoft.com/ja-jp/defender-office-365/email-authentication-dmarc-configure?preserve-view=true&view=o365-worldwide#syntax-for-dmarc-txt-records)


## 送信成功率の維持

メールの失敗率は 1% 未満であることが求められます。
検証目的であったとしても、当該のドメインからの失敗率が高い場合、申請は却下される可能性が高くございます。

- [Azure Communication Services 電子メールの送信者の評判を理解する - An Azure Communication Services article | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/email/sender-reputation-managed-suppression-list#managing-sender-reputation-and-email-complaints-to-enhance-email-delivery)

  > **送信者の評判と電子メールの苦情を管理してメール配信を強化する**
  >
  > Azure Communication Services には、顧客の通信を強化できる電子メール機能が用意されています。 ただし、プラットフォーム経由で送信したメールが顧客の受信トレイに届く保証はありません。 配信の問題を事前に特定して回避するには、次のような評判チェックを実行する必要があります。
  > *   正常に配信された電子メールの一貫性と正常な割合を一定期間にわたって確保します。
  > *   電子メール配信の失敗とバウンスに関する特定の詳細の分析。
  > *   スパムおよび不正使用レポートの監視。
  > *   正常な連絡先リストを維持する。
  > *   ユーザーエンゲージメントとメールボックス配置の理解。
  > *   顧客の苦情を理解し、オプトアウトまたは登録解除のための簡単なプロセスを提供します。

- [Azure Communication Services メール機能でよく頂くご質問 - Japan PaaS Support Team Blog](https://azure.github.io/jpazpaas/2025/09/08/ACS-Email-Onboarding-FAQ.html) 
  > * 送信数のクォータ引き上げ審査でメール送信失敗率は加味されますか？
  > * 失敗率の監視はどこからできますか？


## 段階的な緩和について

カスタムドメインに対するデフォルトのレート制限は、30件/min、100件/hour となります。
具体的な値をお伝えすることができず恐縮ではございますが、制限の緩和は送信実績に基づき段階的に実施させていただいております。
そのため、現在の制限を大幅に上回るような申請をいただいてもご希望に沿える可能性は低くございます。
目安として、現在の制限に抵触する状態が続くようであればその倍の容量をご申請いただければと存じます。

- [メール送信層の制限に達したときに例外をスローする - An Azure Communication Services article | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/send-email-advanced/throw-exception-when-tier-limit-reached?pivots=programming-language-javascript)

- [Azure Communication Services のサービス制限 - An Azure Communication Services article | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/service-limits#email)

  > メールの配信状態を注意深く監視しながら、2 週間から 4 週間かけて、Azure Communication Services Email を使用するメールの量を少しずつ増やすことをお勧めします。 このように段階的に増やすと、サード パーティのメール サービス プロバイダーはドメインのメール トラフィック用の IP の変更に適応できます。 少しずつ変更すると、送信者評価を保護し、メール配信の信頼性を維持する時間が得られます。

一方でシステム要件として予定されている送信数があらかじめ定まっている場合なども多かろうと存じます。
そのような場合、ビジネスインパクトやスケジュール等の詳細をお伺いさせていただき、可能な範囲で調整させていただくこともございます。一度サポートまでご相談くださいませ。

## 弊社からのヒアリング事項
ドキュメントに沿ってサポートリクエスト起票時に記入くださいませ。

[メール ドメインのクォータの引き上げ - An Azure Communication Services concept document | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/email/email-quota-increase)

日本語で記載いただいてもかまいません。
未記載の項目がございますと、審査プロセスを進めるまでに時間がかかってしまうことになります。ご注意くださいませ。

```
1.送信を利用するお客様の会社名 (英語表記) :
2.送信を利用するお客様の会社のウェブサイト URL :
3.送信を利用するお客様の会社の事業内容について簡潔にご説明ください :

4.該当の Azure Communication Services リソースでは、既にカスタムドメインを構成し、カスタムドメインからメールを送信できている状態でしょうか :
5.メール送信に利用されているカスタムドメイン名 :
6.Azure Communication Services リソース名および Email Services リソース名
・Azure Communication Servicesリソース名 :
・Email Services リソース名:
7.サブスクリプション ID :
8.テナント名 :
9.テナント ID :

10.本メール機能をどのような目的でご利用されるか (お客様との取引、売買 (ショッピング等)、宣伝等) :
11.ご希望の上限:
 - 件/1分
 - 件/1時間
 - 件/1日
12.送信先のメールアドレスはどのような方法で取得しましたか :
13.送信先となるユーザーが配信解除を希望、もしくは送信不達となった場合、対象ユーザーをどのように送信先から除外するか :
14.現在該当のドメインにてどのようなプラットフォームをメール送信時に使用されているか教えてください。：例: オンプレミス、Amazon SES 等
```

# クォータ緩和後のガイドライン
  
メール送信量を増やしていく際のガイドラインとなります。ご参考にしてくださいませ。

- MXレコードが未設定の場合、MX レコードの追加を実施くださいませ。

- 低い失敗率と良好なNDR（配信不能通知）は、高いメールクォータを維持するために重要です。以下の2つの重要なガイダンスに従っていることを確認してください。
  [Azure Communication Services のサービス制限 - An Azure Communication Services article | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/service-limits#email)

  - 1. 移行アプローチ
    > メールの配信状態を注意深く監視しながら、2 週間から 4 週間かけて、Azure Communication Services Email を使用するメールの量を少しずつ増やすことをお勧めします。 このように段階的に増やすと、サード パーティのメール サービス プロバイダーはドメインのメール トラフィック用の IP の変更に適応できます。 少しずつ変更すると、送信者評価を保護し、メール配信の信頼性を維持する時間が得られます。

   - 2. 失敗率の管理
      > 失敗率の要件
      > 
      > 高いメール クォータを有効にするには、メールの失敗率が 1% 未満である必要があります。 失敗率が高い場合は、クォータの引き上げを要求する前に問題を解決する必要があります。 お客様は、失敗率を積極的に監視する必要があります。
      > 
      > クォータの引き上げ後に失敗率が上がった場合、Azure Communication Services はお客様に連絡して、直ちに対処することを求め、解決のタイムラインを確認します。 極端なケースでは、指定されたタイムライン内で失敗率が管理されない場合、Azure Communication Services は、問題が解決されるまでサービスを減らしたり中断したりする可能性があります。

下記のドキュメント群もご参考にしていただけますと幸いです。
- [Azure Communication Services 電子メールの送信者の評判を理解する - An Azure Communication Services article | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/email/sender-reputation-managed-suppression-list)
- [メールの分析情報 - An Azure Communication Services article | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/analytics/insights/email-insights)
- [Azure Monitor でログ記録を有効にする - An Azure Communication Services article | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/analytics/enable-logging)
- [メールイベントを処理する - Azure Communication Services | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/handle-email-events)
- [Azure Communication Services - Email イベント - Azure Event Grid | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/event-grid/communication-services-email-events)
- [Management SDK を使用してドメイン抑制リストを管理する - An Azure Communication Services article | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/manage-suppression-list-management-sdks?pivots=programming-language-javascript)
- [Azure Communication Services メール機能でよく頂くご質問 - Japan PaaS Support Team Blog](https://azure.github.io/jpazpaas/2025/09/08/ACS-Email-Onboarding-FAQ.html)
