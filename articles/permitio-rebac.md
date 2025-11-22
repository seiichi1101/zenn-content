---
title: "Permit.ioでReBAC(Relationship-Based Access Control)をやってみた"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["accesscontrol", "permitio", "authorization", "rebac"]
published: true
---

皆さん、こんにちは。

今回は以前紹介した [Permit.io で ABAC(Attribute-Based Access Control)をやってみた](https://zenn.dev/seiichi1101/articles/permitio-abac) の続編として、Permit.io で **ReBAC(Relationship-Based Access Control)** を実装してみた内容を共有します。

## ReBAC (Relationship-Based Access Control)とは？

ReBAC はユーザーとリソースの「**関係(Relationship)**」に基づいてアクセス可否を決定するモデルです。
RBAC(Role-Based Access Control) のように事前に役割へ権限を割り当てるのではなく、ユーザーとリソースの関係がそのまま権限の根拠になります。

例えば、あるユーザーが「プロジェクト A のメンバー」であれば、そのメンバー関係を持つリソースに対して閲覧や編集権限を付与するといった具合です。

Google Drive などで利用されている **[Zanzibar](http://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/)** も ReBAC の代表例です。
Permit.io の ReBAC 機能は Zanzibar 論文で紹介されている Tuple 形式（`resource:instance#role` のような表現）と互換があり、似たようなアクセス制御モデルを実現できます。

- 参考にした資料
  - ドキュメント: https://docs.permit.io/how-to/build-policies/rebac/overview/
  - 解説動画: https://youtu.be/ZoGnYERyZgg?si=q8ZY6pU-diD0sVmj
  - Zanzibar 論文: http://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/

## Permit.io での ReBAC 実装手順

Permit.io では

1. 各 Resource と ReBAC 用の Resource Role の作成
2. Resource 間の関係 (Relationship) を定義
3. Permission Policy の定義 (Role Derivation を定義)
4. User の作成 と Instant Access の割り当て (同時に Resource Instance も作成)
5. Resource Instance 間の関係 (Relationship) を作成

を行うことで ReBAC を実現できます。

要点をまとめると「**Resource ごとに固有の Role を定義し、Action と Role の対応を Permission Policy に記述した上で、Resource 同士の関係を結んで Role Derivation を定義する**」イメージです。
実際のアクセス対象は Resource Instance（特定の Resource の具体的なインスタンス）であり、ユーザーはこのインスタンスに紐づく Role を持つことで権限を得ます。

文章で説明するとかなり複雑に感じますが、実際にやってみると理解しやすいと思います。

## やってみる

前回と同様に、Zenn のようなブログサービスのアクセス制御を考えたいと思います。
今回は Zenn の Publication 機能をモデルに ReBAC を設定します。

### シナリオ

Publication には、Publication 自体の管理を行う Admin Role と、Publication の閲覧・更新のみが許可された Member Role を設定します。
また、Publication 配下に Blog リソースが存在し、Blog には Owner Role を設定します。
**このとき、Publication の Admin Role を持つユーザーは、その Publication に紐づく Blog の Owner Role も自動的に付与されるようにします。**
一方で、Publication の Member Role を持つユーザーは、Publication 自体へのアクセスのみが許可され、自身が Owner Role を持つ Blog 以外にはアクセスできないようにします。

作成するリソースおｔユーザー構成は以下の通りです。

- Resource
  - Publication
    - Resource Role
      - `Admin`
      - `Member`
    - Resource Instance
      - `Classmethod`
  - Blog
    - Resource Role
    - `Owner` Role
    - Resource Instance
      - `blog1`
      - `blog2`
- User
  - `cm_seiichi1101_admin` (Publication:Classmethod の Admin Role を持つ、かつ Blog:blog1 の Owner Role も派生で割当てられる)
  - `cm_seiichi1101_member1` (Publication:Classmethod の Member Role を持つ)
  - `cm_seiichi1101_member2` (Publication:Classmethod の Member Role と Blog:blog1 の Owner Role を持つ)

Publication と Blog に対して CRUD 操作を考えたときのアクセス要件は以下の通りです。

| User \ Resource        | Publication:Classmethod  | Blog:blog1                                |
| ---------------------- | ------------------------ | ----------------------------------------- |
| cm_seiichi1101_admin   | CRUD (Admin Role をもつ) | CRUD (派生で Owner Role が割り当てられる) |
| cm_seiichi1101_member1 | RU (Member Role をもつ)  | -（権限なし）                             |
| cm_seiichi1101_member2 | RU (Member Role をもつ)  | CRUD（Owner Role をもつ）                 |

### 各 Resource と ReBAC 用の Resource Role の作成

まず、最初に Publication と Blog のリソースを登録し、同時に ReBAC Options で Resource Role も定義します。

- Publication Resource の作成

![](/images/permitio-rebac/2025-11-21-11-01-34.png)

`Admin` Role と `Member` Role を定義しています。

- Blog Resource の作成

![](/images/permitio-rebac/2025-11-21-10-48-43.png)

`Owner` Role を定義しています。

### Resource 間の関係 (Relationship) を定義

続いて Resource 間の親子関係を定義します。

- Publication Resource と Blog Resource の関係を定義

![](/images/permitio-rebac/2025-11-21-11-11-43.png)

`Publication` Resource が `Blog` Resource の親リソースであることを定義しています。

定義が完了すると、下記のように表示されます。

![](/images/permitio-rebac/2025-11-21-11-12-05.png)

### Permission Policy の定義 (Role Derivation を定義)

Action と Role の対応を設定し、どの Role がどの操作を実行できるかを明確にします。

- Action の定義

![](/images/permitio-rebac/2025-11-21-10-59-41.png)

![](/images/permitio-rebac/2025-11-21-11-12-56.png)

![](/images/permitio-rebac/2025-11-21-11-04-17.png)

Member Role に対しては Publication Resource の Read/Update のみを許可しています。

- Publication Resource と Blog Resource の Derivation を定義

![](/images/permitio-rebac/2025-11-21-11-06-44.png)

**ReBAC の要となるのがこの Role Derivation です**。
Publication Resource の Admin Role を Blog Resource の Owner Role に**派生(Derivation)**させることで、Publication の Admin ユーザーは紐づく Blog に対して自動的に Owner 権限を得られます。

### User の作成 と Instant Access の割り当て (同時に Resource Instance も作成)

ここまでで抽象的な定義が完了したので、実際のユーザーと Resource Instance を紐づけます。
Permit.io ではユーザー作成後に「Add Instant Role」を実行すると、Resource Instance を作りながら Role を割り当てられます。

![](/images/permitio-rebac/2025-11-21-11-20-29.png)

![](/images/permitio-rebac/2025-11-21-11-20-47.png)

![](/images/permitio-rebac/2025-11-21-11-19-58.png)

設定が完了すると、下記のように表示されます。

![](/images/permitio-rebac/2025-11-21-11-19-19.png)

これで、実際の Resource Instance の作成と、ユーザーがその Resource Instance に対して持つ Role の割当てが完了しました。

### Resource Instance 間の関係 (Relationship) を作成

最後に、Resource Instance 同士の関係をセットします。
前述の Role Derivation は Resource Instance 間の関係が存在して初めて効いてくるため、`Publication:Classmethod` と `Blog:blog1` を結びつけます。

![](/images/permitio-rebac/2025-11-21-11-22-04.png)

完了すると、下記のように表示されます。

![](/images/permitio-rebac/2025-11-21-11-23-18.png)

### コーディング

Hono フレームワークを使った検証コードを GitHub に置いています。

https://github.com/seiichi1101/permitio-hono/tree/rebac

ReBAC を利用する際は [PDP](https://docs.permit.io/concepts/pdp/overview/)（Policy Decision Point）を経由して Permit.io にアクセスする必要があります。PDP がポリシー判定を担当し、アプリケーションと Permit.io クラウドの橋渡しを行う仕組みです。今回必要な ReBAC 機能は Cloud PDP ではまだ提供されていないため、Docker コンテナでローカル PDP を立ち上げます。

- docker compose の設定

https://github.com/seiichi1101/permitio-hono/blob/rebac/docker-compose.yaml

- メインコード

https://github.com/seiichi1101/permitio-hono/blob/rebac/src/index.ts

### 動作確認

`docker compose up -d` で PDP サーバーを起動し、`npx wrangler dev` で Hono アプリケーションを起動した後、curl コマンドでいくつかのケースの動作確認を行います。

- `cm_seiichi1101_admin` ユーザーは `Publication:Classmethod` にアクセス可能 ✅

```sh
$ curl -X GET "localhost:8787/auth/publications/Classmethod" -u cm_seiichi1101_admin:password

{"message":"Blog read successfully","user":"cm_seiichi1101_admin","pubId":"Classmethod","action":"read"}
```

- `cm_seiichi1101_admin` ユーザーは `Publication:Classmethod` の `Blog:blog1` にアクセス可能 ✅ **(Role Derivation により Owner Role が派生しているため)**

```sh
$ curl -X GET "localhost:8787/auth/publications/Classmethod/blogs/blog1" -u cm_seiichi1101_admin:password

{"message":"Blog read successfully","user":"cm_seiichi1101_admin","pubId":"Classmethod","blogId":"blog1","action":"read"}
```

- `cm_seiichi1101_member1` ユーザーは `Publication:Classmethod` にアクセス可能 ✅

```sh
$ curl -X GET "localhost:8787/auth/publications/Classmethod" -u cm_seiichi1101_member1:password

{"message":"Blog read successfully","user":"cm_seiichi1101_member1","pubId":"Classmethod","action":"read"}
```

- `cm_seiichi1101_member1` ユーザーは `Publication:Classmethod` の `Blog:blog1` にアクセス不可 ❌

```sh
$ curl -X GET "localhost:8787/auth/publications/Classmethod/blogs/blog1" -u cm_seiichi1101_member1:password


Forbidden: User 'cm_seiichi1101_member1' cannot 'read' resource 'Blog' (blog1)
```

- `cm_seiichi1101_member2` ユーザーは `Publication:Classmethod` にアクセス可能 ✅

```sh
$ curl -X GET "localhost:8787/auth/publications/Classmethod" -u cm_seiichi1101_member2:password

{"message":"Blog read successfully","user":"cm_seiichi1101_member2","pubId":"Classmethod","action":"read"}
```

- `cm_seiichi1101_member2` ユーザーは `Publication:Classmethod` の `Blog:blog1` にアクセス可能 ✅ **(Owner のため)**

```sh
$ curl -X GET "localhost:8787/auth/publications/Classmethod/blogs/blog1" -u cm_seiichi1101_member2:password

{"message":"Blog read successfully","user":"cm_seiichi1101_member2","pubId":"Classmethod","blogId":"blog1","action":"read"}
```

## まとめ

いかがだったでしょうか。

Permit.io の ReBAC を使えば 今回の様な Publication/Blog のような多階層構造でも、関係性の設計さえ工夫すれば柔軟にアクセス制御を拡張できます。

一方で、権限チェック時に必要な属性情報をまとめて渡す ABAC(Attribute-Based Access Control) と違って、ReBAC では Resource Instance を実際に Permit.io 上に事前登録する必要があるため、運用面での工夫が必要になると感じました。
今後も Permit.io の新機能や面白い使い方を見つけたら共有していきたいと思います。
