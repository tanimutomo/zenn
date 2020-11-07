---
title: "Rails(client)とGo(server)間でgRPC通信"
emoji: "🚆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gRPC", "Go", "Ruby", "Rails"]
published: true
---

# Rails(client) <- gRPC -> Go(server) のサンプル
先日，マイクロサービス間通信をRestからgRPCにすることになり，初めてgRPCを触りました．
公式のtutorialと複数の記事を参考にしながら開発を進めて，`.proto`ファイルの書き方や簡単なサンプルコードの書き方は，理解できました，
一方，ほとんどの記事では，以下のような構成の簡単なサンプルで，ソースコード全体を公開している記事も多くありませんでした．
そこで，本記事ではレポジトリの構成・各レポジトリのディレクトリ構成や，それに応じたprotoファイルのoptionの書き方などをメインで紹介しようと思います．
(あくまで一例です．より良い書き方があったら是非教えてください！)

※ Protocol Buffersの書き方や，gRPCそのものに関する説明はこの記事ではしません．

# Source Code
articleというリソースを，作成(Create), 取得(Get)，一覧(List) できる機能を備えた簡単なアプリケーションです．

## Repositories
proto, go(server), rails(client) の3つのレポジトリを用意しました．
go, railsのレポジトリでは，protoをsubmoduleとして利用する方針を取っています．

- https://github.com/tanimutomo/grpcapi-proto
- https://github.com/tanimutomo/grpcapi-go-server
- https://github.com/tanimutomo/grpcapi-rails-client

## proto
- protoファイルを配置

```
.
└── grpcapi-proto/
    └── article.proto
```

### `article.proto`
ここでの， `option go_package` と `option ruby_package` の書き方が，`protoc`, `grpc_tools_ruby_protoc` コマンドのオプションの書き方と関連しています．

```protobuf
syntax = "proto3";

package article;

// server
option go_package = "github.com/tanimutomo/grpcapi-go-server/pkg/grpcs/article";

// client
option ruby_package = "Grpcs::GrpcapiGoServer::Article";

import "google/protobuf/timestamp.proto";

service ArticleService {
  rpc GetArticle (GetArticleRequest) returns (GetArticleResponse) {}
  rpc ListArticles (ListArticlesRequest) returns (ListArticlesResponse) {}
  rpc CreateArticle (CreateArticleRequest) returns (CreateArticleResponse) {}
}

message GetArticleRequest {
  uint64 id = 1;
}

message GetArticleResponse {
  Article article = 1;
}

message ListArticlesRequest {}

message ListArticlesResponse {
  repeated Article articles = 1;
}

message CreateArticleRequest {
  string title = 1;
}

message CreateArticleResponse {
  Article article = 1;
}

message Article {
  uint64 id = 1;
  string title = 2;
  google.protobuf.Timestamp created_at = 3;
  google.protobuf.Timestamp updated_at = 4;
}
```


## Go (server)
- proto/ に grpcapi-proto をサブモジュールとして配置
- pkg/grpcs以下に，protoから生成されるコードを格納
- cmd/api/adapter/grpc/ で，リクエストのハンドリング

```
.
└── grpcapi-go-server/
    ├── cmd/
    │   └── api/
    │       ├── server/
    │       │   └── main.go (server起動)
    │       └── adapter/
    │           └── grpc/
    │               └── article/
    │                   └── handler.go (grpcのリクエストハンドラー)
    ├── pkg/
    │   ├── data (mapをdbの代わりにしてます)
    │   └── grpcs/
    │       └── article (protocで生成されたボイラープレートコード)/
    │           ├── article.pb.go
    │           └── article_grpc.pb.go
    └── proto (@grpcapi-proto)
```

### ボイラープレートコードの生成
以下のコマンドで生成
`protoc --go_out=$GOPATH/src  --go-grpc_out=$GOPATH/src proto/article.proto`

- `option go_package` で，goのパッケージのFull Pathを指定してあるので，`--go_out`, `--go_grpc_out` のオプションは，それに合わせて指定する．


## Rails (client)
- proto/grpcapi-go-server に grpcapi-proto をサブモジュールとして配置 (クライアント側は，複数のprotoファイルを配置できるようなディレクトリ構成にしている．)
- lib/grpcs 以下にprotoからの生成コードを配置
- app/services に，lib/grpcs 以下を呼び出すサービスを配置
- 必要に応じて，app/services を呼び出して，grpc経由でリクエストを送る (今回はcontrollerから呼び出している)

```
.
└── grpcapi-rails-client/
    ├── app/
    │   ├── controllers/
    │   │   └── articles_controller.rb (必要に応じて， services/article_api を呼び出す)
    │   └── services/
    │       ├── article_api (lib/grpcs/grpcapi-go-server 以下を呼び出して結果を返すサービス)/
    │       │   ├── article/
    │       │   │   ├── create_service.rb
    │       │   │   ├── get_service.rb
    │       │   │   └── list_service.rb
    │       │   └── article.pb
    │       └── article_api.rb  
    ├── lib/
    │   └── grpcs/
    │       └── grpcapi-go-server (protocで生成されたボイラープレートコード)/
    │           ├── article_pb.rb
    │           └── article_service_pb.rb
    └── proto/
        └── grpcapi-go-server (@grpcapi-proto)
```

### ボイラープレートコードの生成
以下のコマンドで生成
`grpc_tools_ruby_protoc -I ./proto --ruby_out=lib/grpcs --grpc_out=lib/grpcs ./proto/grpcapi-go-server/article.proto`

- 上記のように指定すると， `protoファイルへのパス` -(minus) `-I で指定したprotoのルート` 以下のパス (`grpcapi-go-server/article.proto`) に合わせて， `--ruby_opt`, `--grpc_out` で指定したディレクトリ以下にコードを生成してくれる．
- 結果的に， `lib/grpcs/grpcapi-go-server/*_pb.rb` というファイルを生成してくれる．


# まとめ
本記事で紹介したソースコードでは対応できないケース (go-server がgRPCのクライアントも担う場合のprotoファイルの置き場など) があるので，ベストプラクティス的な物があれば，是非知りたいです．
自分でもいろいろ模索してみようと思います．

開発するときは，以下の手順でやるのがいいのかなと思いました． (特にgrpcurlは便利です．)
- `.proto` を作成
- `server` を作成
- `grpcurl` で挙動確認
- `client` を作成

また，今回はvalidatorを入れてないので，次試してみたいと思います．
