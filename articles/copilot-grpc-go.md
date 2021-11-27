---
title: "AWS Copilot CLIでECSにgRPCサーバーをデプロイできるようになったよ"
emoji: "🐫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["copilot","grpc","go"]
published: true
---

こちらの[v1.13.0リリース](https://github.com/aws/copilot-cli/releases)にてCopilotでECSにgRPCサーバーをデプロイできるようになりました。

自分で出したPRということもあり、せっかくなので試してみたいとおもいます。

## せっかちな人へ

GitHubに[ソースコード](https://github.com/seiichi1101/copilot-grpc-go)あげてます。

## セットアップ

- Copilot CLIインストール

`brew install aws/tap/copilot-cli`

既にインストール済みの方は

`brew upgrade aws/tap/copilot-cli`

- protobufインストール

`brew install protobuf`

- go install

`go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.26`
`go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.1`

- grpcurlのインストール

`brew install grpcurl`

## gRPCサーバーの作成

- mod init

`go mod init helloworld`

- go get

`go get google.golang.org/grpc google.golang.org/protobuf`

- protobuf

```protobuf:proto/helloworld.proto
syntax = "proto3";
package helloworld;
option go_package = "./;helloworld";

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (Empty) returns (HelloReply) {}
}

message Empty {}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

- generate code

```
protoc -I proto \
  --go_out proto/gen \
  --go_opt paths=source_relative \
  --go-grpc_out proto/gen \
  --go-grpc_opt paths=source_relative \
  proto/helloworld.proto
```

- main code

```go:main.go
package main

import (
 "context"
 "log"
 "net"

 pb "helloworld/proto/gen"

 "google.golang.org/grpc"
)

type server struct {
 pb.UnimplementedGreeterServer
}

func (s *server) SayHello(ctx context.Context, in *pb.Empty) (*pb.HelloReply, error) {
 return &pb.HelloReply{Message: "Hello World"}, nil
}

func main() {
 lis, err := net.Listen("tcp", ":50051")
 if err != nil {
  log.Fatalf("failed to listen: %v", err)
 }
 s := grpc.NewServer()
 pb.RegisterGreeterServer(s, &server{})
 if err := s.Serve(lis); err != nil {
  log.Fatalf("failed to serve: %v", err)
 }
}
```

- Dockerfile

```dockerfile:Dockerfile
FROM golang:1.17 AS builder
RUN mkdir /app
ADD . /app
WORKDIR /app
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64  go build -o grpc ./

FROM alpine:latest
COPY --from=builder /app ./
RUN chmod +x ./grpc
ENTRYPOINT ["./grpc"]
EXPOSE 50051
```

- docker build

`docker build -t helloworld .`

- docker run

`docker run --rm helloworld`

- send request

```sh
$ grpcurl \
-plaintext \
-import-path proto \
-proto helloworld.proto \
localhost:50051 helloworld.Greeter/SayHello
{
  "message": "Hello World"
}
```

## デプロイ

[こちら](https://aws.amazon.com/jp/about-aws/whats-new/2020/10/application-load-balancers-enable-grpc-workloads-end-to-end-http-2-support/)の記事にある通り、ターゲットグループのプロトコルバージョン`GRPC`では、リスナープロトコルとして**HTTPSが必須**になります。

つまりドメインやSSL証明書が必要になります。

ただ、既にRoute53でドメインを登録している場合は、Copilotの設定でACMでの証明書の発行やRoute53でのDNSレコードの登録をイイ感じでやってくれます。

詳細は[ドキュメント Domain](https://aws.github.io/copilot-cli/docs/developing/domain/)をご覧ください。

- copilot app init

`copilot app init --domain cm-arai.com`

app nameは`arai`にします。

workload typeは`Load Balanced Web Service`を選びます。

![](/images/copilot-grpc-go/2021-11-26_21h24_30.png)

- copilot svc init

`copilot svc init`

service nameは`helloworld`にします。

- copilot env init

`copilot env init`

env nameは`test`にします。

- copilot deploy

`copilot deploy --name helloworld --env test`

- send request

エンドポイントは`${SvcName}.${EnvName}.${AppName}.${DomainName}`の関係です。

```sh
$ grpcurl \
-import-path proto \
-proto helloworld.proto \
helloworld.test.arai.cm-arai.com:443 helloworld.Greeter/SayHello
{
  "message": "Hello World"
}
```

## まとめ

[v1.13.0リリース](https://github.com/aws/copilot-cli/releases)では、同日に発表された[Fargate Graviton2](https://aws.amazon.com/jp/about-aws/whats-new/2021/11/aws-fargate-amazon-ecs-aws-graviton2-processors/)のサポートもされています。

まだまだ使いづらい部分はあるともいますが、リリース頻度も高く更新も活発なため、今後に期待したいと思います。
