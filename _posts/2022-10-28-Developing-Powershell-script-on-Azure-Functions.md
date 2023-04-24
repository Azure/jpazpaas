---
title: "任意のPowerShellモジュールを利用したAzure Functions の環境構築ガイド"
author_name: "Hayato Kuroda"
tags:
    - Function Apps
    - PowerShell
---

# 質問
PowerShellモジュールを利用して Azure Functions から Azure 上のリソースへアクセスを行いたい。

# 回答
以下に Azure Functions から特定のリソース グループの LogicApps リソース一覧を参照する場合の環境構築手順例を示します。
## 構築手順
1. 任意のプランで Azure Functions と LogicApps を作成します。<br>
今回は Azure Functions の従量課金プランと Logic Apps の消費プランを作成します。

2. 任意の関数アプリを作成します。<br>
今回は HTTP トリガーを作成します。<br>

![image-f732f4ae-a83e-4db7-9317-60dcb0db60bc.png]({{site.baseurl}}/media/2022/10/image-f732f4ae-a83e-4db7-9317-60dcb0db60bc.png)

3. PowerShell にて利用する依存モジュールを宣言します。<br>
Azure Functions の PowerShell では、以下にご案内がございますように requirements.psd1 にて依存モジュールを宣言するかモジュール群をダウンロードし既定のフォルダに配置することでアプリケーション コードで利用することができます。<br>

■ [Azure Functions の PowerShell 開発者向けガイド - 依存関係管理](
https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-reference-powershell?tabs=portal#dependency-management)

※使用したいモジュールに必要なパッケージは [PowerShell Gallery](https://www.powershellgallery.com/) から検索いただけます。

Get-AzLogicApp コマンドを利用したいため Az.LogicApp を requirements.psd1 に宣言します。PowerShell の場合には Azure ポータルで編集できるため簡単のために今回はアプリ ファイル ブレードから編集しました。

![image-52ed6e2a-577c-47b7-a386-7ec5eac931ac.png]({{site.baseurl}}/media/2022/10/image-52ed6e2a-577c-47b7-a386-7ec5eac931ac.png)

requirements.psd1 に宣言されたモジュール群は \home\data\ManagedDependencies 配下にアーカイブされて保存されます。

![image-0452a7a9-ea69-4d87-8f91-b1d2cba95a31.png]({{site.baseurl}}/media/2022/10/image-0452a7a9-ea69-4d87-8f91-b1d2cba95a31.png)

4. アプリケーション コードを編集します。
Get-AzLogicApp を利用する簡易なコードを記載します。特定のリソース グループ配下にある LogicApps の一覧を取得し、HTTP の応答として返却します。

```
using namespace System.Net

# Input bindings are passed in via param block.
param($Request, $TriggerMetadata)

$body = Get-AzLogicApp -ResourceGroupName hakurodablogpost

# Associate values to output bindings by calling 'Push-OutputBinding'.
Push-OutputBinding -Name Response -Value ([HttpResponseContext]@{
    StatusCode = [HttpStatusCode]::OK
    Body = $body
})
```

5. アプリケーション コードがリソース グループへアクセスするための権限を付与します。<br>
今回はマネージド ID を利用します。ID ブレードからシステム割り当て済みマネージド ID をオンに変更し、保存します。

![image-a765051f-dd18-480b-8773-27f64de43e20.png]({{site.baseurl}}/media/2022/10/image-a765051f-dd18-480b-8773-27f64de43e20.png)

次に、有効化したマネージド ID にリソース グループへのアクセス権限を付与します。リソース グループの「アクセス制御(IAM)」から「ロールの割り当ての追加」から、下図の操作例に従って権限を追加します。
![image-7cdfc8d1-9a77-423e-a1a5-ab2b67c1780d.png]({{site.baseurl}}/media/2022/10/image-7cdfc8d1-9a77-423e-a1a5-ab2b67c1780d.png)

6. アプリケーションを実行します。<br>
リソース グループに含まれる LogicApps の一覧を応答データとして取得することができました。

![image-6aa48eeb-c7a4-4918-b791-4181b4fd057a.png]({{site.baseurl}}/media/2022/10/image-6aa48eeb-c7a4-4918-b791-4181b4fd057a.png)

## 発生するエラー例
今回の構成において発生し得るエラーと回避策をまとめます。手元環境の構築時に検出したエラーとなっており、すべての場合を網羅しているわけではないのでご了承ください。


### A. 依存モジュールが不足している場合
項番3の依存モジュール宣言が不足しているや設定誤りしている場合には、アプリケーション実行時に以下のようなモジュールが見つからない旨エラーが発生します。

```
YYYY-MM-DDTHH:MI:SSZ   [Error]   ERROR: The term 'Get-AzLogicApp' is not recognized as a name of a cmdlet, function, script file, or executable program.
Check the spelling of the name, or if a path was included, verify that the path is correct and try again.

Exception             : 
    Type        : System.Management.Automation.CommandNotFoundException
    ErrorRecord : 
        Exception             : 
            Type    : System.Management.Automation.ParentContainsErrorRecordException
            Message : The term 'Get-AzLogicApp' is not recognized as a name of a cmdlet, function, script file, or executable program.
                      Check the spelling of the name, or if a path was included, verify that the path is correct and try again.
            HResult : -2146233087
        TargetObject          : Get-AzLogicApp
        CategoryInfo          : ObjectNotFound: (Get-AzLogicApp:String) [], ParentContainsErrorRecordException
        FullyQualifiedErrorId : CommandNotFoundException
        InvocationInfo        : 
            ScriptLineNumber : 6
            OffsetInLine     : 9
            HistoryId        : 1
            ScriptName       : C:\home\site\wwwroot\HttpTrigger1\run.ps1
            Line             : $body = Get-AzLogicApp -ResourceGroupName hakurodablogpost
                               
            PositionMessage  : At C:\home\site\wwwroot\HttpTrigger1\run.ps1:6 char:9
                               + $body = Get-AzLogicApp -ResourceGroupName hakurodablogpost
                               +         ~~~~~~~~~~~~~~
            PSScriptRoot     : C:\home\site\wwwroot\HttpTrigger1
            PSCommandPath    : C:\home\site\wwwroot\HttpTrigger1\run.ps1
            InvocationName   : Get-AzLogicApp
            CommandOrigin    : Internal
        ScriptStackTrace      : at <ScriptBlock>, C:\home\site\wwwroot\HttpTrigger1\run.ps1: line 6
    CommandName : Get-AzLogicApp
    TargetSite  : 
        Name          : LookupCommandInfo
        DeclaringType : System.Management.Automation.CommandDiscovery, System.Management.Automation, Version=7.2.6.500, Culture=neutral, PublicKeyToken=31bf3856ad364e35
        MemberType    : Method
        Module        : System.Management.Automation.dll
    Message     : The term 'Get-AzLogicApp' is not recognized as a name of a cmdlet, function, script file, or executable program.
                  Check the spelling of the name, or if a path was included, verify that the path is correct and try again.
    Data        : System.Collections.ListDictionaryInternal
    Source      : System.Management.Automation
    HResult     : -2146233087
    StackTrace  : 
   at System.Management.Automation.CommandDiscovery.LookupCommandInfo(String commandName, CommandTypes commandTypes, SearchResolutionOptions searchResolutionOptions, CommandOrigin commandOrigin, ExecutionContext context)
   at System.Management.Automation.CommandDiscovery.LookupCommandInfo(String commandName, CommandOrigin commandOrigin, ExecutionContext context)
   at System.Management.Automation.CommandDiscovery.LookupCommandInfo(String commandName, CommandOrigin commandOrigin)
   at System.Management.Automation.CommandDiscovery.LookupCommandProcessor(String commandName, CommandOrigin commandOrigin, Nullable`1 useLocalScope)
   at System.Management.Automation.ExecutionContext.CreateCommand(String command, Boolean dotSource)
   at System.Management.Automation.PipelineOps.AddCommand(PipelineProcessor pipe, CommandParameterInternal[] commandElements, CommandBaseAst commandBaseAst, CommandRedirection[] redirections, ExecutionContext context)
   at System.Management.Automation.PipelineOps.InvokePipeline(Object input, Boolean ignoreInput, CommandParameterInternal[][] pipeElements, CommandBaseAst[] pipeElementAsts, CommandRedirection[][] commandRedirections, FunctionContext funcContext)
   at System.Management.Automation.Interpreter.ActionCallInstruction`6.Run(InterpretedFrame frame)
   at System.Management.Automation.Interpreter.EnterTryCatchFinallyInstruction.Run(InterpretedFrame frame)
   at System.Management.Automation.Interpreter.EnterTryCatchFinallyInstruction.Run(InterpretedFrame frame)
TargetObject          : Get-AzLogicApp
CategoryInfo          : ObjectNotFound: (Get-AzLogicApp:String) [], CommandNotFoundException
FullyQualifiedErrorId : CommandNotFoundException
InvocationInfo        : 
    ScriptLineNumber : 6
    OffsetInLine     : 9
    HistoryId        : 1
    ScriptName       : C:\home\site\wwwroot\HttpTrigger1\run.ps1
    Line             : $body = Get-AzLogicApp -ResourceGroupName hakurodablogpost
                       
    PositionMessage  : At C:\home\site\wwwroot\HttpTrigger1\run.ps1:6 char:9
                       + $body = Get-AzLogicApp -ResourceGroupName hakurodablogpost
                       +         ~~~~~~~~~~~~~~
    PSScriptRoot     : C:\home\site\wwwroot\HttpTrigger1
    PSCommandPath    : C:\home\site\wwwroot\HttpTrigger1\run.ps1
    InvocationName   : Get-AzLogicApp
    CommandOrigin    : Internal
ScriptStackTrace      : at <ScriptBlock>, C:\home\site\wwwroot\HttpTrigger1\run.ps1: line 6
```

#### [回避策A]
・requirements.psd1 を確認します。また requirements.psd1 が誤っていない場合には、\home\data\ManagedDependecies に配置された依存モジュールのアーカイブ ファイルに利用予定のモジュールが含まれているか確認します。

・手動で依存モジュールを宣言できるため、Save-Module で依存モジュールをダウンロードしていただき \home\site\wwwroot\Modules に配置ください。

手動で依存モジュールを追加する方法を解説します。

今回の例では、Save-Module コマンドをローカル マシン上の任意のディレクトリで実行します。

> コマンド例: Save-Module -Name Az.LogicApp -Path ./

Az.LogicApps 及びその依存モジュールがダウンロードされるため、ダウンロードされたファイル一式を \home\site\wwwroot\Modules に配置します。Kudu をご利用の場合には、ドラッグ＆ドロップでアップロードいただけます。最終的には以下のようなフォルダ構成となります。Windows OS での Kudu 表示画面例となります。

![image-2e2fe095-8864-4c69-8c60-d7939d2b35ce.png]({{site.baseurl}}/media/2022/10/image-2e2fe095-8864-4c69-8c60-d7939d2b35ce.png)

### B. 依存モジュールはあるが Get-AzLogicApp コマンドのリソース グループへのアクセス設定が全くない場合
項番5のアクセス制御を実施していない場合には Get-AzLogicApp 実行時にリソース グループへのアクセス権限がない旨エラーが発生します。

```
YYYY-MM-DDTHH:MI:SSZ   [Error]   ERROR: No subscription found in the context.  Please ensure that the credentials you provided are authorized to access an Azure subscription, then run Connect-AzAccount to login.

Exception             : 
    Type            : Microsoft.Azure.Commands.Common.Exceptions.AzPSApplicationException
    ErrorKind       : User
    ErrorLineNumber : 61
    ErrorFileName   : ClientFactory
    TargetSite      : 
        Name          : CreateArmClient
        DeclaringType : Microsoft.Azure.Commands.Common.Authentication.Factories.ClientFactory
        MemberType    : Method
        Module        : Microsoft.Azure.PowerShell.Authentication.dll
    Message         : No subscription found in the context.  Please ensure that the credentials you provided are authorized to access an Azure subscription, then run Connect-AzAccount to login.
    Data            : System.Collections.ListDictionaryInternal
    Source          : Microsoft.Azure.PowerShell.Authentication
    HResult         : -2146232832
    StackTrace      : 
   at Microsoft.Azure.Commands.Common.Authentication.Factories.ClientFactory.CreateArmClient[TClient](IAzureContext context, String endpoint)
   at Microsoft.Azure.Commands.LogicApp.Utilities.LogicAppClient..ctor(IAzureContext context)
   at Microsoft.Azure.Commands.LogicApp.Utilities.LogicAppBaseCmdlet.get_LogicAppClient()
   at Microsoft.Azure.Commands.LogicApp.Cmdlets.GetAzureLogicAppCommand.ExecuteCmdlet()
   at Microsoft.WindowsAzure.Commands.Utilities.Common.CmdletExtensions.<>c__3`1.<ExecuteSynchronouslyOrAsJob>b__3_0(T c)
   at Microsoft.WindowsAzure.Commands.Utilities.Common.CmdletExtensions.ExecuteSynchronouslyOrAsJob[T](T cmdlet, Action`1 executor)
   at Microsoft.WindowsAzure.Commands.Utilities.Common.CmdletExtensions.ExecuteSynchronouslyOrAsJob[T](T cmdlet)
   at Microsoft.WindowsAzure.Commands.Utilities.Common.AzurePSCmdlet.ProcessRecord()
CategoryInfo          : CloseError: (:) [Get-AzLogicApp], AzPSApplicationException
FullyQualifiedErrorId : Microsoft.Azure.Commands.LogicApp.Cmdlets.GetAzureLogicAppCommand
InvocationInfo        : 
    MyCommand        : Get-AzLogicApp
    ScriptLineNumber : 6
    OffsetInLine     : 9
    HistoryId        : 1
    ScriptName       : C:\home\site\wwwroot\HttpTrigger1\run.ps1
    Line             : $body = Get-AzLogicApp -ResourceGroupName hakurodablogpost
                       
    PositionMessage  : At C:\home\site\wwwroot\HttpTrigger1\run.ps1:6 char:9
                       + $body = Get-AzLogicApp -ResourceGroupName hakurodablogpost
                       +         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    PSScriptRoot     : C:\home\site\wwwroot\HttpTrigger1
    PSCommandPath    : C:\home\site\wwwroot\HttpTrigger1\run.ps1
    InvocationName   : Get-AzLogicApp
    CommandOrigin    : Internal
ScriptStackTrace      : at <ScriptBlock>, C:\home\site\wwwroot\HttpTrigger1\run.ps1: line 6
PipelineIterationInfo : 
```

#### [回避策B]
手順5のアクセス権限の付与手順を実施します。

### C. 依存モジュールはあり Get-AzLogicApp コマンドのリソース グループへのアクセス権限がない場合
A(依存モジュールの宣言不足) 及び B(アクセス権限の付与設定漏れ) を順番に解消後、Azure Functions のマネージド ID を有効化したにもかかわらずリソース グループへのアクセス権を付与できていない場合にはヌル オブジェクトに対する操作である旨エラーが発生します。

```
YYYY-MM-DDTHH:MI:SSZ   [Error]   ERROR: Object reference not set to an instance of an object.

Exception             : 
    Type       : System.NullReferenceException
    TargetSite : Void .ctor(Microsoft.Azure.Commands.Common.Authentication.Abstractions.IAzureContext)
    Message    : Object reference not set to an instance of an object.
    Source     : Microsoft.Azure.PowerShell.Cmdlets.LogicApp
    HResult    : -2147467261
    StackTrace : 
   at Microsoft.Azure.Commands.LogicApp.Utilities.LogicAppClient..ctor(IAzureContext context)
   at Microsoft.Azure.Commands.LogicApp.Utilities.LogicAppBaseCmdlet.get_LogicAppClient()
   at Microsoft.Azure.Commands.LogicApp.Cmdlets.GetAzureLogicAppCommand.ExecuteCmdlet()
   at Microsoft.WindowsAzure.Commands.Utilities.Common.CmdletExtensions.<>c__3`1.<ExecuteSynchronouslyOrAsJob>b__3_0(T c)
   at Microsoft.WindowsAzure.Commands.Utilities.Common.CmdletExtensions.ExecuteSynchronouslyOrAsJob[T](T cmdlet, Action`1 executor)
   at Microsoft.WindowsAzure.Commands.Utilities.Common.CmdletExtensions.ExecuteSynchronouslyOrAsJob[T](T cmdlet)
   at Microsoft.WindowsAzure.Commands.Utilities.Common.AzurePSCmdlet.ProcessRecord()
CategoryInfo          : CloseError: (:) [Get-AzLogicApp], NullReferenceException
FullyQualifiedErrorId : Microsoft.Azure.Commands.LogicApp.Cmdlets.GetAzureLogicAppCommand
InvocationInfo        : 
    MyCommand        : Get-AzLogicApp
    ScriptLineNumber : 6
    OffsetInLine     : 9
    HistoryId        : 1
    ScriptName       : C:\home\site\wwwroot\HttpTrigger1\run.ps1
    Line             : $body = Get-AzLogicApp -ResourceGroupName hakurodablogpost
                       
    PositionMessage  : At C:\home\site\wwwroot\HttpTrigger1\run.ps1:6 char:9
                       + $body = Get-AzLogicApp -ResourceGroupName hakurodablogpost
                       +         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    PSScriptRoot     : C:\home\site\wwwroot\HttpTrigger1
    PSCommandPath    : C:\home\site\wwwroot\HttpTrigger1\run.ps1
    InvocationName   : Get-AzLogicApp
    CommandOrigin    : Internal
ScriptStackTrace      : at <ScriptBlock>, C:\home\site\wwwroot\HttpTrigger1\run.ps1: line 6
PipelineIterationInfo : 
```

#### [回避策C]
設定が手順5のアクセス権限の付与手順において有効化したマネージド ID にリソース グループへのアクセス権限を付与を実施します。

以上で任意の PowerShell モジュールを利用して Azure Functions から Azure 上のリソースへアクセスを行えました。他の PowerShell モジュールを利用の場合にも、同様の手順で構成可能ですので是非お試しください。

---

<br>
<br>

2023 年 04 月 24 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>