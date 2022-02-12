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

クライアントからgRPC GateWayにhttpで送られたものがgRPC通信でgRPCサーバに転送。gRPCサーバがPostgresにSQLを実行する流れを確認したい。ただしフォーカスとしては
![regular](images/example-grpc-database.svg)

