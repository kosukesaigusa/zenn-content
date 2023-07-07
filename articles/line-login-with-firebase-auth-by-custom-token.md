---
title: "カスタムトークンによる Firebase Authentication と LINE ログインの連携"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "firebaseauth", "line"]
published: false
---

## カスタムトークンによる Firebase Authentication と LINE ログインの連携

この記事は、2023-07-07 PORT Firebase meetup の登壇資料です。

@[card](https://connpass.com/event/285741/)

カスタムトークンによる、Firebase Authentication と LINE ログインの連携について、サンプルコードと一緒に解説します。

## Firebase Authentication のカスタムトークン認証について

はじめに、Firebase Authentication のカスタムトークンについての公式ドキュメントを掲載します。詳細はこちらをご確認ください。

@[card](https://firebase.google.com/docs/auth/admin/create-custom-tokens?hl=ja)

Firebase Authentication は様々な認証プロバイダをネイティブにサポートしています。下記はその一例です。Firebase コンソールの Authentication > Sign-in method からその一覧を確認することができます。

- Email/Password
- Phone
- Anonymous
- Google
- Apple
- Facebook

Firebase Authentication のカスタムトークン認証は、Firebase Authentication がネイティブにサポートしていない認証プロバイダと Firebase Authentication を連携するために使用することができます。たとえば自社の認証システムや、今回取り上げる LINE ログインを使用する場合などが該当します。

基本的な使い方は、下記の通りです。

- 第 1 引数（必須）：認証中のユーザーまたはデバイスを一意に識別できる ID
- 第 2 引数（任意）：追加のカスタムクレーム

```ts
import * as admin from 'firebase-admin'

const uid = 'some-uid'
const additionalClaims = { premiumAccount: true };

const createCustomToken = async (uid: string): Promise<void> => {
  const customToken = await admin.auth().createCustomToken(uid, additionalClaims)
}
```

よって、LINE ログインと連携する際には、`uid` として、LINE のユーザー ID を指定すると良いでしょう。

## LINE ログインについて

LIN Eログイン APIリファレンスについての公式ドキュメントを掲載します。詳細はこちらをご確認ください。

@[card](https://developers.line.biz/ja/reference/line-login/)

LINE ログインを組み込んだアプリを実装する際のセキュリティチェックリストも公式のドキュメントがあるので必ずご確認ください。

@[card](https://developers.line.biz/ja/docs/line-login/security-checklist/)

Firebase Authentication と LINE ログインの連携にあたり、必要となる各 LINE API については「実装方針」の章で説明します。

また、LINE ログインを使用するには、LINE Developers コンソールからプロバイダや LINE ログインチャネルの設定が必要です。

@[card](https://developers.line.biz/console/)

下記の公式記事（「LINEログインを始めよう」）に必要な設定内容が書かれています。

@[card](https://developers.line.biz/ja/docs/line-login/getting-started/#page-title)

以下では、LINE のログインチャネルの設定が LINE Developers コンソール上で済んだことを前提に話を進めます。

## 実装方針

実装方針と処理の流れは次の通りです。

1. クライアントアプリで LINE ログインし、アクセストークンを取得する（今回は Flutter をクライアントアプリとし、公式からリリースされている [flutter_line_sdk](https://pub.dev/packages/flutter_line_sdk) というパッケージを使用します）
2. アクセストークンをバックエンドサーバ（今回は Firebase Functions の onCall を使用します）に送信する
3. バックエンドサーバで、アクセストークンの検証を行う
4. 検証済みのアクセストークンを使用して、LINE のプロフィール情報（LINE のユーザー ID を含む）を取得する
5. 得られた LINE のユーザー ID を用いて、カスタムトークンを作成し、クライアントアプリにレスポンスする
6. クライアントアプリで、カスタムトークンを用いてログインする

下記のドキュメント「アプリとサーバーの間で安全なログインプロセスを構築する」に記載されている通り、クライアントからバックエンドサーバに送信して良いのは、ユーザー ID ではなくアクセストークンであることに注意してください。

@[card](https://developers.line.biz/ja/docs/line-login/secure-login-process/)

脆弱性を伴うプロセスの例が公式ドキュメントに図示されています。

![脆弱性を伴うプロセス](/images/articles/line-login-with-firebase-auth-by-custom-token/flow-vulnerability.png)

安全な方法は下図の通りです。

![安全な方法](/images/articles/line-login-with-firebase-auth-by-custom-token/flow-secure.png)

前述のセキュリティチェックリストを確認し、上の安全な方法の図に従いながら実装を進めます。

## 実装

### クライアントアプリの実装

今回は Flutter アプリをクライアントアプリと、かんたんな説明に留めます。

Flutter アプリで LINE ログインをするために、公式からリリースされている [flutter_line_sdk](https://pub.dev/packages/flutter_line_sdk) というパッケージを使用します。セットアップ方法の詳細などはパッケージの README を確認してください。

Flutter アプリからバックエンドサーバの Firebase Functions をコールするために [cloud_functions](https://pub.dev/packages/cloud_functions) というパッケージも使用します。

@[card](https://pub.dev/packages/flutter_line_sdk)

@[card](https://pub.dev/packages/cloud_functions)

事前に作成しておいた LINE のチャネル ID を用いて、エントリポイントに下記のような記述をすることで、Flutter アプリで LINE SDK を使用することができます。

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  LineSDK.instance.setup('YOUR-CHANNEL-ID-HERE').then((_) {
    print("LineSDK Prepared");
  });
  runApp(App());
}
```

LINE SDK を用いて LINE ログインを行い、得られたアクセストークンを Firebase Functions のバックエンドサーバに送ることでカスタムトークンをレスポンスとして受け取り、それを用いて Firebase Authentication にサインインするための処理は次の通りです。

```dart
Future<void> signInWithLine() async {
  // LineSDK の login メソッドをコールする
  final loginResult = await LineSDK.instance.login(scopes: ['profile', 'openid', 'email'])

  // 得られる LoginResult 型の値にアクセストークン文字列が入っている。
  final accessToken = loginResult.accessToken.data['access_token'] as String;

  // Firebase Functions の httpsCallable を使用してバックエンドサーバと通信する。
  // リクエストボディに上で得られたアクセストークンを与える。
  final callable = FirebaseFunctions.instanceFor(region: 'asia-northeast1')
      .httpsCallable('createfirebaseauthcustomtoken');
  final response = await callable.call<Map<String, dynamic>>(
    <String, dynamic>{'accessToken': accessToken},
  );

  // バックエンドサーバで作成されたカスタムトークンを得る。
  final customToken = response.data['customToken'] as String;

  // カスタムトークンを用いて Firebase Authentication にサインインする。
  await FirebaseAuth.instance.signInWithCustomToken(customToken);
}
```

