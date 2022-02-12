---
title: "Opentelemetryで基本的なトレーシングパターンを実装する"
date: 2022-02-12T13:25:22+09:00
draft: true
tags: ["Go"]
ShowToc: true
TocOpen: true
---

## モチベーション
Go言語のOpentelemetryで適当なアプリにトレーシング機能をInstrumentation（計装）した際に、分割して色々なものは対応していたのだが、まとめてやってみたというものがなく、結構苦戦したので忘備録がてら残しておこうと思う

## トレースしたいシステム

クライアントからgRPC GateWayにhttpで送られたものがgRPC通信でgRPCサーバに転送。gRPCサーバがPostgresにSQLを実行する流れを確認したい。ただしフォーカスとしてはgRPC GateWayからとする。
![regular](images/example-grpc-database.svg)

クライアントからは`/`に対してbodyに`{"name": message}` の形式のJSONをPostすることで、データベースにその内容がINSERTされる。またクエリパラメータにそのnameを付与してGETで送ることで、そのnameを持つ行全てを戻す。

## 実装のポイント
### クライアント側
標準ライブラリを用いたhttpサーバの構成方法は他の記事に譲るとして、、、
- TraceProviderの追加  
  
TracerProvider provides access to instrumentation Tracers.
TracerProviderとはTracerへのアクセスを提供する。TracerとはStartメソッドを持つInterfaceであり、contextとNameを引数にSpanとContextを作成する機能を持つ。もしこのcontextの中にSpanがあればその子spanとして作成される  
参考:   
https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider
https://pkg.go.dev/go.opentelemetry.io/otel/trace#Tracer

TracerProviderへの追加はプロセスセーフではないといけないので、**main関数の中でかくこと**。間違ってもhandlerとか多数のスレッドで呼び出されるところでやってはいけない。
後続のサービスへspanのContextを伝播するには、`otel.SetTextMapPropagator`を追記すること。これはリクエスト送る側だけではなく、**受け取る側にも必要**

```go:client.go
package main

import (
	"github.com/lottotto/stdgrpc/utils"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/propagation"
)

func main() {
	// トレースの追加
	tp, err := utils.InitTraceProviderStdOut("gRPC", "1.0.0")
	otel.SetTracerProvider(tp)
	// 後続のサービスにつなげるためにpropagaterを追加
	otel.SetTextMapPropagator(
		propagation.NewCompositeTextMapPropagator(
			propagation.TraceContext{},
			propagation.Baggage{},
		),
	)
	....
  }
  ```

- gRPCインターセプタの追加  

gRPCで後続のサービスにリクエストを送るので、gRPCのインターセプタを利用して、gRPCの情報などをtraceの値に入れる。もちろん後続へspanを教えるためにこのメソッドは必要。
`grpc.WithUnaryInterceptor`を利用し、opentelemetryのUnary通信用のインターセプタを入れる。複数Interceptorを入れる場合はChain何ちゃらを使うこと。
（おまけ）gRPCのコネクションをプールとして利用する場合は、Controllerなりの構造体を作成し、その中にgRPCのconnectionを入れるのが良い。global変数として使うと、変数定義時に死ぬ。

```go
package main

import (
    ...
	"go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

type Controller struct {
	conn *grpc.ClientConn
}

func main() {
	...

	grpcHost := utils.GetEnv("GRPC_HOST", "localhost")
	grpcPort := utils.GetEnv("GRPC_PORT", "50051")

	conn, err := grpc.Dial(grpcHost+":"+grpcPort,
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithUnaryInterceptor(otelgrpc.UnaryClientInterceptor()),
	)
	defer conn.Close()
    c := Controller{conn: conn}
    ...

}

```

- (おまけ)otelhttpの利用

今回は標準ライブラリを用いて構築したので、`otelhttp`を利用する。これを使うことで、spanの作成や終了、メタデータの入れ込みなどを隠蔽できる。これを使う場合はオプションに`otelhttp.WithPropagators(otel.GetTextMapPropagator())`を追記しないといけない。
ただ実際はなくてもよかった。おそらくhttp送信する際にトレース情報を付与しているので、httpサーバ→httpサーバの場合は必要と思われる。ただし落としておく必要もないと思うのでそのままにする。

```go
package main

import (
    ...
)


func (c *Controller) userHandler(w http.ResponseWriter, r *http.Request) {
     ...
}

func main() {

	c := Controller{conn: conn}
	// otelhttp用のオプションが必要
	otelOptions := []otelhttp.Option{
		otelhttp.WithTracerProvider(otel.GetTracerProvider()),
		otelhttp.WithPropagators(otel.GetTextMapPropagator()),
	}
	otelUserHandler := otelhttp.NewHandler(
		http.HandlerFunc(c.userHandler),
		"UserHandler",
		otelOptions...,
	)

	http.HandleFunc("/", otelUserHandler.ServeHTTP)
	fmt.Println("start http server")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

```


### サーバ側
gRPCサーバの構成方法は他の記事に譲るとして、、、
- TracerProviderの追加
- gRPCインターセプタの追加
- SQLの追加


## 結果
Todo: Jaegarの画面を貼る！

## まとめ
まとめいい感じにできたね。




