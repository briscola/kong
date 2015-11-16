![Alt text](https://getkong.org/assets/images/branding.svg)
## kongの利用方法

APIマーケットプレイスとして有名な[mashape](https://www.mashape.com/)が開発しているオープンソースソフトウェア版のAPI Managementである[kong](https://getkong.org/)の利用方法をまとめます。

ここではkong設定ファイル（/etc/kong/kong.yml）のproxy_portをデフォルトの8000から80に変更し、kongに登録したAPIをhttpでアクセスできるように設定しています。

また、コマンド例に出てくる[jqコマンド](https://stedolan.github.io/jq/)は出力されるjson文字列を整形するコマンドです。


### API関連

バックエンドのAPIをkong経由でアクセス可能にするための定義を行う。kongではAPIにアクセスしたクライアントのDNSアドレスによってバックエンドAPIを指定する方法とAPIのURIのパスに従ってバックエンドAPIを指定する方法があり、ここではAPIのURIのパスに従ってバックエンドAPIを指定する方法を採用する。詳細は[kongドキュメント](https://getkong.org/docs/0.5.x/admin-api)を参照。


#### APIの登録
詳細は[kongドキュメント](https://getkong.org/docs/0.5.x/admin-api/#add-api)を参照
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


#### APIの更新(URIパスを /mockbin -> /mockbin2 に変更します)
詳細は[kongドキュメント](https://getkong.org/docs/0.5.x/admin-api/#update-api)を参照
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


#### APIへのアクセス試験(URIパスが変更されています)
```
$ curl -s -X GET http://localhost:8000/mockbin2 | jq .
[
  {
      "color": "red",
          "value": "#f00"
	    },
...
```


#### APIのURIパスを元に戻します(/mockbin2 -> /mockbin)
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

登録されたAPIに認証方式やセキュリティ設定、流量制御を設定するには、APIにこれらのプラグインを追加する形で設定します。設定ステップは下記の通りです。

1. 登録されたAPIに認証プラグインを追加します。認証プラグインは複数設定することが可能ですが、同一APIで複数の認証が求められ、ユーザーの混乱を避けるため、一つのAPIに一つの認証プラグインに限定することをお勧めします。
2. このままだと認証キーを発行したユーザーはすべてのAPIにアクセス可能になるため、APIにアクセスコントロール(ACL)のプラグインを追加します。ACLプラグイン登録時にグループ名を設定します。ユーザーがこのグループに所属している場合のみ、ユーザーが発行した認証キーを使ってグループの所属APIにアクセスすることが可能です。この時、GET/POST/PUT/DELETEの区別をしないため、注意が必要です(元のAPIがGETとDELETEで同じURIを使っている場合、参照も削除も可能になってしまうため、API設計時に考慮が必要です)。
3. 開発者としてユーザーを登録し、認証キーを発行します。また、アクセスを許可するAPIに登録したグループをユーザーのグループとして設定します。
4. セキュリティ設定および流量制御のプラグインについては、APIに追加する際、ユーザーIDを指定することが可能です。これによって、どのユーザーについて制御を行うかを指定します。


#### 認証プラグイン

ここでは、基本認証プラグイン、キー認証プラグイン、OAuth2認証プラグインについて説明します。認証プラグインはAPIに追加され、開発者にキーが発行されると利用可能になります。一つのAPIにこの複数の認証プラグインを追加するとすべての認証をクリアしないとAPIにアクセスできずユーザーの混乱を招くため、一つのAPIには一つの認証プラグインに限定することをお勧めします。


##### 基本認証プラグイン追加
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
※ Basic認証例のみを説明するため削除します。
```
$ curl -s -X DELETE http://localhost:8001/apis/mockbin/plugins/adbf776a-75b4-4d78-cdac-b4f672976c68
```


#### ACLプラグイン登録

APIに認証プラグインを追加しただけでは、認証キーを発行したユーザーはすべてのAPIにアクセス可能になります。特定のユーザーが特定のAPIにのみアクセス可能にするために、APIにアクセスコントロール(ACL)のプラグインを追加します。ACLプラグインは、登録時にグループ名を設定します(コマンドの引数で"config.whitelist"に指定する文字列がグループ名になります)。ユーザーがこのグループに所属している場合のみ、ユーザーが発行した認証キーを使ってグループの所属APIにアクセスすることが可能になります。この時、GET/POST/PUT/DELETEの区別をしないため、注意が必要です(元のAPIがGETとDELETEで同じURIを使っている場合、参照も削除も可能になってしまうため、API設計時に考慮が必要です)。

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

#### ユーザーの登録

開発者としてユーザーを登録し、認証キーを発行します。また、アクセスを許可するAPIに登録したグループをユーザーのグループとして設定します。


##### ユーザーの登録
ユーザ名user1で作成
```
$ curl -s -X POST http://localhost:8001/consumers -d "username=user1" | jq .
{
  "username": "user1",
  "created_at": 1447671543000,
  "id": "a7a95439-481a-422c-c2d2-f590ff847a4d"
}
```

##### アクセスを許可するグループの設定
ユーザuser1にmockbinグループを追加
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
ユーザuser1のBasic認証用ユーザ名(user1)とパスワード(password)を発行
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
```
$ curl -s -X GET http://localhost/mockbin | jq .
{
  "message": "Unauthorized"
}
```

##### APIにアクセス(認証あり)
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


##### CORS設定
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

##### IP Restriction設定
```
$ curl -X POST http://localhost:8001/apis/mockbin/plugins -d "name=ip-restriction&config.whitelist=0.0.0.0/255"
```

##### SSL設定 [鍵の作り方等はこちらを参照](https://getkong.org/plugins/ssl/)
```
$ curl -X POST http://localhost:8001/apis/mockbin/plugins -d "name=ssl&config.cert=/etc/kong/server.crt&config.key=/etc/kong/server.key&config.only_https=true"
```


##### Request Size Limiting設定
```
$ curl -X POST http://localhost:8001/apis/mockbin/plugins -d "name=request-size-limiting"
```

##### Rate Limiting設定
```
$ curl -X POST http://localhost:8001/apis/mockbin/plugins -d "name=rate-limiting&config.second=1"
```


#### HTTP Log設定
```
$ curl -X POST http://localhost:8001/apis/mockbin/plugins -d "name=http-log&config.http_endpoint=http://localhost:3000/kmonitor"
```







プラグインのリスト(プラグインの更新・削除のためにidが必要なため実行)
```
$ curl -s -X GET http://localhost:8001/apis/mockbin/plugins | jq .
{
  "data": [
    {
      "api_id": "dd8bcf0e-f9f0-4d16-cd95-bf153b5a2f66",
      "id": "cf74fb66-81ea-47e1-c7eb-f1477d337333",
      "created_at": 1447155759000,
      "enabled": true,
      "name": "rate-limiting",
      "config": {
        "second": 1
      }
    },
    {
      "api_id": "dd8bcf0e-f9f0-4d16-cd95-bf153b5a2f66",
      "id": "388dcec1-ae3e-4e33-ce89-d552461fae01",
      "created_at": 1447212689000,
      "enabled": true,
      "name": "acl",
      "config": {
      "whitelist": [
        "mockbin"
      ]
    }
  },
  {
    "api_id": "dd8bcf0e-f9f0-4d16-cd95-bf153b5a2f66",
    "id": "dd73687e-c55f-471b-c792-a33c70f8751a",
    "created_at": 1447153680000,
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
      "provision_key": "608de92880934cffccddd18d526d4ab9"
    }
  },
  {
    "api_id": "dd8bcf0e-f9f0-4d16-cd95-bf153b5a2f66",
    "id": "c5b63cae-0ec3-4204-c257-884061d93e30",
    "created_at": 1447154332000,
    "enabled": true,
    "name": "request-size-limiting",
    "config": {
      "allowed_payload_size": 128
    }
  },
  {
    "api_id": "dd8bcf0e-f9f0-4d16-cd95-bf153b5a2f66",
    "id": "7160ceaf-86e7-4568-c584-8e5c8734c216",
    "created_at": 1447154396000,
    "enabled": true,
    "name": "http-log",
    "config": {
      "method": "POST",
      "http_endpoint": "http://localhost:3000/kmonitor",
      "timeout": 10000,
      "keepalive": 60000
    }
  },
  {
    "api_id": "dd8bcf0e-f9f0-4d16-cd95-bf153b5a2f66",
    "id": "285a4543-7a2b-4fea-cf1c-1be58490dcc2",
    "created_at": 1447154535000,
    "enabled": true,
    "name": "ip-restriction",
    "config": {
      "_whitelist_cache": [
        [
          0,
          255
        ]
      ],
      "whitelist": [
        "0.0.0.0/24"
      ]
    }
  },
  {
    "api_id": "dd8bcf0e-f9f0-4d16-cd95-bf153b5a2f66",
    "id": "b40f977a-986e-4076-c0df-b555a635227d",
    "created_at": 1447153742000,
    "enabled": true,
    "name": "cors",
    "config": {
      "preflight_continue": false,
      "credentials": false
    }
  },
  {
    "api_id": "dd8bcf0e-f9f0-4d16-cd95-bf153b5a2f66",
    "id": "2bb33e2d-2079-4fad-cf8b-252a3659f10e",
    "created_at": 1447153598000,
    "enabled": true,
    "name": "key-auth",
    "config": {
      "key_names": [
        "apikey"
      ],
      "hide_credentials": false
    }
  },
  {
    "api_id": "dd8bcf0e-f9f0-4d16-cd95-bf153b5a2f66",
    "id": "51cf6990-df83-4e60-cead-c1f27d88cad0",
    "created_at": 1447153511000,
    "enabled": true,
    "name": "basic-auth",
    "config": {
      "hide_credentials": false
    }
  }
 ]
}
```


基本認証プラグイン更新(config.hide_credentials(認証キー情報を上位URLに渡さない)を修正できる)
```
$ curl -X PATCH http://localhost:8001/apis/mockbin/plugins/51cf6990-df83-4e60-cead-c1f27d88cad0 -d "config.hide_credentials=true"
```


キー認証プラグイン更新(config.key_names(apikeyというデフォルト名を修正,複数設定可能)、config.hide_credentials(認証キー情報を上位URLに渡さない)を修正できる)
```
$ curl -X PATCH http://localhost:8001/apis/mockbin/plugins/2bb33e2d-2079-4fad-cf8b-252a3659f10e -d "config.hide_credentials=true&config.key_names=apikey,client_id"
```


OAuth2認証プラグイン更新(たくさんあるのであとまわし)


CORS情報更新(config.origin(Access-Control-Allow-Originの値),config.methods(Access-Control-Allow-Methodsの値(GET,POSTなど)),)
