---
title: "アクセスコントロール界隈のトレンドまとめ 2025"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["accesscontrol", "permitio", "authorization"]
published: false
---

## 流派

これはまだ若いエコシステムであり、今後の進化が期待される分野です。特に、Google の Zanzibar や Amazon の Cedar などの新しいアプローチが注目されています。

## ポリシーエンジン

表現が正しいのかはわからないですが、アクセスコントロールの設定と評価のやり方にはいくつか思想があり、それぞれの生まれた背景や目的があるようです。

下記は、完全に競合するものではなく、重複する部分もあれば、補完しあう部分もあると思います。

- Open Policy Agent (OPA)
  - ポリシーをコードとして管理するためのオープンソースプロジェクトであり、Kubernetes や Envoy などの環境で使用される。
  - ポリシーを Rego という DSL で記述する。
  - CNCF のプロジェクトで、Kubernetes や Envoy などの環境で使用される。
- OpenFGA
  - OpenFGA は、Google の Zanzibar をベースにしたオープンソースのアクセス制御システムであり、分散型のデータベースを使用している。
  - ReBAC (Relationship-Based Access Control) モデルを採用しており、ユーザー間の関係性に基づいてアクセス権限を決定する。
  - OpenFGA は、特にマイクロサービスアーキテクチャやソーシャルネットワークに適している。
- Cedar
  - Cedar は、Amazon が開発したアクセス制御システムであり、ポリシーをコードとして管理することを目的としている。
  - Cedar は、ポリシー言語としてフォーマル検証済みのポリシー言語を使用しており、セキュリティが重要なシステムで使用される。
  - Cedar は、特に Amazon Cognito などの AWS サービスと連携して使用されることが多い。
- Opal
  - ???
  - <https://github.com/permitio/opal>

## モデル

- ACL (Access Control List)
  - リソースに対するアクセス権限を定義するためのリストであり、ユーザーやグループに対して特定の操作を許可または拒否するために使用される。
  - ACL は、リソースごとにアクセス権限を定義するため、リソースの数が増えると管理が複雑になることがある。
- RBAC (Role-Based Access Control)
  - ユーザーの役割に基づいてアクセス権限を管理する方法であり、役割ごとにアクセス権限を定義する。
  - RBAC は、ユーザーの役割が明確な場合に効果的であり、役割の変更が少ないシステムに適している。
- ABAC (Attribute-Based Access Control)
- ユーザーやリソースの属性に基づいてアクセス権限を決定する方法であり、より柔軟なポリシーを実現できる。
  - ABAC は、ユーザーやリソースの属性が多様な場合に効果的であり、複雑なポリシーを実現できる。
- ReBAC (Relationship-Based Access Control)
  - ユーザー間の関係性に基づいてアクセス権限を決定する方法であり、特にソーシャルネットワークやマイクロサービスアーキテクチャに適している。
  - ReBAC は、ユーザー間の関係性が重要な場合に効果的であり、動的なポリシーを実現できる。
- PBAC (Policy-Based Access Control)
  - ポリシーに基づいてアクセス権限を決定する方法であり、ポリシーの柔軟性と再利用性を高めることができる。
  - PBAC は、ポリシーの変更が頻繁な場合に効果的であり、ポリシーの管理が容易になる。
- FGAC (Fine-Grained Access Control)
  - リソースやユーザーの属性に基づいて、より詳細なアクセス権限を定義する方法であり、特にセキュリティが重要なシステムで使用される。
  - FGAC は、セキュリティが重要な場合に効果的であり、詳細なポリシーを実現できる。
- XACML (eXtensible Access Control Markup Language)
  - XML ベースのポリシー言語であり、アクセス制御ポリシーを定義するための標準的な方法を提供する。
  - XACML は、XML ベースのシステムで使用されることが多く、ポリシーの再利用性が高い。
- OPA (Open Policy Agent)
  - ポリシーをコードとして管理するためのオープンソースプロジェクトであり、Kubernetes や Envoy などの環境で使用される。
  - OPA は、ポリシーを Rego という DSL で記述する。
- Zanzibar
  - Google が開発した分散型のアクセス制御システムであり、グローバルな一貫性を持つアクセス制御を実現することを目的としている。
  - Zanzibar は、特に大規模なシステムで使用されることが多く、分散型のデータベースを使用している。
- Cedar
  - Amazon が開発したアクセス制御システムであり、ポリシーをコードとして管理することを目的としている。
  - Cedar は、ポリシー言語としてフォーマル検証済みのポリシー言語を使用しており、セキュリティが重要なシステムで使用される。
  - PBAC (Policy-Based Access Control) モデルやPARC (Policy Attribute-Based Access Control) モデルとも言われる。
    - Role Based Access Control (RBAC) - Permissions are based on role assignments
    - Attribute Based Access Control (ABAC) - Permissions are based on attribute values of the user and/or the resources
    - Relationship Based Access Control (ReBAC) - Permissions are based on a relationship between the user and the resource.

- Rego
  - クエリ言語で、Datadog の監査ログや、Kubernetes のポリシーエンジンなどで使用される。
  - <https://www.openpolicyagent.org/docs/latest/policy-language/>

## policy as code vs policy as data

## 中央集権化 vs. 分散化

## プロジェクト

| プロジェクト | モデル | 特徴 | 出典 |
| - | - | - | - |
| OPA | ReBAC | Go |Google の Zanzibar をベースにしたオープンソースのアクセス制御システムであり、分散型のデータベースを使用している。ReBAC (Relationship-Based Access Control) モデルを採用しており、ユーザー間の関係性に基づいてアクセス権限を決定する。OpenFGA は、特にマイクロサービスアーキテクチャやソーシャルネットワークに適している。 | OpenFGA |
| OpenFGA | Go | ReBAC | Google の Zanzibar をベースにしたオープンソースのアクセス制御システムであり、分散型のデータベースを使用している。ReBAC (Relationship-Based Access Control) モデルを採用しており、ユーザー間の関係性に基づいてアクセス権限を決定する。OpenFGA は、特にマイクロサービスアーキテクチャやソーシャルネットワークに適している。 | OpenFGA |
| Cedar | FGAC | Rust | Amazon が開発したアクセス制御システムであり、ポリシーをコードとして管理することを目的としている。Cedar は、ポリシー言語としてフォーマル検証済みのポリシー言語を使用しており、セキュリティが重要なシステムで使用される。Cedar は、特に Amazon Cognito などの AWS サービスと連携して使用されることが多い。 | Amazon Verified Permissions |

## プロダクト

| プロダクト | 分類 | モデル | 主要言語・備考 | 配布 | 出典 |
| - | - | - | - | - | - |
| Oso / Oso Cloud | ライブラリ＋SaaS | RBAC / ABAC / ReBAC | Polar DSL。マイクロサービス用に最適化。トレーサビリティ豊富 | OSS + SaaS | Oso |
| Permify | サーバー型 | RBAC / ABAC / ReBAC | Go 製。Zanzibar 風 ReBAC & UI 付き。Postgres 等へ保存 | OSS + (今後 SaaS) | Permify | Fine-Grained Authorization |
| Cerbos | サーバー型 PDP | RBAC / ABAC | YAML ポリシー。gRPC/HTTP API、監査ログ、SPIFFE ID 対応 | OSS / “Cerbos Hub” SaaS | Cerbos |
| SpiceDB | サーバー型 DB | ReBAC | Google Zanzibar OSS 実装。高スケール／強整合 | OSS / Authzed Cloud | GitHub |
| Auth0 FGA / Okta FGA | ReBAC | Zanzibar インスパイアの Fine‑Grained Auth。Auth0 とは独立利用も可 | auth0.com |
| AWS Verified Permissions (Cedar) | Cedar FGAC | Amazon Cognito 等と連携。フォーマル検証済みポリシー言語 Cedar で記述 | docs.aws.amazon.com |
| Permit.io |  |  |  | permit.io |
| Casbin |  |  |  | casbin.org |
| Ory Keto / Permissions | サーバー型 | ReBAC + ACL/RBAC | Go 製。Ory Network と統合可。API‑first | OSS / SaaS | GitHub |
| Warrant (WorkOS FGA) | サーバー型 | RBAC / ABAC / ReBAC | UI で顧客が権限編集可。React HOC など SDK 充実 | OSS / WorkOS SaaS | warrant.dev |

## SaaSプロダクト

| プロダクト | 分類 | モデル | 主要言語・備考 | 配布 | 出典 |
| - | - | - | - | - | - |
| Oso / Oso Cloud | ライブラリ＋SaaS | RBAC / ABAC / ReBAC | Polar DSL。マイクロサービス用に最適化。トレーサビリティ豊富 | OSS + SaaS | Oso |
| Permify | サーバー型 | RBAC / ABAC / ReBAC | Go 製。Zanzibar 風 ReBAC & UI 付き。Postgres 等へ保存 | OSS + (今後 SaaS) | Permify | Fine-Grained Authorization |
| Cerbos | サーバー型 PDP | RBAC / ABAC | YAML ポリシー。gRPC/HTTP API、監査ログ、SPIFFE ID 対応 | OSS / “Cerbos Hub” SaaS | Cerbos |
| SpiceDB | サーバー型 DB | ReBAC | Google Zanzibar OSS 実装。高スケール／強整合 | OSS / Authzed Cloud | GitHub |
| Auth0 FGA / Okta FGA | ReBAC | Zanzibar インスパイアの Fine‑Grained Auth。Auth0 とは独立利用も可 | auth0.com |
| AWS Verified Permissions (Cedar) | Cedar FGAC | Amazon Cognito 等と連携。フォーマル検証済みポリシー言語 Cedar で記述 | docs.aws.amazon.com |
| Permit.io |  |  |  | permit.io |
| Casbin |  |  |  | casbin.org |
| Ory Keto / Permissions | サーバー型 | ReBAC + ACL/RBAC | Go 製。Ory Network と統合可。API‑first | OSS / SaaS | GitHub |
| Warrant (WorkOS FGA) | サーバー型 | RBAC / ABAC / ReBAC | UI で顧客が権限編集可。React HOC など SDK 充実 | OSS / WorkOS SaaS | warrant.dev |
| Authzed | サーバー型 | ReBAC | Google Zanzibar OSS 実装。高スケール／強整合 | OSS / Authzed Cloud | GitHub |

## Ref

<https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/>

<https://www.permit.io/blog/policy-engine-showdown-opa-vs-openfga-vs-cedar>

<https://www.youtube.com/watch?v=AVA32aYObRE&embeds_referring_euri=https%3A%2F%2Fwww.permit.io%2F&source_ve_path=MjM4NTE>

amaozon verified permissions

<https://tech.techtouch.jp/entry/verified-permissions-introduction>
