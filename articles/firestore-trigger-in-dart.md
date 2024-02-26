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

サンプルリポジトリはこちら：

https://github.com/kosukesaigusa/full_dart_monorepo

です。

簡単なものですが、アーキテクチャ図にすると下記のような内容をとりあげます。

![architecture.png](/images/articles/firestore-trigger-in-dart/architecture.png)

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

## Eventarc で Firestore のイベントを Cloud Run に転送する

Eventarc に関する説明を引用します。

> Eventarc を使用すると、基盤となるインフラストラクチャを実装、カスタマイズ、またはメンテナンスすることなく、[イベントドリブンアーキテクチャ](https://cloud.google.com/eventarc/docs/event-driven-architectures?hl=ja)を構築できます。Eventarc は、分離されたマイクロサービス間の状態変更（イベント）を管理する標準化されたソリューションを提供します。トリガーされると、Eventarc は配信、セキュリティ、認可、オブザーバビリティ、エラー処理を行いながら、これらのイベントをさまざまな宛先に転送します（このドキュメントの[イベントの宛先](https://cloud.google.com/eventarc/docs/overview?hl=ja#targets)を参照）。

https://cloud.google.com/eventarc/docs/overview?hl=ja

Eventarc は GCP のさまざまなソースで利用できるイベント機能を提供します。

Eventarc を使用することで、[CloudEvents](https://cloudevents.io/) 形式のイベントを、あるサービス（イベントプロバイダ）からあるサービス（イベントレシーバ・コンシューマ）へ転送することができます。

[CloudEvents](https://cloudevents.io/) はそのようなイベントのメタデータを記述する標準的な仕様です。

https://cloudevents.io/

たとえば、gcloud CLI を用いて、

- Cloud Firestore の `(default)` データベースの `foos/{fooId}` ドキュメントに
- ドキュメントが作成された (`google.cloud.firestore.document.v1.created`) イベントを
- Cloud Run の `oncloudevent` サービス（関数）にイベントを転送する

トリガーを作成することができます。

```sh
gcloud eventarc triggers create oncloudevent \
  --destination-run-service=oncloudevent \
  --event-filters="type=google.cloud.firestore.document.v1.created" \
  --event-filters="database=(default)" \
  --event-filters="namespace=(default)" \
  --event-filters-path-pattern="document=foos/{fooId}" \
  --event-data-content-type="application/protobuf" \
  --service-account="your-service-account-name@project-id.iam.gserviceaccount.com" \
```

もちろん Google Cloud コンソール の GUI を通じて同等の操作を行うことも容易にできます。または Eventarc API を使用して管理することもできます。

## Secrete Manager

Secret Manager に関する説明を引用します。

> Secret Manager を使用すると、[シークレット](https://cloud.google.com/secret-manager/docs/overview?hl=ja#secret)をバイナリ blob またはテキスト文字列として保存、管理、アクセスできます。適切な権限を使用して、シークレットのコンテンツを表示できます。
>
> Secret Manager は、実行時にアプリケーションが必要とする構成情報（データベース パスワード、API キー、TLS 証明書など）を保存するのに便利です。

https://cloud.google.com/secret-manager/docs/overview?hl=ja

Cloud Run にデプロイした関数で Firebase Admin SDK を利用するための機密情報（private key など）は Secret Manager で管理・使用できるようにするのが良いでしょう。

Cloud Run へのサービスのデプロイ時に、たとえば下記のように `--set-secrets` オプションにキー名・値とバージョンを列挙することで、Cloud Run のサービスが参照すべき機密情報を設定できます。

```sh
gcloud run deploy hello \
  --source=. \                    # can use $PWD or . for current dir
  --platform=managed \            # for Cloud Run
  --no-allow-unauthenticated \    # for restricted access
  --set-secrets=PROJECT_ID=PROJECT_ID:latest,CLIENT_ID=CLIENT_ID:latest,CLIENT_EMAIL=CLIENT_EMAIL:latest,PRIVATE_KEY=PRIVATE_KEY:latest
```

## Workload Identity 連携

Workload Identity 連携に関する説明を引用します。

> このドキュメントでは、外部ワークロードのための ID 連携の概要について説明します。ID 連携を使用することで、サービスアカウントキーを使用せずに、Google Cloud リソースへのアクセス権を、オンプレミスまたはマルチクラウドのワークロードに付与できます。
>
> ID 連携は、アマゾン ウェブ サービス（AWS）や、OpenID Connect（OIDC）をサポートする任意の ID プロバイダ（IdP）（Microsoft Azure など）、SAML 2.0 で使用できます。
>
> Google Cloud の外部で実行されているアプリケーションは、[サービスアカウントキー](https://cloud.google.com/iam/docs/service-account-creds?hl=ja#key-types)を使用して Google Cloud リソースにアクセスできます。ただし、サービス アカウント キーは強力な認証情報であり、正しく管理しなければセキュリティ上のリスクとなります。
>
> ID 連携を使用すると、Identity and Access Management（IAM）を使用し、外部 ID に対して、サービスアカウントになりすます機能を含む [IAM ロール](https://cloud.google.com/iam/docs/overview?hl=ja#roles)を付与できます。これにより、サービスアカウントキーに関連するメンテナンスとセキュリティの負担がなくなります。

https://cloud.google.com/iam/docs/workload-identity-federation?hl=ja

具体的な設定内容などは下記の記事などが非常に参考になります。

https://zenn.dev/cloud_ace/articles/7fe428ac4f25c8

Workload Identity 連携を行うことで、サービスアカウントキーを発行することなく、特定のリポジトリの GitHub Actions からのみ所定の操作を行うことができるように設定することができます。

```yaml
jobs:
  deploy:
    steps:
      # 省略

      - name: Google Auth
        id: auth
        uses: 'google-github-actions/auth@v2'
        with:
          project_id: ${{ secrets.PROJECT_ID }}
          workload_identity_provider: ${{ secrets.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.SERVICE_ACCOUNT }}
```

## GitHub Actions でデプロイワークフローを組む

従来通りの方法で Node.js の SDK を用いて Firebase Functions を書く場合は、firebase CLI の `firebase deploy --only functions(:関数名)` コマンドを実行することで、関数をデプロイすることができます。

また、それが Firestore のトリガー関数である場合は、下記のように `firebase-functions` パッケージを使用して記述されていれば、 `foos` コレクションのドキュメントが作成されたことをトリガーとする関数がデプロイされます。

```ts
import * as functions from 'firebase-functions'

export const onCreateSubmission = functions
    .region(`asia-northeast1`)
    .firestore.document(`/foos/{fooId}`)
    .onCreate(async (snapshot, context) => {
        /** 省略 */
    })
```

Dart で関数を書いてそれを Cloud Run にデプロイする場合、firebase CLI を使用することはできず、`firebase-functions` パッケージのようなものも存在しません。

そこで、ここでは GitHub Actions で、ある程度快適なデプロイワークフローを組む例を示すことにします。

具体的な内容は折りたたんでいるので適宜展開して確認してください。

また、ワークフローのサンプルの全体は下記で確認できます。

https://github.com/kosukesaigusa/full_dart_monorepo/tree/main/.github

ワークフローでは下記の内容を行うことにします。

- gcloud CLI で Dart で書いた関数（HTTP や Firestore トリガー）を Cloud Run にデプロイする
- それが Firestore トリガーならば、それに対応する Eventarc トリガーを作成する
- 不要になった（Cloud Run にデプロイされているが、手元にもう存在しない）関数を削除する

まず、`.github/actions/services.yml` に下記のように関数名と一緒に `signature_type` や、Firestore トリガー関数の場合には、`event_type` や `path_pattern` を列挙することにします。

:::details .github/actions/services.yml

```yml
services:
  - service: hello
    signature_type: http
  - service: oncloudevent
    signature_type: cloudevent
    event_type: google.cloud.firestore.document.v1.created
    path_pattern: document=foos/{fooId}
```

:::

上記の `services.yml` を下記のような `deploy.yml` でパースして `$GITHUB_OUTPUT` に格納し、`reusable_deploy_workflow.yml` の `services` に渡すようにします。

:::details deploy.yml

```yml
jobs:
  set_services:
    runs-on: ubuntu-latest

    outputs:
      services: ${{ steps.set-services.outputs.services }}

    steps:
      - uses: actions/checkout@v4

      - name: Set services
        id: set-services
        run: |
          services=$(cat ${{ github.workspace }}/.github/actions/services.yml | ruby -ryaml -rjson -e 'puts YAML.load(STDIN.read)["services"].to_json')
          echo $services
          echo "services=${services}" >> $GITHUB_OUTPUT

  call_workflow:
    needs: set_services
    uses: ./reusable_deploy_workflow.yml
    secrets: inherit
    permissions:
      contents: 'read'
      id-token: 'write'
    with:
      services: ${{ needs.set_services.outputs.services }}
```

:::

`deploy.yml` から呼び出される `reusable_deploy_workflow.yml` では、下記のようにして、`strategy.matrix` に `.github/actions/services.yml` の内容を反映させます。

:::details reusable_deploy_workflow.yml

```yml
on:
  workflow_call:
    inputs:
      services:
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJSON(inputs.services) }}
```

:::

事前準備として、下記のようにリポジトリと Dart の環境を設定し、`google-github-actions/auth@v2` で Workload Identity を用いて認証をし、`google-github-actions/setup-gcloud@v2` で gcloud SDK のセットアップを済ませます。

:::details reusable_deploy_workflow.yml

```yml
# 省略

jobs:
  deploy:
    # 省略

    steps:
      - name: Checkout
        uses: 'actions/checkout@v4'

      - name: Set up Dart
        uses: 'dart-lang/setup-dart@v1'
        with:
          sdk: stable

      - name: Google Auth
        id: auth
        uses: 'google-github-actions/auth@v2'
        with:
          project_id: ${{ secrets.PROJECT_ID }}
          workload_identity_provider: ${{ secrets.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.SERVICE_ACCOUNT }}

       - name: Set up Cloud SDK
        uses: 'google-github-actions/setup-gcloud@v2'
```

:::

その後、`google-github-actions/deploy-cloudrun@v2` を用いて関数を Cloud Run にデプロイします。

`gcloud run deploy` コマンドを実行するのと同等の内容です。

:::details reusable_deploy_workflow.yml

```yml
jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:

      # 省略

      - name: Deploy to Cloud Run
        id: deploy
        uses: 'google-github-actions/deploy-cloudrun@v2'
        with:
          source: ./
          service: ${{ matrix.service }}
          region: ${{ env.REGION }}
          flags: ${{ env.FLAGS }}
          env_vars: キー名=値
          secrets: キー名=値:latest
```

:::

デプロイした関数が Firestore トリガーならば、それに対応する Eventarc トリガーを作成します。

また、すでに対応するトリガーが作成済みならばスキップするために、`gcloud eventarc triggers describe` で作成済みのトリガー一覧を取得して使用しています。

:::details reusable_deploy_workflow.yml

```yml
jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:

      # 省略

      - name: Create Eventarc Trigger
        if: ${{ matrix.signature_type == 'cloudevent' }}
        run: |
          if ! gcloud eventarc triggers describe ${{ matrix.service }} --location=${{ env.REGION }} --project=${{ secrets.PROJECT_ID }} > /dev/null 2>&1; then
            echo "Trigger does not exist. Creating trigger."
            gcloud eventarc triggers create ${{ matrix.service }} \
              --location=${{ env.REGION }} \
              --destination-run-service=${{ matrix.service }} \
              --event-filters="type=${{ matrix.event_type }}" \
              --event-filters="database=(default)" \
              --event-filters="namespace=(default)" \
              --event-filters-path-pattern="${{ matrix.path_pattern }}" \
              --event-data-content-type="application/protobuf" \
              --service-account="${{ secrets.EVENTARC_SERVICE_ACCOUNT }}" \
              --project=${{ secrets.PROJECT_ID }}
          else
            echo "Trigger already exists. Skipping creation."
          fi
```

:::

さらに、`deploy` ジョブ後の `cleanup` ジョブとして、Cloud Run にデプロイ済みだが `.github/actions/services.yml` にはもう含まれていない関数を削除する処理を行っています。

デプロイ済みの関数は `gcloud run services list --platform managed` で一覧にすることができます。

:::details reusable_deploy_workflow.yml

```yml
jobs:
  deploy:

    # 省略

  cleanup:
    needs: deploy

    runs-on: ubuntu-latest

    steps:
      # 省略

      - name: Cleanup Services
        run: |
          services=$(echo '${{ inputs.services }}' | ruby -rjson -e 'puts JSON.parse(STDIN.read).map { |s| s["service"] }.join(" ")')
          deployed_services=$(gcloud run services list --platform managed --region ${{ env.REGION }} --project ${{ secrets.PROJECT_ID }} --format="value(name)")

          services_to_delete=()
          for service in $deployed_services; do
            if [[ ! $services =~ $service ]]; then
              services_to_delete+=($service)
            fi
          done

          if [ ${#services_to_delete[@]} -eq 0 ]; then
            echo "No services to delete."
          else
            echo "Services to delete: ${services_to_delete[@]}"
            for service in "${services_to_delete[@]}"; do
              echo "Deleting service: $service"
              gcloud run services delete $service \
                --platform managed \
                --region ${{ env.REGION }} \
                --project ${{ secrets.PROJECT_ID }} \
                --quiet
            done
          fi
```

:::

## 実例

### HTTP 関数：カスタムトークン認証で Firebase Auth と LINE ログインを連携する

個人的によく実装するもので Zenn でも記事を書いてまとめているのですが、HTTP 関数の実例として、カスタムトークン認証で Firebase Auth と LINE ログインを連携するための、サーバサイドでやる必要のある典型的な実装例を紹介します。

https://zenn.dev/kosukesaigusa/articles/line-login-with-firebase-auth-by-custom-token

なお、dart_firebase_admin パッケージが提供する `createCustomToken` メソッドにはそのカスタムトークンのフォーマットが誤っているというバグがあり、ちょうど先日 PR を作ったところです。

PR レビューが遅れていてまだマージされていませんが、同様の報告をしていた方からもこれで解決されるとの確認もされています。

https://github.com/invertase/dart_firebase_admin/pull/23

`lib/functions.dart` に `@CloudFunction()` を用いて HTTP 関数を定義します。

Cloud Run の関数名には大文字を使えないなどの制約があるので、すべて小文字の関数名にしています。

```dart
@CloudFunction()
Future<Response> createfirebaseauthcustomtoken(Request request) =>
    CreateFirebaseAuthCustomTokenFunction(
      firestore: firestore,
      auth: auth,
      request: request,
    ).call();
```

`firestore` や `auth` は下記のように初期化した `AdminApp` を用いてインスタンス化した Cloud Firestore や Firebase Auth の機能を利用可能にするために `dart_firebase_admin` パッケージが提供しているものです。

```dart
final environmentVariable = EnvironmentVariable(EnvironmentProvider());

final _adminApp = FirebaseAdminApp.initializeApp(
  environmentVariable.projectId,
  Credential.fromServiceAccountParams(
    clientId: environmentVariable.clientId,
    privateKey: environmentVariable.privateKey,
    email: environmentVariable.clientEmail,
  ),
);

final firestore = Firestore(_adminApp);

final auth = Auth(_adminApp);
```

下記がカスタムトークン認証で Firebase Auth と LINE ログインを連携する処理の内容です。

private メソッドの内容は省略していますが、上記で紹介した記事の TypeScript の実装と対応する形で Dart でサーバサイドの処理が実装できています。

```dart
class CreateFirebaseAuthCustomTokenFunction {
  const CreateFirebaseAuthCustomTokenFunction({
    required this.firestore,
    required this.auth,
    required this.request,
  });

  final Firestore firestore;

  final Auth auth;

  final Request request;

  /// カスタムトークン認証で Firebase Auth と LINE ログインを連携する。
  ///
  /// [request] のリクエストボディから得られる `accessToken` を [_verifyAccessToken]
  /// メソッドで検証し、LINE プロフィールを [_getLineProfile] メソッドで取得する。
  ///
  /// その後、Firebase Auth で LINE のユーザー ID からカスタムトークンを作成し、Firestore
  /// にプロフィール情報を保存して、カスタムトークンを返す。
  Future<Response> call() async {
    try {
      final json =
          jsonDecode(await request.readAsString()) as Map<String, dynamic>;
      final accessToken = json['accessToken'] as String?;
      if (accessToken == null) {
        return Response.badRequest(
          body: jsonEncode({'message': 'accessToken is required.'}),
        );
      }

      await _verifyAccessToken(accessToken);

      final profile = await _getLineProfile(accessToken);

      final customToken = await auth.createCustomToken(profile.lineUserId);

      await firestore.collection('users').doc(profile.lineUserId).set({
        'displayName': profile.displayName,
        'imageUrl': profile.imageUrl,
      });

      return Response.ok(jsonEncode({'customToken': customToken}));
    } on Exception catch (e) {
      return Response.badRequest(body: jsonEncode({'message': e.toString()}));
    }
  }

  Future<void> _verifyAccessToken(String accessToken) async { /** 省略 */ }

  Future<({String lineUserId, String displayName, String? imageUrl})>
      _getLineProfile(String accessToken) async { /** 省略 */ }
}
```

### Firestore トリガー関数：ドキュメントの作成をトリガーに当該ドキュメントを更新する

今度は Firestore トリガー関数の実例として、`submissions` コレクションにドキュメント（稟議や経費申請のような提出物データを想定）が作成された時に発火し、そのドキュメントに `submittedByUserId`（提出者の ID）フィールドが含まれていれば、`isVerified` フィールドが `true` に更新される実装をしてみましょう。

`lib/functions.dart` に `@CloudFunction()` を用いて `CloudEvent` を引数とする関数を定義します。

```dart
@CloudFunction()
Future<void> oncreatesubmission(CloudEvent event, RequestContext context) =>
    OnCreateSubmissionFunction(
      firestore: firestore,
      auth: auth,
      event: event,
      context: context,
    ).call();
```

下記が処理の内容です。

クライアント SDK とほとんど同じ感覚で Admin SDK による処理を記述することができます。

```dart
class OnCreateSubmissionFunction {
  const OnCreateSubmissionFunction({
    required this.firestore,
    required this.auth,
    required this.event,
    required this.context,
  });

  final Firestore firestore;

  final Auth auth;

  final CloudEvent event;

  final RequestContext context;

  /// `submissions` コレクションに新たなドキュメントが作成されたときに呼び出される。
  ///
  /// そのドキュメントに非 null な `submittedByUserId` が含まれている場合、当該ドキュメント
  /// の `isVerified` フィールドを `true` に更新する。
  Future<void> call() async {
    final documentCreatedEvent = DocumentCreatedEvent.fromCloudEvent(
      firestore: firestore,
      event: event,
    );
    final documentId = documentCreatedEvent.id;
    final submittedByUserId =
        documentCreatedEvent.value.data['submittedByUserId'] as String?;
    if (submittedByUserId != null) {
      await firestore
          .collection('submissions')
          .doc(documentId)
          .update({'isVerified': true});
      context.logger.debug(
        'submission $documentId submitted by $submittedByUserId is verified',
      );
    } else {
      context.logger.debug(
        '''submission $documentId is not verified because submittedByUserId is null''',
      );
    }
  }
}
```

なお、下記のように `CloudEvent` 型の `event` からドキュメント ID などを取り出せるようにした `DocumentCreatedEvent` の実装は独自のものです。

具体的な内容はサンプルリポジトリを確認してください。

```dart
final documentCreatedEvent = DocumentCreatedEvent.fromCloudEvent(
  firestore: firestore,
  event: event,
);
final documentId = documentCreatedEvent.id;
```

Node.js の SDK では、下記のように記述するだけで、作成されたドキュメントのデータ (`snapshot`) やその他のメタデータ (`context`) を関数内で直ちに利用することができます。

npm の `firebase-functions` 相当のパッケージは Dart にないので、それとほぼ同等の仕組みのパッケージを近いうちにリリースしたいとも考えています。

```ts
import * as functions from 'firebase-functions'

export const onCreateSubmission = functions
    .region(`asia-northeast1`)
    .firestore.document(`/submissions/{submissionId}`)
    .onCreate(async (snapshot, context) => {
        /** 省略 */
    })
```

## おわりに

「GCP を活用して Firestore トリガー関数を Dart で動かす」というタイトルでやや挑戦的な取り組みをしてみました。

このような取り組みが可能なのは Cloud Run で Dart が動かせることや、この数年で Eventarc が一般公開されるようになったことや、`dart_firebase_admin` パッケージや `functions_framework` パッケージが登場したことが背景にあります。

個人的には Flutter と Firebase を組み合わせたアプリの開発に従事するようになった 4〜5 年前から「Dart で Firebase のサーバサイドが書けたら良いな」の世界観が実現できつつあることに非常にわくわくしています。

また、npm の `firebase-functions` 相当のパッケージはまだ Dart にないなど、今後の趣味の OSS 開発のネタが見つかり、とても楽しみに思っています。

これを機に Dart でサーバサイドを書く取り組みをしてみたり、Dart によるサーバサイドの開発に関連するパッケージを見ていきたいとも考えています。

同じような取り組みに関心のある方の参考になれば幸いです。
