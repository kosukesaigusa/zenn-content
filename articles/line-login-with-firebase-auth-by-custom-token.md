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

