---
title: "Azure Media Services で、字幕付き VOD 配信する方法"
author_name: "Yudai Kurashita"
tags:
    - Azure Media Service
---
# 質問
Azure Media Services の VOD 配信で、字幕付きの動画を配信するにはどのように設定すればよろしいですか？

# 回答
配信予定のアセットに、字幕ファイル（.vtt ファイル、あるいは .ttml ファイル）をアップロードすることで、Azure Media Services で字幕付き配信が可能です。<br>
下記に字幕ファイルの作成手順、VOD 字幕付き配信手順をご案内いたします。

## 字幕ファイルの作成手順
1.   Azure ポータルより、当該 Azure Media Services を開きます。
2.   左のブレードメニューから [資産] を選択します。
3.   資産の一覧の中から、字幕を作成したい資産（VOD 字幕配信を行う資産）を選択します。<br>
![image-1186459d-47bc-4e4c-bc8c-35c2d3d6465e.png]({{site.baseurl}}/media/2022/05/image-1186459d-47bc-4e4c-bc8c-35c2d3d6465e.png)
4.   上部にあるタブから [+ ジョブの追加] を選択します。<br>
![image-92ca33d6-f74d-4bcf-8244-22bf6428c4dd.png]({{site.baseurl}}/media/2022/05/image-92ca33d6-f74d-4bcf-8244-22bf6428c4dd.png)
5.   [変換] の項目で “新規作成” 、[変換の種類] の項目で “音声の文字起こし”、[言語] の項目で文字起こしを行う言語を選択し、ジョブを作成します。
![iage1_2c381262-f7ef-4d1a-aaeb-513d6d987e37.png]({{site.baseurl}}/media/2022/05/iage1_2c381262-f7ef-4d1a-aaeb-513d6d987e37.png)<br>※例として、[言語] を “英語 – 米国（en-US）” に選択しております。
6.   再度 [資産] ブレードに戻り、5. で作成したジョブの実行完了後、新たに作成された資産を開きます。
7.   [ストレージ コンテナー] を押下し、ストレージアカウントへ移動します。<br>![image2-500629d2-7521-42d6-bc18-c58ba1be4403.png]({{site.baseurl}}/media/2022/05/image2-500629d2-7521-42d6-bc18-c58ba1be4403.png)
8.   字幕ファイルである transcript.vtt、あるいは transcript.ttml をダウンロードします。<br>![image4-56edf38c-c94d-4894-9334-4b71941d56cd.png]({{site.baseurl}}/media/2022/05/image4-56edf38c-c94d-4894-9334-4b71941d56cd.png)
 <br>※ご利用されるプレイヤーに併せて、.vtt ファイル、あるいは .ttml ファイルのいずれかをご選択ください。
下記の ”VOD 字幕付き配信手順” では、例として .vtt ファイルを利用する場合を紹介いたします。）

## VOD 字幕付き配信手順
1.   Azure ポータルより、当該 Azure Media Services を開きます。
2.   左のブレードメニューから [資産] を選択します。
3.   資産の一覧の中から、字幕ファイルを取得した動画が格納されており、エンコードされた資産を開きます。
4.   上部にあるタブから “字幕のアップロード” を選択します。<br>![image5-b22b71d4-6ba0-4c58-8dd0-0982a2fa19ef.png]({{site.baseurl}}/media/2022/05/image5-b22b71d4-6ba0-4c58-8dd0-0982a2fa19ef.png)
5.   該当アセットの字幕ファイル（[字幕ファイルの作成手順] の8. でダウンロードした transcript.vtt）をアップロードします。<br>![image6-12e70306-5742-4a2b-b914-f2f007fffb0e.png]({{site.baseurl}}/media/2022/05/image6-12e70306-5742-4a2b-b914-f2f007fffb0e.png)
<br>※注意書きで記載されている通り、.vtt ファイル、.ttml ファイルのみ指定可能です。

6.   上記タブの、[+ 新しいストリーミング ロケーター] を選択します。
7.   [ストリーミング ポリシー] に “Predefined_DownloadAndClearStreaming” を選択し、ストリーミング ロケーターを追加します。<br>![image7-d29beab8-1ecb-480a-a63d-2d3a64c4dbba.png]({{site.baseurl}}/media/2022/05/image7-d29beab8-1ecb-480a-a63d-2d3a64c4dbba.png)
<br>※字幕ファイル（.vtt ファイル、.ttml ファイル）を取得するため、上記のストリーミング ポリシーを選択する必要があります。

8.   作成したストリーミング ロケーターで、ストリーミングエンドポイントを起動します。
9.   Azure Media Player “https://ampdemo.azureedge.net/” を開きます。
<br>※例としてプレイヤーは Azure Media Player を利用します。

10.  [URL] に、8. で起動した “ストリーミング URL” を入力します。<br>![image8-fae6c35f-55bc-4c59-bd18-db3766787999.png]({{site.baseurl}}/media/2022/05/image8-fae6c35f-55bc-4c59-bd18-db3766787999.png)
11.  [Add Track] を選択し、下記の通り入力します。<br>![image9-c9ec48d2-679e-4202-aac3-fbd81c503d0a.png]({{site.baseurl}}/media/2022/05/image9-c9ec48d2-679e-4202-aac3-fbd81c503d0a.png)

| 項目 | 値 |
|--|--|
| Kind | Captions |
| Track Label | 動画再生画面で字幕選択をする際の表示文字列（入力無しでも可） |
| Language | 字幕表示する言語 |
| WebVTT URL | ストリーミング URL の “/xxxx.ism/manifest” を削除し、“/<.vtt or .ttml ファイル名>” に変更した URL<br>例）https://test-jpea.streaming.media.azure.net/0000000-0000-0000-0000-000000000000/transcript.vtt |

12. 下部の “Update Player” を押下します。
13. 動画を再生し、下記の赤枠内のボタンを押下し、10. で入力した Track Label（スクリーンショットでは “en”）を選択します。<br>![image10-b1e0370f-32d7-401a-8a9b-e2f701293560.png]({{site.baseurl}}/media/2022/05/image10-b1e0370f-32d7-401a-8a9b-e2f701293560.png)
14. 字幕付きで動画が再生されます。<br>![image11-71a17dfd-ee66-49b0-81d6-b08abcc02ded.png]({{site.baseurl}}/media/2022/05/image11-71a17dfd-ee66-49b0-81d6-b08abcc02ded.png)

# 補足
- 字幕の言語は複数設定することが可能です。
- Web アプリケーションへストリーミング URL を埋め込み、VOD 配信を再生される場合、上記手順の ”WebVTT URL” を指定することで、VOD 字幕付き配信が可能です。<br>[Azure-Samples/azure-media-player-samples](https://github.com/Azure-Samples/azure-media-player-samples/blob/master/html/dynamic_webvtt.html)
にサンプルコードが記載されているのでご参照ください。
- ストリーミング ポリシーを “Predefined_DownloadAndClearStreaming” に選択できない場合、VOD 配信用のストリーミングロケーターと、字幕ファイルダウンロード用のストリーミングロケーター（“Predefined_DownloadAndClearStreaming”）をそれぞれ用意することで、VOD 字幕付き配信が可能です。
- ライブイベントで字幕付き配信を行うには、上記方法ではなく、ライブイベントで文字起こしを有効にする必要があります。<br>[ライブ文字起こし (プレビュー)](https://docs.microsoft.com/ja-jp/azure/media-services/latest/live-event-live-transcription-how-to)に手順が記載されているのでご参照ください。

# 参考ドキュメント
- [Media Services の機能](https://docs.microsoft.com/ja-jp/azure/media-services/latest/media-services-overview#what-can-i-do-with-media-services)
- [キャプションと字幕](https://docs.microsoft.com/ja-jp/azure/media-services/azure-media-player/azure-media-player-accessibility#captions-and-subtitles)
<br>
<br>
---
<br>
<br>
2022 年 5 月 11 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。
<br>
<br>