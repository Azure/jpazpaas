---
title: "Azure Export for Terraform で Azure リソースの管理に Terraform を導入する"
author_name: "yiwenchen"
tags:
    - ARM
---

こんにちは、Azure PaaS Developer サポート担当の陳です。

Terraform を使用してリソースを管理する際の操作方法や注意点などの質問を受けましたので、Terraform で Azure リソースを管理する方法をまとめました。 

# 質問
Terraform で Azure リソースを管理したいのですが、既に Azure リソースが作成されています。既存の Azure リソースを Terraform で管理するにはどのようにしたらいいでしょうか。

# 回答
既存の Azure リソースを Terraform で管理するには 公式ツールの Azure Export for Terraform を使用できます。

#### Azure Export for Terraform のインストール
[インストール](https://learn.microsoft.com/ja-jp/azure/developer/terraform/azure-export-for-terraform/export-terraform-overview#installation) のドキュメントをご確認ください。

#### PowerShell でエクスポート
エクスポートしたいサブスクリプションにログインしてから、下記のコマンドでリソースグループやリソースをエクスポートできます。

```powershell
# リソースグループを指定してエクスポート
aztfexport resource-group myResourceGroup

# リソースを個別に指定してエクスポート
aztfexport resource <resourceId>
```

エクスポートが終わりましたら、以下のファイルが作成されます。

![terraformFile_vscodeView-ccabac61-03b4-490f-8cbe-354105ff607f.png]({{site.baseurl}}/media/2024/04/terraformFile_vscodeView-ccabac61-03b4-490f-8cbe-354105ff607f.png)

`main.tf` には各リソースの詳細情報を記録しています。
`provider.tf` では各リソースをどのサブスクリプションに紐づけるかをサブスクリプションの ID と認証情報を追加できます。
`terraform.tf` には Terraform 自身の情報などが入っています

リソースを Terraform にインポートしたら、次は情報を編集し、ほかのサブスクリプションに導入する部分となります。

#### リソース情報の編集

リソース情報の編集は、全部 `main.tf` で行えます。ここで、下の画像を例とします。
リソースグループ名を変更したい場合は、一番上の `"azurerm_resource_group"` ブロックで `name` を編集します。

ただし、リソースグループ内にほかのリソースも入っているので、その依存関係も考慮しないといけません。

ここでの例だと、`"azurern_storage_account"` ブロック内の `"resource_group_name"` も変更しないと導入時に `"resourceGroupNotFound"` のエラーが出てしまいます。

そのため、リソース情報を編集する際には相互の依存関係にご注意ください。

![main_tf-4c89ac05-dfb7-4361-ad06-6e2a5a38c0f8.png]({{site.baseurl}}/media/2024/04/main_tf-4c89ac05-dfb7-4361-ad06-6e2a5a38c0f8.png)

- **依存関係を明示する方法**<a id="dependency"></a>

以下の画像のように、ほかのリソースと関連する情報を直接文字列で記載するではなく、ほかのリソースの変数の形で記載すれば Terraform が理解してくれます。

※ 依存関係を解決しないと、新しいサブスクリプションにデプロイする際のリソースの前後順番は考慮されず、パラレルでデプロイされるため、デプロイエラーは発生してしまいます。

![image-7ecc26e1-3aef-4b8c-86c4-5d849bc90248.png]({{site.baseurl}}/media/2024/04/image-7ecc26e1-3aef-4b8c-86c4-5d849bc90248.png)

#### 導入サブスクリプションの認証設定

公式ドキュメントでは、主に Microsoft アカウントを使用する認証とサービスプリンシパルを使用する認証方法を紹介しています。

ここでは、サービスプリンシパルを使用する認証方法を解説しますが、参考ドキュメントにもリンクを添付します。

1. 導入サブスクリプションでサービスプリンシパルを作成し、必要な権限を付与する（ドキュメントでは共同作成者を付与しています）
2. 以下のように、必要な認証情報を `provider.tf` の azurerm ブロックに追加します。

```
provider "azurerm" {
  features {}
  // alias = "app-sub"
  subscription_id   = "<azure_subscription_id>"
  tenant_id         = "<azure_subscription_tenant_id>"
  client_id         = "<service_principal_appid>"
  client_secret     = "<service_principal_password>"
}

```

ここでの `alias` の部分はマルチサブスクリプションに一気に導入する際の識別子です。

２つのサブスクリプションにリソースをそれぞれ導入したい場合は、２つの azurerm ブロックに異なる `alias` とサブスクリプションの認証情報を追加し、さらに `main.tf` のリソース情報に下記のように `provider` を指定します。

![main_provider-722e5119-5522-40d2-af49-1663a9c7249c.png]({{site.baseurl}}/media/2024/04/main_provider-722e5119-5522-40d2-af49-1663a9c7249c.png)

#### 他のサブスクリプションに Terraform を導入します

他のサブスクリプションを Terraform で管理する場合、新しいフォルダで `main.tf`、`provider.tf`、`terraform.tf` を作成します。
理由としては、エクスポート時に生成された `.tfstate` ファイルはエクスポート時のリソース状態を記録しているので、新しいサブスクリプションの空の状態から導入する際に、`.tfstate` で記録されているリソース状態と現在の状態とコンフリクトしてしまうためです。

さらに前述した [**依存関係**](#dependency) を導入時に注意します。

エクスポート時に生成された `main.tf` ファイルを見ればわかりますが、すべての name などは直接文字列で記載されております。

そのままで導入してしまうと、Terraform はこれらのリソースをどういう順番で作成したらいいかわからないため、並列処理で作成しようとします。その場合は、例にあるようなリソースグループとリソースグループ内のリソースの依存関係（先にリソースグループを作成しないと、リソースを作成できません）を考慮されず、途中で `resourceGroupNotFound` エラーになってしまいます。

リソースグループとリソースの間の依存関係だけでなく、ストレージアカウントとコンテナなども依存関係を考慮しないといけないので、導入するリソースによって依存関係に注意しましょう。詳細は上部の[依存関係](#dependency)の部分をご参照ください。


依存関係について修正したら、新しいフォルダで以下のコマンドでサブスクリプションに導入します。

```powershell
terraform init

# 作成、変更されるリソースを確認します。
terraform plan

# 実際に作成、変更処理を開始します。
terrform apply
```

# 参考ドキュメント
- [Azure に対して Terraform を認証する](https://learn.microsoft.com/ja-jp/azure/developer/terraform/get-started-cloud-shell-powershell?tabs=bash#5-authenticate-terraform-to-azure)
- Terraform 公式の Azure ドキュメント [Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)

<br>
<br>

---

<br>
<br>

2024 年 04 月 09 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>