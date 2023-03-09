---
title: "App Serviceのアプリケーション設定で改行を含む値を登録する方法"
author_name: "kento yokozuka"
tags:
    - App Service
---

# はじめに
App Service の アプリケーション設定における改行を含む値を登録する方法についてご紹介いたします。

# 概要
App Service のアプリケーション設定における改行を含む値を登録するには、

- 高度な編集を利用する
- Base64 エンコードした値を登録し、アプリケーション側では Base64 デコードして利用する
- Kay Vault を利用する

のいずれかの方法ご利用いただくこと可能となります。詳細については、以下にてご説明いたします。   

# この記事で解消したい問題
ポータル上の アプリケーション設定の追加/編集機能からは"\n"や"\r\n"を設定しても改行として反映されません。

## 失敗する例
### 失敗例1) ポータルのアプリケーション設定から編集
改行コードをそれぞれ（\r\n, \n）設定、保存してみます。

![image-f2d233c5-843a-47c8-a149-b3d1c13bdcb7.png]({{site.baseurl}}/media/2023/03/image-f2d233c5-843a-47c8-a149-b3d1c13bdcb7.png)

![image-e4125481-be9d-47c3-b4c8-4b12727e91c0.png]({{site.baseurl}}/media/2023/03/image-e4125481-be9d-47c3-b4c8-4b12727e91c0.png) 

高度な編集で確認すると、改行コードがエスケープされていることが確認できます。

![image-2de9bec5-4fbe-40a2-8131-29d6eeeffdfe.png]({{site.baseurl}}/media/2023/03/image-2de9bec5-4fbe-40a2-8131-29d6eeeffdfe.png)

また、後述の検証コードの実行結果を確認すると改行コードが文字列として表示されます。

![image-fa7b6f64-cba5-48af-978a-831de599bb1e.png]({{site.baseurl}}/media/2023/03/image-fa7b6f64-cba5-48af-978a-831de599bb1e.png)

### 失敗例2) az cli を使用してアプリケーション設定を設定
az cli を使用して以下のコマンドを実行してみます。

```bash
az webapp config appsettings set -g appservicejapanrg -n linux-python310-webapp --settings LF=aaa\nbbb\nccc
```

アプリケーション設定を確認すると以下のように"\\"がエスケープされ"n"のみ文字列として登録されます。

![image-39fb5041-9ee4-44e5-8c71-f8d593c5f05c.png]({{site.baseurl}}/media/2023/03/image-39fb5041-9ee4-44e5-8c71-f8d593c5f05c.png)

次に、"\\\n"を追加してコマンドを実行します。
```
az webapp config appsettings set -g appservicejapanrg -n linux-python310-webapp --settings LF=aaa\\nbbb\\nccc
```

アプリケーション設定を確認すると、一つの"\\"だけエスケープされ、"\n"が文字列として登録されます。

![image-929ed8fd-c8bf-40f7-96b2-e707e8ae5832.png]({{site.baseurl}}/media/2023/03/image-929ed8fd-c8bf-40f7-96b2-e707e8ae5832.png)

※ **CRLFの場合も同様です**


---

以下、回避策についてご案内します。

# 検証用ソースコード
今回検証に使用したソースコード(Python3.10)は以下になります。

~~~:python
import base64
import logging
import os

from flask import Flask

app = Flask(__name__)
logger = logging.getLogger('azure.mgmt.resource')
logger.setLevel(logging.INFO)

@app.route('/')
def index():
    """ 環境変数より取得した値をログ出力します
    """
    lf = os.environ['LF']
    print('LF:')
    print(lf)
    
    crlf = os.environ['CRLF']
    print('CRLF:')
    print(crlf)
    
    return f'<h2>LF:</h2><br><pre>{lf}</pre><br><h2>CRLF:</h2><br><pre>{crlf}</pre>'


@app.route('/encode')
def newline():
    """ Base64でエンコードされた値をデコードしてログ出力します
    """
    encode_lf = os.environ['LF']
    print('LF:')
    print(encode_lf)

    # base64 デコード処理
    decode_lf = base64.b64decode(encode_lf).decode()
    print(decode_lf)

    encode_crlf = os.environ['CRLF']
    print('CRLF:')
    print(encode_crlf)

    # base64 デコード処理
    decode_crlf = base64.b64decode(encode_crlf).decode()
    print(decode_crlf)

    return f'<h2>LF:</h2><br><pre>{decode_lf}</pre><br><h2>CRLF:</h2><br><pre>{decode_crlf}</pre>'
~~~ 

# 回避策
## 回避策1) アプリケーション設定の「高度な編集」を利用する方法
アプリケーション設定の「高度な編集」で文字列に改行コード(CRLF/LF)を追加することにより、改行をすることが可能となります。

![image-4066ad75-f1cc-4b03-83b2-daf96ca3ed59.png]({{site.baseurl}}/media/2023/03/image-4066ad75-f1cc-4b03-83b2-daf96ca3ed59.png)

実行結果

![image-9e34b7e5-4be2-476b-9498-bc43d904669a.png]({{site.baseurl}}/media/2023/03/image-9e34b7e5-4be2-476b-9498-bc43d904669a.png)


### 注意点

「高度な編集」にて文字列に改行コード(CRLF/LF)を設定して保存した場合においても、アプリケーション設定で確認する場合においては表示が異なります。

アプリケーション設定で確認をすると空白に変換され、表示されます。

![image-0c3ea911-ee96-4542-947e-7d8b662c8292.png]({{site.baseurl}}/media/2023/03/image-0c3ea911-ee96-4542-947e-7d8b662c8292.png)

また、アプリケーション設定の編集で確認すると、改行コードも空白もなく連結されて表示されます。

![image-aaa77a69-7f9f-47dc-b437-256b996fae15.png]({{site.baseurl}}/media/2023/03/image-aaa77a69-7f9f-47dc-b437-256b996fae15.png)

![image-5240bc09-44f8-493c-8960-7d64cd62b673.png]({{site.baseurl}}/media/2023/03/image-5240bc09-44f8-493c-8960-7d64cd62b673.png)

「高度な編集」で保存後にアプリケーション設定の追加/編集機能で値を編集をして保存した場合、改行コードの設定が削除されてしまい、アプリケーション設定で設定した値に上書きされてしまいます。 

![image-1ebfe567-e76d-49b3-8580-f30a1c1459e2.png]({{site.baseurl}}/media/2023/03/image-1ebfe567-e76d-49b3-8580-f30a1c1459e2.png)
 
あらためて「高度な編集」で確認してみると、改行コードも空白もない文字列として登録されています。

![image-d929275b-6370-476a-8dfe-6b8970a1dac5.png]({{site.baseurl}}/media/2023/03/image-d929275b-6370-476a-8dfe-6b8970a1dac5.png)

実行結果

![image-6ec3e71d-e748-4dac-b2af-20440754fb46.png]({{site.baseurl}}/media/2023/03/image-6ec3e71d-e748-4dac-b2af-20440754fb46.png)

※ **CRLFの場合も同様です**

## 回避策2) Base64 エンコードした値を登録し、アプリケーション側では Base64 デコードして利用する方法
次の改行(CRLF/LF)を含む値を base64 でエンコードし、エンコードした値をアプリケーション設定に設定し、プログラム側ではデコード処理することにより改行を含む値を取得することが可能となります。

```
aaa
bbb
ccc
```

上記改行(CRLF/LF)を含む文字列をエンコードするとそれぞれ以下の文字列になります。

```bash
echo -n "aaa\r\nbbb\r\nccc" | base64
YWFhDQpiYmINCmNjYw==

$ echo -n "aaa\nbbb\nccc" | base64
YWFhCmJiYgpjY2M=
```

実行結果(/encodeエンドポイント)

![image-c9fdf990-a5d9-41a7-9925-c575963408b9.png]({{site.baseurl}}/media/2023/03/image-c9fdf990-a5d9-41a7-9925-c575963408b9.png)

## 回避策3) Kay Vault を利用する方法

Key Vault に改行(CRLF/LF)を含むデータを登録し、アプリケーションから利用します。

###前提条件
Key Vault が作成されており、App Service からKey Vaultへのアクセスはできていること 

### Key Vault シークレットへの設定手順
1. 改行(CRLF/LF)を含むデータを secretfile.txt (ファイル名は任意)として保存します。
1. az Cli を使用して、Key Vault シークレットに設定します。

    ```bash
    az keyvault secret set --vault-name "<your-unique-keyvault-name>" --name "MultilineSecret" --file "secretfile.txt"
    ```

     登録後、Key Vault にて確認すると以下のようになります。

    ![image-364c241a-3d4b-443d-a58a-5607eb5484ab.png]({{site.baseurl}}/media/2023/03/image-364c241a-3d4b-443d-a58a-5607eb5484ab.png)

1. 作成した シークレットの「シークレット識別子」をコピーして、App Service -> 構成 -> アプリケーション設定 にて環境変数を設定します。<br>
    ```
    値: @Microsoft.KeyVault(SecretUri=●シークレット識別子●)
    ```

実行結果

![image-650168d6-0cd8-4740-8fb4-946d7ec27d6a.png]({{site.baseurl}}/media/2023/03/image-650168d6-0cd8-4740-8fb4-946d7ec27d6a.png)

※ **CRLFの場合も同様です**

# まとめ
以上のように、高度な編集、Base64、Key Vault のいずれかをご利用いただくことで、改行を含む値を登録することが可能となります。

# Appendix
アプリケーション設定に関する詳細については以下の公式ドキュメントを参照していただけますと幸いです。

< App Service アプリを構成する ><br>
https://learn.microsoft.com/ja-jp/azure/app-service/configure-common?tabs=portal

< Azure Key Vault に複数行のシークレットを格納する ><br>
https://learn.microsoft.com/ja-jp/azure/key-vault/secrets/multiline-secrets


<br>
<br>

---

<br>
<br>

2023 年 03 月 10 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
