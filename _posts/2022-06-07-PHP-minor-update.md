---
title: "PHP 7.2 および 7.3 の PHP 7.4 へのアップデートに関して"
author_name: "Yusuke Tobo"
tags:
    - App Service
---

お世話になっております。App Service サポート担当の東方です。

Windows の PHP スタックでよく頂く疑問点にお答えしたく本記事を作成しました。

みなさまの疑問の解消に役立てましたら嬉しいです。

# 質問
Windows の App Service で PHP 7.2 や PHP 7.3 で動作しているアプリケーションが急にサーバーエラーを返すようになったのはなぜですか？

# 回答
Windows の App Service から PHP 7.2 や PHP 7.3 のバイナリが削除されたためです。

App Service の PHP スタックにおきましては、サポートが切れたマイナーバージョンから、サポートされているマイナーバージョンへのマイナーバージョンアップデートが App Service によって実施されます。

▼ [End of Extended Support](https://github.com/Azure/app-service-linux-docs/blob/master/Runtime_Support/php_support.md#end-of-extended-support) より抜粋
```
Once a version of PHP has reached it's end of extended support your application will be upgraded to the next recommended supported minor version.

For example on February 01, 2020 any application running on PHP 7.0 or PHP 7.1 will be upgraded to PHP 7.3
```

PHP 7.2 や PHP 7.3 は既にサポート終了を迎えておりますので、サポートされているマイナーバージョン PHP 7.4 にマイナーバージョンアップデートが発生しました。

本アップデートにおきましては、PHP 7.2 や PHP 7.3 のバイナリが Windows の App Service から削除されますが、後述の例外を除き、お客様のアプリケーションは PHP 7.4 のバイナリを使って動作するようになります。

例外は、お客様のアプリケーションやその設定ファイルが PHP 7.2 や PHP 7.3 のバイナリを直接参照している場合です。
例えば、web.config 等で PHP 7.2 や PHP 7.3 を直接参照されている場合は、お客様の App Service でサーバーエラーが発生するようになります。 

上記の例外に該当する場合におけるサーバーエラーの解消方法といたしましては、PHP 7.2 や PHP 7.3 をどのようにお客様がご利用されているのかに依ります一方で、下記 PHP 7.4 のフォルダ内のバイナリをご利用いただくようにアプリケーションや設定ファイルの修正をお客様で実施いただくことになると考えられます。


|プラットフォーム| PHP 7.4 のフォルダ [1] |
|--|--|
| 32 bit | C:\Program Files (x86)\PHP\v7.4 |
| 64 bit | C:\Program Files\PHP\v7.4 |




[注釈1] お客様の環境によっては、ドライブレターが D ドライブになる場合がございます。`where php` を高度なツールから実行し、PHP 7.4 のバイナリが存在するパスをご確認ください。  


<br>
<br>

---

<br>
<br>

2022 年 06 月 07 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>