---
title: "koでビルドしたGoのコンテナをGithubプライベートコンテナレジストリに入れて、KindからPullする"
date: 2023-10-15T18:47:09+09:00
draft: false
tags: ["CICD","Go"]
ShowToc: true
TocOpen: true
---


## この記事の目的と対象者
この記事の対象者としては、個人の開発利用目的にて、下記の人たちのしたいことができるようになることを目指します。

- goのコンテナを`ko`でビルドしたい人
- ビルドしたコンテナをgithubのコンテナレジストリのプライベートリポジトリにプッシュしたい人
- Kubernetes(今回はkindで立てたクラスタ)にそのビルド or プッシュしたコンテナを展開したい人

今回題材にしているものは下記のリポジトリに格納しました。いろいろ適当なところがありますが、ご容赦ください。  
https://github.com/lottotto/ko-sample

## koについて
Goの標準的なプロジェクト構成としては、cmd配下に`<サービス名>/main.go`をおくケースが多いかなと個人的に感じています。  

参考: https://go.dev/doc/modules/layout

この時のバイナリをビルドするコマンドとしては下記になります。ディレクトリ名は相対パスにしてください。

```bash
go build ./cmd/<サービス名>
```

このとき、koでビルドするにはこのコマンドを利用すればいい感じに出ます。

```bash
ko build --base-import-paths ./cmd/kosample --local
```
`ko build`を実行すると、コンテナをビルドして、指定されたレジストリにプッシュします。これを実行すると下記のイメージがビルドされます。正直なところなんのハッシュ値がつくかはわかっていないですが、ユニークなものがつくと考えています。
```bash
ko.local/kosample:latest
ko.local/kosample:<package-hash>
```

ここでは、`ko`の詳細は書きませんが、代表的なオプションとしては公式ドキュメントを参照ください。  
https://ko.build/configuration/#naming-images  
今回は下記の理由を元に`--base-import-paths`を利用しています。
1. プロダクション利用ではないので、ハッシュ値はいらない。
2. バイナリ名をそのままコンテナ名につけたい

## github actionの設定

まずworkflow定義としては下記の通りになります。とりあえず個人用に使うためのものになっています。

```yaml
name: Publish

on:
  push:
    branches: ['main']

jobs:
  publish:
    name: Publish
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-go@v4
        with:
          go-version: '1.20.x'
      - uses: actions/checkout@v3
      - uses: ko-build/setup-ko@v0.6
      - run: ko publish --bare ./cmd/kosample
```

今回`--bare`オプションを利用しているのは、`ko-build/setup-ko@v0.6`にて`$KO_DOCKER_REPO`が`ghcr.io/[owner]/[repo]`に設定されるためです。
`--bare`オプションを利用すると、そのまま`$KO_DOCKER_REPO`がイメージ名に利用されます。
これによって、下記のコンテナが作成されます。
```bash
ghcr.io/lottotto/kosample:latest
ghcr.io/lottotto/kosample:sha256-XXXXXXXXXx.sbom
```

しかしこれでは、githubのコンテナレジストリにプッシュされません。リポジトリのSettings>Actions>GeneralからWorkflow permissionsの設定にて、**Read and Write Permissions**を有効にしてください。

## kindでビルドしたコンテナを扱う

ここまで来れば、ローカルでも、プライベートなリポジトリにもビルドされたコンテナがあります。それぞれの方法でkindに展開してみます。

### ローカルビルドされたコンテナをkindのK8sクラスタにデプロイする

kindで作成したk8sクラスタにローカルビルドしたコンテナを展開できません。下記のコマンドを実施して、kindのノード上にimportする必要があります。

```bash
kind load docker-image ko.local/kosample/cmd/kosample:latest
```
注意事項としては、Kubernetesの使用として、イメージタグにlatestが利用されている場合は、imagePullPolicyがAlwaysに指定されます。kindで指定する場合は、ko.localからpullしようとして、そのようなリポジトリはありませんとして、失敗します。そのため、下記のどちらかに設定する必要があります。
- イメージタグのlatest指定を止める
- imagePullPolicyをIfNotPresentかNeverにする

詳しくは公式を参照してください。
https://kind.sigs.k8s.io/docs/user/quick-start/#loading-an-image-into-your-cluster


### リモートのprivateリポジトリからpullする

まずは、github のトークンを発行します。手順は下記の通りです。

1. Settings>Developer Settingsにアクセス
2. Personal Access Tokens>Token(classic)にアクセス
3. 右上のGenerate new Tokenからトークンを作成
4. read:packagesをチェックしてpackageからダウンロードできる権限を与える

上記のトークンが取得できたら下記のコマンドでトークンを作成します。

```bash
kubectl create secret docker-registry regcred \
--docker-server=https://ghcr.io \
--docker-username=lottotto \
--docker-password=$GITHUB_TOKEN
```
最後に上記で作成したsecretを利用するようにdeploymentのマニフェストにimagepullsecretを追記します。
```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
	  ==
+      imagePullSecrets:
+      - name: regcred
```

## kind上に展開したアプリにホストから疎通する

kindで展開したアプリケーションに疎通する方法は主に２つあります。
- nodeportにて穴あけしたところに対して穴あけするようにクラスタを構築する
- port-forwoardにて、つながるようにする。

前者は、穴あけするポートに従ってクラスタの作理直しが発生するため、とりあえず疎通したい人であれば、port-forwordすることをお勧めします。

ここまで来れば、githubのプライベートなレジストリにgoで作成したコンテナをプッシュして、それを色々なところに展開できます。
