---
title: "Permit.ioでABAC(Attribute-Based Access Control)をやってみた"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["accesscontrol", "permitio", "authorization"]
published: true
---

皆さん、こんにちは。

今回は以前紹介した [Permit.io で Web API のアクセス制御を実装してみた](https://zenn.dev/seiichi1101/articles/95f04cd5668718) の続編として、Permit.io で **ABAC(Attribute-Based Access Control)** を実装してみた内容を共有します。

## ABAC (Attribute-Based Access Control)とは？

実際にやってみる前に、ABAC について簡単に説明します。

ABAC は属性ベースのアクセス制御モデルで、ユーザーやリソースに属性に基づいてアクセス権限を決定します。

例えば、Zenn のようなブログサービスを考えた場合、ユーザーの属性（例: スタッフ、一般ユーザー、プレミアムユーザー）やリソースの属性（例: 無料ブログ、有料ブログ、下書きブログ）に基づいてアクセス権限を設定できます。

JSON で表現すると以下のような情報なります。

```json
{
  "user": {
    "attributes": {
      "is_staff": false,
      "role": "premium_user"
    }
  }
}


{
  "blog": {
    "attributes": {
      "paid_blog": true,
      "is_published": true
    }
  }
}
```

これにより、**属性の組み合わせ**に基づいて、ユーザーが特定のリソースに対してどのような操作（例: 読む、書く、削除する）が許可されるかを柔軟に制御できます。

## Permit.IO での ABAC 実装手順

Permit.IO では

1. User の作成、及び User の Attribute の定義（属性名や型の設定）
2. Resource の作成、及び Resource の Attribute の定義（属性名や型の設定）
3. ABAC Dynamic Role (User Sets) の作成 (Attribute に基づいて Role が割当てられる設定)
4. ABAC Dynamic Resource (Resource Sets) の作成 (Attribute に基づいて Resource が割当てられる設定)
5. Permission の設定

を行うことで、ABAC を実現できます。

少しわかりにくいのですが、かなり噛み砕いて説明すると、

ユーザーの属性値に基づいて動的に割り当てる Role (User Sets) と リソースの属性値に基づいて動的に割り当てる Resource (Resource Sets)を事前に作成しておき、その上で各 Role が各 Resource に対してどの Action を実行できるかを Permission で設定するイメージです。

例えば、実際の PermitIO のアクセス制御の際には下記のような呼び出しを行います。

```ts
const hasPermission = await permit.check(
  {
    key: "seiichi1101", // User ID
    attributes: {
      is_staff: false,
      subscription_plan: "premium",
    },
  },
  action,
  {
    type: "Blog", // Resource Type
    attributes: {
      is_paid: false,
      is_published: true,
      owner: "seiichi1101_staff",
    },
  }
);
```

ユーザーの属性とリソースの属性を一緒に渡し、Permit.io 側で事前に設定した User Sets と Resource Sets にマッチするセットを見つけ出し、最終的な Permission 判定を行う形になります。

User と Resource の属性はクライアント側から渡して、バリデーションを Permit.io 側で行うイメージです。

詳しくは、下記の公式ドキュメントも参照してください。

<https://docs.permit.io/how-to/build-policies/abac/building-abac-policy/>

## やってみる

ここでは Zenn のようなブログサービスのアクセス制御を考えたいと思います。

### シナリオ

ブログの記事には有料・無料・下書きなどの属性があり、管理者（スタッフ）はすべての記事にはアクセスできるが、一般ユーザーは無料記事のみ閲覧可能、プレミアムユーザーは有料記事も閲覧可能、などのシナリオを考えてみます。

以下のようなユーザーとリソースの属性を考えることができます。

- User
  - Staff
  - User
  - Premium User
- Resource (**Blog**)
  - Free Blog
  - Paid Blog
  - Draft Blog
  - My Blog

こちらに対して CRUD 操作を考えたとき、下記のようなアクセス制御になると考えられます。

| User \ Resource | Free Blog | Paid Blog | Draft Blog | My Blog |
| --------------- | --------- | --------- | ---------- | ------- |
| Staff           | CRUD      | CRUD      | CRUD       | CRUD    |
| User            | R         | -         | -          | CRUD    |
| Premium User    | R         | R         | -          | CRUD    |

これをもう少し細分化して、ユーザーとリソースにどの様な属性があるか？を考えると以下のようになります。

```md
- User
  - is_staff: boolean
  - subscription_plan: "free" | "premium"
- Resource (**Blog**)
  - is_paid: boolean
  - is_published: boolean
  - owner: string // userId と一致する場合に My Blog とみなす
```

マトリックスにすると下記の様な割当になります。

- User

| User         | is_staff | subscription_plan |
| ------------ | -------- | ----------------- |
| Staff        | true     | any               |
| User         | false    | free              |
| Premium User | false    | premium           |

- Resource (Blog)

| Resource   | is_paid | is_published | owner  |
| ---------- | ------- | ------------ | ------ |
| Free Blog  | false   | true         | any    |
| Paid Blog  | true    | true         | any    |
| Draft Blog | false   | false        | any    |
| My Blog    | any     | any          | userId |

### User の作成、及び User の Attribute の定義

「Directory > Users」 より 新規で３名の User を作成します。（ユーザーの作成は必須ではありませんが、後述する Resource での`ref`を使うため追加しています。）

- seiichi_staff
- seiichi_user
- seiichi_premium

![](/images/permitio-abac/2025-11-14-10-46-51.png)

「Directory > Users > Settings」 より Attribute を下記のように定義します。

ここでは、ユーザーが持つ属性を設定しているだけです。

![](/images/permitio-abac/2025-11-09-19-31-08.png)

### Resource の作成、及び Resource の Attribute の定義

リソースは、前回作った Blog Resource Type を使って、Attribute を下記のように定義します。

ここでは、Resource が持つ属性を設定しているだけです。

![](/images/permitio-abac/2025-11-09-19-32-06.png)

### ABAC Dynamic Role (User Sets) の作成

「Policies > ABAC Rules」 より 新規で３つの ABAC Dynamic Role (User Sets) を作成します。

- Staff
  - `user.is_staff equals True`
- User
  - `user.is_staff equals False AND user.subscription_plan equals "free"`
- Premium User
  - `user.is_staff equals False AND user.subscription_plan equals "premium"`

これが、実際に PermitIO にアクセス制御を問い合わせた際に、ユーザーの属性に基づいて動的に割り当てられる Role になります。

![](/images/permitio-abac/2025-11-09-19-34-04.png)

### ABAC Dynamic Resource (Resource Sets) の作成

同じく、「Policies > ABAC Rules」 より 新規で４つの ABAC Dynamic Resource (Resource Sets) を作成します。

- Free Blog
  - `resource.is_paid equals False AND resource.is_published equals True`
- Paid Blog
  - `resource.is_paid equals True AND resource.is_published equals True`
- Draft Blog
  - `resource.is_paid equals False AND resource.is_published equals False`
- My Blog (※ `ref` を使うには PermitIO 内に User が必要: 詳しくは[こちら](https://docs.permit.io/how-to/build-policies/abac/patterns/#ownership-via-list-on-resource))
  - `resource.owner equals(ref) user.key`

![](/images/permitio-abac/2025-11-09-19-36-56.png)

![](/images/permitio-abac/2025-11-14-10-38-07.png)

### Permission の設定

各 User Sets が各 Resource Sets に対してどの Action を実行できるかを Permission で設定します。

![](/images/permitio-abac/2025-11-09-19-38-24.png)

![](/images/permitio-abac/2025-11-09-19-38-48.png)

![](/images/permitio-abac/2025-11-09-19-38-36.png)

### コーディング

前回の続きで、Hono フレームワークを使ったのサンプルコードを示します。

GitHub リポジトリはこちらです。: https://github.com/seiichi1101/permitio-hono/tree/abac

ABAC を利用するためには、[PDP](https://docs.permit.io/concepts/pdp/overview/)（Policy Decision Point）と呼ばれるサーバーを経由して Permit.io にアクセスする必要があります。PDP はアクセス制御のポリシー判定を行うエンジンで、クライアントアプリケーションと Permit.io クラウド間の橋渡し役を担います。
起動方法はいくつかあるのですが、今回はローカルの Docker コンテナで起動する方法で行います。

- docker compose の設定

https://github.com/seiichi1101/permitio-hono/blob/abac/docker-compose.yaml

- メインコード

属性に応じた 3 人のユーザーと、属性に応じた複数のブログリソースを用意して、Permit.io にアクセス制御の問い合わせを行っています。

https://github.com/seiichi1101/permitio-hono/blob/abac/src/index.ts

### 動作確認

`docker compose up -d` で PDP サーバーを起動し、`npx wrangler dev` で Hono アプリケーションを起動した後、curl コマンドでいくつかのケースの動作確認を行います。

- Staff ユーザーは 他人の有料ブログにアクセス可能 ✅

```sh
$ curl -X GET "localhost:8787/auth/blog?blogId=11" -u seiichi1101_staff:password

{"message":"Blog read successfully","user":"seiichi1101_staff","blogId":"11","action":"read"}
```

- Staff ユーザーは 他人の下書きブログにアクセス可能 ✅

```sh
$ curl -X GET "localhost:8787/auth/blog?blogId=10" -u seiichi1101_staff:password

{"message":"Blog read successfully","user":"seiichi1101_staff","blogId":"10","action":"read"}
```

- User ユーザーは 無料ブログにアクセス可能 ✅

```sh
$ curl -X GET "localhost:8787/auth/blog?blogId=1" -u seiichi1101_user:password

{"message":"Blog read successfully","user":"seiichi1101_user","blogId":"1","action":"read"}
```

- User ユーザーは 他人の有料ブログにアクセス不可 ❌

```sh
$ curl -X GET "localhost:8787/auth/blog?blogId=11" -u seiichi1101_user:password

Forbidden: User 'seiichi1101_user' cannot 'read' blog '11'
```

- User ユーザーは 自身の有料ブログにアクセス可能 ✅

```sh
$ curl -X GET "localhost:8787/auth/blog?blogId=7" -u seiichi1101_user:password

{"message":"Blog read successfully","user":"seiichi1101_user","blogId":"7","action":"read"}
```

- Premium User ユーザーは 有料ブログにアクセス可能 ✅

```sh
$ curl -X GET "localhost:8787/auth/blog?blogId=3" -u seiichi1101_premium:password

{"message":"Blog read successfully","user":"seiichi1101_premium","blogId":"3","action":"read"}
```

- Premium User ユーザーは 他人の下書きブログにアクセス不可 ❌

```sh
$ curl -X GET "localhost:8787/auth/blog?blogId=6" -u seiichi1101_premium:password

Forbidden: User 'seiichi1101_premium' cannot 'read' blog '6'
```

期待通り動いてますね。

## まとめ

いかがだったでしょうか。

ABAC では、ユーザーとリソースの属性数の組み合わせ分だけアクセス制御が可能なので、RBAC(Role-Based Access Control)よりも柔軟なアクセス制御が実現できます。また、RBAC と ABAC は独立した判定を行うので、両方を組み合わせて利用することも可能です。

仮にすでにアプリケーション内にユーザーとリソースの属性がある場合には、その情報を元に User Sets と Resource Sets を作成して Action のマッピングを行うだけなので、比較的簡単に導入できるシステムも多いのではないかと感じました。

今回は Attribute の階層構造を使いませんでしたが、こちらを使うことで、親子関係がある Entity のアクセス制御も実現できるので、さらに柔軟なアクセス制御が可能になります。

ぜひ皆さんも Permit.io で ABAC を試してみてください！
