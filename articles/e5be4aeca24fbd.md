---
title: "Next.js 13のapp dirでチケットアプリを作った"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## アプリ概要

### なぜ作ったのか
彼女とダイエットに勤しんでいまして、ただ月に何回かはチートデイ的に1食好きなものを食べる権利をお互い持っているのですが(彼女は2回、私は1回)、この権利を行使したことをDiscordに記録していました。
これアプリ化したいなーと思っていたので今回作りました。

###　使用ライブラリ

#### Next.js@13.3.1
今回はNext.js 13で登場したapp directoryを使ってみたいなと思ったのがきっかけだったので当然採用しています。

#### 


## 遭遇した問題

### You're importing a component that needs useState. It only works in a Client Component but none of its parents are marked with "use client", so they're Server Components by default.

これは何回も見たエラーです。
サーバーコンポーネントでクライアントコンポーネントでしか使えないモジュールをimportした時に発生します。
特に考えさせられたのはutilファイルみたいなものを作成しようとしたとき。
個人的にはhooksと純粋な関数や定数みたいな分け方より、`utils/form.ts`みたいな機能ベースでファイルを作ることが多いんですが、こうするとサーバーコンポーネントでも使えるモジュールとクライアントコンポーネントでしか使えないモジュール(hooksなど)が混在してしまい、よくこのエラーで怒られていました。
なので、今までのやり方をそのまま踏襲すると問題が起きるので、app dirの気持ちになることが大事ですね。。。

### unhandledRejection: Error [FirebaseError]: Messaging: This browser doesn't support the API's required to use the Firebase SDK. (messaging/unsupported-browser).

Firebase Cloud Messaging関連のモジュール実装時に発生した問題です。
サーバーサイドでのレンダリング時にFirebase Cloud Messagingが要求するAPIがブラウザにないぞと怒られています(当然ですね)。
[isSupported](https://firebase.google.com/docs/reference/js/messaging_.md?hl=ja#issupported)という関数が用意されているのでこれを使用して回避します。
戻り値がPromiseなので注意。

### @supabase/auth-helpersのcreateRouteHandlerSupabaseClientでセッションが取得できない

これはQiitaの方で書きました
https://qiita.com/sky_t/items/e41e46ea071a09c8dce1
