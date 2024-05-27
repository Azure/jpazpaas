---
title: "Python logging ライブラリで出力したログが全て AppServiceConsoleLogs にて Error レベルと記録される"
author_name: "Gakushi Ihsii"
tags:
    - App Service
---

# 質問
Python ランタイムの Azure Web Apps で診断設定の AppServiceConsoleLogs を有効にすると、 logging ライブラリのログレベルに関わらず、出力しているログが全て Error レベルで記録される。AppServiceConsoleLogs のログレベルを制御することは可能か。

![image-d7e395ee-f602-4200-85ec-e5e756305640.png]({{site.baseurl}}/media/2024/05/image-d7e395ee-f602-4200-85ec-e5e756305640.png)

# 回答
AppServiceConsoleLogs のログレベルは Informational, Error のみとなっておりますので、この2つのレベルに制御することは可能です。AppServiceConsoleLogs は標準出力/標準エラーのログを出力するための機能として設計されおります。このような設計思想より、恐縮ながら logging ライブラリが独自に定義しているログレベル (debug, info, warning, error, critical) でログを分類することは叶いません。ログレベルの情報はログの本文の先頭に付与されますので、logging ライブラリで出力したログのフィルタリングが必要な場合はこちらの情報をご活用ください。（上記画像内の ResultDescription 参照） 


## 全てのログが Error レベルで記録される理由
AppServiceConsoleLogs は [標準出力 (stdout) と標準エラー (stderr) をキャプチャ](https://learn.microsoft.com/ja-jp/azure/app-service/troubleshoot-diagnostic-logs#supported-log-types) しており、 標準出力は Informational レベル, 標準エラーは Error レベルと記録いたします。

![image-e71f70b1-bbe7-48a3-bbc2-f5a4567c3115.png]({{site.baseurl}}/media/2024/05/image-e71f70b1-bbe7-48a3-bbc2-f5a4567c3115.png)

参考：[診断ログの有効化 - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/troubleshoot-diagnostic-logs#supported-log-types)

一方で、 Python logging ライブラリは [ログ出力先としてファイルパスを指定せず、既定のままとした場合、 StreamHandler が使用](https://docs.python.org/3.10/library/logging.html#logging.basicConfig)され、[StreamHandler は既定で 標準エラー (stderr) にログを出力](https://docs.python.org/3.10/library/logging.handlers.html#logging.StreamHandler)いたします。

![image-3-0f2e4f63-aa4e-4451-814a-379ac19b20c8.png]({{site.baseurl}}/media/2024/05/image-3-0f2e4f63-aa4e-4451-814a-379ac19b20c8.png)

参考：<https://docs.python.org/3.10/library/logging.html#logging.basicConfig>


![image-2-e9ace92a-46d8-4e7a-b3c9-b73a13638f7d.png]({{site.baseurl}}/media/2024/05/image-2-e9ace92a-46d8-4e7a-b3c9-b73a13638f7d.png)

参考：<https://docs.python.org/3.10/library/logging.handlers.html#logging.StreamHandler>

そのため、 logging の設定が既定だった場合、ログレベルに依らず全てのログが標準エラーへ出力され、 AppServiceConsoleLogs にて Error レベルと記録されますので、App Service としては想定された正常な動作となります。


## Informational レベルでログを記録する方法はありますか？
stream の出力先を標準出力 (stdout) とすることで、logging ログを Informational レベルで記録いただけます。

具体的には、下記コード例のように basicConfig() の引数に stream=sys.stdout と指定することで標準出力にログを出力することが可能となります。

```python
import logging

app = Flask(__name__)

@app.route('/')
def index():
    # logging.basicConfig()
    logging.basicConfig(stream=sys.stdout)
    logger = logging.getLogger(__name__)
    logger.setLevel("DEBUG")
    logger.debug("debug log")
    logger.info("info log")
    logger.warning("warning log")
    logger.error("error log")
    logger.critical("critical log")
    print('Request for index page received')
    return render_template('index.html')
```

※標準出力に出力されたログは Informational レベルで AppServiceConsoleLogs に記録される。 

![image-5-c4692b97-1e6d-4a5c-90f8-2c11d358806d.png]({{site.baseurl}}/media/2024/05/image-5-c4692b97-1e6d-4a5c-90f8-2c11d358806d.png)

本記事が参考になりましたら幸いでございます。
<br>
<br>

---

<br>
<br>

2024 年 05 月 23 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>