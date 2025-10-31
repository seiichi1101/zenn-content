---
title: "アクセスコントロールSaaS Permit.io でWeb APIのアクセス制御を実装してみた"
emoji: "🙌"
type: "tech"
topics: ["acl", "permitio", "authorization", "accessControl"]
published: false
---


みなさん、こんにちは。

近年の`OpenFGA`や`Cedar`などのアクセスコントロールの統一した規格が登場してきており、注目を集めています。

今回はそんな中、イスラエルにある会社が提供している **Permit.IO** というSaaS型のアクセスコントロールサービスを利用してみたので、その紹介をしたいと思います。

![](/images/permitio-tutorial/2025-10-30_15-35-18.png)

## アクセスコントロール界隈（余談）

一言にアクセスコントロールと言っても、様々なシチュエーションとアプローチがあるかと思います。

古くはUnixのUser/Groupに対するファイルパーミッションから認証システムに付随するユーザーのアクセス制御機能、近年では認証基盤が提供しているRBAC(Role-Based Access Control)やABAC(Attribute-Based Access Control)などがあります。
中でも、Google の `Zanzibar` や Amazon の `Cedar` などのアプローチが注目されています。

下記の記事では、主要なポリシーエンジンの開発者同士がそれぞれの特徴を比較して対談しているので、興味がある方はぜひ読んでみてください。
これらはまだ若いエコシステムで、今後の進化が期待される分野です。

<https://www.permit.io/blog/policy-engine-showdown-opa-vs-openfga-vs-cedar>

## やってみた

今回は、Permit.ioの無料プランにサインアップし、チュートリアルを実施してみました。

### Workspaceの作成

まずはPermit.ioにサインアップし、Workspaceを作成します。
無料プランである程度利用できるので、気軽に試すことができます。

「Getting Started」がチュートリアルになっているので、こちらを順番に進めていきます。

![alt text](/images/permitio-tutorial/2025-10-30_15-53-14.png)

### PolicyおよびResourceの作成

Policyはアクセスコントロールのルールを定義するもので、Resourceはアクセス対象となるリソース自体を定義します。

Policyを作るためにはまずResourceを作成する必要があるので、Resourceから作成していきます。

![alt text](/images/permitio-tutorial/2025-10-30_15-59-56.png)

リソースは今回 `Blog` として作成しました。
ブログのCRUD操作に対して、Web APIでアクセス制御を行う想定です。

![alt text](/images/permitio-tutorial/2025-10-30_16-00-12.png)

Resourceを作成したら、自動でデフォルトのPolicyを作成してくれます。

![alt text](/images/permitio-tutorial/2025-10-30_16-00-24.png)

`admin` `editor` `viewer` の3つのRoleが作成され、それぞれに対してCRUD操作の権限が割り当てられています。

![alt text](/images/permitio-tutorial/2025-10-30_16-00-43.png)

### ユーザーの作成

次にユーザーを登録します。
一旦このユーザーには`admin`ロールを割り当てておきます。

![](/images/permitio-tutorial/2025-10-30-18-37-47.png)

### サンプルアプリを動かす

次に、Permit.ioが提供しているサンプルアプリを動かしてみます。
言語やプラットフォームのサポートもかなり充実しています。

![alt text](/images/permitio-tutorial/2025-10-30_16-04-59.png)

Node.js では Express.js を利用した下記のようなサンプルコードが提供されていました。
固定ユーザーのWebAPIアクセスに対して、アクセス権限をPermit.ioに問い合わせているだけのシンプルなコードです。

```javascript
const { Permit } = require('permitio');

const express = require('express');
const app = express();
const port = 4000;

// This line initializes the SDK and connects your Node.js app
// to the Permit.io PDP container you've set up in the previous step.
const permit = new Permit({
  // in production, you might need to change this url to fit your deployment
  pdp: 'https://cloudpdp.api.permit.io',
  // your api key
  token:
  'permit_key_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
});

// You can open http://localhost:4000 to invoke this http
// endpoint, and see the outcome of the permission check.
app.get('/', async (req, res) => {
  // This user was defined by you in the previous step and
  // is already assigned with a role in the permission system.
  const user = {
    id: 'seiichi1101',
    firstName: 'Seiichi',
    lastName: 'Arai',
    email: 'undefined',
  }; // in a real app, you would typically decode the user id from a JWT token

// After we created this user in the previous step, we also synced the user's identifier
// to permit.io servers with permit.write(permit.api.syncUser(user)). The user identifier
// can be anything (email, db id, etc) but must be unique for each user. Now that the
// user is synced, we can use its identifier to check permissions with 'permit.check()'.
const permitted = await permit.check('seiichi1101', 'read', 'Blog');
  if (permitted) {
    res.status(200).send(`${user.firstName} ${user.lastName} is PERMITTED to 'read' 'Blog' !`);
  } else {
    res.status(403).send(`${user.firstName} ${user.lastName} is NOT PERMITTED to 'read' 'Blog' !`);
  }
});

app.listen(port, () => {
  console.log('Example app listening at http://localhost:'+port);
});
```

ページ下部の「Test Connection」で実際にアプリがPermit.ioと通信できるか確認できます。

![alt text](/images/permitio-tutorial/2025-10-30-18-40-17.png)

### Honoアプリで試してみる

これだけだと面白くないので、Honoアプリに認証機能を追加して、Permit.ioを利用してみます。

GitHub: <https://github.com/seiichi1101/permitio-hono>

メインのコードは下記の通り。
ベーシック認証を通過したユーザーに対して、アクセス権限をPermit.ioに問い合わせています。

ここでのポイントは、HTTPの`method`と`path`をそれぞれ、`action`と`resource`にマッピングしている点です。

```typescript
import { type Context, Hono } from "hono";
import { basicAuth } from "hono/basic-auth";
import { Permit } from "permitio";

const app = new Hono<{ Bindings: Env }>();
let permit: Permit | undefined;

type Env = {
 PERMITIO_TOKEN: string;
};
type Action = "read" | "create" | "delete" | "update";
type Resource = "Blog";
type Permission = {
 action: Action;
 resource: Resource;
};

const urlMapping = (method: string, path: string): Permission => {
 const mapping: { [key: string]: Permission } = {
  "get-/auth/blog": { action: "read", resource: "Blog" },
  "post-/auth/blog": { action: "create", resource: "Blog" },
  "delete-/auth/blog": { action: "delete", resource: "Blog" },
  "put-/auth/blog": { action: "update", resource: "Blog" },
 };
 const key = `${method}-${path}`;
 return mapping[key];
};

app.use(
 "/auth/*",
 basicAuth({
  username: "seiichi1101",
  password: "password",
 }),
);

// Access Control Middleware
app.use("/auth/*", async (c, next) => {
 const path = c.req.path;
 const method = c.req.method.toLowerCase();
 const requiredPermissions = urlMapping(method, path);
 const authHeader = c.req.header("Authorization");
 const username = atob(authHeader?.substring(6) || "").split(":")[0];

 if (!permit) {
  permit = new Permit({
   pdp: "https://cloudpdp.api.permit.io",
   token: c.env.PERMITIO_TOKEN,
  });
 }

 const hasPermission = await permit.check(
  username,
  requiredPermissions.action,
  requiredPermissions.resource,
 );

 if (!hasPermission) {
  return c.text("Forbidden", 403);
 }

 return next();
});

app.get("/auth/blog", (c) => {
 return c.text(
  `You are authorized to access: ${c.req.method} - ${c.req.path}`,
 );
});

app.post("/auth/blog", (c) => {
 return c.text(
  `You are authorized to access: ${c.req.method} - ${c.req.path}`,
 );
});

app.delete("/auth/blog", (c) => {
 return c.text(
  `You are authorized to access: ${c.req.method} - ${c.req.path}`,
 );
});

app.put("/auth/blog", (c) => {
 return c.text(
  `You are authorized to access: ${c.req.method} - ${c.req.path}`,
 );
});

export default app;
```

`npm run dev`で起動し、`curl`でアクセスしてみます。

```sh
# read Blog
$ curl -X GET localhost:8787/auth/blog -u seiichi1101:password -w "\n"
You are authorized to access: GET - /auth/blog

# create Blog
$ curl -X POST localhost:8787/auth/blog -u seiichi1101:password -w "\n"
You are authorized to access: POST - /auth/blog

# delete Blog
$ curl -X DELETE localhost:8787/auth/blog -u seiichi1101:password -w "\n"
You are authorized to access: DELETE - /auth/blog

# update Blog
$ curl -X PUT localhost:8787/auth/blog -u seiichi1101:password -w "\n"
You are authorized to access: PUT - /auth/blog
```

すべての操作が許可されました。

### Roleを変更してみる

次に、Permit.ioのコンソールからユーザーのRoleを`viewer`に変更してみます。

![](/images/permitio-tutorial/2025-10-31-14-44-52.png)

その後、再度`curl`でアクセスしてみます。

```sh
# read Blog
$ curl -X GET localhost:8787/auth/blog -u seiichi1101:password -w "\n"
You are authorized to access: GET - /auth/blog

# create Blog
$ curl -X POST localhost:8787/auth/blog -u seiichi1101:password -w "\n"
Forbidden

# delete Blog
$ curl -X DELETE localhost:8787/auth/blog -u seiichi1101:password -w "\n"
Forbidden

# update Blog
$ curl -X PUT localhost:8787/auth/blog -u seiichi1101:password -w "\n"
Forbidden
```

GET以外の操作が拒否されましたね。
Permit.ioのポリシーで設定した通りに動作しています。

## まとめ

いかがだったでしょうか。

Permit.ioはSaaS型のアクセスコントロールサービスであり、簡単にアクセス制御を実装できる点が魅力的でした。
コンソールもモダンなUIで使いやすく、ポリシーの管理も直感的に行えます。
また、様々な言語やフレームワークに対応しており、既存のアプリケーションにも容易に組み込むことができます。

今後もアクセスコントロールの分野は進化が期待されるので、引き続き注目していきたいと思います。

## 備忘録

試しながら気づいた点や疑問に思った点をメモしておきます。(今後のネタになればと)

### PermitIOのサンプル集

<https://docs.permit.io/category/learn-by-example>

### Prisma連携

<https://docs.permit.io/sdk/permit-prisma-extension/>

### ユーザーの同期

<https://docs.permit.io/how-to/sync-users/>

### PDP(Policy Decision Point)のローカルホスティング

アプリケーションが自前でホストするPDPにアクセスすることが可能

PDPはコンテナとして提供されており、サイドカーで動作させることが可能

<https://docs.permit.io/concepts/pdp/overview/>
