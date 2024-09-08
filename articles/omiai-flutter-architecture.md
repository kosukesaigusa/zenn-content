---
title: "Omiai の Flutter プロジェクトのアーキテクチャ"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "Flutter"]
published: false
published_at: 2024-09-20 00:00
---

## はじめに

株式会社 Omiai の Flutter テックリードの [@kosukesaigusa](https://github.com/kosukesaigusa) です。

株式会社 Omiai では、マッチングアプリの Omiai を、長年の間 iOS, Android (, Web) それぞれのプラットフォームで開発・運営してきています。

... 年のサービス提供開始からの長い歴史の中で、多くのユーザーに利用していただいているサービスになっている一方で、コードベースはかなり古くなり、技術的な負債や、それぞれのプラットフォーム間の仕様の差異など、多くの課題が蓄積してきました。

そこで、Flutter を新規導入することを決定し、既存の  iOS, Android (, Web) アプリを Flutter で少しずつリプレイスしていくプロジェクトが 2024 年 5 月頃から始まりました。

これから Omiai の Flutter プロジェクトにおける様々な取り組みを紹介していきます。

