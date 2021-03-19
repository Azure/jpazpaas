---
title: "Azure Lighthouseの制限について"
author_name: "Taiyo Kato"
tags:
    - Lighthouse
---

「Azure Lighthouseでオンボードしたリソースを操作できない」というお問い合わせをたびたび頂戴しております。
Azure Lighthouseを使用する機会や場面が少なく、現在も成長中の製品である為制限事項がございます。今回はこちらの主な制限についてご紹介します。


登場人物は以下になります。

- ベンダー/サービスプロバイダー: 代わりに運用する側
- 顧客/お客様: 運用を委任する側


## Lighthouseの制限事項

現在Azure Lighthouseにはいくつかの制限があります。Azure Lighthouseドキュメントを参照しますと、以下の制限事項があります。

> - Azure Resource Manager で処理される要求は、Azure Lighthouse を使用して実行できます。 これらの要求の操作 URI は、`https://management.azure.com` で始まります。 ただし、リソースの種類のインスタンス (Key Vault のシークレット アクセスやストレージのデータ アクセスなど) によって処理される要求は、Azure Lighthouse ではサポートされていません。 これらの要求の操作 URI は、通常、`https://myaccount.blob.core.windows.net` や `https://mykeyvault.vault.azure.net/` など、実際のインスタンスに固有のアドレスで始まります。 また、通常、後者は管理操作ではなくデータ操作です。
> - ロールの割り当てには [Azure 組み込みロール](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles)を使用する必要があります。 現在、組み込みロールはすべて、Azure の委任されたリソース管理によってサポートされています。ただし、所有者または [`DataActions`](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/role-definitions#dataactions) アクセス許可を持つ組み込みロールは除きます。 [マネージド ID へのロールの割り当て](https://docs.microsoft.com/ja-jp/azure/lighthouse/how-to/deploy-policy-remediation#create-a-user-who-can-assign-roles-to-a-managed-identity-in-the-customer-tenant)において、ユーザー アクセス管理者ロールは、限定された用途のみに対してサポートされています。 カスタム ロールと[従来のサブスクリプション管理者ロール](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/classic-administrators)はサポートされていません。
> - Azure Databricks を使用するサブスクリプションをオンボードすることはできますが、現時点では、管理テナントのユーザーは、委任されたサブスクリプションで Azure Databricks ワークスペースを起動することはできません。
> - リソース ロックがあるサブスクリプションとリソース グループをオンボードすることはできますが、このようなロックがあっても、管理テナントのユーザーによるアクションの実行は妨げられません。 Azure マネージド アプリケーションまたは Azure Blueprints (システム割り当ての拒否割り当て) によって作成されたものなど、システムの管理対象リソースを保護する[拒否割り当て](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/deny-assignments)がある場合、管理テナントのユーザーはそれらのリソースを操作できません。ただし、現時点では、顧客テナントのユーザーは自分の拒否割り当て (ユーザー割り当て拒否割り当て) を作成できません。
> - [各国のクラウド](https://docs.microsoft.com/ja-jp/azure/active-directory/develop/authentication-national-cloud)と Azure パブリック クラウドにわたって行われる、または 2 つの独立した国内クラウドにわたって行われるサブスクリプションの委任はサポートされていません。

[テナント間の管理エクスペリエンス - Azure Lighthouse \| Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/lighthouse/concepts/cross-tenant-management-experience#current-limitations)



このうち、一つ目の制限に注目します。

> Azure Resource Manager で処理される要求は、Azure Lighthouse を使用して実行できます。 これらの要求の操作 URI は、`https://management.azure.com` で始まります。 **ただし、リソースの種類のインスタンス (Key Vault のシークレット アクセスやストレージのデータ アクセスなど) によって処理される要求は、Azure Lighthouse ではサポートされていません。** これらの要求の操作 URI は、通常、`https://myaccount.blob.core.windows.net` や `https://mykeyvault.vault.azure.net/` など、実際のインスタンスに固有のアドレスで始まります。 また、通常、後者は管理操作ではなくデータ操作です。
>
> ※注: 管理操作=コントロールプレーンーAzure上のリソース自体の操作、データ操作=データプレーン操作ーリソース内のデータの操作

[コントロール プレーンとデータ プレーンの操作 - Azure Resource Manager \| Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/azure-resource-manager/management/control-plane-and-data-plane)


これまで一部のリソースで、顧客テナント側で操作を行った際に正常にできたにも関わらず、ベンダーテナント側で同じコントロールプレーン操作をすると、一部の製品がクロステナントデータプレーン操作が行えず、承認エラーが発生することがございます。現在も引き続き弊社にて調整を進めておりますため、ご承知おきいただけますと幸いです。

### 例： Azure PolicyとNSG

以下のスクリーンショットでは、Azure PolicyにてポリシーにとあるNSGの非準拠原因について確認する時に出たエラーです。
顧客テナントでは同じ画面では正常に非準拠原因が確認できたものの、ベンダーテナントを通して確認すると下記エラーが表示されました。

Azure Policyでは、認可処理実行時に、リクエスト元（=ベンダーテナント）の認可トークンを使用して操作をしようとしますが、操作対象のテナントは自身と異なるテナントIDを検知し、承認エラーが発生します。

![image.png]({{site.baseurl}}/media/2021/03/2021-03-19-policy-compliance-detail-auth-error.png)

正常に閲覧可能な場合、以下の様にコンプライアンス違反の理由や違反箇所が表示されます。

![image.png]({{site.baseurl}}/media/2021/03/2021-03-19-policy-compliance-detail.png)


### ワークアラウンドについて


2020年11月頃に一部のコントロールプレーン操作で権限認証認可が正しく行われない問題がありましたが、再現環境を用意して検証したところ正常に表示された為修正されたかと思われます。


もし現在も尚特定の操作ができないと言った場面に直面した場合、顧客テナントでベンダーテナントのユーザーをIAM追加することをお薦めします。
Azure Lighthouseの様に複数の顧客テナントを一つのベンダーテナント画面で管理することができませんが、
顧客テナントと同一テナント内での操作という扱いになり、正常に操作できるようになりますので、ご参考いただけると幸いです。


<br>
<br>

---

<br>
<br>

2021 年 3 月 19 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>