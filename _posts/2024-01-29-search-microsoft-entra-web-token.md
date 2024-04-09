---
title: "Microsoft Entra 認証でログインする Web アプリケーションで、ユーザーの権限に応じて AI Search を検索させたい"
author_name: "Takumi Nagaya"
tags:
    - "AI Search"
---

# 質問
Microsoft Entra 認証でログインする Web アプリケーションがあり、AI Search のインデックスを検索させたいです。  
ただし、ユーザーが AI Search のインデックスを検索する権限がない場合は、検索できないようにしたいです。

# 回答
端的に申し上げますと、Web アプリケーションが利用する Microsoft Entra ID アプリケーションの設定で、AI Search サービスの API に対する「委任されたアクセス許可」を追加いただき、  
認証スコープを `https://search.azure.com/user_impersonation` としてトークンを取得すると、ユーザーの権限に応じて AI Search のインデックスを検索させることができます。

## AI Search 側で必要な前提条件
アプリケーションの設定前に、AI Search 側で必要な対応を記載いたします。

- [ロールの割り当て](https://learn.microsoft.com/ja-jp/azure/search/search-security-rbac?tabs=config-svc-portal%2Croles-portal%2Ctest-portal%2Ccustom-role-portal%2Cdisable-keys-portal#assign-roles) のドキュメントを参考に、インデックスもしくは AI Search スコープで、ユーザーに対して権限を付与します。
  - AI Search のインデックスを検索するための組み込みロールとして「検索インデックス データ閲覧者」がございます。アプリケーション経由で AI Search を検索する場合はこちらのロールが便利と存じます。
- [データ プレーンにロールベースのアクセスを構成する](https://learn.microsoft.com/ja-jp/azure/search/search-security-rbac?tabs=config-svc-portal%2Croles-portal%2Ctest-portal%2Ccustom-role-portal%2Cdisable-keys-portal#configure-role-based-access-for-data-plane) のドキュメントを参考に、AI Search でロールベースのアクセス制御を有効にします。

## Web アプリケーション および Microsoft Entra ID アプリケーションの設定

### 手順 1. Microsoft Entra ID 認証でログインする Web アプリケーションを用意します。
サンプルとして、[Microsoft ID プラットフォームのコード サンプル](https://learn.microsoft.com/ja-jp/entra/identity-platform/sample-v2-code?tabs=apptype#samples-and-guides) にある React アプリケーション「[Azure REST API と Azure Storage を呼び出す](https://github.com/Azure-Samples/ms-identity-javascript-react-tutorial/tree/main/2-Authorization-I/2-call-arm)」を使用します。  

### 手順 2. Microsoft Entra ID アプリケーションを設定します。
手順 1. のサンプルをご利用の場合は、[Step 3: Register the sample application(s) in your tenant](https://github.com/Azure-Samples/ms-identity-javascript-react-tutorial/tree/main/2-Authorization-I/2-call-arm#step-3-register-the-sample-applications-in-your-tenant) の手順に従います。  
以下の 1 ~ 6 の手順は、[Step 3: Register the sample application(s) in your tenant](https://github.com/Azure-Samples/ms-identity-javascript-react-tutorial/tree/main/2-Authorization-I/2-call-arm#step-3-register-the-sample-applications-in-your-tenant) に基づく内容となりますが、Microsoft Entra ID アプリケーションの設定方法を参考までに記載いたします。  
<br/>
**既に Microsoft Entra ID アプリケーションを構成いただいている場合は、以下の 4. 5. の「Azure Cognitive Search」の API へのアクセス許可設定のみ実施いただければと存じます。**

#### 1. Azure Portal で Microsoft Entra ID を開き、「アプリの登録」から「新規登録」をクリックします。
 
![image-7b9f23e3-2518-4cf7-8cdd-17b03244f091.png]({{site.baseurl}}/media/2024/01/image-7b9f23e3-2518-4cf7-8cdd-17b03244f091.png)

#### 2. アプリケーションの表示名を入力し、「この組織ディレクトリのみに含まれるアカウント」を選択し、アプリケーションを作成します。

![image-a470d374-8ae2-4754-b934-950d2e1e9d8d.png]({{site.baseurl}}/media/2024/01/image-a470d374-8ae2-4754-b934-950d2e1e9d8d.png)

#### 3. 「認証」ブレードより、「プラットフォームを追加」をクリックします。サンプルはシングルページアプリケーションなので、「シングルページアプリケーション」を選択し、リダイレクト URI に以下を設定し保存します。

- `http://localhost:3000/redirect`
- `http://localhost:3000/`

![image-d8dbbf12-41c9-442d-9e1b-20aa44b32d80.png]({{site.baseurl}}/media/2024/01/image-d8dbbf12-41c9-442d-9e1b-20aa44b32d80.png)

#### 4. 「API のアクセス許可」ブレードを開き、「アクセス許可の追加」から「Azure Cognitive Search」という名前の API を探します。

![image-d7829107-6711-4b9b-975c-7575826262a0.png]({{site.baseurl}}/media/2024/01/image-d7829107-6711-4b9b-975c-7575826262a0.png)

#### 5. 「Azure Cognitive Search」をクリックし、「委任されたアクセス許可」を選択し、「user_impersonation」にチェックを入れアクセス許可を追加します。

![image-e83f389b-f7ff-45d5-a425-9d78b59bd563.png]({{site.baseurl}}/media/2024/01/image-e83f389b-f7ff-45d5-a425-9d78b59bd563.png)

API のアクセス許可が以下のようになっていれば問題ありません。

![image-e23ea208-d76a-4adf-b4dd-5f05997ff018.png]({{site.baseurl}}/media/2024/01/image-e23ea208-d76a-4adf-b4dd-5f05997ff018.png)

#### 6. 「トークン構成」のブレードより、「オプションの要求の追加」をクリックし、トークンの種類 「ID」で「acct」にチェックを入れて保存します。

![image-0c092b76-47cf-4290-a3da-e428d41a5253.png]({{site.baseurl}}/media/2024/01/image-0c092b76-47cf-4290-a3da-e428d41a5253.png)

### 手順 3. Web アプリケーションの構成
恐れ入りますが、以下の内容はあくまでサンプルアプリケーションをベースにしたものになりますので、お客様のアプリケーションに応じて適宜検証の上開発いただけますと幸いです。

#### 1. 手順 2 で作成した Microsoft Entra ID アプリケーションの概要ページより、アプリケーション ID とディレクトリ ID を取得します。

![image-122d9417-c126-49ec-b821-5c9db7162739.png]({{site.baseurl}}/media/2024/01/image-122d9417-c126-49ec-b821-5c9db7162739.png)

#### 2. Web アプリケーションの設定ファイルにて、アプリケーション ID とディレクトリ ID を設定します。サンプルでは [SPA/src/authConfig.js#L16](https://github.com/Azure-Samples/ms-identity-javascript-react-tutorial/blob/main/2-Authorization-I/2-call-arm/SPA/src/authConfig.js#L16) 内の以下の部分に設定します。

```javascript
export const msalConfig = {
    auth: {
        clientId: 'Enter_the_Application_Id_Here', // This is the ONLY mandatory field that you need to supply.
        authority: 'https://login.microsoftonline.com/Enter_the_Tenant_Info_Here', // Defaults to "https://login.microsoftonline.com/common"
```

#### 3. AI Search 用のトークンを取得するため、[SPA/src/authConfig.js#L71](https://github.com/Azure-Samples/ms-identity-javascript-react-tutorial/blob/main/2-Authorization-I/2-call-arm/SPA/src/authConfig.js#L71) に以下のようにスコープの定義を追加します。

```javascript
    armBlobStorage: {
        scopes: ['https://storage.azure.com/user_impersonation'],
    },
    armSearch: {
        scopes: ['https://search.azure.com/user_impersonation'],
    }
```

#### 4. [SPA/src/authConfig.js](https://github.com/Azure-Samples/ms-identity-javascript-react-tutorial/blob/main/2-Authorization-I/2-call-arm/SPA/src/authConfig.js) に接続先の AI Search サービス名と、インデックス名を指定します。

```javascript
 export const searchInformation = {
    serviceName: 'search-service-name',
    indexName: 'hotels-sample-index'
};
```

#### 5. サンプルの [SPA/src/pages/BlobStorage.jsx](https://github.com/Azure-Samples/ms-identity-javascript-react-tutorial/blob/main/2-Authorization-I/2-call-arm/SPA/src/pages/BlobStorage.jsx) と同様に、`Search.jsx` を用意し、`request` 定数の `scopes` に 3. で作成した `armSearch` の `scopes` を指定します。

```javascript
export const Search = () => {
// 中略

    const request = {
        scopes: protectedResources.armSearch.scopes,
        account: account,
    };

    const { login, result, error } = useMsalAuthentication(InteractionType.Popup, {
        ...request,
        redirectUri: '/redirect',
    });
```

#### 6. [SPA/src/azureManagement.js](https://github.com/Azure-Samples/ms-identity-javascript-react-tutorial/blob/main/2-Authorization-I/2-call-arm/SPA/src/azureManagement.js#L22) にある、Blob Storage クライアントの初期化処理と同様に、以下のように AI Search の検索用クライアントを初期化する処理を記載します。ここで 4. で設定した値を使用します。

```javascript
export const getSearchClient = async (accessToken) => {

  const credential = new StaticTokenCredential({
      token: accessToken,
      expiresOnTimestamp: accessToken.exp,
  });

  const client = new SearchClient(`https://${searchInformation.serviceName}.search.windows.net`, searchInformation.indexName, credential);
  return client;
};
```

#### 7. 検索結果を取得するコードを記載します。

```javascript
export const Search = () => {
    const { instance } = useMsal();
    const [searchWord, setsearchWord] = useState(null);

// 中略
    const handleSubmit = async (e) => {
        e.preventDefault();
        try {
            const client = await getSearchClient(result.accessToken);
            const searchResults = await client.search(searchWord);
            let r = "";
            for await (const result of searchResults.results) {
                r += result.document.HotelName;
            }
            setMessage(r);
            setShowMessage(true);

// 中略
    return (
        <Container>
            <Row>
                <Form onSubmit={handleSubmit}>
                    <input id="searchWord" name="searchWord" placeholder="search word" type="text" onChange={handleChangesearchWord} />
                    <Button type="submit">Submit</Button>
                </Form>
            </Row>
```

例えば以下のようにフォームに記載した内容に基づいて検索し、検索結果を取得することができます。<br/>

![image-222496ff-aa66-4b58-a445-165a436ce1df.png]({{site.baseurl}}/media/2024/01/image-222496ff-aa66-4b58-a445-165a436ce1df.png)
# 参考ドキュメント
- [Microsoft ID プラットフォームのコード サンプル](https://learn.microsoft.com/ja-jp/entra/identity-platform/sample-v2-code?tabs=apptype#samples-and-guides)

<br>
<br>

---

<br>
<br>

2024 年 04 月 09 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>