---
title: "My First Post"
date: 2020-02-03T09:33:57+09:00
draft: false
---
こんにちは、SREチームの成田です。
今回は[Serverless Framework](https://serverless.com/)を使用したAWS Lambdaの作成についてご紹介したいと思います。

# Lambdaについて

ご存じの方も多いと思いますが、LambdaはAWSが提供しているサーバレスコンピューティングサービスです。

[https://aws.amazon.com/jp/lambda/:embed:cite]

Lambdaは、サーバを管理することなくコードを実行することができ、コードはイベントドリブンに実行されます。
コードはGo、Python、JavaScriptなど多くの言語が公式でサポートされています。

[AWS Lambda ランタイム](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/lambda-runtimes.html)

また、Lambdaは無期限の無料枠も十分あるのが嬉しいところです。

[https://aws.amazon.com/jp/lambda/pricing/:title]

> AWS Lambda の無料利用枠には、1 か月に 1,000,000 件の無料リクエストおよび 400,000 GB-秒のコンピューティング時間が含まれます。


# Serverless Frameworkについて

LambdaはAWSコンソールから作成できますが、コードのGit管理、ローカルでのデバッグ、デプロイの自動化などを行おうとする場合、何かしらのツールを使用することが一般的です。

その際に使用する代表的なツールとして、Serverless FrameworkやAWS SAMがありますが、今回はServerless Frameworkを使用した方法についてご紹介します。

Serverless Frameworkは、AWS以外のプラットフォームでも使用できるマルチクラウド対応のサーバレスアプリケーション開発支援ツールです。

[https://serverless.com/:embed:cite]

先に挙げた、コードのGit管理、ローカルでのデバッグ、デプロイの自動化などの用途では、CLIの利用のみで事足りるので無料で使用できます。


# HTTPサーバの作成例

それでは実際にServerless FrameworkでLambdaの開発をしてみましょう。
今回は以下のようなHTTPサーバを作ってみます。

|項目|内容|
|:--|:--|
|エンドポイント|POST /hello|
|リクエストJSON|{"person": "`名前`"}|
|レスポンスJSON|{"text": "Hello `名前`"}|

## 準備

※ 途中、npm、Go、Dockerなどを使用しますが、必要に応じて準備してください。

まずはServerless FrameworkのCLIをインストールします。インストール方法は[インストールガイド](https://serverless.com/framework/docs/getting-started/)を参考にnpmでインストールします。

```sh
$ npm install -g serverless
```

作業用ディレクトリを作成し、slsコマンドを使用してサーバレスサービスを作成します。
サービス作成の際に、使用する言語に応じたテンプレートを作成することができます。
今回はGoで実装しようと思うので、aws-go-modテンプレートを指定します。

```sh
$ mkdir hello
$ cd hello/
$ sls create --template aws-go-mod
$ tree
.
├── Makefile
├── gomod.sh
├── hello
│   └── main.go
├── serverless.yml
└── world
    └── main.go
```

いくつかファイルが作成されたと思いますが、一旦`hello/main.go`について注目してみます。
`hello/main.go`はhelloというLambda関数のデモ実装になっています。
実際にローカルで実行してみましょう。

GoのLambdaでは、Goをビルドした実行ファイルが必要なのでGoのビルドを行う必要がありますが、Makefileがすでに作成されていると思うので、`make build`でビルドを行うことができます。
ビルドした後、`sls invoke local -f "Lambda関数名"`でLambdaのローカル実行を行うことができます。

```sh
$ make build
chmod u+x gomod.sh
./gomod.sh
export GO111MODULE=on
env GOOS=linux go build -ldflags="-s -w" -o bin/hello hello/main.go
env GOOS=linux go build -ldflags="-s -w" -o bin/world world/main.go
$ sls invoke local -f hello
Serverless: Packaging service...
Serverless: Excluding development dependencies...
Serverless: Building Docker image...
START RequestId: a50fcbae-8dfa-1d45-adcd-e250c62cce4c Version: $LATEST

END RequestId: a50fcbae-8dfa-1d45-adcd-e250c62cce4c

REPORT RequestId: a50fcbae-8dfa-1d45-adcd-e250c62cce4c	Init Duration: 160.91 ms	Duration: 5.31 ms	Billed Duration: 100 ms	Memory Size: 1536 MB	Max Memory Used: 20 MB


{"statusCode":200,"headers":{"Content-Type":"application/json","X-MyCompany-Func-Reply":"hello-handler"},"body":"{\"message\":\"Go Serverless v1.0! Your function executed successfully!\"}"}
```

一番下の段の出力がLambdaからのレスポンスになります。

`sls invoke local`では、localでLambda環境をエミュレートして実行するためにDockerコンテナを使用しています。
Dockerコンテナとしてlocalに起動したLambda環境でhello関数が実行されます。


## HTTPサーバの実装

それでは目的のHTTPサーバを`hello/main.go`に実装していきます。

HTTPサーバとしてLambdaを実行させる場合はAPI Gatewayと連携する必要があります。
API Gatewayと連携するLambdaはAWSが用意している[サンプル](https://github.com/aws/aws-lambda-go/blob/master/events/README_ApiGatewayEvent.md)を参考に、`hello/main.go`を以下のように実装します。

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

// リクエストのJSONの構造体
type RequestBody struct {
	Person string `json:"person"`
}

// レスポンスのJSONの構造体
type ResponseBody struct {
	Text string `json:"text"`
}

func Handler(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
	fmt.Printf("Processing request data for request %s.\n", request.RequestContext.RequestID)

	// リクエストのJSONをパース
	reqBody := &RequestBody{}
	if err := json.Unmarshal([]byte(request.Body), reqBody); err != nil {
		return events.APIGatewayProxyResponse{StatusCode: 500}, err
	}

	// レスポンスBodyとなるJSON文字列を作成
	resBody := &ResponseBody{
		Text: "Hello " + reqBody.Person,
	}
	if bytes, err := json.Marshal(resBody); err != nil {
		return events.APIGatewayProxyResponse{StatusCode: 500}, err
	} else {
		return events.APIGatewayProxyResponse{Body: string(bytes), StatusCode: 200}, nil
	}
}

func main() {
	lambda.Start(Handler)
}

```

[API Gatewayのイベントの型定義](https://github.com/aws/aws-lambda-go/blob/master/events/apigw.go)を見ると、HTTPリクエストBodyとレスポンスBodyはそれぞれ、`APIGatewayProxyRequest.Body`と`APIGatewayProxyResponse.Body`にstringとして定義してあるので、JSON APIの場合は注意が必要です。

また、Lambdaは標準出力に出力するだけでログの出力を行えます。

それではこのLambdaをlocalで実行してみます。
このLambdaはAPI Gatewayのイベントを入力として受け取るので、`sls generate-event`でイベントデータを生成して、`sls invoke local`で実行します。

`sls generate-event`は`-t`でイベントタイプ、`-b`でリクエストBodyを指定することができます。
`sls invoke local`でイベントデータを入力して実行するには`--path`オプションでイベントデータファイルを指定します。

```sh
$ make build
chmod u+x gomod.sh
./gomod.sh
export GO111MODULE=on
env GOOS=linux go build -ldflags="-s -w" -o bin/hello hello/main.go
env GOOS=linux go build -ldflags="-s -w" -o bin/world world/main.go
$ sls generate-event -t aws:apiGateway -b '{"person": "Narita"}' > apigw_event.json
$ sls invoke local -f hello --path apigw_event.json
Serverless: Packaging service...
Serverless: Excluding development dependencies...
Serverless: Building Docker image...
START RequestId: abf507c5-db7e-16c1-d089-4151e82e70b4 Version: $LATEST

Processing request data for request 41b45ea3-70b5-11e6-b7bd-69b5aaebc7d9.

END RequestId: abf507c5-db7e-16c1-d089-4151e82e70b4
REPORT RequestId: abf507c5-db7e-16c1-d089-4151e82e70b4	Init Duration: 188.79 ms	Duration: 4.96 ms	Billed Duration: 100 ms	Memory Size: 1536 MB	Max Memory Used: 21 MB


{"statusCode":200,"headers":null,"body":"{\"text\":\"Hello Narita\"}"}
```

リクエスト`{"person": "Narita"}`に対してレスポンス`{"text":"Hello Narita"}`が帰ってきているので、期待した動作をしていることが分かります。


## デプロイ

最後に作成したLambdaをAWSへデプロイしてみます。

今回はhelloというLambda関数のみデプロイしますが、現状不要なファイルが存在するので先にそちらを削除しておきます。
`sls create`で作成されたテンプレートの中にworldというLambda関数のデモ実装があり、それは今回は不要なので削除します。

```sh
$ rm -rf world/
```

Makefileも、以下のように`world`をビルドする行を削除しておきます。

```make
.PHONY: build clean deploy gomodgen

build: gomodgen
	export GO111MODULE=on
	env GOOS=linux go build -ldflags="-s -w" -o bin/hello hello/main.go

clean:
	rm -rf ./bin ./vendor Gopkg.lock

deploy: clean build
	sls deploy --verbose

gomodgen:
	chmod u+x gomod.sh
	./gomod.sh
```

次にデプロイの設定をしていきます。

Lambdaのデプロイには、Lambdaが起動するトリガーとなるイベント、LambdaにアタッチされるIAMロールなど設定をする必要がありますが、そういった設定はすべて`serverless.yml`に記述します。
また、今回のようにAPI Gatewayと連携するLambdaの場合は、API Gatewayの作成と設定も行う必要がありますが、それも`serverless.yml`で行うことができます。
`serverless.yml`を以下の内容に修正します。

```yaml
service: hello
frameworkVersion: '>=1.28.0 <2.0.0'

provider:
  name: aws
  runtime: go1.x
  region: ap-northeast-1

package:
  exclude:
    - ./**
  include:
    - ./bin/**

functions:
  hello:
    handler: bin/hello
    events:
      - http:
          path: hello
          method: post
```

必要最小限の設定しかしてませんが、`events`の箇所に注目すると、API Gatewayが`/hello`にPOSTでリクエストを受けた時にhello関数を起動するように、API GatewayとLambdaを作成する設定しています。
詳しくは[こちら](https://serverless.com/framework/docs/providers/aws/events/apigateway/)を御覧ください。

今回は不要なので設定していませんが、Lambdaの処理の中でS3やDynamoDBなどのAWSリソースを使用する際にはIAMの設定をする必要があります。
その際は[こちら](https://serverless.com/framework/docs/providers/aws/guide/iam/)が参考になります。

それではいよいよ`sls deploy`コマンドでデプロイしますが、デプロイには`AWS CLI`のインストールとセットアップがされている必要があるので、インストールされていない場合は[ユーザガイド](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-chap-welcome.html)を参考に準備してください。

```sh
$ make clean build
rm -rf ./bin ./vendor Gopkg.lock
chmod u+x gomod.sh
./gomod.sh
export GO111MODULE=on
env GOOS=linux go build -ldflags="-s -w" -o bin/hello hello/main.go
$ sls deploy
Serverless: Packaging service...
Serverless: Excluding development dependencies...
Serverless: Creating Stack...
Serverless: Checking Stack create progress...
........
Serverless: Stack create finished...
Serverless: Uploading CloudFormation file to S3...
Serverless: Uploading artifacts...
Serverless: Uploading service hello.zip file to S3 (3.06 MB)...
Serverless: Validating template...
Serverless: Updating Stack...
Serverless: Checking Stack update progress...
..............................
Serverless: Stack update finished...
Service Information
service: hello
stage: dev
region: ap-northeast-1
stack: hello-dev
resources: 11
api keys:
  None
endpoints:
  POST - https://bbahz0mqoc.execute-api.ap-northeast-1.amazonaws.com/dev/hello
functions:
  hello: hello-dev-hello
layers:
  None
Serverless: Run the "serverless" command to setup monitoring, troubleshooting and testing.
```

デプロイが成功すると、`https://bbahz0mqoc.execute-api.ap-northeast-1.amazonaws.com/dev/hello`のようにエンドポイントのURLが表示されます。
実際にエンドポイントに対してリクエストを送ってみます。

```sh
$ curl -X POST -H "Content-Type: application/json" -d '{"person": "Narita"}' https://bbahz0mqoc.execute-api.ap-northeast-1.amazonaws.com/dev/hello
{"text":"Hello Narita"}
```

Lambdaで実装したように`{"text":"Hello Narita"}`かレスポンスされました。
これにてLambdaのデプロイは完了です。


## 最後に

今回の例のようにHTTPサーバのバックエンドはリクエストが来たときにだけ処理が実行されれば良く、そういったイベンドドリブンな処理にはLambdaは適しています。

Lambdaの起動トリガーには、今回使用したAPI Gatewayの他にも、S3、SNS、SQS、CloudWatch Eventsなど様々なイベントを設定することができます。

各イベントを使用するLambdaの例は[AWS Lambda 開発者ガイド](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/lambda-services.html)に乗ってあるので、参考にしてみてください。


