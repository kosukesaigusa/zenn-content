---
title: "Flutter Web x Cloudflare Pages でプレビュー用のアプリを配信する"
emoji: "🚚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "Flutter", "Cloudflare"]
published: true
published_at: 2024-04-14 00:00
---

## はじめに

Flutter ではクロスプラットフォームな開発が可能です。

iOS, Android のモバイルアプリを開発するとき、GitHub Actions や Codemagic などの CI から、App Distribution や TestFlight を通じてアプリを配信することで、開発中の内容を PM やデザイナー、レビュワーなどに確認して貰えるようにすることはよくあります。

もちろん、細かい体験などは再現できませんが、Flutter は同じコードベースから Web アプリとしてもビルドすることができます（※ Web 非対応のパッケージを利用している場合などには工夫が必要です）。

そこで、この記事では、より簡易なプレビュー用の環境として、Flutter Web アプリを wrangler CLI を使用して Cloudflare Pages にデプロイする方法を紹介します。

特に設定する必要もなく、プレビュー用のデプロイをするたびに異なる URL が毎回すぐに生成されることが強みです。また、今回は紹介しませんが、Basic 認証の設定なども簡単に行うことができます。

今回紹介する GitHub Actions のワークフローでは、`main` ブランチにマージされた時に本番用の Flutter Web アプリを配信するのはもちろんですが、PR 内で `preview` とコメントすることで、その PR の状態で Flutter Web アプリとして配信し、プレビュー用 URL をコメントに返す内容も含まれます。

![pr](/images/flutter-preview-ci/pr.png)

| 本番 | プレビュー |
| ---- | ---- |
| ![production](/images/flutter-preview-ci/production.png) | ![preview](/images/flutter-preview-ci/preview.png) |

## wrangler CLI でデプロイする

まずは手元のマシンから wrangler CLI でデプロイしてみましょう。

wrangler CLI をインストールします。

```sh
npm install -g wrangler
```

ログインします。

Cloudflare のアカウントを持っていない場合にはこの流れ作るのが良いかもしれません。

```sh
wrangler login
```

Flutter アプリのあるディレクトリに移動し、Flutter アプリを Web でビルドします。

```sh
cd path/to/your-flutter-app
flutter build web
```

`flutter build web` の成果物は `build/web` にあります。

それを `wrangler pages deploy` コマンドでデプロイすることができます。

```sh
wrangler pages deploy build/web
✔ Select an account › <your-account-name>'s Account
No project selected. Would you like to create one or use an existing project?
❯ Create a new project
  Use an existing project
✔ Enter the name of your new project: … <your-project-name>
✔ Enter the production branch name: … main
✨ Successfully created the '<your-project-name>' project.
🌎  Uploading... (29/29)

✨ Success! Uploaded 29 files (5.04 sec)

✨ Deployment complete! Take a peek over at https://<random-strings>.<your-project-name>.pages.dev
```

![wrangler](/images/flutter-preview-ci/wrangler.png)

上記では、対話式のやりとりの中で、production branch name として `main` を選択しています。

こうすることで、`main` ブランチからデプロイされた場合は本番用の URL が、その他からはプレビュー用の URL が毎回発行されて割り当てられるようになります。

## GitHub Actions から配信する

### 準備

GitHub Actions から Cloudflare Pages にデプロイするワークフローを組むために、下記の準備が必要です。

#### Cloudflare の API トークンを発行する

GitHub Actions の中で wrangler CLI を使用するために、Cloudflare の API トークンを発行します。

My Profile > API Token から進んで、Cloudflare Pages への Edit 権限を付与したトークンを作成してください。

これを GitHub Actions の secrets に `CLOUDFLARE_API_TOKEN` という名前で保存することにします。

![api-token](/images/flutter-preview-ci/api-token.png)

また、Workers & Pages > Overview から Account ID も控えておきます。

今回はこれも GitHub Actions の secrets に `CLOUDFLARE_ACCOUNT_ID` という名前で保存することにします。

![account-id](/images/flutter-preview-ci/account-id.png)

#### （必要に応じて）GitHub Actions の Workflow permissions を変更する

この記事で作成するワークフローでは、PR に GitHub Actions からコメントを作成する内容を含むので、Settings > Actions > Workflow Permissions で書き込み権限も与えます。

必要に応じてより適切に権限を絞るなどもしてください。

![workflow-permissions](/images/flutter-preview-ci/workflow-permissions.png)

### ワークフローを記述する

`.github/workflows/preview.yml` にワークフローを記述します。

ワークフローの全体は下記です。

https://github.com/kosukesaigusa/flutter_preview_ci/blob/ed86b0c8cbe0c20ee8a57416eb564215fdb47dc2/.github/workflows/preview.yml#L1

ここから部分ごとに説明していきます。

トリガーの設定をします。

今回は

- `main` ブランチへの push
- Issue コメント（PR への `preview` のコメント）

をトリガーとします。

```yml
on:
  push:
    branches:
      - main
  issue_comment:
    types: [created]
```

ジョブの設定です。`if` の条件に注目してください。以下の OR 条件になっています。

- push（`main` ブランチへの push）
- PRへの (Issue) コメントで、その内容が `preview` である

```yml
jobs:
  preview:
    runs-on: ubuntu-latest
    if: >
      github.event_name == 'push' ||
      (github.event_name == 'issue_comment' && github.event.comment.body == 'preview' && github.event.issue.pull_request)
```

各ステップを記述していきます。

冒頭で `if: github.event_name == 'issue_comment'` の場合、つまり前述の条件と合わせて PR へのコメントがトリガーである場合に、「Preview CI が開始されました。」のコメントを PR に書き込みます。

```yml
- name: Comment on PR
  if: github.event_name == 'issue_comment'
  uses: actions/github-script@v6
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `Preview CI が開始されました。`
      })
```

次にリポジトリをチェックアウトします。

PR へのコメントがトリガーである場合には、その PR の最新の内容をチェックアウトします。

```yml
- uses: actions/checkout@v4

- name: Checkout PR
  if: github.event_name == 'issue_comment'
  run: gh pr checkout ${{ github.event.issue.number }}
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Flutter SDK をセットアップして Flutter Web アプリをビルドします。

```yml
- name: Setup Flutter
  id: flutter-action
  uses: subosito/flutter-action@v2
  with:
    flutter-version: <YOUR-FLUTTER-SDK-VERSION>

- name: Build Web app
  run: flutter build web
```

`cloudflare/wrangler-action@v3` を用いて成果物を Cloudflare Pages にデプロイします。

`<YOUR-PROJECT-NAME>` の部分には Cloudflare Pages のプロジェクト名を入力してください。

`deployment-url` という名前で本番またはプレビュー用の URL が発行されます。

```yml
- name: Publish to Cloudflare Pages
  id: deploy
  uses: cloudflare/wrangler-action@v3
  with:
    apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
    command: pages deploy build/web --project-name=<YOUR-PROJECT-NAME>
```

お好みで、

- `main` ブランチへのマージの場合はそのコミットステータスに `deployment-url` を紐づける
- PR のコメントにプレビュー用の `deployment-url` を書き込む

ステップを追加します。

```yml
- name: Add publish URL as commit status
  if: github.event_name == 'push'
  uses: actions/github-script@v6
  with:
    script: |
      const sha = context.payload.pull_request?.head.sha ?? context.sha;
      await github.rest.repos.createCommitStatus({
        owner: context.repo.owner,
        repo: context.repo.repo,
        context: 'Preview URL',
        description: 'Cloudflare Pages deployment',
        state: 'success',
        sha,
        target_url: "${{ steps.deploy.outputs.deployment-url }}",
      });

- name: Comment PR with preview URL
  if: github.event_name == 'issue_comment'
  uses: actions/github-script@v6
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: "Preview URL: ${{ steps.deploy.outputs.deployment-url }}",
      });
```

## おわりに

以上で、main にマージされた場合は本番 URL として、PR に `preview` のコメントがあった場合にはその都度発行されるプレビュー URL として Flutter Web アプリを配信する方法を説明しました。

Flutter Web や Cloudflare の特性も活かして、iOS や Android のモバイルアプリを配信するのよりも簡易で素早い方法として使ってみてください！

https://github.com/kosukesaigusa/flutter_preview_ci

![pr](/images/flutter-preview-ci/pr.png)

| 本番 | プレビュー |
| ---- | ---- |
| ![production](/images/flutter-preview-ci/production.png) | ![preview](/images/flutter-preview-ci/preview.png) |
