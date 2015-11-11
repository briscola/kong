![Alt text](https://getkong.org/assets/images/branding.svg)
## kongの利用方法

APIマーケットプレイスとして有名な[mashape](https://www.mashape.com/)が開発しているオープンソースソフトウェア版のAPI Managementであるkongの利用方法をまとめます。


- APIの登録 [kong docs](https://getkong.org/docs/0.5.x/admin-api/#add-api)
```
$ curl -i -X POST http://localhost:8001/apis -d "name=mockbin&upstream_url=http://mockbin.com/bin/800a818b-5fb6-40d4-a342-75a1fb8599db&request_path=mockbin&strip_request_path=true"
```
strip_request_path=trueになっているので、<http://localhost:8000/mockbin>とアクセスすると上位URLにはURIから/mockbinがはずれて<http://mockbin.com/bin/800a818b-5fb6-40d4-a342-75a1fb8599db>にアクセスする。これを入れないと<http://mockbin.com/mockbin/bin/800a818b-5fb6-40d4-a342-75a1fb8599db>にアクセスしてしまう。


APIへのアクセス試験
```
$ curl -i -X GET http://localhost:8000/mockbin
```


[APIの更新](https://getkong.org/docs/0.5.x/admin-api/#update-api)
```
$ curl -i -X PATCH http://localhost:8001/apis/mockbin -d "name=mockbin&upstream_url=http://mockbin.com/bin/800a818b-5fb6-40d4-a342-75a1fb8599db&request_path=mockbin2&strip_request_path=true"
```


APIへのアクセス試験
```
$ curl -i -X GET http://localhost:8000/mockbin2
```


API名を元に戻す
```
$ curl -i -X PATCH http://localhost:8001/apis/mockbin -d "name=mockbin&upstream_url=http://mockbin.com/bin/800a818b-5fb6-40d4-a342-75a1fb8599db&request_path=mockbin&strip_request_path=true"
```


基本認証プラグイン追加
```
$ curl -i -X POST http://localhost:8001/apis/mockbin/plugins -d "name=basic-auth"
```


キー認証プラグイン追加
```
$ curl -i -X POST http://localhost:8001/apis/mockbin/plugins -d "name=key-auth"
```


OAuth2認証プラグイン追加
```
$ curl -i -X POST http://localhost:8001/apis/mockbin/plugins -d "name=oauth2"
```


CORS設定
```
$ curl -i -X POST http://localhost:8001/apis/mockbin/plugins -d "name=cors"
```

Request Size Limiting設定
```
$ curl -i -X POST http://localhost:8001/apis/mockbin/plugins -d "name=request-size-limiting"
```


HTTP Log設定
```
$ curl -i -X POST http://localhost:8001/apis/mockbin/plugins -d "name=http-log&config.http_endpoint=http://localhost:3000/kmonitor"
```


IP Restriction設定
```
$ curl -i -X POST http://localhost:8001/apis/mockbin/plugins -d "name=ip-restriction&config.whitelist=0.0.0.0/24"
```


SSL設定 [鍵の作り方等はこちらを参照](https://getkong.org/plugins/ssl/)
```
$ curl -i -X POST http://localhost:8001/apis/mockbin/plugins -d "name=ssl&config.cert=/etc/kong/server.crt&config.key=/etc/kong/server.key&config.only_https=true"
```


Rate Limiting設定
```
$ curl -i -X POST http://localhost:8001/apis/mockbin/plugins -d "name=rate-limiting&config.second=1"
```


ACL登録
```
$ curl -i -X POST http://localhost:8001/apis/mockbin/plugins -d "name=acl&config.whitelist=mockbin"
```


プラグインのリスト(プラグインの更新・削除のためにidが必要なため実行)
```
$ curl -i -X POST http://localhost:8001/apis/mockbin/plugins
Response Body
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
$ curl -i -X PATCH http://localhost:8001/apis/mockbin/plugins/51cf6990-df83-4e60-cead-c1f27d88cad0 -d "config.hide_credentials=true"
```


キー認証プラグイン更新(config.key_names(apikeyというデフォルト名を修正,複数設定可能)、config.hide_credentials(認証キー情報を上位URLに渡さない)を修正できる)
```
$ curl -i -X PATCH http://localhost:8001/apis/mockbin/plugins/2bb33e2d-2079-4fad-cf8b-252a3659f10e -d "config.hide_credentials=true&config.key_names=apikey,client_id"
```


OAuth2認証プラグイン更新(たくさんあるのであとまわし)


CORS情報更新(config.origin(Access-Control-Allow-Originの値),config.methods(Access-Control-Allow-Methodsの値(GET,POSTなど)),)
