---
title: "AI Search アナライザーのトークナイズと正規表現による検索の動作について"
author_name: "yiwenchen"
tags:
    - "Azure Resource Manager (ARM)"
---
こんにちは、PaaS Developer チームの陳です。

今回は、Azureサービスをバックアップする、IaC観点でサービスを管理する際のベストプライスを紹介します。

# 質問

Azure リソースの設定のバックアップを取る方法がありますか？

障害等で Azure リソース自体のリストア方法があるか？

# 回答

Azure リソースのバックアップを考える際に、Azure リソースの設定自体をバックアップするのか、それとも Azure リソースに保存されているデータのバックアップをするのかを分けて考える必要があります。

**データのバックアップ**の場合は、ローカルで保存するなどではなく、サービスのパックアップ機能を使用します。

 ### 例：

ストレージアカウントのデータバックアップには、冗長性を構成することで、保存機器レベル、またはデータセンターレベルの障害、災害からデータを保護できます。

[Azure ストレージのディザスター リカバリー計画とフェールオーバー - Azure Storage | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-disaster-recovery-guidance?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&bc=%2Fazure%2Fstorage%2Fblobs%2Fbreadcrumb%2Ftoc.json#plan-for-failover)

**リソース設定情報のバックアップ**の場合、ARM テンプレート、または Bicep テンプレート、Terrform テンプレートを活用するのが有効です。

こちらのテンプレートを管理し、使用してリソースの管理、あるいは更新をしていくことが IaC の考え方となります。

Azure のリソースの ARM テンプレート、 Bicep テンプレート、Terrform テンプレートのスキーマは以下の公式ドキュメントにあります。

[Azure resource reference - Bicep, ARM template & Terraform AzAPI reference | Microsoft Learn](https://learn.microsoft.com/en-us/azure/templates/)

既存リソースの設定情報をバックアップするには、先にリソースの 「テンプレートのエクスポート」 より ARM テンプレートを出力します。

※以前は ARM テンプレートの出力のみサポートされていましたが、最近 Bicep、Terrform テンプレートの出力もサポートされるようになりました

ただし、テンプレートのエクスポートはすべてのコンポーネントをエクスポートできなかったり、エクスポートされた情報に欠落がある場合があるため、テンプレートエクスポートする場合はチェックが必要となります。

上記のテンプレートスキーマを参照しながらチェックする必要があるため、リソースの機能などについて一定程度の理解が必要となります。

![image-6575f3df-6052-4a68-8fc2-9e90323e7f3b.png]({{site.baseurl}}/media/2025/04/image-6575f3df-6052-4a68-8fc2-9e90323e7f3b.png)

エクスポートされたテンプレートには、人によるチェックや修正が必要であることについて、公式ドキュメントにも以下のような注意書きがあります。

[Azure portal でテンプレートをエクスポートする - Azure Resource Manager | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/templates/export-template-portal)

![image-95125f45-ee74-484f-b298-fbc09f8078b1.png]({{site.baseurl}}/media/2025/04/image-95125f45-ee74-484f-b298-fbc09f8078b1.png)


ARM テンプレートとしてエクスポートした場合、以下の方法で手動で Bicep テンプレートに変換することも可能です。

ただし、Bicep への逆コンパイルはベストエフォートでのサポートとなり、逆コンパイルは情報が欠落したり、失敗したりする場合があるため、チェックが必要です。

[JSON Azure Resource Manager テンプレートを Bicep に逆コンパイルする - Azure Resource Manager | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/decompile?tabs=azure-cli#decompile-from-json-to-bicep)

![image-579ece23-2379-4702-abcf-183550894e50.png]({{site.baseurl}}/media/2025/04/image-579ece23-2379-4702-abcf-183550894e50.png)

単純なバックアップであれば、テンプレートのエクスポート、チェックを行えば、バックアップ完了となり、リソースのリストアは Azure PowerShell コマンドなどでテンプレートを使用してリソースをデプロイできます。

[New-AzResourceGroupDeployment (Az.Resources) | Microsoft Learn](https://learn.microsoft.com/ja-jp/powershell/module/az.resources/new-azresourcegroupdeployment?view=azps-0.10.0)

IaC としてリソースのデプロイなどを管理したい場合は、リソーステンプレートを以下の処理を行います。

1. テンプレートエクスポート
2. テンプレートチェック

ここまでは上記内容で説明しました

3. テンプレートのリファクタリング（モジュール化など）
4. テスト
5. デプロイ

![image-966141ff-85ce-4719-8422-e15e3c2daaf9.png]({{site.baseurl}}/media/2025/04/image-966141ff-85ce-4719-8422-e15e3c2daaf9.png)

[Bicep を使用するために Azure リソースと JSON ARM テンプレートを移行する - Azure Resource Manager | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/migrate)

### リファクタリング

テンプレートのリファクタリングは、テンプレートの変数などを読みやすく変更したり、コメントを追加してテンプレートの可読性を向上したりすることができます。

また、複数のリソースといった複雑なシステム構成の場合、個々のリソースをモジュール化することで、主テンプレートに集約することが可能で、管理を簡略化できます。

Bicep テンプレートはモジュール、ARM テンプレートはリンクされたテンプレートを使用します。

### テスト

更新したテンプレートでリソースをアップデートする場合、動作の検証が必要となってくるため、いきなり本番環境へのデプロイではなく、

あらかじめ用意されたテスト環境へデプロイし、動作の検証をすることを推奨します。

また、実際のデプロイを行わず、デプロイによる変更をプレビューすることも可能ですので、実際にデプロイする前に今回のデプロイで変更された箇所を記録することもいざとなった時に参考となれます。

[テンプレート デプロイの what-if - Azure Resource Manager | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/templates/deploy-what-if?tabs=azure-powershell)

### デプロイ

テンプレートの検証が終わりましたら、実際にデプロイを行います。

デプロイは、コマンドで実行するか、ポータルで実行できますが、ポータルではファイルのアップロードなどが必要となるため、

複数のテンプレートでリソースのデプロイを行う場合は、ローカルでのコマンド実行の方が迅速で、便利となります。


