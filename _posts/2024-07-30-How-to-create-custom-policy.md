---
title: "カスタムポリシーの作成手順について"
author_name: "Hirotaka Shionoiri"
tags:
    - Policy
---

# 質問
私たちの要件を満たすカスタムポリシーを作成したいです。<br>
サポートリクエストを起票することで要件を満たすポリシーを作成してくれませんか？

# 回答
大変恐縮ながら、サポートリクエストでは、<br>
Azure Policy や ARM Template、各種クエリ等の開発依頼を受け付けておりません。<br>
必要に応じてサンプル等のご提供をさせて頂く場合もございますが、<br>
基本的には、製品のご利用方法や仕様などについて公開情報をもととしたご案内をいたしておりまして、<br>
お客様が Azure Policy の定義や ARM テンプレートの作成をするにあたりお役に立つ情報のご提供を行っております。<br>
<br>
「カスタムポリシーでの対応を行いたいが、作成方法が分からない」という方も多いかと思います。<br>
本記事では、カスタムポリシーの作成手順についてご案内します。<br>

## ＜Azure Policy の作成の基礎について＞
カスタムポリシーの作成方法の大まかな流れは下記の資料が参考になるかと思います。<br>
[チュートリアル:カスタム ポリシー定義の作成 - Azure Policy | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/governance/policy/tutorials/create-custom-policy-definition)<br>
<br>
Azure Policy の構造については下記の資料を一度ご確認下さい。<br>
[ポリシー定義の構造の基本の詳細 - Azure Policy | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/governance/policy/concepts/definition-structure-basics)<br>
<br>
Azure Policy の効果については下記の資料をご確認下さい。<br>
[Azure Policy 定義の効果の基本 - Azure Policy | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/governance/policy/concepts/effect-basics)<br>
<br>
カスタムポリシーを作成する際には、上記の資料、<bR>
特に Azure Policy の構造に関する資料と、Azure Policy の効果に関する資料を参照頂きながら、<br>
作業をするケースが多いかと思います。
## ＜ビルトインポリシーについて＞
ビルトイン（組み込みの）ポリシーも数多く用意されております。<br>
まずは、目的のポリシーに極力近しいビルトインポリシーを探してみると良いかと思います。<br>
下記の資料では、提供されている組み込みのポリシーが記載されております。<br>
[組み込みのポリシー定義の一覧 - Azure Policy | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/governance/policy/samples/built-in-policies)<br>
<br>
サービスによっては、サービスの資料中にて、そのサービスに関するポリシーをまとめている例もございます。<br>
こちらの方が目的に近いポリシーを探すことが容易かとも思います。<br>
一例として、Azure VM や Azure Storage の場合は下記の様な資料がございました。<br>
[Azure Virtual Machines 用の組み込みポリシー定義 - Azure Virtual Machines | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/virtual-machines/policy-reference)<br>
[Azure Storage 用の組み込みポリシー定義 | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/storage/common/policy-reference)<br>
<br>
監査対象のプロパティが異なっていたとしても、<br>
対象の type など、ある程度は一致する部分があるかとは思いますので、<br>
ビルトインポリシーをひな型として、カスタムポリシーを作成されると良いかと思います。<br>

## <カスタムポリシーの作成例について>
下記のWebサイト、"AzAdvertizer" ではカスタムポリシーの作成例を確認することが可能です。<br>
[AzPolicyAdvertizer (azadvertizer.net)](https://www.azadvertizer.net/azpolicyadvertizer_all.html)<br>
<br>
AzAdvertizer は Microsoft の従業員によって開発されていますが、<br>
Microsoft のサービスや製品ではないことにご注意ください。<br>
つまり、ここで掲載されているポリシーについてサポートなどは提供されておりません。<br>
また、このサイトの問題や掲載されているポリシーの問題についてはお問い合わせを受け付けておりません。<br>
<br>
ご利用者様自身で検証頂く必要はございますが、<br>
より目的に近しいポリシーのひな型が見つかる可能性もございます。<br>
ビルトインポリシーと合わせて、AzPolicyAdvertizer もご確認頂き、<br>
ご希望のポリシーにより近いカスタムポリシーを、<br>
作成されるカスタムポリシーのひな型として選択頂くと良いかと思います。<br>

## ＜エイリアスの確認方法について＞
そもそも、Azure Policy はデプロイされる ARM Template、<br>
あるいは既存のリソースの構成を ARM Template で取得することによって、プロパティを確認しています。<br>
プロパティを確認した際に、条件に一致しているかどうかを確認の上、制限している構成に一致していれば、<br>
ポリシーの効果を適用するというものです。<br>
<br>
よって、カスタムポリシーを作成する上では、<br>
作成したいリソースの ARM Template のリファレンスもご確認頂くと良いかと思います。<br>
一例として、Azure VM や Azure Storage の ARM Template リファレンスを下記に記載します。<br>
[Microsoft.Compute/virtualMachines - Bicep, ARM template & Terraform AzAPI reference | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/templates/microsoft.compute/virtualmachines?pivots=deployment-language-arm-template)<br>
[Microsoft.Storage/storageAccounts - Bicep, ARM template & Terraform AzAPI reference | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/templates/microsoft.storage/storageaccounts?pivots=deployment-language-arm-template)<br>
<br>
「（対象のサービス名） ARM Template リファレンス」<br>
などと検索頂くと、上記の様なリファレンスがご確認頂けるかと思います。<br>
<br>
上述のビルトインポリシーや ARM Template リファレンスから、<br>
対象のリソースのタイプを特定します。<br>
> 例）Microsoft.Storage/storageAccounts

また、ARM Template リファレンスを確認し、監査したい目的のプロパティのパスを特定します。<br>
> 例）properties.publicNetworkAccess

対象のプロパティに入る値についても確認しておきます。<br>
> 例）publicNetworkAccess の場合、'Disabled' または 'Enabled'

対象のプロパティのパスを確認しましたが、<br>
実は、Azure Policy の定義において、監査をするプロパティの指定方法は ARM Template と同じパスではなく、<br>
Azure Policy 用のエイリアスが定義されており、このエイリアスを利用して対象の項目を監査します。<br>
<br>
エイリアスの確認方法については下記の資料をご確認下さい。<br>
[ポリシー定義構造の別名の詳細 - Azure Policy | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/governance/policy/concepts/effect-deny)<br>
<br>
端的には、例えば Azure Storage のエイリアスの場合、<br>
下記の様なPowershellコマンドでご確認頂くことが可能です。<br>
```
$allAliases = Get-AzPolicyAlias -NamespaceMatch 'Microsoft.storage' -ResourceTypeMatch 'storageAccounts'
$allAliases.Aliases
```
エイリアスの中から、上記の例に挙げたような、<br>
properties.publicNetworkAccess を監査するためのエイリアスを探すと、<br>
> Microsoft.Storage/storageAccounts/publicNetworkAccess

![image-5431be42-f4d8-4f48-adad-2a12742276a9.png]({{site.baseurl}}/media/2024/07/image-5431be42-f4d8-4f48-adad-2a12742276a9.png)

であることが分かります。<br>
なお、エイリアスが提供されていない場合もございます。<br>
その場合はそのプロパティを監査するカスタムポリシーを作成することができません。<br>
<br>
## ＜実際に作成してみる＞
以上がカスタムポリシーを作成する際のヒントとなりますが、<br>
最後に、実際にカスタムポリシーを作成してみます。<br>
<br>
今回は、Azure Storage のポリシーを作成してみます。<br>
「ストレージアカウントのSKUを制限しつつ、ストレージアカウントのパブリックネットワークアクセスも制限したい」<br>
という想定で、<br>
「ストレージアカウントの SKU が許可されたものではない、またはパブリックネットワークアクセスが有効になっていたら、作成や更新を拒否する」<br>
というカスタムポリシーを作成してみます。<br>
<br>
まずはひな型となるポリシーを探してみます。<br>
下記のポリシーが目的のポリシーに近そうです。<br>
<br>
「ストレージ アカウントを許可されている SKU で制限する必要がある」<br>
（/providers/Microsoft.Authorization/policyDefinitions/7433c107-6db4-4ad1-b57a-a76dce0154a1）<br>
<br>
ここに、<br>
「またはパブリックネットワークアクセスが有効になっていたら」<br>
という条件を追加する方針でカスタムポリシーを作ります。<br>
パブリックネットワークアクセスを設定している Azure Storage 上のプロパティは、<br>
"エイリアスの確認方法について" の章に記載した方法で確認をすると、
> properties.publicNetworkAccess

であることが分かります。また、その項目を監査するためのエイリアスは、
> Microsoft.Storage/storageAccounts/publicNetworkAccess

でした。<br>
よって、ポリシー定義の条件に、<br>
「"Microsoft.Storage/storageAccounts/publicNetworkAccess" が、Enabled であれば」<br>
という条件を追加すればよいことが分かります。<br>
<br>
複数の条件を取り入れる際には、allOf、または anyOf を利用してルールを定義します。<br>
詳細は下記の資料をご参照下さい。<br>
[ポリシー定義の構造ポリシー ルールの詳細 - Azure Policy | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/governance/policy/concepts/definition-structure-policy-rule)<br>
<br>
簡潔には、allOf は論理式で言う"AND"、文章で表現するところの "かつ" であり、<br>
anyOf は論理式で言う"OR"、文章で表現するところの "または" に該当します。<br>
<br>
これらのことを踏まえて、<br>
「ストレージアカウントの SKU が許可されたものではない、またはパブリックネットワークアクセスが有効になっていたら」<br>
という条件は、Azure Policy の条件としては下記の3つの条件に分解できます。<br>

1. リソースのタイプが "Microsoft.StorageAccounts" である
2. "Microsoft.Storage/storageAccounts/sku.name" が許可された SKU に当てはまらない
3. "Microsoft.Storage/storageAccounts/publicNetworkAccess" が、Enabledである

1、2は既に発見しているビルトインポリシーに記載があるので、その形式を参考にします。<br>
3の条件を1、2とどのようにつなぐと正しいかを考えると、<br>
1 AND (2 OR 3)<br>
であることが分かります。<br>
また「拒否する」という効果は "deny" が該当します。<br>
<br>
以上の事から、ビルトインポリシー「ストレージ アカウントを許可されている SKU で制限する必要がある」をもとに、<br>
下記の様なポリシーを作ります。
    
```
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Storage/storageAccounts"
        },
        {
          "anyOf": [
            {
              "not": {
                "field": "Microsoft.Storage/storageAccounts/sku.name",
                "in": "[parameters('listOfAllowedSKUs')]"
              }
            },
            {
              "field": "Microsoft.Storage/storageAccounts/publicNetworkAccess",
              "equals": "Enabled"
            }
          ]
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {
    "listOfAllowedSKUs": {
      "type": "Array",
      "metadata": {
        "displayName": "許可されている SKU",
        "description": "ストレージ アカウントに指定できる SKU の一覧。",
        "strongType": "StorageSKUs"
      }
    }
  }
}
```
anyOf の中身、及び effect の部分がビルトインポリシーから変更した箇所となります。<br>
<br>
=========================<br>
<br>
上述の様な流れでカスタムポリシーを作成していきます。<br>
なお、ポリシーは割り当ててから実際に評価を実施して、コンプライアンス結果に反映するまでに時間がかかります。<br>
ポリシーを割り当てた後、明示的に下記の監査を実行するコマンドを実施頂き、<br>
実行完了をお待ち頂くと良いかと思います。<br>
[az policy state | Microsoft Learn](https://learn.microsoft.com/en-us/cli/azure/policy/state?view=azure-cli-latest#az-policy-state-trigger-scan)<br>
[Start-AzPolicyComplianceScan (Az.PolicyInsights) | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/az.policyinsights/start-azpolicycompliancescan?view=azps-11.6.0&viewFallbackFrom=azps-4.7.0)<br>
<br>
上記の様な流れでカスタムポリシーを作成します。<br>
カスタムポリシーを作成した上で、ご希望のような動作にならない、<br>
という場合にはサポートリクエストを起票頂き、作成されたカスタムポリシーと共にお問い合わせ頂けますと幸いです。

# 参考ドキュメント
[Policy 作成のヒント集 - Japan PaaS Support Team Blog (azure.github.io)](https://azure.github.io/jpazpaas/2020/10/01/hint-for-creating-policy.html)

<br>
<br>

---

<br>
<br>

2024 年 7 月 18 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>