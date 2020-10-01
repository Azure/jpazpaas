---
title: "Policy 作成のヒント集"
author_name: "Hirotaka Shionoiri"
tags:
    - Policy
---

# 質問
Azure Policy のポリシーを作成しようと思うのですが、type などをどう入力していいのか分かりません。<br>
また、どういうったパラメーターが条件式に使えるかもわかりません<br>
ポリシー作成時のコツなどあれば教えてください。

# 回答
今回はポリシー作成時の Tips について紹介します。

type や条件式に使えるプロパティの確認方法は下記のページに記載されています。<br>
[チュートリアル:カスタム ポリシー定義の作成 - リソースのプロパティを判別する](https://docs.microsoft.com/ja-jp/azure/governance/policy/tutorials/create-custom-policy-definition#determine-resource-properties)

筆者が良く利用する確認方法として下記の 3 つです。
- 公式ドキュメントによる ARM Templates の確認
- 利用しているリソースにおけるテンプレートのエクスポート
- CLI コマンドや PowerShell コマンドでエイリアスを見つける

また、動作確認時のコツとして、ポリシーによる評価をすぐに実行する CLI コマンド等についてもご紹介します。

## 公式ドキュメントによる ARM Templates の確認
公式ドキュメントより、各種リソースの ARM Templates を最初に確認します。<br>

例えば、ストレージアカウントの blob であれば下記のページで参照できます。<br>
https://docs.microsoft.com/ja-jp/azure/templates/microsoft.storage/storageaccounts/blobservices

例: "type": "Microsoft.Storage/storageAccounts/blobServices",

## 利用しているリソースにおける ARM Templates のエクスポート
Azure Portal より、各サービスのインスタンスの現在の状態を ARM Templates としてエクスポートできます。<br>
ここで出力される ARM Templates も基本的に公式ドキュメントに記載された ARM Templates と変わらないのですが、<br>
条件式に使いたいプロパティが実際どうなっているか（どう変化するか）などを確認するときにこちらを使っています。

Azure Portal より下記の部分からテンプレートのエクスポートができます。
![図1.jpg]({{site.baseurl}}/media/2020/10/2020-10-01-export-template.jpg)

## CLI コマンドや PowerShell コマンドでエイリアスを見つける
下記のページで紹介されている CLI コマンドや PowerShell コマンドでエイリアスを取得できます。<br>
CLI: https://docs.microsoft.com/ja-jp/azure/governance/policy/tutorials/create-custom-policy-definition#azure-cli 
<br>PowerShell: https://docs.microsoft.com/ja-jp/azure/governance/policy/tutorials/create-custom-policy-definition#azure-powershell

しかし、そのまま使うと、かなり行数を使われて流されてしまうので以下のように特定の項目をフィルタリングして出力しています。<br>
`(Get-AzPolicyAlias -NamespaceMatch 'Microsoft.Storage').Aliases.Name`<br>
ただ、ここまでの対応を行うことはほぼなく、公式ドキュメントかテンプレートのエクスポートで解決します。

## CLI コマンドによるコンプライアンス評価の実行
ポリシー定義を書き換えた後、変更後の定義による監査が有効になる前に動作確認などをされてしまい、<br>
ポリシーの検証結果がおかしかった、という経験があるかもしれません。<br>
下記のコマンドをご利用頂くと、ポリシーのスキャンを確実に実行してくれます。<br>
https://docs.microsoft.com/en-us/cli/azure/policy/state?view=azure-cli-latest#az-policy-state-trigger-scan

実行完了までに時間はかかりますが、コンプライアンス状況を見てみることで、実際のポリシー式の挙動の把握に役立ちます。<br>
<br>
なお、PowerShell 用の同様のコマンドも存在します。<br>
https://docs.microsoft.com/en-us/powershell/module/az.policyinsights/start-azpolicycompliancescan?view=azps-4.7.0

# まとめ
以上がポリシー作成時の Tips となります。<br>
ポリシー作成の流れとしては、筆者は下記の様に行っています。
1. type を公式ドキュメントのテンプレートで確認する
1. 条件式に利用する type や値を公式ドキュメントのテンプレートやテンプレートのエクスポートで確認する
1. 定義を保存し、割り当てが有効になっていることを確認したら、`az policy state trigger-scan` で確実に監査を有効にする
1. コンプライアンス結果を見たり、新規リソースを作ろうとして、意図したようにポリシーが働いているかを確認する 

<br>
<br>

---

<br>
<br>

2020 年 10 月 1 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>