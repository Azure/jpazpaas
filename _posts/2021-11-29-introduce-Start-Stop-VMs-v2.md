---
title: "Start/Stop VMs v2 のご紹介"
author_name: "Kohei Hosono"
tags:
    - Function App
---

[Start/Stop VMs v2](https://docs.microsoft.com/ja-jp/azure/azure-functions/start-stop-vms/overview) は、複数のサブスクリプションにわたって Azure 仮想マシン (VM) を開始または停止できる、複数の Azure サービスからなるソリューションです。

本記事では、 Start/Stop VMs v2 を作成し、スケジュールに従って VM を起動または停止する方法をご紹介します。

なお、本記事においては、 Start/Stop VMs v2 の基本的な使い方をご紹介し、すべての機能をご紹介するわけではございませんので予めご了承ください。また、現在 Start/Stop VMs v2 はプレビュー段階の機能であり、通常のサービスレベル契約は適用されませんので、ご注意ください。

Start/Stop VMs v2 は、以下の Azure サービスで構成されます。

| Azure リソース        |                                           機能                                                       |
| :------------------: | :--------------------------------------------------------------------------------------------------- |
|     Function App     | スケジュールや使用状況に応じて、 VM の起動と停止を行います。                                         |
|      Logic App       | スケジュールの設定と VM に作用するペイロードを設定します。                                           |
| Application Insights | 実行された操作のテレメトリをログに記録します。                                                       |
|      Dashboard       | Application Insights で収集したログをビューとしてカスタムできます。                                  |
|       Storage        | 実行されたトランザクションをテーブルに記録、キューを使用してオペレーションオブジェクトを保存します。 |
|       Monitor        | VM の状態が変化した際にアラートやメール通知を行います。                                              |

Start/Stop VMs v2 は、 Azure Automation で使用できる以前のバージョンと同じ機能をすべて提供しますが、より新しい Azure のテクノロジを活用するように設計されています。

# Start/Stop VMs v2 を作成し、スケジュールに従って VM を起動/停止する方法

本記事で紹介する環境は、検証のために作成したものとなりますので、活用の際にはお客様の要件に合わせて十分なテスト・検証の上ご利用いただきますようお願い申し上げます。

## 1. Start/Stop VMs v2 を作成する

Azure ポータルの [リソースの作成] を利用して、 Function App や Logic App を含む、複数の Azure サービスからなるリソースグループを新規作成します。

1-1. Azure ポータルを開き、 [リソースの作成] をクリックします。

![リソースの作成]({{site.baseurl}}/media/2021/11/2021-11-29-1_1_create_resource-0761ce8b-3d23-4240-b7b6-12012ac3a754.png)

1-2. 検索ボックスに [Start/Stop] と入力し、 Start/Stop VMs during off hours - V2 を選択、作成します。

![リソースを検索]({{site.baseurl}}/media/2021/11/2021-11-29-1_2_search_resource-1d4afd89-0bdc-4188-9360-d84fad17b277.png)

![Start/Stop VMs v2 を作成]({{site.baseurl}}/media/2021/11/2021-11-29-1_3_create_resource_ssvmv2-72b93f70-4c10-44ba-bf11-060c6ecde9c9.png)

1-3. 各種リソース名等を入力し、 [作成] をクリックしてリソース群を作成します。

![リソース名等を入力]({{site.baseurl}}/media/2021/11/2021-11-29-1_4_type_resource_info-c91141f1-7ae3-4b15-8f77-ed8d14e77184.png)

![確認及び作成]({{site.baseurl}}/media/2021/11/2021-11-29-1_5_create_ssvmv2-c48fad1b-b205-403a-8b5d-176136f027ba.png)

1-4. [サブスクリプションに移動] を選択すると、作成されたリソースを確認できます。

![リソースの表示]({{site.baseurl}}/media/2021/11/2021-11-29-1_6_list_resourcegroup-55ade3e8-73c3-447e-a26b-691b58ffe222.png)

## 2. サブスクリプションに対してロールを割り当てる

Start/Stop VMs v2 では、作成された Function App が VM の起動/停止を行うため、該当 Function App に対して、VM が属するサブスクリプションにロールを割り当てる必要があります。

2-1. VM が属するサブスクリプションに移動します。

![サブスクリプションを選択]({{site.baseurl}}/media/2021/11/2021-11-29-2_1_list_subscription-bab55d2c-ceec-45e8-b305-87802b5bdf04.png)

2-2. [アクセス制御(IAM)] に移動し、 [追加] から [ロールの割り当ての追加] をクリックします。

![追加を選択]({{site.baseurl}}/media/2021/11/2021-11-29-2_2_subscription_add-73475e26-0b70-494c-9686-d72c9b9a0189.png)

![ロールの割り当ての追加]({{site.baseurl}}/media/2021/11/2021-11-29-2_3_subscription_add_role-1fca1995-13d3-4481-9945-a25ae1fe838f.png)

2-3. [共同作成者] を選択し、 [次へ] をクリックします。

![共同作成者を選択]({{site.baseurl}}/media/2021/11/2021-11-29-2_4_select_cocreator-f5e09458-15fc-4376-9886-4229a9483691.png)

2-4. アクセスの割当先として [マネージド ID] を選択し、 [メンバーを追加する] から [サブスクリプション] を選択、マネージド ID は [関数アプリ] を選択し、 1-4 で作成された Fucntion App を選択します。

![Function App を選択]({{site.baseurl}}/media/2021/11/2021-11-29-2_5_add_role_function-1bcfdab2-3efd-4165-8d4b-129c85facdfe.png)

2-5. [レビューと割り当て] を選択し、 Function App にロールを割り当てます。

![レビューと割り当てを選択]({{site.baseurl}}/media/2021/11/2021-11-29-2_6_confirm_add_role-a93cc575-08ee-4457-bc17-40a1308ab13e.png)

## 3. スケジュールを構成する

VM の起動/停止を制御するには、要件に基づいて、 1-4 で作成された Logic App を 1 つ以上構成します。 Start/Stop VMs v2 では、 VM を起動/停止するトリガーが 3 種類あります。

| トリガー種類 |                                 機能                                                                                                                                    |
| :-------: | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Scheduled | 指定したスケジュールに基づいて、 VM を起動/停止します。 <br/>Azure Resource Manager とクラシック VM の両方に対して実行できます。                                          |
| Sequenced | 事前に定義されたシーケンス処理タグの付いた VM に対して順番に起動/停止を行います。<br/>Azure Resource Manager の VM のみがサポートされ、クラシック VM では実行できません。 |
| AutoStop  | CPU 使用率に基づき、 VM の停止を行います。<br/>Azure Resource Manager とクラシック VM の両方に対して実行できます。                                                        |

ここでは、スケジュールに基づいて VM を起動する方法をご紹介します。

3-1. 作成された Logic App の一覧から [ststv2_vms_Scheduled_start] を選択し、 [ロジック アプリ コードビュー] に移動します。 `body` オブジェクトで起動/停止したいターゲットを設定します。

ここでは、 `rgvm` というリソースグループに属する Azure Resource Manager VM すべてを起動するために、デフォルトで `ResourceGroups` に記述されている `"/subscriptions/11111111-2222-3333-4444-555555555555/resourceGroups/rg1/"` を削除し、`"/subscriptions/12345678-1234-1234-1234-123456789012(該当サブスクリプションIDに変更してください)/resourceGroups/rgvm/"` を追加しています。コードの編集が完了したら、 [保存] をクリックします。

![コードの編集]({{site.baseurl}}/media/2021/11/2021-11-29-3_1_edit_logicapp_code-c5651a5e-4617-450e-b1d5-ea4898eec04e.png)

起動/停止する VM のスコープとして、サブスクリプション、リソースグループ、VM の3種類が用意されています。 `RequestScopes` を以下のフォーマットに従って記述することで、複数の VM を指定できます。

|       Key       | Value (例)                                                                                                             |
| :-------------: | :--------------------------------------------------------------------------------------------------------------------- |
|  Subscriptions  | /subscriptions/12345678-1234-5678-1234-123456781234/                                                                   |
| ResourceGroups  | /subscriptions/12345678-1234-5678-1234-123456781234/resourceGroups/rg1/                                                |
|     VMLists     | /subscriptions/12345678-1234-5678-1234-123456781234/resourceGroups/rg1/providers/Microsoft.Compute/virtualMachines/vm1 |
| ExcludedVMLists | /subscriptions/12345678-1111-2222-3333-123456781234/resourceGroups/rg1/providers/Microsoft.Compute/virtualMachines/vm1 |

`ExcludedVMLists` で指定した仮想マシンを対象から除外することができます。例えば、リソースグループ `rg1` の VM `vm1` 以外の VM すべてを起動したい場合は以下のように記述します。 `ExcludedVMLists` は、 `VMLists` とは併用できませんのでご注意ください。

```json
{
    "body": {
        "Action": "start",
        "RequestScopes": [
            "/subscriptions/12345678-1234-5678-1234-123456781234/resourceGroups/rg1/"
        ],
        "ExcludedVMLists": [
            "/subscriptions/12345678-1111-2222-3333-123456781234/resourceGroups/rg1/providers/Microsoft.Compute/virtualMachines/vm1"
        ]
    }
}
```

3-2. [ロジック アプリ デザイナー] に移動し、 [Recurrence] をクリックし、スケジュールを設定します。ここでは、毎時 0 分と 30 分に スケジュールされるよう設定します。

繰り返しオプションについては、以下のドキュメントで詳細に記載されておりますので、ご参照ください。

[Azure Logic Apps で繰り返しトリガーを使用して繰り返しタスクおよびワークフローを作成、スケジュール設定、および実行する Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/connectors/connectors-native-recurrence#add-the-recurrence-trigger)

![スケジュールの構成]({{site.baseurl}}/media/2021/11/2021-11-29-3_2_edit_logicapp_schedule-fa51447e-7cd6-41bc-8ade-45a7a3807ee5.png)

3-3. [概要] に移動し、 [有効] をクリックしてスケジュールを有効化します。

![Logic App を有効化]({{site.baseurl}}/media/2021/11/2021-11-29-3_3_apply_logicapp-84b85e9d-5dbf-403e-95a8-52975d77c263.png)

3-4. 以上で、 Start/Stop VMs V2 を作成し、スケジュールに基づいて VM を起動/停止することができました。 [最新の情報に更新] をクリックし、 [実行の履歴] から実行履歴を確認できます。また、該当 VM の [アクティビティ ログ] に移動すると、 "Start Virtual Machine" の状態が成功になっていることから、 VM の起動が成功していることが確認できます。

![実行の履歴を確認]({{site.baseurl}}/media/2021/11/2021-11-29-3_4_list_ssvm_logs-2a37d117-02b3-4d0e-aa8e-dea5f7ecf30e.png)

![VM のアクティビティログを確認]({{site.baseurl}}/media/2021/11/2021-11-29-3_5_list_vm_activity_logs-051dcbb2-f4ce-477a-adaf-08e89176444c.png)

# 参考ドキュメント

- [Start/Stop VMs v2 (プレビュー) の概要 Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/azure-functions/start-stop-vms/overview) 
- [Start/Stop VMs v2 (プレビュー) のデプロイ Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/azure-functions/start-stop-vms/deploy)
- [Start/Stop VMs v2 (プレビュー) を管理する方法 Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/azure-functions/start-stop-vms/manage)
- [VM の開始/停止 (プレビュー) に関する一般的な問題のトラブルシューティング Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/azure-functions/start-stop-vms/troubleshoot)
- [Azure portal でロジック アプリを管理する Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/logic-apps/manage-logic-apps-with-azure-portal)

<br>
<br>

---

<br>
<br>

2021 年 11 月 29 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
