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

### バックエンドサーバの実装

次にバックエンドサーバの実装を行います。

せっかくなので先日 GA となった 2nd gen の firebase-functions を使用することにしました。

`functions.https.onCall` で `createfirebaseauthcustomtoken` という Callable 関数を定義しています（※ 2nd gen の Firebase Functions では関数名はすべて小文字である必要があります）。

詳細は JSDoc にも記載しており、前述の「処理の流れ」で説明したとおりですが、再度まとめると次の通りです。

1. `verifyAccessToken`: クライアントから送られてきたアクセストークンの検証を行う
2. `getLINEProfile`: 検証済みのアクセストークンを用いて、該当する LINE ユーザーのプロフィール情報（ユーザー ID を含む）を取得する
3. 得られた `lineUserId` でカスタムトークンを作成する
4. （任意）得られたプロフィール情報から Cloud Firestore にユーザードキュメントを作成する
5. カスタムトークンをクライアントに返す

サンプルでは HTTP クライアントには `axios` を使用していますが、`fetchAPI` でも何でも構いません。

```ts
import * as admin from 'firebase-admin'
import axios from 'axios'
import * as functions from 'firebase-functions/v2'

/**
 * LINE アクセストークンを検証し、LINE プロフィール情報を取得して、Firebase Auth のカスタムトークンを生成し、ユーザードキュメントを設定する Firebase Functions の HTTPS Callable Function.
 * @param {Object} callableRequest - Firebase Functions から提供されるリクエストオブジェクト。
 * @param {string} callableRequest.data.accessToken - ユーザーから提供される LINE アクセストークン。
 * @returns {Promise<{customToken: string}>} 生成された Firebase Auth のカスタムトークンを含むオブジェクト。
 * @throws {Error} LINE アクセストークンの検証に失敗した場合、または LINE プロフィール情報の取得に失敗した場合、またはカスタムトークンの生成に失敗した場合、またはユーザードキュメントの設定に失敗した場合にエラーをスローする。
 */
export const createfirebaseauthcustomtoken = functions.https.onCall<{ accessToken: string }>(
    async (callableRequest) => {
        const accessToken = callableRequest.data.accessToken
        await verifyAccessToken(accessToken)
        const { lineUserId, name, imageUrl } = await getLINEProfile(accessToken)
        const customToken = await admin.auth().createCustomToken(lineUserId)
        await setAppUserDocument({ lineUserId, name, imageUrl })
        return { customToken }
    }
)
```

中で呼ばれているそれぞれの関数について解説していきます。

#### アクセストークンの検証

LINE の GET verify API を用いて、下記のようにクライアントから送られてきたアクセストークンを検証します。

LINE の GET verify API のレスポンスデータに含まれる

- `client_id`: LINE Login チャネル ID が正しいこと（`process.env.LINE_CHANNEL_ID` と一致すること）
- `expires_in`: アクセストークンの有効期限が切れていないこと

を確認しています。

```ts
/**
 * LINE の Verify API を呼び出して、アクセストークンの有効性を確認する。
 * @param {string} accessToken - 検証する LINE のアクセストークン。
 * @throws {Error} API のレスポンスステータスが 200 でない場合、または LINE チャネル ID が正しくない場合、またはアクセストークンの有効期限が過ぎている場合にエラーをスローする。
 * @returns {Promise<void>} アクセストークンが有効であると確認された場合に解決する Promise.
 */
const verifyAccessToken = async (accessToken: string): Promise<void> => {
    const response = await axios.get<LINEGetVerifyAPIResponse>(
        `https://api.line.me/oauth2/v2.1/verify?access_token=${accessToken}`
    )
    if (response.status !== 200) {
        throw new Error(`[${response.status}]: GET /oauth2/v2.1/verify`)
    }

    const channelId = response.data.client_id
    if (channelId !== process.env.LINE_CHANNEL_ID) {
        console.error(`channelId: ${channelId}, process.env.LINE_CHANNEL_ID: ${process.env.LINE_CHANNEL_ID}`)
        throw new Error(`LINE Login チャネル ID が正しくありません。`)
    }

    const expiresIn = response.data.expires_in
    if (expiresIn <= 0) {
        throw new Error(`アクセストークンの有効期限が過ぎています。`)
    }
}
```

### LINE ユーザーのプロフィール情報（ユーザー ID を含む）の取得

LINE の profile API を用いて、検証済みのアクセストークンから、対応するユーザーのプロフィール情報を取得します。

```ts
/**
 * LINE のプロフィール情報を取得する。
 * @param {string} accessToken - LINE のアクセストークン。
 * @returns {Promise<{ lineUserId: string; name: string; imageUrl?: string }>} ユーザーの LINE ID、名前、画像URL（存在する場合）を含むオブジェクトを返す Promise.
 * @throws エラーが発生した場合、エラーメッセージが含まれる Error オブジェクトがスローされる。
 */
const getLINEProfile = async (
    accessToken: string
): Promise<{ lineUserId: string; name: string; imageUrl?: string }> => {
    const response = await axios.get<LINEGetProfileResponse>(`https://api.line.me/v2/profile`, {
        headers: { Authorization: `Bearer ${accessToken}` }
    })
    if (response.status !== 200) {
        throw new Error(`[${response.status}]: GET /v2/profile`)
    }
    return {
        lineUserId: response.data.userId,
        name: response.data.displayName,
        imageUrl: response.data.pictureUrl
    }
}
```

### （任意）Cloud Firestore にユーザードキュメントを作成する

得られたプロフィール情報から Cloud Firestore にユーザードキュメントを作成します。

```ts
/**
 * LINE のユーザー情報を使用して Firestore に 'users' ドキュメントを作成または更新する。
 * @param {Object} params - ユーザー情報パラメータ。
 * @param {string} params.lineUserId - LINE のユーザーID。
 * @param {string} params.name - LINE のユーザー名。
 * @param {string} [params.imageUrl] - LINE のユーザー画像のURL。提供されていない場合は null を設定する。
 * @returns {Promise<void>} 作成または更新操作が完了した後に解決する Promise.
 */
const setAppUserDocument = async ({
    lineUserId,
    name,
    imageUrl
}: {
    lineUserId: string
    name: string
    imageUrl?: string
}): Promise<void> => {
    await admin
        .firestore()
        .collection(`users`)
        .doc(lineUserId)
        .set({ name: name, imageUrl: imageUrl ?? null })
}
```

## おわりに

この記事では、LINE ログインの公式ドキュメントに従いながら、カスタムトークンを用いて Firebase Authentication と LINE ログインを連携する方法について説明しました。

クライアントアプリは Flutter で、バックエンドサーバは Firebase Functions のコーラブル関数で実装する例を示しました。

今後の展望として、バックエンドサーバの実装はそれほど複雑ではありませんが、Firebase Extensions の実装・公開の仕方を学んで、Firebase Extensions として公開することができると良いなと考えています。
