---
title: "Azure App Service Triggered WebJob がデプロイ直後に即実行される場合がある — 10 分間のグレース期間について"
author_name: "Takeharu Oshida"
tags:
    - App Service
    - Web Apps
    - WebJobs
date: 2026-04-01 00:00:00
---

# はじめに
お世話になっております。Azure App Service サポート担当の押田です。
本記事は、Azure App Service の **Triggered WebJob** において、デプロイ直後に WebJob が意図せず即実行されるケースについて解説します。

Triggered WebJob は Cron 式でスケジュールを設定できますが、削除方法とデプロイのタイミングによっては、次のスケジュール時刻を待たずに直後に実行されることがあります。これは **Kudu のスケジューラに実装されている「10 分間のグレース期間（grace period）」** に起因する設計上の動作です。

公式ドキュメントにはこのグレース期間に関する詳細な記載がないため、本記事ではソースコードの実装を根拠として動作を説明し、回避策を紹介します。


# 背景
## Triggered WebJob のスケジュール実行の仕組み

Azure App Service の Triggered WebJob では、`settings.job` ファイルに Cron 式を記述することでスケジュール実行が可能です。

```json
{
  "schedule": "0 0,30 * * * *"
}
```

このスケジュールは Kudu（App Service のデプロイエンジン / 管理サイト）内のスケジューラ（`TriggeredJobsScheduler`）によって管理されます。WebJob がデプロイされると、スケジューラはジョブの **前回実行時刻（`LatestRun`）** と **Cron 式** をもとに次回実行タイミングを計算します。

  
## 問題 — デプロイ直後の意図しない即実行

以下のようなシナリオで問題が発生します。

1. Cron 式 `0 0,30 * * * *`（毎時 00 分と 30 分に実行）で WebJob を運用中
2. 13:00 に WebJob がスケジュール通り実行される（次回スケジュールは 13:30）
3. VFS API や FTP、Kudu ファイルブラウザ等で **WebJob のバイナリファイルを手動削除する**
   * ファイルウォッチャーが削除を検知し、スケジューラが 13:30 のスケジュールを **即座に解除** する（ログ: `Removing current schedule from WebJob`）
   * ただし実行履歴データ（`C:\home\data\`）は残るため `LatestRun = 13:00` が保持される
4. **13:35 に WebJob を再デプロイする**
5. スケジューラが `LatestRun = 13:00` を基に次回スケジュール（13:30）を再計算し、10 分以内の過去と判定して **即実行する**

※ 3 のステップにおいて、ポータルからの操作や `DELETE /api/triggeredwebjobs/{job name}` を用いた場合は実行履歴データも削除されるため、この記事で紹介する内容には該当しません。


## 再現検証

以下の 2 つのケースで検証を行いました。

**検証環境**: App Service（B1 プラン, Always On 有効）
**Cron 式**: `0 */30 * * * *`（毎時 00 分と 30 分）

前述 1 ~ 3 のステップ実行後、4 のステップを行うタイミングで動作が変わることを確認します。

### Case A — スケジュール時刻の 5 分後にデプロイ → 即実行される

```
[03/31/2026 12:35:16 > 11d75b: SYS INFO] Scheduling WebJob with 0 0,30 * * * * next schedule expected in 00:00:00
[03/31/2026 12:35:16 > 11d75b: SYS INFO] Invoking triggered web job on instance 11d75bcbb6c00295b0add383ae24b636e2d6a516557a878b08e12c19251084e4 at UTC datetime 3/31/2026 12:35:16 PM
[03/31/2026 12:35:16 > 11d75b: SYS INFO] WebJob invoked
```

`next schedule expected in 00:00:00` — 次回までの待機時間が 0 と判定され、即実行されています。

  
### Case B — スケジュール時刻の 15 分後にデプロイ → 即実行されない

```
[03/31/2026 13:15:18 > 11d75b: SYS WARN] Missed 1 schedules, most recent at 3/31/2026 1:00:00 PM
[03/31/2026 13:15:18 > 11d75b: SYS INFO] Scheduling WebJob with 0 0,30 * * * * next schedule expected in 00:14:41.7471271
```

スケジュールのミスが検出されましたが、即実行はされず、次回スケジュール（約 15 分後）まで待機しています。

 

## なぜこのような動作となるのか

### 実装から確認

Kudu のスケジューラ実装（[Schedule.cs](https://github.com/projectkudu/kudu/blob/master/Kudu.Core/Jobs/Schedule.cs)）の `GetNextInterval` メソッドに、以下のロジックがあります。

```csharp
if (ignoreMissed || nextOccurrence >= now - TimeSpan.FromMinutes(10))
{
    return TimeSpan.Zero;
}
```

このコードは次のように動作します：

- スケジューラが次回の実行予定時刻（`nextOccurrence`）を計算する
- その時刻が **現在時刻から 10 分以内の過去** であれば、待機時間を `TimeSpan.Zero`（= 即実行）として返す
- 10 分を超えて過去であれば、ミスしたスケジュールとして扱い、次の未来のスケジュールまで待機する

  
この **10 分間のグレース期間**が設けられていることから、Case A と Case B のようにデプロイタイミングによって動作が異なるものとなります。


## 削除方法による LatestRun への影響

  

WebJob の削除方法によって、実行履歴（`LatestRun`）の残り方が異なります。これはグレース期間の判定に影響するため注意が必要です。

  

### Kudu DELETE API / Azure ポータルからの削除

Kudu DELETE API(` DELETE /api/triggeredwebjobs/{job name}`)を呼びだした場合、 [`DeleteJob` メソッド](https://github.com/projectkudu/kudu/blob/master/Kudu.Core/Jobs/JobsManagerBase.cs#L245)は、ジョブのバイナリと実行履歴データの **両方** を削除します。

```csharp
// Remove both job binaries and data directories
FileSystemHelpers.DeleteDirectorySafe(jobDirectory.FullName, ignoreErrors: false);
FileSystemHelpers.DeleteDirectorySafe(jobsSpecificDataPath, ignoreErrors: false);
```

この方法で削除した場合、再デプロイ時に `LatestRun` は `2026 年 04 月 01 日時点の内容となります。<br>`（= `DateTime.MinValue`）となります。`GetNextInterval` メソッドでは `DateTime.MinValue` を `now` に置き換えるため、`GetNextOccurrence(now)` は常に未来の時刻を返し、**グレース期間による即実行は発生しません**。

### VFS / FTP でファイルを手動削除した場合

  
VFS API や FTP を使ってジョブのバイナリファイルのみを手動削除した場合、実行履歴データ（`C:\home\data\jobs\triggered\<job-name>\` 配下）は残ります。そのため再デプロイ時に前回の `LatestRun` が保持され、グレース期間の判定に影響します。

## 回避策

デプロイ直後の意図しない即実行を防ぐには、以下の方法があります。

### 方法 1: WebJob DELETE API で削除してから再デプロイする（推奨）

VFS / FTP による手動削除ではなく、Kudu の DELETE API または Azure ポータルから WebJob を削除することで、実行履歴（`LatestRun`）も含めてクリーンな状態にできます。再デプロイ時に `LatestRun = null` となるため、グレース期間による即実行は発生しません。

  

```powershell
# WebJob を DELETE API で削除（バイナリ + 実行履歴の両方が削除される）
Invoke-RestMethod -Uri "https://<your-app-name>.scm.azurewebsites.net/api/triggeredwebjobs/<your-job-name>" `
    -Method Delete `
    -Headers @{ Authorization = "Basic <credentials>" }

# WebJob を再デプロイ（任意のタイミングで実行可能、即実行されない）
Invoke-RestMethod -Uri "https://<your-app-name>.scm.azurewebsites.net/api/triggeredwebjobs/<your-job-name>" `
    -Method Put `
    -Headers @{
        Authorization = "Basic <credentials>"
        "Content-Disposition" = "attachment; filename=<your-job-name>.zip"
    } `
    -InFile "<your-job-name>.zip" `
    -ContentType "application/zip"
```

  

> **注意**: 実行履歴も削除されるため、履歴を保持したい場合は事前に [Kudu の History API](https://github.com/projectkudu/kudu/wiki/WebJobs-API)（`GET /api/triggeredwebjobs/<name>/history`）でバックアップしてください。

  

### 方法 2: `WEBJOBS_STOPPED` アプリケーション設定を使用する

デプロイ前にすべての WebJob を停止し、デプロイ完了後に解除します。

1. Azure ポータルまたは Azure CLI でアプリケーション設定 `WEBJOBS_STOPPED` を `1` に設定
2. WebJob をデプロイ
3. デプロイ完了後、`WEBJOBS_STOPPED` を削除（または `0` に設定）

```powershell
# デプロイ前: WebJob を停止
az webapp config appsettings set --name <your-app-name> --resource-group <your-resource-group> --settings WEBJOBS_STOPPED=1

# デプロイ実行（zip deploy 等）

# デプロイ後: WebJob を再開
az webapp config appsettings delete --name <your-app-name> --resource-group <your-resource-group> --setting-names WEBJOBS_STOPPED
```

> **注意**: `WEBJOBS_STOPPED` は App 上のすべての WebJob に影響します。

  
### 方法 3: デプロイタイミングを調整する

スケジュール時刻から **10 分以上経過してから** デプロイすることで、グレース期間を回避できます。

- 例: Cron 式が `0 0,30 * * * *` の場合、xx:10〜xx:20 または xx:40〜xx:50 の間にデプロイする

ただし、CI/CD パイプラインの実行タイミングを正確に制御することが難しい場合もあるため、方法 1 または方法 2 を推奨します。

  
## 参考情報


- [Azure App Service で WebJob を使用してバックグラウンド タスクを実行する - Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/app-service/webjobs-create)

- [Azure WebJobs SDK の使用方法 - Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/app-service/webjobs-sdk-how-to)

- [Kudu - Schedule.cs（GetNextInterval メソッド）](https://github.com/projectkudu/kudu/blob/master/Kudu.Core/Jobs/Schedule.cs)

- [Kudu - TriggeredJobsScheduler.cs](https://github.com/projectkudu/kudu/blob/master/Kudu.Core/Jobs/TriggeredJobsScheduler.cs)

- [Kudu - JobsManagerBase.cs（DeleteJob メソッド）](https://github.com/projectkudu/kudu/blob/master/Kudu.Core/Jobs/JobsManagerBase.cs#L245)
