# OAuth 2.0

OAuth 2.0 の認可コードフローに基づいた API が用意されています。

[The OAuth 2.0 Authorization Framework](http://openid-foundation-japan.github.io/rfc6749.ja.html)
[The OAuth 2.0 Authorization Framework: Bearer Token Usage（日本語）](http://openid-foundation-japan.github.io/rfc6750.ja.html)

OAuth 2.0 のフローで取得したアクセストークンは、目的の API リクエストの `Authorization` ヘッダーに Bearer トークンとして付与する形で使ってください。

```console
$ curl -H 'Authorization: Bearer ${アクセストークン}' https://api.stores.dev/xxx/xxx
```

## エンドポイント

https://api.id.stores.jp

- 認可エンドポイント `GET /oauth2/auth`
- トークンエンドポイント `POST /oauth2/token`
- トークンの無効化 `POST /oauth2/revoke`

## 認可エンドポイント `GET /oauth2/auth`

STORES ID の認可フローを開始し、認可コードを取得します。

| パラメータ    | 概要                                                                                            |
| :------------ | :---------------------------------------------------------------------------------------------- |
| response_type | `code` を指定                                                                                   |
| client_id     | アプリ登録時に取得したクライアント ID                                                           |
| redirect_uri  | アプリ登録時に指定したリダイレクト URL                                                          |
| scope         | スコープをスペース区切りで指定。有効な値の一覧は項目「[スコープ](#スコープ)」を参照してください |
| state         | CSRF 対策として検証用の値。アプリへのリダイレクト時にそのままパラメータとして返します           |
| prompt        | （省略可能） `login` を指定すると、STORES ID はユーザーに再認証を要求します                     |

以上のクエリパラメータ付きの URL は次のようになります。

```
https://api.id.stores.jp/oauth2/auth?client_id={client_id}&redirect_uri={redirect_uri}&response_type=code&scope={scope}&state={state}&prompt={prompt}
```

このエンドポイントへのアクセス時、ユーザーがログイン済みでない場合はログイン画面を表示します。

ショップへの指定の `scope` の操作をユーザーがアプリに認可した場合、以下の情報をクエリパラメータに付与して `redirect_url` にリダイレクトします。

| パラメータ | 概要                                                             |
| :--------- | :--------------------------------------------------------------- |
| code       | 認可コード。有効期限は 10 分                                     |
| scope      | ユーザーが認可した scope                                         |
| state      | 認可エンドポイントへのリクエストでアプリから受け取った検証用の値 |

リダイレクト先では必ず state の値の有効性を検証し、[CSRF 攻撃への対策](http://openid-foundation-japan.github.io/rfc6749.ja.html#CSRF)を行ってください。
例えば、認可エンドポイントに付与した state 値をセッションに紐付けておき、リダイレクト時の state 値との一致を確認します。

## トークンエンドポイント `POST /oauth2/token`

認可コードやリフレッシュトークンと、アクセストークンを交換します。

| パラメータ            | 概要                                                                            |
| :-------------------- | :------------------------------------------------------------------------------ |
| client_id             | アプリ登録時に取得したクライアント ID                                           |
| grant_type            | `authorization_code` を指定。リフレッシュトークンを使用する際は `refresh_token` |
| code                  | 認可エンドポイントで取得した認可コード                                          |
| redirect_uri          | アプリ登録時に指定したリダイレクト URL                                          |
| client_assertion_type | `urn:ietf:params:oauth:client-assertion-type:jwt-bearer` を指定                 |
| client_assertion      | クライアント認証のための情報を含む、署名済み JWT                                |

クライアント認証が必要です。
OpenID Connect のクライアント認証方式のひとつである `private_key_jwt` 方式をサポートしています。
https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#ClientAuthentication

`private_key_jwt` 方式では、 [JWK](https://openid-foundation-japan.github.io/rfc7517.ja.html) で表現されるキーペアを使ってデジタル署名した、JWT 形式の[アサーション](https://datatracker.ietf.org/doc/html/rfc7521)を含めてリクエストを送信します。

アサーションに指定するペイロードの例

```json
{
  "iss": "776b0ba3-c00f-4953-b840-cbca7010f5e8",
  "sub": "776b0ba3-c00f-4953-b840-cbca7010f5e8",
  "aud": "https://api.id.stores.jp/oauth2/token",
  "jti": "1654514191",
  "exp": 1654517791,
  "iat": 1654514191
}
```

署名アルゴリズムは ES256 にのみ対応しています。
署名用の秘密鍵は、[パートナー登録](development-partner-signup.md)が完了したら STORES が発行して JWK 形式でお渡しします。
ご希望があればご指定の JWK を登録することも可能ですのでお問い合わせください。

ペイロードを署名し、得られた JWT をトークンリクエストの `client_assertion` クエリパラメータに指定して送信してください。

リクエストの例

```console
$ curl -X POST \
-H 'Content-Type: application/x-www-form-urlencoded; charset=utf-8' \
--data-urlencode 'grant_type=authorization_code' \
--data-urlencode 'code={code}' \
--data-urlencode 'client_id={client_id}' \
--data-urlencode 'redirect_uri={redirect_uri}' \
--data-urlencode 'client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer' \
--data-urlencode 'client_assertion={client_assertion}' \
https://api.id.stores.jp/oauth2/token
```

アサーションに成功すれば、アクセストークンとリフレッシュトークンが発行され、レスポンスとして返却されます。

レスポンスの例

```json
{
  "access_token": "DXzQzbwWh7amWhMWdcK1_b_JioNulRyDlGWTvQIBK7s.fZvUA8ju4aoRsyYvfKi1c3GcKAwEjc2zBlRpy6NnqP0",
  "expires_in": 3600,
  "refresh_token": "qve0a_9YRxweyC3v_5-JtFt-Ak6-KUH2FB9t1UWeQng.8-Akk15jgYPC7db7BZnNxJjNlE8TsT0Ik5OcfIxvtWE",
  "scope": "retail.shop.read offline_access",
  "token_type": "bearer"
}
```

アクセストークンの有効期限は 1 時間、リフレッシュトークンの有効期限は 35 日です。

### リフレッシュトークンを利用してアクセストークンを再発行

`grant_type=refresh_token` を指定し、 `refresh_token` パラメータにリフレッシュトークンを入れてください。

```console
$ curl -X POST \
-H 'Content-Type: application/x-www-form-urlencoded; charset=utf-8' \
--data-urlencode "grant_type=refresh_token" \
--data-urlencode "refresh_token={refresh_token}" \
--data-urlencode "client_id={client_id}" \
--data-urlencode 'client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer' \
--data-urlencode 'client_assertion={client_assertion}' \
https://api.id.stores.jp/oauth2/token
```

新しいアクセストークンと、新しいリフレッシュトークンが再発行されます。
有効期限は再発行した時点から起算して、アクセストークンが 1 時間、リフレッシュトークンが 35 日となります。
古いリフレッシュトークンは無効化されます。

## トークンの無効化 `POST /oauth2/revoke`

[OAuth 2.0 Token Revocation](https://openid-foundation-japan.github.io/rfc7009.ja.html) をサポートしています。

トークンが不要になった場合などに無効化することで、万が一トークンが漏洩したときに有効な認可が残っている状態を防げます。

リクエストの例

```
$ curl -X POST \
-H 'Content-Type: application/x-www-form-urlencoded; charset=utf-8' \
--data-urlencode 'token={token}' \
--data-urlencode 'token_type_hint=refresh_token' \
--data-urlencode 'client_id={client_id}' \
--data-urlencode 'client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer' \
--data-urlencode 'client_assertion={client_assertion}' \
https://api.id.stores.jp/oauth2/revoke
```

アクセストークンを指定した場合、そのアクセストークンと、同じ認可を元とするリフレッシュトークンを無効にします。
リフレッシュトークンを指定した場合、そのリフレッシュトークンと、同じ認可を元とするアクセストークンを無効にします。

## スコープ

以下のスコープを利用して、アプリがアクセス可能なリソースの範囲を定義できます。

| Scope             | 説明                                                                                                   |
| :---------------- | :----------------------------------------------------------------------------------------------------- |
| retail.shop.read  | STORES のショップデータの読み取りを許可します                                                          |
| retail.shop.write | STORES のショップデータの書き込みを許可します                                                          |
| offline_access    | リフレッシュトークンを発行して、ユーザーがオフラインでもアプリの長期的なリソースアクセスを可能にします |

[▶︎TOP に戻る](README.md)
