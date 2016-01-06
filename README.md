![Alt text](https://getkong.org/assets/images/branding.svg)
## kongの利用方法

APIマーケットプレイスとして有名な[mashape](https://www.mashape.com/)が開発しているオープンソースソフトウェア版のAPI Managementである[kong](https://getkong.org/)の利用方法をまとめます。

kongの各種設定はAPIを使って行います。現時点でkongは設定機能に認証のしくみを持たないため、誰でも管理者として設定情報の新規登録、更新、参照、削除ができてしまうため、kongのベースとなっているWebサーバであるnginxを設定し、リモートからの接続を禁止したり、ベーシック認証を設定するなどの対応が必要です。

本書では、kongを設定するユーザーを「API管理者」と「開発者」に分け、それぞれがどう設定を行うとうまく機能するかをまとめています。

また本書では、kong設定ファイル（/etc/kong/kong.yml）のproxy_portをデフォルトの8000から80に変更し、kongに登録したAPIをhttpでアクセスできるように設定しており、それに従ってコマンドサンプルを記述しています。

コマンド例に出てくる[jqコマンド](https://stedolan.github.io/jq/)は出力されるjson文字列を整形するコマンドです。


### API関連

API管理者は、アップストリームのAPIをkong経由でアクセス可能にするための定義を行います。kongには、APIにアクセスしたクライアントのDNSアドレスによってアップストリームAPIを指定する方法とAPIのURIのパスに従ってアップストリームAPIを指定する方法がありますが、本書ではAPIのURIのパスに従ってアップストリームAPIを指定する方法を採用します（DNSアドレスによって切り替える方法を採用するケースは少ないと考えるため）。詳細は[kongドキュメント](https://getkong.org/docs/0.5.x/admin-api)を参照してください。


#### APIの登録
API管理者は、アップストリームAPIを指定してkongにAPIを登録します。nameはAPI名、request_pathはkongに設定されるAPIのURIパスを意味します。これらは一意の値である必要があるため、既存の値を登録しようとするとエラーになります。詳細は[kongドキュメント](https://getkong.org/docs/0.5.x/admin-api/#add-api)を参照してください。
```
$ curl -s -X POST http://localhost:8001/apis -d "name=mockbin&upstream_url=http://mockbin.com/bin/800a818b-5fb6-40d4-a342-75a1fb8599db&request_path=mockbin&strip_request_path=true" | jq .
{
  "upstream_url": "http://mockbin.com/bin/800a818b-5fb6-40d4-a342-75a1fb8599db",
  "request_path": "/mockbin",
  "id": "d2b2a575-b9a9-4f6d-cfae-289460a38d7c",
  "created_at": 1447658806000,
  "strip_request_path": true,
  "name": "mockbin"
}
```

※ strip_request_path=trueになっているため、<http://localhost:8000/mockbin>とアクセスすると上位URLにはURIから/mockbinがはずれて<http://mockbin.com/bin/800a818b-5fb6-40d4-a342-75a1fb8599db>にアクセスします。これを入れないと<http://mockbin.com/mockbin/bin/800a818b-5fb6-40d4-a342-75a1fb8599db>にアクセスします。

※ レスポンスボディのcreated_atは1970/01/01からのミリ秒を表します。


#### APIへのアクセス試験
```
$ curl -s -X GET http://localhost/mockbin | jq .
[
  {
      "color": "red",
          "value": "#f00"
	    },
...
```


#### APIの更新
(URIパスを /mockbin -> /mockbin2 に変更します)

API管理者は、登録したAPIの情報を更新します。詳細は[kongドキュメント](https://getkong.org/docs/0.5.x/admin-api/#update-api)を参照してください。
```
$ curl -s -X PATCH http://localhost:8001/apis/mockbin -d "name=mockbin&upstream_url=http://mockbin.com/bin/800a818b-5fb6-40d4-a342-75a1fb8599db&request_path=mockbin2&strip_request_path=true" | jq .
{
  "upstream_url": "http://mockbin.com/bin/800a818b-5fb6-40d4-a342-75a1fb8599db",
  "request_path": "/mockbin2",
  "id": "d2b2a575-b9a9-4f6d-cfae-289460a38d7c",
  "strip_request_path": true,
  "name": "mockbin",
  "created_at": 1447658806000
}
```


#### APIへのアクセス試験
(URIパスが変更されています)
```
$ curl -s -X GET http://localhost:8000/mockbin2 | jq .
[
  {
      "color": "red",
          "value": "#f00"
	    },
...
```


#### APIのURIパスを元に戻します
(/mockbin2 -> /mockbin)
```
$ curl -s -X PATCH http://localhost:8001/apis/mockbin -d "name=mockbin&upstream_url=http://mockbin.com/bin/800a818b-5fb6-40d4-a342-75a1fb8599db&request_path=mockbin&strip_request_path=true" | jq .
{
  "upstream_url": "http://mockbin.com/bin/800a818b-5fb6-40d4-a342-75a1fb8599db",
  "request_path": "/mockbin",
  "id": "d2b2a575-b9a9-4f6d-cfae-289460a38d7c",
  "strip_request_path": true,
  "name": "mockbin",
  "created_at": 1447658806000
}
```


### プラグイン関連

kongではマイクロサービスの設計方針を採用しており、小さな機能をプラグインに分け、必要に応じてプラグインを有効化することで機能を実現しています。ここでは、[認証プラグイン](https://getkong.org/plugins/#authentication)、[セキュリティ・プラグイン](https://getkong.org/plugins/#security)、[流量制御プラグイン](https://getkong.org/plugins/#traffic-control)、[ロギング・プラグイン](https://getkong.org/plugins/#logging)を利用します。

APIに認証やセキュリティ、流量制御を設定するには、APIにこれらのプラグインを追加します。設定ステップは下記の通りです。

1. API管理者は、APIに「認証プラグイン」を追加し、許可したユーザーだけがAPIにアクセス可能になるように設定します。認証プラグインは、一つのAPIに複数種類を追加することが可能ですが、同一APIで複数種類の認証が求められるとAPI利用者を混乱させるため、一つのAPIに設定する認証プラグインは一つにすることをお勧めします。
2. 1の設定だけでは、認証キーを持つユーザーは誰でもこのAPIにアクセス可能になってしまいます。これを回避するために、API管理者はAPIにアクセスコントロール(ACL)のプラグインを追加します。ACLプラグインをAPIに登録する際、whitelistパラメータにグループ名を設定ることで、このグループに所属しているユーザーだけが対象のAPIにアクセス可能になるよう設定することができます。API管理者は、開発者としてユーザーを作成し、アクセスを許可するAPIのグループに追加することでACLを管理することができます。
3. API管理者は、APIに「セキュリティ・プラグイン」を追加し、CORS設定（クロスサイトスクリプティングの制限回避のためのサーバー側の設定）やIP制限の設定（アクセス可能もしくは不可能なIPアドレスを指定）を行います。CORS設定以外は、プラグイン追加時にユーザーIDを指定し、どのユーザーについて制御を行うかを指定することができます。
4. API管理者は、APIに「流量制御プラグイン」を追加し、指定時間内のHTTPリクエスト数制限の設定やリクエストサイズ制限の設定（DOS攻撃に対応するためリクエストデータのサイズ量を制限）を行います。プラグイン追加時にはユーザーIDを指定し、どのユーザーについて制御を行うかを指定します。
5. 開発者は、自らの認証キーを発行し、API管理者に許可されたAPIにアクセスを行います。


#### 認証プラグイン

ここでは、基本認証プラグイン、キー認証プラグイン、OAuth2認証プラグインについて説明します。


##### 基本認証プラグイン追加

詳細は[こちら](https://getkong.org/plugins/#authentication)を参照。

```
$ curl -s -X POST http://localhost:8001/apis/mockbin/plugins -d "name=basic-auth" | jq .
{
  "api_id": "d2b2a575-b9a9-4f6d-cfae-289460a38d7c",
  "id": "25cdc2dd-d37e-4856-c83e-736e009342a9",
  "created_at": 1447662697000,
  "enabled": true,
  "name": "basic-auth",
  "config": {
    "hide_credentials": false
  }
}
```

##### 基本認証プラグイン更新
config.hide_credentials(認証キー情報を上位URLに渡さない)を修正することができます。
```
$ curl -X PATCH http://localhost:8001/apis/mockbin/plugins/51cf6990-df83-4e60-cead-c1f27d88cad0 -d "config.hide_credentials=true"
```

##### キー認証プラグイン追加
```
$ curl -s -X POST http://localhost:8001/apis/mockbin/plugins -d "name=key-auth" | jq .
{
  "api_id": "d2b2a575-b9a9-4f6d-cfae-289460a38d7c",
  "id": "90edc438-e6be-4a75-cabe-11f974aff431",
  "created_at": 1447662748000,
  "enabled": true,
  "name": "key-auth",
  "config": {
    "key_names": [
      "apikey"
    ],
    "hide_credentials": false
  }
}
```

##### キー認証プラグイン更新
config.key_names(apikeyというデフォルト名を修正,複数設定可能)、config.hide_credentials(認証キー情報を上位URLに渡さない)を修正できます。
```
$ curl -X PATCH http://localhost:8001/apis/mockbin/plugins/2bb33e2d-2079-4fad-cf8b-252a3659f10e -d "config.hide_credentials=true&config.key_names=apikey,client_id"
```

※ Basic認証例のみを説明するため削除します。
```
$ curl -s -X DELETE http://localhost:8001/apis/mockbin/plugins/90edc438-e6be-4a75-cabe-11f974aff431
```

##### OAuth2認証プラグイン追加
```
$ curl -s -X POST http://localhost:8001/apis/mockbin/plugins -d "name=oauth2" | jq .
{
  "api_id": "d2b2a575-b9a9-4f6d-cfae-289460a38d7c",
  "id": "adbf776a-75b4-4d78-cdac-b4f672976c68",
  "created_at": 1447662808000,
  "enabled": true,
  "name": "oauth2",
  "config": {
    "hide_credentials": false,
    "mandatory_scope": false,
    "token_expiration": 7200,
    "enable_password_grant": false,
    "enable_implicit_grant": false,
    "enable_authorization_code": true,
    "enable_client_credentials": false,
    "provision_key": "cb6559d8dce84ea0ce61804edd29b09c"
  }
}
```

##### OAuth2認証プラグイン更新
たくさんあるのであとまわし

※ Basic認証例のみを説明するため削除します。
```
$ curl -s -X DELETE http://localhost:8001/apis/mockbin/plugins/adbf776a-75b4-4d78-cdac-b4f672976c68
```


#### ACLプラグイン登録

API管理者は、特定のユーザーが特定のAPIにのみアクセス可能にするために、APIにアクセスコントロール(ACL)のプラグインを追加します。ACLプラグインは、登録時にグループ名を設定します(コマンドの引数で"config.whitelist"に指定する文字列がグループ名になります)。ユーザーがこのグループに所属している場合のみ、ユーザーが発行した認証キーを使ってグループの所属APIにアクセスすることが可能になります。ただし、GET/POST/PUT/DELETEの区別をしないため、注意が必要です(例えば、アップストリームAPIがGETとDELETEで同じURIを使っている場合、参照も削除も可能になってしまいます)。

```
$ curl -s -X POST http://localhost:8001/apis/mockbin/plugins -d "name=acl&config.whitelist=mockbin" | jq .
{
  "api_id": "d2b2a575-b9a9-4f6d-cfae-289460a38d7c",
  "id": "363c9bcf-cfb8-4e3f-ce26-56190cd99f93",
  "created_at": 1447663720000,
  "enabled": true,
  "name": "acl",
  "config": {
    "whitelist": [
      "mockbin"
    ]
  }
}
```

#### ユーザーのAPIグループへの登録と認証キーの発行

API管理者は、開発者用にユーザーを作成し、アクセスを許可するAPIのグループに登録します。また、開発者は認証キーを発行し、API管理者に許可されたAPIへのアクセスを行います。


##### ユーザーの登録
API管理者は、開発者用にユーザー「user1」を作成します。
```
$ curl -s -X POST http://localhost:8001/consumers -d "username=user1" | jq .
{
  "username": "user1",
  "created_at": 1447671543000,
  "id": "a7a95439-481a-422c-c2d2-f590ff847a4d"
}
```

##### アクセスを許可するグループの設定
API管理者は、ユーザー「user1」にmockbinグループを追加します。複数のグループを追加する場合はカンマ(,)でつなげて設定します。
```
$ curl -s -X POST http://localhost:8001/consumers/user1/acls -d "group=mockbin" | jq .
{
  "id": "7de12a76-612b-4348-c381-8a6016e705f7",
  "consumer_id": "a7a95439-481a-422c-c2d2-f590ff847a4d",
  "created_at": 1447671732000,
  "group": "mockbin"
}	
```

##### 認証キーの発行
開発者は、ユーザー「user1」のBasic認証用ユーザー名(user1)とパスワード(password)を発行します。
```
$ curl -s -X POST http://localhost:8001/consumers/user1/basic-auth -d "username=user1&password=password" | jq .
{
  "password": "49850a5a4674b354e4a47421e6f850f2e4cd67e0",
  "consumer_id": "a7a95439-481a-422c-c2d2-f590ff847a4d",
  "id": "65af901e-180d-48de-c858-1c7dbfeb4999",
  "username": "user1",
  "created_at": 1447671914000
}
```

##### APIにアクセス(認証なし)
認証なしでAPIにアクセスすると失敗します。
```
$ curl -s -X GET http://localhost/mockbin | jq .
{
  "message": "Unauthorized"
}
```

##### APIにアクセス(認証あり)
Basic認証に成功するとAPIにアクセスできます。
```
$ curl -s -X GET http://localhost/mockbin --user user1:password | jq .
[
  {
    "color": "red",
    "value": "#f00"
  },
...
```

#### セキュリティ設定プラグイン

API管理者は、「セキュリティ・プラグイン」として「CORS設定」や「IP制限の設定」を行います。プラグイン追加時にはユーザーIDを指定し、どのユーザーについて制御を行うかを指定します。

##### CORS設定
CORS設定とは、クロスサイトスクリプティングの制限回避のためのサーバー側に設定するものです。これを設定すると、CORSを許可するブラウザであればJavaScriptが配布されたホスト以外から提供されるAPIにもアクセスが可能になります。
```
$ curl -s -X POST http://localhost:8001/apis/mockbin/plugins -d "name=cors" | jq .
{
  "api_id": "d2b2a575-b9a9-4f6d-cfae-289460a38d7c",
  "id": "a490fab1-2b8f-4a7e-ca51-d1acbe2ce575",
  "created_at": 1447663036000,
  "enabled": true,
  "name": "cors",
    "config": {
      "preflight_continue": false,
      "credentials": false
    }
  }
```

##### CORS情報更新
config.origin(Access-Control-Allow-Originの値),config.methods(Access-Control-Allow-Methodsの値(GET,POSTなど)),を修正できます。

##### IP制限の設定
IP制限の設定では、アクセス可能もしくは不可能なIPアドレスを指定します。プラグイン追加時にはユーザーIDを指定し、どのユーザーについて制御を行うかを指定します。
```
$ curl -X POST http://localhost:8001/apis/mockbin/plugins -d "name=ip-restriction&config.whitelist=0.0.0.0/255&consumer_id=a7a95439-481a-422c-c2d2-f590ff847a4d"
```

#### セキュリティ設定プラグイン

API管理者は、「流量制御プラグイン」として、「HTTPリクエスト数制限の設定」や「リクエストサイズ制限の設定」を行います。プラグイン追加時にはユーザーIDを指定し、どのユーザーについて制御を行うかを指定します。

##### リクエストサイズ制限の設定
リクエストサイズ制限の設定では、DOS攻撃に対応するためリクエストデータのサイズ量を制限します。プラグイン追加時にはユーザーIDを指定し、どのユーザーについて制御を行うかを指定します。
```
$ curl -X POST http://localhost:8001/apis/mockbin/plugins -d "name=request-size-limiting&consumer_id=a7a95439-481a-422c-c2d2-f590ff847a4d"
```

##### HTTPリクエスト数制限の設定
HTTPリクエスト数制限の設定では、単位時間当たりのHTTPリクエストの最大数の設定を行います。プラグイン追加時にはユーザーIDを指定し、どのユーザーについて制御を行うかを指定します。
```
$ curl -X POST http://localhost:8001/apis/mockbin/plugins -d "name=rate-limiting&config.second=1&consumer_id=a7a95439-481a-422c-c2d2-f590ff847a4d"
```
