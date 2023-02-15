---
title: App Service における Docker User Namespace remapping issues について
author_name: "Takeharu Oshida"
tags:
    - App Service
---


# はじめに
お世話になっております。App Service サポート担当の押田です。

本記事は 2022 年 6 月 30 日に公開されました [Docker User Namespace remapping issues](https://azureossd.github.io/2022/06/30/Docker-User-Namespace-remapping-issues/index.html) の日本語訳です。
翻訳内容には細心の注意を払っておりますが、もし原文内容と齟齬がある場合は、原文の内容を優先くださいませ。

<!--
Docker has a concept of **User namespace** and **User namespace remapping** (Otherwise known as `userns`). When this is used, this remaps users in the container to less privileged users on the host machine. 

Sometimes, a user may be explicitly defined in a `Dockerfile` that has a `UID` mapped outside of the allowed range of id's. Othertimes, a file itself for example can have a high `UID`, which may happen without purposely doing this, as this would depend on how the Image is ultimately built (base Image, Third Party Image, etc.)

When these `UID`s are mapped to an id out of range, errors can occur, as explained below.

You can read more of the official documentation [here](https://docs.docker.com/engine/security/userns-remap/#user-namespace-known-limitations).

# Important note
The concept of username spaces is a typical Linux construct. The error described in this article is **NOT** an App Service and/or an Azure issue. This can easily be reproduced on essentially any machine that supports user namespaces and remapping, and is trying to create an ID out of the range set on the machine.

> **NOTE**: The range of these id's can **not** be changed on App Services. It is ultimately up to the developer or maintainer of the Image to resolve what is set with too high of an id.
-->

Docker には**ユーザ名前空間**と**名前空間の再割り当て**(userns とも呼ばれます)の概念があります。これを使用すると、コンテナ内のユーザーをホストマシン上の特権のないユーザーに再割当てできます。

`Dockerfile` で明示的に定義されたユーザーが、許可された ID 範囲の外の `UID` を持っている場合や、ファイル自体が大きな `UID` を持っている場合があります。これは、最終的にイメージがどのように構築されるか（ベースイメージ、サードパーティイメージなど）に依存するため、意図せずに発生する可能性があります。

許可された範囲外の `UID` が割当てされた場合、以下に示すようなエラーが発生する場合があります。

詳細については [こちら](https://docs.docker.com/engine/security/userns-remap/#user-namespace-known-limitations) のオフィシャルドキュメントもご参照ください。

# 重要事項

ユーザー名前空間の概念は、典型的な Linux の仕組みです。この記事で説明されているエラーは、App Service や Azure の問題ではありません。これは、ユーザー名前空間と再割当てをサポートし、マシン上の設定範囲外の ID を作成しようとしているマシンでも再現できます。

> **注意**: App Service において、ID の範囲の変更は**できません**。非常に大きな ID の解決は、最終的にはイメージの開発者または管理者にて行う必要があります。

<!--
# Errors 

## Types of errors
Sometimes if a user, group or file is mapped to a high UID or GUID you may see errors like the following:

- `failed to register layer: Error processing tar file(exit status 1): Container ID 1000000 cannot be mapped to a host IDErr: 0, Message: failed to register layer: Error processing tar file(exit status 1): Container ID 1000000 cannot be mapped to a host ID`
- `OCI runtime create failed: container_linux.go:380: starting container process caused: setup user: cannot set uid to unmapped user in user namespace: unknown`

There may be other variations of this message, but ultimately it will show that a user id cannot be mapped to the user namespace or hose.

## Where this may show
On App Services, this will show on the Image pull. Encountering these errors will cause the pull to fail - causing an `Application Error :(` message or a HTTP 502 when attempting to browse the site.

Below is an example of a pull that fails:

```yaml
2022-07-01T00:02:29.030Z INFO  - 6f199d2c4d2f Extracting 32KB / 308KB
2022-07-01T00:02:29.930Z INFO  - 6f199d2c4d2f Extracting 192KB / 308KB
2022-07-01T00:02:30.500Z INFO  - 6f199d2c4d2f Extracting 224KB / 308KB
2022-07-01T00:02:32.193Z INFO  - 6f199d2c4d2f Extracting 256KB / 308KB
2022-07-01T00:02:34.247Z INFO  - 6f199d2c4d2f Extracting 308KB / 308KB
2022-07-01T00:02:35.021Z INFO  - 6f199d2c4d2f Extracting 308KB / 308KB
2022-07-01T00:02:48.845Z INFO  - 6f199d2c4d2f Pull complete
2022-07-01T00:02:48.874Z INFO  - d163dfb13dd8 Extracting 193B / 193B
2022-07-01T00:02:48.876Z INFO  - d163dfb13dd8 Extracting 193B / 193B
2022-07-01T00:02:49.002Z INFO  - d163dfb13dd8 Pull complete
2022-07-01T00:02:49.017Z INFO  - 8820d67a46eb Extracting 202B / 202B
2022-07-01T00:02:49.019Z INFO  - 8820d67a46eb Extracting 202B / 202B
2022-07-01T00:02:49.113Z INFO  -  
2022-07-01T00:02:49.114Z ERROR - failed to register layer: Error processing tar file(exit status 1): Container ID 1000000 cannot be mapped to a host IDErr: 0, Message: failed to register layer: Error processing tar file(exit status 1): Container ID 1000000 cannot be mapped to a host ID
2022-07-01T00:02:49.124Z INFO  - Pull Image failed, Time taken: 0 Minutes and 55 Seconds
```

You can view log files to further correlate this through **Log Stream** or the **Kudu** site by viewing log files ending with `_docker.log`.
-->
# エラー

## エラーの種類

ユーザー、グループまたはファイルが大きな `UID` または、`GUID` に再割当てされる場合、以下のようなエラーを目にする可能性があります。

- `failed to register layer: Error processing tar file(exit status 1): Container ID 1000000 cannot be mapped to a host IDErr: 0, Message: failed to register layer: Error processing tar file(exit status 1): Container ID 1000000 cannot be mapped to a host ID`
- `OCI runtime create failed: container_linux.go:380: starting container process caused: setup user: cannot set uid to unmapped user in user namespace: unknown`

様々なパターンのメッセージがありますが、根本原因としては、uid を ユーザー名前空間に再割当てできないことに起因します。

## エラーの発生箇所

App Sericeでは、イメージの pull 実行時に発生します。これらのエラーが発生した場合、pull に失敗し、サイトアクセス時に `Application Error :(` が表示されたり、HTTP ステータス 502 が返却されることになります。

以下は、pull 失敗時のエラー例となります。

```yaml
2022-07-01T00:02:29.030Z INFO  - 6f199d2c4d2f Extracting 32KB / 308KB
2022-07-01T00:02:29.930Z INFO  - 6f199d2c4d2f Extracting 192KB / 308KB
2022-07-01T00:02:30.500Z INFO  - 6f199d2c4d2f Extracting 224KB / 308KB
2022-07-01T00:02:32.193Z INFO  - 6f199d2c4d2f Extracting 256KB / 308KB
2022-07-01T00:02:34.247Z INFO  - 6f199d2c4d2f Extracting 308KB / 308KB
2022-07-01T00:02:35.021Z INFO  - 6f199d2c4d2f Extracting 308KB / 308KB
2022-07-01T00:02:48.845Z INFO  - 6f199d2c4d2f Pull complete
2022-07-01T00:02:48.874Z INFO  - d163dfb13dd8 Extracting 193B / 193B
2022-07-01T00:02:48.876Z INFO  - d163dfb13dd8 Extracting 193B / 193B
2022-07-01T00:02:49.002Z INFO  - d163dfb13dd8 Pull complete
2022-07-01T00:02:49.017Z INFO  - 8820d67a46eb Extracting 202B / 202B
2022-07-01T00:02:49.019Z INFO  - 8820d67a46eb Extracting 202B / 202B
2022-07-01T00:02:49.113Z INFO  -  
2022-07-01T00:02:49.114Z ERROR - failed to register layer: Error processing tar file(exit status 1): Container ID 1000000 cannot be mapped to a host IDErr: 0, Message: failed to register layer: Error processing tar file(exit status 1): Container ID 1000000 cannot be mapped to a host ID
2022-07-01T00:02:49.124Z INFO  - Pull Image failed, Time taken: 0 Minutes and 55 Seconds
```

これらのログは `ログストリーム` 機能や、`Kudu` にて `_docker.log.` で終わるファイル名のログファイルから確認することができます。

<!--
# Resolution
## Find the file

**This must be ran locally and not on App Services**.

Start a shell into the locally running container from the Image in question and run `find / \( -uid 1000000 \)  -ls 2>/dev/null`.

Replace `1000000` with the id that fails in the error message during the Image pull.

Ideally, the below should be seen, in this case with a file owned by a user of too high of a `UID`:

```shell
# find / \( -uid 1000000 \)  -ls 2>/dev/null
  4758012      0 -rw-r--r--   1 veryhigh veryhigh        0 Jun 30 21:24 /var/www/html/file-with-high-id
```

> **NOTE**: If a user with a high `UID` is set as the `USER` of the container, then all files may appear owned by said user.

## Confirm the file 

**This must be ran locally and not on App Services**.

If a specific file is found to be the offender, it can be confirmed to have a high ID mapping by running:

`ls -ln file-with-high-id`

An example of output would be:

```shell
# ls -ln file-with-high-id
-rw-r--r-- 1 1000000 1000000 0 Jun 30 21:24 file-with-high-id
```

## Fix the file or user
To do this, the file(s) ownership must be changed. This can be done in the `Dockerfile` by using `chown` for example on the file.

If the user is defined in the Image like the below, update the id it is mapped to - below is an example:

```Dockerfile
RUN groupadd veryhigh -g 1000000
RUN useradd -r -u 1000000 -g veryhigh veryhigh
RUN touch file-with-high-id
RUN chown veryhigh:veryhigh file-with-high-id
```

If the Image is a 3rd party image, contact the maintainer about the `UID` used. Alternatively it may be possible to create your own Image and use the 3rd party image as a base to change ownership on the affect file(s) or user.

After resolving this, **the image must be rebuilt** and then can be deployed. This **must** be done locally.

-->

# 回避方法


## ファイルの特定

**次の操作は App Service 上ではなく、ローカル環境で実施する必要があります。**

問題のあるコンテナイメージをローカル環境で実行し `find / \( -uid 1000000 \) -ls 2>/dev/null` を shell から実行します。

`1000000 ` は実際に pull 失敗時のエラーメッセージに含まれる値に置き換えます。

理想的には、以下のように表示されるはずです。この場合、UID が大きすぎるユーザーが所有するファイルがあります。

```
# find / \( -uid 1000000 \)  -ls 2>/dev/null
  4758012      0 -rw-r--r--   1 veryhigh veryhigh        0 Jun 30 21:24 /var/www/html/file-with-high-id
```

> **注意**: もし、大きな `UID` がコンテナ内の `USER` として定義されている場合、全てのファイルが該当のユーザーに所有されていると表示される場合があります。

## ファイルの確認

**次の操作は App Service 上ではなく、ローカル環境で実施する必要があります。**

特定のファイルが特定できた場合、次のコマンドを実行することで、大きな ID マッピングが実行されていることがわかります。

`ls -ln file-with-high-id`

サンプル実行例:

```shell
# ls -ln file-with-high-id
-rw-r--r-- 1 1000000 1000000 0 Jun 30 21:24 file-with-high-id
```
## ファイルまたはユーザーの修正
このためには、ファイルの所有権を変更する必要があります。例えばファイルに対しては `Dockerfile` 内で `chown` コマンドを利用します。
もし、以下のようにユーザーがイメージ内に定義されている場合、以下のように割り当てられるIDをセットする必要があります。

```Dockerfile
RUN groupadd veryhigh -g 1000000
RUN useradd -r -u 1000000 -g veryhigh veryhigh
RUN touch file-with-high-id
RUN chown veryhigh:veryhigh file-with-high-id
```

もし、イメージがサードパーティ製の場合は、`UID` を使用しているメンテナーに連絡してください。あるいは、サードパーティ製イメージをベースとし、問題のあるファイルあるいはユーザーを変更したイメージを作成することも可能です。

問題を解消した後に、**イメージを再ビルドし**、デプロイすることができます。この作業は**ローカルで実施する必要**があります。 

## NPM 利用プロジェクトにおける ユーザー名前空間再割当てエラー
特定の NPM バージョンでは、`node_modules` インストール時に、非常に大きな ID をファイルの所有者/作成者として割り当てることがあります。これは **npm 9** 系において導入されました。npm コミュニティにおいてもこの問題に関連するディスカッションが以下の 3 つのスレッドで確認できます。 一読することをお勧めします。

- [RFC thread ([RRFC] Clean up file ownership story)](https://github.com/npm/rfcs/issues/546)
- [NPM 9 change log (Breaking Changes)](https://docs.npmjs.com/cli/v9/using-npm/changelog?v=true#900-pre6-2022-10-19)
- [This GitHub issue (GitHub Issue 5900)](https://github.com/npm/cli/issues/5900)

**npm 9** 系を利用する以下の `Dockerfile` を App Service にデプロイした場合、ユーザー名前空間再割当てに関するエラーが発生します。

> **注意**: [npm 5](https://github.com/projectkudu/kudu/issues/2512)系でも発生することもあります。

**(Dockerfile)**:
```Dockerfile
FROM node:18.9.0-alpine3.15

WORKDIR /app
COPY package.json ./
RUN npm i -g npm@9.1.1 && \
    npm i

COPY . ./

EXPOSE 8080 

CMD [ "node", "/app/server.js" ]
```

App Service で発生するエラー:

```
ERROR - failed to register layer: Error processing tar file(exit status 1): Container ID 1516583083 cannot be mapped to a host IDErr: 0, Message: failed to register layer: Error processing tar file(exit status 1): Container ID 1516583083 cannot be mapped to a host ID
```

前述のアプローチと同様に、**ローカル環境で調査する必要があります。**　ただし、`find` コマンドでは、問題を起こしている UID を特定することができません。 ID 変更が発生しているためです。（詳細については前述のスレッド内に書かれています。）

NPM と node_modules に関連する問題がわかっているため、以下のアプローチで特定します。
実際には、利用するプロジェクト内の `node_modules` のパスに置き換えてください。

`node_modules`:

```bash
FILES=$(find /app/node_modules/ ! -user root)
ls -lrta $FILES
```
エラーに示される ID と一致する以下の所有者名を確認することができます。

```bash
-rw-r--r--    1 501      dialout       1490 Nov 29 17:18 /app/node_modules/cookie-signature/Readme.md
-rw-r--r--    1 501      dialout        695 Nov 29 17:18 /app/node_modules/cookie-signature/History.md
-rw-r--r--    1 501      dialout         29 Nov 29 17:18 /app/node_modules/cookie-signature/.npmignore
-rw-r--r--    1 15165830 root          1070 Nov 29 17:18 /app/node_modules/content-type/package.json
-rw-r--r--    1 15165830 root          4809 Nov 29 17:18 /app/node_modules/content-type/index.js
-rw-r--r--    1 15165830 root          2796 Nov 29 17:18 /app/node_modules/content-type/README.md
-rw-r--r--    1 15165830 root          1089 Nov 29 17:18 /app/node_modules/content-type/LICENSE
-rw-r--r--    1 15165830 root           436 Nov 29 17:18 /app/node_modules/content-type/HISTORY.md
-rw-r--r--    1 501      dialout        703 Nov 29 17:18 /app/node_modules/asynckit/stream.js
-rw-r--r--    1 501      dialout       1751 Nov 29 17:18 /app/node_modules/asynckit/serialOrdered.js
-rw-r--r--    1 501      dialout        501 Nov 29 17:18 /app/node_modules/asynckit/serial.js
-rw-r--r--    1 501      dialout       1017 Nov 29 17:18 /app/node_modules/asynckit/parallel.js
```

回避策としては, `chown` コマンドによってディレクトリ配下のファイルの所有者を変更します。

**重要**: `chown` を実行する場合、それは `npm install` を実行するレイヤと**必ず**同じレイヤで実行する必要があります。もし 2 つの異なるレイヤで実行された場合、所有者の変更は正しく動かず、名前空間再割当てに関するエラーは解消しません。

(Do)
```
RUN npm i -g npm@9.1.1 && \
    npm i && \
    find /app/node_modules/ ! -user root | xargs chown root:root
```

(Dont)
```
RUN npm i -g npm@9.1.1 && \
    npm i 

RUN find /app/node_modules/ ! -user root | xargs chown root:root
```

### UID を特定できない場合はどうすると良いか?
コンテナを Pull し作成エラーとなった際に、エラーメッセージから UID が特定できない場合もあります。

常にそうなるわけではありませんが、**同様のアプローチ** で解決することができます。

なぜなら、 `find /app/node_modules/ ! -user root` を実行しているためです。例えば、ユーザーが **50x** の範囲にあり、グループ **dialout** に含まれる node_module 関連のファイルが表示される場合があります。 ファイルシステム全体を検索すると、通常、これらはユーザー/グループとして、 `root` と `node` 以外に表示される唯一のものです。
このようなシナリオの場合、50x ユーザーと dialout グループを **root** (または、許容範囲内にあるUID) に変更することで問題が解消します。これは、NPM インストール実行時にユーザー切り替えが起きているためとなります。**NPM** に関連する当セクションの先頭に記載したスレッドのリンクを参照ください。

2023 年 02 月 15 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
