---
title: "OpenTelemetryのログ仕様をZAPで実装してみる"
date: 2022-05-03T21:06:46+09:00
draft: false
tags: ["Go"]
ShowToc: true
TocOpen: true
---

## この記事について
まだOpenTelemetryのログに対しては仕様が固まっていない状況だが、情報がまとまっていないのでいろいろGithub見てまとめてみることを目的としている。

## ロギングとは
wikipedia によると下記のように書かれています
> In computing, a log file is a file that records either events that occur in an operating system or other software runs, or messages between different users of a communication software.　Logging is the act of keeping a log.
https://en.wikipedia.org/wiki/Logging_(software)

コンピューティングでは、ログファイルとは、OSや他のソフトウェアが稼働するときに発生するイベント,またはコミュニケーションソフトウェアでの異なるユーザ間でのメッセージを記録するファイルである。ロギングとはログを保持する行為であるといいます。


## ログの目的
上記のログの定義や私個人の経験からログの目的については主に下記の4つが考えられます
- システムのイベントをログから解読し、稼働中か開発中のソフトウェアの障害根本原因を分析するために利用する
- 法律対応などの背景をもとにソフトウェアが提供する機能の一つとして、データの操作の監査証跡として利用する
- ログを集計することで、ソフトウェアのメトリックを出したり、レイテンシを出したりするためのプロファイルとして利用する
- ログを統計分析することで、実行しているユーザの動作やエラーの予防検知などの状態予想などに利用する


## ログのベストプラクティス
上記の目的を達成するためにはどのようなログのベストプラクティスが必要なのかを考えていきます。それと私なりの解釈を書いておきます
1. **ログライブラリを利用すること**  
自前でprintfするのは管理が厳しくなるのでやめましょうという話。ベストプラクティスの前の基本動作. javaだったらlogbackだったり、goだったらzapだったり、いろいろ存在する。  
ただしlog4jのようなログライブラリへのバグが見つかるケースもあるので注意しておこう
2. **適切なレベルでログを記録すること**  
infoがなにとか、error が何とかはこちらを参照すること。https://ja.wikipedia.org/wiki/Log4j
いつも悩むINFOレベルの具体的なケースとしては、サービスにアクセスされたときの開始/終了情報, サービスから他のサービスやDBにアクセスしに行った時の開始/終了情報とかになるのかなぁ。  
WARNについては、例えばDBのアクセスが遅くなったり、キャッシュができなかった時に出すのがよくあるパターンっぽい
3. **適切なログのカテゴリを指定すること**  
ログを出すときにカテゴリを指定する。例えばモジュールごとだったり、DBのアクセスだったり、監査要件を満たす内容だったりでカテゴリつけると追うのが楽になるはず。
1. **意味のあるログメッセージを出力すること**  
session startとかだけだと何にもわからないので、意味のあるログメッセージを出しておく。運用側の意見が欲しいのでこことか聞いてみるとDevOpsになっていいかも。
5. **ログメッセージは英語で記載すること**  
あたりまえ体操
6. **ログメッセージにはコンテキスト情報を記載すること**  
transaction start とかにもcontext情報を付加して、transaction ID: 1qazxsw2 startとかつけるとよりわかりやすくなる。
7. **プログラムが解析しやすいフォーマットでログを出すこと**  
構造化しとけ
8. **同時に人間も読めるログのフォーマット形式にすること**  
障害分析するのは人間なんだから機械によめるだけじゃなくて人間にも読みやすくしようね。最近はjsonで出すのがトレンド
9.  **ログに出す量は多すぎても、少なすぎてもいけないこと**  
といわれてもぶっちゃけ何取ればいいかわからん。そういう時は開発中はとりあえずログ出しておいて、運用しながら減らせばいい。簡単に減らせるようなアプリの作りにしておく
10. **ログを利用する人のことを考えること**  
後日読む人は開発者じゃない。他人のことを考えてログを出そう
11. **ログを障害分析のためだけに使わないこと**  
監査だったり、プロファイラだったり、統計利用だったりする。とくに統計分析とかは手段が目的化しやすいので、分析して何したいのとかを決めておくこと。
12. **ベンダーロックインを避けること**  
log4jとかでバグ見つかるから、ログのライブラリのロックインはきをつけよう。
13. **センシティブな情報をログに出さないこと**  
クレカ番号などの個人情報出ると事業おわるよ。

参考: https://www.dataset.com/blog/the-10-commandments-of-logging/


## OpenTelemetryのログの現状
2022年5月現在のOpenTelemetryとしては、下記の状態。https://opentelemetry.io/status/
API: draft
SDK: draft
Protocol: beta
ログのデータモデルとしては実験的だが定義されている様子。APIやSDKなどすべてstableになるのは2~3年以上かかると考えました。
https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/logs/data-model.md
参考:https://event.cloudnativedays.jp/o11y2022/talks/1347

### データモデル
現在Opentelemetryが例示しているログレコードのフィールドは下記の通り

| Field Name           | Description                                  |
| -------------------- | -------------------------------------------- |
| Timestamp            | Time when the event occurred.                |
| ObservedTimestamp    | Time when the event was observed.            |
| TraceId              | Request trace id.                            |
| SpanId               | Request span id.                             |
| TraceFlags           | W3C trace flag.                              |
| SeverityText         | The severity text (also known as log level). |
| SeverityNumber       | Numerical value of the severity.             |
| Body                 | The body of the log record.                  |
| Resource             | Describes the source of the log.             |
| InstrumentationScope | Describes the scope that emitted the log.    |
| Attributes           | Additional information about the event.      |

特によくわからない、`Resource`については、ログを出しているPodの名前だったり、service名だったり、バージョンだったり、IPアドレスだったりが該当する。また`Attributes`については例えばHTTP上の情報などが該当する。より具体的には、UserAgentや、Statusコード、HTTPのパスなどが該当する。
上記の内容をベストプラクティスに従い、プログラムで読みやすく構造化して記録すると下記のようになる
```json
{
  "Timestamp": "1586960586000000000",
  "Attributes": {
      "http.scheme":"https",
      "http.host":"donut.mycie.com",
      "http.target":"/order",
      "http.method":"post",
      "http.status_code":500,
      "http.flavor":"1.1",
      "http.user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.149 Safari/537.36",
  },
  "Resource": {
    "service.name": "donut_shop",
    "service.version": "2.0.0",
    "k8s.pod.uid": "1138528c-c36e-11e9-a1a7-42010a800198",
  },
  "TraceId": "f4dbb3edd765f620", // this is a byte sequence
                                 // (hex-encoded in JSON)
  "SpanId": "43222c2d51a7abe3",
  "SeverityText": "INFO",
  "SeverityNumber": 9,
  "Body": "20200415T072306-0700 INFO I like donuts",
}
```
### ベストプラクティスとの対応状況考察
このログレコードや将来Opentelemetryでログを実装することを考えた時、ベストプラクティスを達成しているかどうかについて考察してみる
| ベストプラクティス                                      | 達成度 | 根拠                                                            |
| ------------------------------------------------------- | ------ | --------------------------------------------------------------- |
| 1. ログライブラリを利用すること                         | 達成   | 今はまだないけどOpentelemetryのライブラリを使うはずなのでできる |
| 2. 適切なレベルでログを記録すること                     | 未達成 | ソフトウェアの作りに依存するため判断不可能                      |
| 3. 適切なログのカテゴリを指定すること                   | 達成   | `Resource`等にカテゴリ情報を記載することで可能                  |
| 4. 意味のあるログメッセージを出力すること               | 未達成 | ソフトウェアの作り次第                                          |
| 5. ログメッセージは英語で記載すること                   | 達成   | ソフトウェアの作り次第だが流石に大丈夫でしょう                  |
| 6. ログメッセージにはコンテキスト情報を記載すること     | 達成   | Opentelemetryでトレースがあるのでそれを出すだけだと思う         |
| 7. プログラムが解析しやすいフォーマットでログを出すこと | 達成   | JSONで構造化している                                            |
| 8. 同時に人間も読めるログのフォーマット形式にすること   | 達成   | 人間読めると思うし、jqでbodyだけ出せば…。改行が重要かな         |
| 9.  ログに出す量は多すぎても、少なすぎてもいけないこと  | 未達成 | ソフトウェアの作り次第                                          |
| 10. ログを利用する人のことを考えること                  | 未達成 | ソフトウェアの作り次第                                          |
| 11. ログを障害分析のためだけに使わないこと              | 達成   | 運用側次第だが、障害分析以外にも使える準備はできてる            |
| 12. ベンダーロックインを避けること                      | 達成   | Opentelemetryの考え方がベンダロックイン防止                     |
| 13. センシティブな情報をログに出さないこと              | 未達成 | ソフトウェアの作り次第                                          |


ということで、Opentelemetryのフォーマットに従うだけで、8/13は達成しているのかなと思われる。

## Opentelemetryのログ対応を代わりにzapで実装してみる
では上記のログレコードの形式で実装することで、結構ベストプラクティスを満たせそうなのでZapの練習がてらにやってみることとする。

Zapの初期設定に関しては下記の通り。Qiitaなどに記事が載っているのでそちらを参考にしました。ポイントとしては特にありません。KeyをOpentelemetryが現在提唱しているフォーマットに変えたくらいです。
参考: https://qiita.com/emonuh/items/28dbee9bf2fe51d28153
```go:logger.go
package logger

import (
	"context"

	"go.opentelemetry.io/otel/trace"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

func GetZapLogger() (*zap.Logger, error) {

	level := zap.NewAtomicLevel()
	level.SetLevel(zapcore.DebugLevel)
	cfg := zap.Config{
		Level:    level,
		Encoding: "json",
		EncoderConfig: zapcore.EncoderConfig{
			TimeKey:        "TimeStamp",
			LevelKey:       "SeverityText",
			NameKey:        "Name",
			CallerKey:      "Caller",
			MessageKey:     "Body",
			EncodeLevel:    zapcore.CapitalLevelEncoder,
			EncodeTime:     zapcore.ISO8601TimeEncoder,
			EncodeDuration: zapcore.StringDurationEncoder,
			EncodeCaller:   zapcore.ShortCallerEncoder,
		},
		OutputPaths: []string{"stdout", "/tmp/zap.log"},
	}
	logger, err := cfg.Build()
	if err != nil {
		return nil, err
	}
	return logger, nil
}
```
上のloggerを呼び出すところは下記の通り
```go
func main() {
	zaplogger, err := logger.GetZapLogger()
	if err != nil {
		panic(err)
	}

    ...
    zaplogger.Info(
		fmt.Sprintf("server listening at %v", lis.Addr()),
		logger.GetOtelLogMetadataFields(context.Background())...,
	)
}

func GetOtelLogMetadataFields(ctx context.Context) []zap.Field {
	spanContext := trace.SpanContextFromContext(ctx)
	traceID := spanContext.TraceID().String()
	spanID := spanContext.SpanID().String()
	return []zap.Field{
		zap.String("TraceId", traceID),
		zap.String("SpanId", spanID),
		zap.Any("Resource", map[string]interface{}{
			"service.version": "1.0.0",
		}),
		zap.Any("Attribute", map[string]interface{}{
			"http.scheme": "http",
		}),
	}
}

```

ポイントとしては、zapはnamespaceを利用することでログを階層化できるのだが、同じレベルでの階層を2つ作ることができない。  
参考: https://github.com/uber-go/zap/issues/351  
そのため上記Issueに書いてあるようにZap.Anyを使用して同じレベルでの階層化を実現した。  

またspanはContextから獲得できたので、そこからSpanIDやTraceIDを獲得して、ログに出すことにした。
本当はAttributeなども取ってきたかったのだが、取ってくることができなかった...。
ReadOnlySpanとかから取得しているようなのですが、そのReadOnlySpanがどのように獲得されるかがわからず。。。


以上の結果結果下記の通りにログが得ることができました。
```json
{
    "SeverityText":"INFO",
    "TimeStamp":"2022-05-03T04:16:33.474Z",
    "Caller":"server/server.go:105",
    "Body":"server listening at [::]:50051",
    "TraceId":"00000000000000000000000000000000",
    "SpanId":"0000000000000000",
    "Resource":{
        "service.version":"1.0.0"
    },
    "Attribute":{
        "http.scheme":"http"
    }
}
```

今後の改善ポイントとして、`context.Context`をlogの引数に加えることと、Attributeの出力あたりでしょうか？気が向けばヘルパーみたいなの作ってみます。  
またOpenTelemetryでログを対応するためにはOpenTelemetyかなんかでTraceを実装しておくことが前段のステップかなと思っています。その準備をしておけ場いいのかなと思います。






