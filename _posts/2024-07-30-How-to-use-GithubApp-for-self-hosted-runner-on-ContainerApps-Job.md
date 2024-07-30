---
title: "ContainerApps ジョブのセルフホステッドランナーを Github App で運用する方法"
author_name: "satoshiurano"
tags:
    - Container Apps

---

# 質問
以下、公開されているチュートリアルでは、PAT (Personal Access Token) を利用して GitHub のSelf-Hosted RunnerをContainer Appsで構築しているが、GitHub App で運用する方法はあるか。<br>
[チュートリアル:Azure Container Apps ジョブを使用してセルフホスト型 CI/CD ランナーとエージェントをデプロイする](https://learn.microsoft.com/ja-jp/azure/container-apps/tutorial-ci-cd-runners-jobs?tabs=bash&pivots=container-apps-jobs-self-hosted-ci-cd-github-actions#deploy-a-self-hosted-runner-as-a-job)


# 回答
Container Apps ジョブの利用例として、PAT を使ったセルフホスト型の CI/CD ランナーの構築方法をチュートリアルにて紹介していますが、代わりに Github App で運用することも可能となります。<br>
以下に必要な手順について記載いたします。

**===================================================================<br>Github App の作成（および Github App ID、Installation ID、秘密鍵を取得する）<br>===================================================================**

前提となる Github App を作成し、ジョブに必要となる各値を取得します。こちらの手順については以下 Github Docs をご参照ください。<br>
[GitHub App インストールとしての認証](https://docs.github.com/ja/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation)<br>

**===================================================================<br>Github App を利用するコンテナイメージの作成<br>===================================================================**

Github App を利用するようスクリプトを作成します。<br>

チュートリアルの下記 Github リポジトリをフォークします。<br>
https://github.com/Azure-Samples/container-apps-ci-cd-runner-tutorial/tree/main/github-actions-runner

フォークしたリポジトリにおいて、entrypoint.sh を下記のように編集します。サンプルコードは以下の各 GitHub Docs を参考として作成したものとなります。<br>
[GitHub アプリの JSON Web トークン (JWT) の生成](https://docs.github.com/ja/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-json-web-token-jwt-for-a-github-app#example-using-bash-to-generate-a-jwt)
<br>
[インストール アクセス トークンを使ってアプリ インストールとして認証する](https://docs.github.com/ja/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation#using-an-installation-access-token-to-authenticate-as-an-app-installation
)

```
#!/bin/bash -l

# 以下の環境変数を利用しますが、これらの環境変数は後述の手順で container app ジョブに設定します。
# $PEM_KEY
# $GITHUB_APP_ID
# $GITHUB_OWNER
# $REPOS

# GitHub App の秘密鍵の内容をファイルに書き込み利用していきます。
echo "$PEM_KEY" > github_app_private_key.pem

# 下記 Github Docs に記載の bash スクリプトを参考に jwt を取得します。
# https://docs.github.com/ja/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-json-web-token-jwt-for-a-github-app#example-using-bash-to-generate-a-jwt

set -o pipefail
client_id=$GITHUB_APP_ID # Client ID as first argument

now=$(date +%s)
iat=$((${now} - 60)) # Issues 60 seconds in the past
exp=$((${now} + 600)) # Expires 10 minutes in the future

b64enc() { openssl base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n'; }

header_json='{
    "typ":"JWT",
    "alg":"RS256"
}'
# Header encode
header=$( echo -n "${header_json}" | b64enc )

payload_json='{
    "iat":'"${iat}"',
    "exp":'"${exp}"',
    "iss":'"${client_id}"'
}'
# Payload encode
payload=$( echo -n "${payload_json}" | b64enc )

# Signature
header_payload="${header}"."${payload}"
signature=$(
    openssl dgst -sha256 -sign ./github_app_private_key.pem \
    <(echo -n "${header_payload}") | b64enc
)

# Create JWT
jwt="${header_payload}"."${signature}"
printf '%s\n' "JWT: $jwt"

# 取得した jwt トークンを利用して Github App トークンを取得します。下記 Github Docs に記載の REST API を利用しています。
# https://docs.github.com/ja/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation#using-an-installation-access-token-to-authenticate-as-an-app-installation

installation_id="$(curl --location --silent --request GET \
  --url "https://api.github.com/users/$GITHUB_OWNER/installation" \
  --header "Accept: application/vnd.github+json" \
  --header "X-GitHub-Api-Version: 2022-11-28" \
  --header "Authorization: Bearer $jwt" \
  | jq -r '.id'
)"

echo "installation_id is: $installation_id"

token="$(curl --location --silent --request POST \
  --url "https://api.github.com/app/installations/$installation_id/access_tokens" \
  --header "Accept: application/vnd.github+json" \
  --header "X-GitHub-Api-Version: 2022-11-28" \
  --header "Authorization: Bearer $jwt" \
  | jq -r '.token'
)"

echo "token is: $token"

registration_token="$(curl -X POST -fsSL \
  -H 'Accept: application/vnd.github.v3+json' \
  -H "Authorization: Bearer $token" \
  -H 'X-GitHub-Api-Version: 2022-11-28' \
  "https://api.github.com/repos/$GITHUB_OWNER/$GITHUB_REPO/actions/runners/registration-token" \
  | jq -r '.token')"
echo "registration_token is: $registration_token"
echo "url is: https://github.com/$GITHUB_OWNER/$GITHUB_REPO"

# 取得したトークンでGithubリポジトリへアクセスします
./config.sh --url https://github.com/$GITHUB_OWNER/$GITHUB_REPO --token $registration_token --unattended --ephemeral && ./run.sh
```

上記スクリプトを含むコンテナイメージを ACR (Azure Container Registry) に登録します。<br>
ACR の構築手順については、下記チュートリアルの項目をご参照ください。<br>
https://learn.microsoft.com/ja-jp/azure/container-apps/tutorial-ci-cd-runners-jobs?tabs=azure-powershell&pivots=container-apps-jobs-self-hosted-ci-cd-github-actions#build-the-github-actions-runner-container-image<br>

なお、上記資料に記載の az acr build のコマンドにおいては、フォークした URL をご指定ください。
```
az acr build \
    --registry "$CONTAINER_REGISTRY_NAME" \
    --image "$CONTAINER_IMAGE_NAME" \
    --file "Dockerfile.github" \
    "フォークした github リポジトリの URL"
```



以上により、Container Apps ジョブを GitHub App の秘密鍵で運用することが可能となります。<br>


**===================================================================<br>Container App ジョブが参照する Github App の秘密鍵を Key Vault に格納する<br>===================================================================**

上記スクリプトにて利用する Github App の秘密鍵をあらかじめ Key Vault にアップロードし、Container App ジョブが参照できるようにします。
<br>秘密鍵は RSA PRIVATE KEY 形式にて内容が複数行となるため、KEDA スケールルールや Github 側に正常に渡るように、Key Vault にはファイル形式にてアップロードいただければと思います。
 
Key Vault への Github Apps 秘密鍵のアップロードするコマンド：<br>
`az keyvault secret set --vault-name "Key Vault リソース名" --name "シークレットの名称" --file "お手元のGithub App 秘密鍵"`

Container App ジョブ からは、Github Apps  秘密鍵をアップロードしたこの Key Vault にはマネージド ID で接続します。<br>
このため、あらかじめマネージド ID をご用意いただき、Key Vault 側で当該のマネージド ID のアクセスを許可するよう構成しておく必要があります。<br>
マネージド ID で Key Vault へ接続する手順に関しては、以下の公開資料をご参照ください。<br>
https://learn.microsoft.com/ja-jp/azure/container-apps/manage-secrets?tabs=azure-cli#reference-secret-from-key-vault

**===================================================================<br>Github App を利用する Container App ジョブの作成<br>===================================================================**


上記まで完了したら、以下コマンドにて Container App ジョブを作成します。<br>
※ 変数化している箇所は事前に定義が必要です。

```
az containerapp job create -n "$JOB_NAME" -g "$RESOURCE_GROUP" --environment "$ENVIRONMENT" \
    --trigger-type Event \
    --replica-timeout 1800 \
    --replica-retry-limit 0 \
    --replica-completion-count 1 \
    --parallelism 1 \
    --image "$CONTAINER_REGISTRY_NAME.azurecr.io/$CONTAINER_IMAGE_NAME" \
    --min-executions 0 \
    --max-executions 10 \
    --polling-interval 30 \
    --scale-rule-name "github-runner" \
    --scale-rule-type "github-runner" \
    --scale-rule-metadata "githubAPIURL=https://api.github.com" "owner=$REPO_OWNER" "runnerScope=repo" "repos=$REPO_NAME" "targetWorkflowQueueLength=1" "applicationID=$GITHUB_APP_ID" "installationID=$GITHUB_APP_INSTALL_ID" \
    --scale-rule-auth "appKey=$KEY_NAME" \
    --cpu "2.0" \
    --memory "4Gi" \
    --user-assigned "$MANAGED_IDENTITY_ID" \
    --secrets "$KEY_NAME=keyvaultref:$KEY_VAULT_SECRET_URI,identityref:$MANAGED_IDENTITY_ID" \
    --env-vars "PEM_KEY=secretref:$KEY_NAME" "GITHUB_APP_ID=$GITHUB_APP_ID" "GITHUB_OWNER=$REPO_OWNER" "GITHUB_REPO=$REPO_NAME" \
    --registry-server "$CONTAINER_REGISTRY_NAME.azurecr.io"
```
上記にて、Github Apps を利用したセルフホステッドランナーの Container App ジョブが作成されます。<br>
またここで、シークレット（Github Apps 秘密鍵）を appKey としてスケールルールの認証に渡している点に注目します。これは、スケールルールとして利用する "KEDA" のドキュメントにおいて、appKey に Github Apps の秘密鍵を渡すことで認証可能であることが記載されているため、このように指定しています。<br>
[KEDA | Github Runner Scalerについて](https://keda.sh/docs/2.13/scalers/github-runner/#trigger-specification)

![image-7567326f-3aac-4fbf-badc-8f28a4947e03.png]({{site.baseurl}}/media/2024/07/image-7567326f-3aac-4fbf-badc-8f28a4947e03.png) 



# 参考ドキュメント

[セルフホステッドランナーの構築手順](https://learn.microsoft.com/ja-jp/azure/container-apps/tutorial-ci-cd-runners-jobs?tabs=bash&pivots=container-apps-jobs-self-hosted-ci-cd-github-actions#deploy-a-self-hosted-runner-as-a-job)

[KEDA | Github Runner Scalerについて](https://keda.sh/docs/2.13/scalers/github-runner/#trigger-specification)
<br>
<br>

---

<br>
<br>

2024年 06 月 19 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>