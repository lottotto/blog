---
title: "bazel触ってみた"
date: 2022-01-10T17:42:02+09:00
draft: false
tags: ["bazel"]
---
https://github.com/bazelbuild/examples
https://bazel.build/faq.html


## bazelとは
ソースコードをコンパイルして、実行可能形式にしたり、テストをしたりする一連の処理をまとめてやってくれるツールのことをビルドツールと言い、例えばMake,Maven, Gradleなどがある。
もともとgoogle では`Blaze`というビルドツールが利用されていて、これをOSS化したのが`bazel`になる

## bazelの特徴
- 多言語サポート
  - Bazelは多くの言語をサポートしており、任意のプログラミング言語をサポートするように拡張可能。
- 高レベルのビルド言語
  - プロジェクトはBUILDファイルにてで記述。BUILDファイル相互接続された小さなライブラリ、バイナリ、およびテストのセットとしてプロジェクトを記述する簡潔なテキスト形式。対照的に、Makeのようなツールでは、個々のファイルとコンパイラの呼び出しを記述する必要があります。

- マルチプラットフォームのサポート
  - 同じツールと同じBUILDファイルを使用して、さまざまなアーキテクチャ、さらにはさまざまなプラットフォーム用のソフトウェアを構築可能
  - GoogleではBazelを使用してデータセンターのシステムで実行されるサーバーアプリケーションから携帯電話で実行されるクライアントアプリまで、あらゆるものを構築。

- 再現性
  - BUILDファイルでは、各ライブラリ、テスト、およびバイナリで直接の依存関係を完全に指定する必要があるため、Bazelはこの依存関係情報を使用して、ソースファイルに変更を加えたときに何を再構築する必要があるか、およびどのタスクを並行して実行できるかを認識。
  - これはすべてのビルドが常に同じ結果を生成することを意味します。

- スケーラブル
  - Bazelは大規模なビルドを処理可能。 Googleでは、サーバーバイナリに10万のソースファイルがあるのが一般的であり、ファイルが変更されていないビルドには約200ミリ秒かかかるらしい

## Googleはなぜbazelを利用するのか
- Makeの場合
  - 人力で`Makefile`を書くのは結構辛い
- Mavenの場合
  - Javaだけなのが辛い
  - Bazelではコードを再利用可能な単位に細分化することを推奨しており、差分だけをビルドするといったことが可能なので
- gradleの場合
  - Gradleの設定ファイルが読みづらいのが辛い
  - Bazelは各アクションが何を行うかを正確に理解できるのでより並列化され、より再現性が高くなるらしい


## 実際にどのような会社やOSSがbazelを使用しているのか
https://bazel.build/users.html ここを見よう




## インストール方法
自分はMacを使用したのでMacでの説明になります
参考: https://docs.bazel.build/versions/4.2.2/install-os-x.html

```bash:インストーラの獲得.sh
export BAZEL_VERSION=3.2.0
curl -fLO "https://github.com/bazelbuild/bazel/releases/download/${BAZEL_VERSION}/bazel-${BAZEL_VERSION}-installer-darwin-x86_64.sh"

# 実行権限をつけて実行
chmod +x "bazel-${BAZEL_VERSION}-installer-darwin-x86_64.sh"
./bazel-${BAZEL_VERSION}-installer-darwin-x86_64.sh --user
```

```bash:実行結果
 k5-saito@k5-saitonoMacBook-Pro tmp % ./bazel-${BAZEL_VERSION}-installer-darwin-x86_64.sh --user
 
 Bazel installer
 ---------------
 
 Bazel is bundled with software licensed under the GPLv2 with Classpath exception.
 You can find the sources next to the installer on our release page:
    https://github.com/bazelbuild/bazel/releases
 
 # setting authenticate proxy
 # Binary package at HEAD (@15371720ae0c4)
    - [Commit](https://github.com/bazelbuild/bazel/commit/15371720ae0c4)
 Uncompressing......Extracting Bazel installation...
 .
 
 Bazel is now installed!
 
 Make sure you have "/Users/k5-saito/bin" in your path.
 
 For bash completion, add the following line to your :
   source /Users/k5-saito/.bazel/bin/bazel-complete.bash
 
 For fish shell completion, link this file into your
 /Users/k5-saito/.config/fish/completions/ directory:
   ln -s /Users/k5-saito/.bazel/bin/bazel.fish /Users/k5-saito/.config/fish/completions/bazel.fish
 
 See http://bazel.build/docs/getting-started.html to start a new project!
 ```

 ## bazelの始め方
 1. `WORKSPACE`ファイルの用意  
     - bazelにおけるワークスペースとは、ビルドするソフトウェアのソースファイルとビルド出力先を含むディレクトリ.各ワークスペースごとには`WORKSPACE`というテキストファイルがあり、空も場合もあれば、外部依存関係への参照が含まれている.
     - つまりプロジェクトのルートには`WORKSPACE`があればOK
  1. `BUILD`ファイルの用意
     - `Starlark`という言語を利用して、ビルドターゲットを指定することでBUILDファイルを記述.
     ```bazel:BUILD
        java_binary(
            name = "ProjectRunner",
            srcs = ["src/main/java/com/example/ProjectRunner.java"],
            main_class = "com.example.ProjectRunner",
            deps = [":greeter"],
        )

        java_library(
            name = "greeter",
            srcs = ["src/main/java/com/example/Greeting.java"],
        )
      ``` 
     - Bazelがビルドするソースコードとその依存関係、そしてbazelが利用するビルドルールなどが記載される
 1. ビルドする
    - `bazel build //path/to/package:<ビルドターゲット名>`でビルド可能
    ```bash:実行結果
     k5-saito@k5-saitonoMacBook-Pro java-tutorial % bazel build //:ProjectRunner
    INFO: Analyzed target //:ProjectRunner (0 packages loaded, 0 targets configured).
    INFO: Found 1 target...
    Target //:ProjectRunner up-to-date:
    bazel-bin/ProjectRunner.jar
    bazel-bin/ProjectRunner
    INFO: Elapsed time: 0.431s, Critical Path: 0.01s
    INFO: 1 process: 1 internal.
    INFO: Build completed successfully, 1 total action
    ``` 

## サンプルプロジェクトの解説
対象プロジェクトは https://github.com/bazelbuild/examples/tree/main/java-tutorial

### ワークスペースの立ち上げ
### BUILD ファイルの理解
```bash:java-tutorial/BUILD
 java_binary(
     name = "ProjectRunner",
     srcs = glob(["src/main/java/com/example/*.java"]),
 )
```
### プロジェクトのビルド
```bash:実行コマンド
$ bazel build //:ProjectRunner
```
### 依存関係のグラフを出力
このコマンドを実行すること
```bash:実行結果
bazel query  --notool_deps --noimplicit_deps "deps(//:ProjectRunner)" --output graph
```
http://www.webgraphviz.com/ ここで確認する

### bazelビルドを改良する
```bash:java-tutorial/BUILD
load("@rules_java//java:defs.bzl", "java_binary")

java_binary(
    name = "ProjectRunner",
    srcs = ["src/main/java/com/example/ProjectRunner.java"],
    main_class = "com.example.ProjectRunner",
    deps = [":greeter"],
)

java_library(
    name = "greeter",
    srcs = ["src/main/java/com/example/Greeting.java"],
)
```
```bash:実行コマンド
 bazel build //:ProjectRunner
```


## 標準で提供されているビルドルールをみてみよう

https://docs.bazel.build/versions/main/be/java.html

## 対応言語
https://awesomebazel.com/


## Go言語でBazelを利用する場合

https://note.crohaco.net/2020/bazel-golang/

gazelleを利用するのが良い。上の参考サイトがDockerコンテナのビルドまで書いてあって参考になる
