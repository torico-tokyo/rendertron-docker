# 概要

dynamicrendering用のrendertron実行環境をDockerで準備します。

### init

git clone git@github.com:torico-tokyo/rendertron-docker.git

git submodule update --recursive --init

git submodule foreach git pull origin master

### run

docker-compose build

docker-compose up

でlocalhost:80に接続できるようにする。

### deploy

AWSの画面で以下コマンドが表示されているので順番に実行しましょう

$(aws ecr get-login --no-include-email --region ap-northeast-1)

docker build -t dynamicrendering .

docker tag dynamicrendering:latest XXXXXXXXXXX.dkr.ecr.ap-northeast-1.amazonaws.com/dynamicrendering:latest

docker push XXXXXXXXXXX.dkr.ecr.ap-northeast-1.amazonaws.com/dynamicrendering:latest

### Exec in server

実行はAWS Fargateで行う。

新しいタスク定義を作成 > 新しいリビジョンを作成

port: 80

entrypoint: sh,-c

command: npm run start

```
      "entryPoint": [
        "sh",
        "-c"
      ],
      "portMappings": [
        {
          "hostPort": 80,
          "protocol": "tcp",
          "containerPort": 80
        }
      ],
      "command": [
        "npm run start"
      ],
```

クラスタ定義 > 今すぐ始める

コンテナの定義 -> custom

メモリ制限はメモリ全部

ポートマッピング 80


## 技術選定の経緯

DynamicRenderingする上で、様々なパターンを試してみた。
1. AWS Lambda@Edge + ServerlessChrome
2. AWS Lambda
3. Puppeteerを使ったNodejs自作サーバー
4. Rendertron

### 1. AWS Lambda@Edge

HeaderlessChrome自体が40MB以上あるので、CDNエッジサーバーでHeaderlessChromeを動かすのは動作やコストで不安があった。

https://www.npmjs.com/package/@serverless-chrome/lambda

### 2. AWS Lambda
Puppeteer(HeaderlessChrome)の起動に2秒ほど掛かっており、Lambdaでキャッシュ化することが難しかった。

`const browser = await puppeteer.launch();` ← ここで2秒かかる

この２秒を減少させるためにはbrowserインスタンスを開放しないようにしなければならないと考えた。

Lambda上でglobal変数をおいて、browserインスタンスを保持するようにしていたが、ログを見ていると４アクセスくらいでリフレッシュされるので諦めた。

### 3. Puppeteerを使ったNodejs自作サーバー

Expressを使って、NodejsサーバーをEC2上に立ててみた。

途中からLRUCacheとか作っていたらRendertronと同じになってきたので、Rendertronに切り替えた。

### 4. Rendertron

Rendertronは信頼性が高いOSSであるのでこれをベースに開発を行った。

LazyLoad・UserAgent判定・MFI対応のカスタマイズを行った。

RendertronはGCP上でデプロイするのは簡単そうだが、AWS上で使うにはサーバー管理が大変そうだった。

そこは、Fargateを使うことで回避した。

## パフォーマンス
GoogleBotは5~6秒間くらいでレスポンスが返却されないとクロールを中断してしまいます。

AWS Lambdaでも実装できたのですが、5秒以上かかることが多々あり、タイムアウトしていました。

Rendertron + Fargateで平均1.7秒くらいで返却できるようにしたのでOKとしました。

更にCloudFrontでのキャッシュやRendertronのメモリキャッシュも実装しています。
