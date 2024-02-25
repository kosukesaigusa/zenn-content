---
title: "GCP を活用して Firestore トリガー関数を Dart で動かす"
emoji: "🎯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "GCP", "gc24"]
published: false
published_at: 2024-02-25 00:00
---

## この記事について

:::message
この記事は [記事投稿キャンペーン2024 Google Cloud](https://zenn.dev/topics/gc24) としても投稿しています！
:::

この記事では、GCP のいろいろなサービスを活用して Firestore トリガー関数を Dart で動かす方法について説明します。

私自身、個人開発などでは Firebase をサーバサイドとしたアプリケーションを開発することが多いのですが、その際、サーバサイドで行うべき処理はこれまで TypeScript (JavaScript) で実装して Cloud Functions にデプロイしてきました。

普段は Flutter, Dart エンジニアをしており、Dart 言語が大好きなので、Dart でサーバサイドも実装することができれば嬉しいなと日々思っていたところ、それが実現できそうな方法を探ることができたのでまとめます。

GCP の Cloud Run, Eventarc, Secret Manager や Workload Identity などを活用して

- HTTP 関数を Dart で書いてデプロイする
- Firestore トリガー関数を Dart で書いてデプロイする
- GitHub Actions で、ある程度快適なデプロイワークフローを構築する

ことを目標とします。

注意点として、

- この資料で紹介する [dart_firebase_admin](https://pub.dev/packages/dart_firebase_admin) パッケージは、まだ初期段階で対応していない機能やバグが含まれている可能性があり、時には自らそのバグを追って解決する必要がある
- Firebase CLI 内部で高度に抽象化されているデプロイタスクなどを gcloud コマンドで自前で記述する必要がある

などの、根気や開拓精神も一定必要になります。

よって現時点で「今後は Firebase で使用するサーバサイドの処理は Dart で書こう！」と強く推奨する意図はありません。

が、個人的には、まだ試しきれていない機能も触ってみつつ、今後は積極的に Dart で Firebase で使用するサーバサイドの処理を書いていこうと考えていますし、すでにそのくらいは使える水準に達してきていると思っています。

## Cloud Run

Cloud Run に関する説明を引用します：

> Cloud Run は、Google のスケーラブルなインフラストラクチャ上でコンテナを直接実行できるマネージドコンピューティングプラットフォームです。
>
> コンテナイメージをビルドできるものであれば、任意のプログラミング言語で記述されたコードを Cloud Run にデプロイできます。コンテナ イメージのビルドは任意です。Go、Node.js、Python、Java、.NET Core、Ruby を使用している場合は、使用している言語のベスト プラクティスに従って、コンテナをビルドする[ソースベースのデプロイ](https://cloud.google.com/run/docs/deploying-source-code?hl=ja) オプションを使用できます。

https://cloud.google.com/run/docs/overview/what-is-cloud-run?hl=ja

昨年の Google I/O 2023 でも紹介されていましたが（GA になったのは 2022 年ですが）、Cloud Functions (2nd Gen) は裏側では Cloud Run で動くようになっています。

Cloud Functions (1st Gen) を使用する場合と比べても機能的には大差ありません。
`
参考：

https://zenn.dev/cloud_ace/articles/b30971199b392c

重要なことは、Cloud Run の「コンテナイメージをビルドできるものであれば、任意のプログラミング言語で記述されたコードをデプロイできる」という特徴により、Dart で書いた関数（サービス）を Cloud Run にデプロイして動かすことができるということです。

## dart_firebase_admin パッケージ

[dart_firebase_admin](https://pub.dev/packages/dart_firebase_admin) パッケージは、Dart や Flutter に関わるエンジニアにはよく知られている invertase が開発している Dart 向けの Firebase Admin SDK を利用できるようにするパッケージです。

https://pub.dev/packages/dart_firebase_admin

https://github.com/invertase/dart_firebase_admin

[dart_firebase_admin](https://pub.dev/packages/dart_firebase_admin) パッケージの提供する機能のスコープは大まかに

- Firebase Admin SDK を利用した Admin 権限での `adminApp` の初期化
- Admin 権限を用いた Dart 向けの Firebase API（[firebaseapis](https://pub.dev/packages/firebaseapis) パッケージも参照）を通じた Firebase の各種機能の API コール

などです。

下記のようにして、Cloud Firestore, Firebase Auth, Firebase Messaging などの各機能を Admin 権限で利用することができるようになります。

```dart
final adminApp = FirebaseAdminApp.initializeApp(
  'your-project-id',
  Credential.fromServiceAccountParams(
    clientId: 'your-client-id',
    privateKey: 'your-private-key',
    email: 'your-email',
  ),
);

final firestore = Firestore(adminApp);

final auth = Auth(adminApp);

final messaging = Messaging(adminApp);
```

## functions_framework パッケージ

[functions_framework パッケージ](https://pub.dev/packages/functions_framework) は、GoogleCloudPlatform オーガニゼーションで開発されている、Dart で記述して Cloud Run などにデプロイするための仕組みとサンプルを提供します。

https://pub.dev/packages/functions_framework

https://github.com/GoogleCloudPlatform/functions-framework-dart

以下ではいよいよ、[functions_framework パッケージ](https://pub.dev/packages/functions_framework) の公式のサンプルを参考にしながら、Dart で書いた関数を Cloud Run にデプロイする方法を紹介していきます。

## Dart で HTTP 関数を実装し、Cloud Run にデプロイする

下記のようにして後に Cloud Run にデプロイして HTTP エンドポイントとして利用できるようになる関数を記述します。

Dart 製の Web サーバ再度のフレームワークである [shelf](https://pub.dev/packages/shelf) パッケージも、その `Response`, `Request` 型を利用するために import されています。

```dart
import 'package:functions_framework/functions_framework.dart';
import 'package:shelf/shelf.dart';

@CloudFunction()
Response hello(Request request) => Response.ok('Hello, World!');
```

上記は `hello` という関数（サービス名）でリクエストするとステータスコード 200 で "Hello, World!" のメッセージが返ってきます。

上記の `@CloudFunction()` アノテーションは [functions_framework パッケージ](https://pub.dev/packages/functions_framework) が提供するものです。

build_runner によるコード生成を実行することで、

```sh
dart pub run build_runner build --delete-conflicting-outputs
```

`bin/server.dart` に下記のようなファイルが生成されます。

Dart のエントリポイントである `main` 関数とともに、`_nameToFunctionTarget` 関数が生成されています。

`_nameToFunctionTarget` 関数が、`switch` 文によって、引数で受け取った関数名に応じた関数を呼び出すような作りになっています。

引数が `'hello'` の文字列だった場合に呼び出されるのが先ほど上で定義した `hello` 関数  (`function_library.hello`) です。

```dart
import 'package:functions_framework/serve.dart';
import 'package:hello_world_function/functions.dart' as function_library;

Future<void> main(List<String> args) async {
  await serve(args, _nameToFunctionTarget);
}

FunctionTarget? _nameToFunctionTarget(String name) => switch (name) {
      'hello' => FunctionTarget.http(function_library.hello),
      _ => null
    };
```

下記のようにしてこの Dart ファイルを実行すると、Web サーバが起動し、デフォルトでは 8080 番ポートでリクエストを受け付けます。

```sh
$ dart run bin/server.dart
Listening on :8080
```

curl コマンドでリクエストしてみると、確かに "Hello World!" のレスポンスをすることが確認できます。

```sh
$ curl http://localhost:8080
Hello, World!
```

さて、この Dart の Web サーバを Cloud Run にデプロイするためには、下記のような `Dockerfile` を使用します。

次のようなことが行われています。

- 冒頭の `FROM dart:stable AS build` で Dart の環境が構築された軽量な Image を指定する
- build_runner を実行してコード生成を行う
- `dart compile exe` コマンドで指定した Dart ファイルをコンパイルして実行可能ファイル化する
- エントリポイントとして生成した実行可能ファイルを指定し、 `--target` オプションに関数名を、 `--signature-type` オプションに `http` を指定して 8080 番ポートで Web サーバを起動する

至ってシンプルな内容です。

```Dockerfile
FROM dart:stable AS build

# Resolve app dependencies.
WORKDIR /app
COPY pubspec.* ./
RUN dart pub get

# Copy app source code and AOT compile it.
COPY . .
# Ensure packages are still up-to-date if anything has changed
RUN dart pub get --offline
RUN dart pub run build_runner build --delete-conflicting-outputs
RUN dart compile exe bin/server.dart -o bin/server

# Build minimal serving image from AOT-compiled `/server` and required system
# libraries and configuration files stored in `/runtime/` from the build stage.
FROM scratch
COPY --from=build /runtime/ /
COPY --from=build /app/bin/server /app/bin/

# Start server.
EXPOSE 8080
ENTRYPOINT ["/app/bin/server", "--target=hello", "--signature-type=http"]
```

デプロイするときには gcloud CLI の下記のコマンドを実行します。

`source` オプションに `Dockerfile` の位置を指定し、 `--platform=managed` （デフォルト値）とすることでその設定・構成で Cloud Run に関数をデプロイすることができます。

```sh
gcloud run deploy hello \
  --source=. \               # can use $PWD or . for current dir
  --platform=managed \       # for Cloud Run
  --allow-unauthenticated    # for public access
```

以上で Dart で書いた HTTP 関数を Cloud Run にデプロイすることができました。

curl コマンドで実際にデプロイされた HTTP 関数にリクエストをしてみましょう。

※ public な関数として誰でもリクエストできるようになっているので注意してください。

```sh
$ curl https://hello-<生成された URL に対応する文字列>-an.a.run.app
Hello, World!
```

## Dart で Cloud Event トリガー関数を実装し、Cloud Run にデプロイする

Cloud Run には HTTP 関数だけではなく、Cloud Event をトリガーとする関数をデプロイすることもできます。

後ほど、この方法で Cloud Run にデプロイした関数に、Firestore のドキュメントの Create, Update, Delete, Write などのイベントを転送できるようにします。

[functions_framework パッケージ](https://pub.dev/packages/functions_framework) では、そのような Cloud Event をトリガーとする関数を簡単に書く方法も提供してくれています。

HTTP 関数の定義と同様に `@CloudFunction()` アノテーションを使用しつつ、引数に `CloudEvent event` と `RequestContext context` を指定するだけです。

`CloudEvent` 型の `event` パラメータに、転送されてきた Cloud Event の内容が入っています。

```dart
@CloudFunction()
void oncloudevent(CloudEvent event, RequestContext context) {
  context.logger.info('[CloudEvent] source: ${event.source}, subject: ${event.subject}');
}
```

デプロイする際の `Dockerfile` は下記の通りで、HTTP 関数との違いは `--signature-type` オプションに `cloudevent` を指定するところだけです。

```Dockerfile
FROM dart:stable AS build

# Resolve app dependencies.
WORKDIR /app
COPY pubspec.* ./
RUN dart pub get

# Copy app source code and AOT compile it.
COPY . .
# Ensure packages are still up-to-date if anything has changed
RUN dart pub get --offline
RUN dart pub run build_runner build --delete-conflicting-outputs
RUN dart compile exe bin/server.dart -o bin/server

# Build minimal serving image from AOT-compiled `/server` and required system
# libraries and configuration files stored in `/runtime/` from the build stage.
FROM scratch
COPY --from=build /runtime/ /
COPY --from=build /app/bin/server /app/bin/

# Start server.
EXPOSE 8080
ENTRYPOINT ["/app/bin/server", "--target=oncloudevent", "--signature-type=cloudevent"]
```

デプロイコマンドもほぼ同様です。

Cloud Event は GCP の同一プロジェクト内の他のサービスから発生するもので、public access にする必要がないので `--no-allow-unauthenticated` のオプションを与えています。

```sh
gcloud run deploy oncloudevent \
  --source=. \                  # can use $PWD or . for current dir
  --platform=managed \          # for Cloud Run
  --no-allow-unauthenticated    # for restricted access
```

