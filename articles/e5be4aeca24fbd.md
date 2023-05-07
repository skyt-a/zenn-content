---
title: "Next.js13のapp directoryでチケット共有アプリを作ってみた"
emoji: "🎫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react", "vercel", "firebase", "個人開発", "typescript", "firebasecloudme"]
published: true
---

## はじめに
今回GWという時間を利用して、`Katacky`というチケット共有アプリを開発しました。

## アプリ概要

https://katacky.xyz/

ソースはこちら
https://github.com/sky-a/Katacky

家族やカップル、友達の間で使えるオリジナルチケットを作成、譲渡、使用できるWebアプリです。
スケジュール設定をすることであるユーザーに月に2枚チケットを配布するみたいなこともできます。
サービス名の`Katacky`というのはこういう身内だけで運用するチケットといえば、肩叩き券だなと思ったのでそこから考えました。
ちなみに`Katacky`のロゴは[BRANDMARK](https://brandmark.io/)というサイトで作成したものです。


![](/images/e5be4aeca24fbd/ticket.png =250x)
*こんな感じのチケットが作れます*

### なぜ作ったのか
彼女とダイエットに勤しんでいまして、ただ月に何回かはチートデイ的に1食好きなものを食べる権利をお互い持っているのですが(彼女は2回、私は1回)、この権利を行使したことをDiscordに記録していました。
これアプリ化したいなーと思っていたので今回作ってみました。

## 技術スタック

### フロントエンド + APIサーバー
アプリを作るにあたって、フレームワーク・UIライブラリを何にするかは悩みましたが、今回は`Next.js` 13で登場した`app directory`を使ってみたいなと思ったので`Next.js`に決定しました。
（`Remix`や`Nuxt`、`Solid.js`、`SvelteKit`あたりが他の候補でした)
ちょうどGW中にGAになりましたね。
https://nextjs.org/blog/next-13-4
またAPIサーバーとフロントのインターフェースには[tRPC](https://trpc.io/)を採用しています。

### スタイリング・UIフレームワーク
普段は個人的に`emotion`や`styled-components`といったCSS in JS系のライブラリを使うことが多いのですが、今回`app directory(React Server Component)`を使うということでこれらのライブラリは相性が悪いため、初めて巷で話題の`TailwindCSS`に触れてみることにしました。
またUIフレームワークとして[shadcn/ui](https://github.com/shadcn/ui)というフレームワークを採用しています。
これは`Radix UI`というHeadlessなUIフレームワークと`TailwindCSS`を組み合わせたもので、カスタマイズが容易そうだったので採用しました。
[Mantine](https://mantine.dev/)も良さそうだったんですが、`emotion`依存が気になったので今回は見送りました。
(ただ`Mantine`は今後`emotion`依存をやめる可能性があるようです: [参考](https://github.com/mantinedev/mantine/issues/2815#issuecomment-1509758301))

### 認証
認証は`Firebase Auth`のメール＋パスワード認証を採用しました。(今後他のプロバイダーも追加していく予定です)
またNext.jsと合わせて簡単に認証情報を利用するために`Auth.js(NextAuth.js)`も採用しています。

### ホスティング
ホスティング先は`Next.js`との相性の良さから`Vercel`にしました。
`Firebase`を使っているので、[Firebase Hosting](https://firebase.google.com/docs/hosting/frameworks/nextjs?hl=ja)もありかなと思ったんですが、まだまだ安定版ではないようなので今回は見送りました。

### データベース
今回は[PlanetScale](https://planetscale.com/)を採用しています。
Firebaseに依存しているから`FireStore`もありかなと思ったんですが、個人的には`RDB`の方が好みなので今回は採用を見送りました。
ちなみに開発当初はFirebaseではなく、[Supabase](https://supabase.com/)を使っていて、DBも`Supabase`の`PostgreSQL`を使用していたのですが、後述する`Firebase Cloud Messaging`が使いたくなった関係で`Supabase`依存をやめ、`PlanetScale`へ移行することになりました(Authも合わせて移行しました)。
技術選定の前にちゃんと要件固めとかないと。。。
またORマッパーとして[Prisma](https://www.prisma.io)を採用しています。

結果的に[T3Stack](https://create.t3.gg/en/introduction)と呼ばれる構成になりました。

## こだわりポイント

### グループ登録をQRコードでできるようにした
このアプリではグループという概念があり、ユーザーは同じグループのユーザーにのみチケットを譲渡できるようになっています。
このグループはユーザーが自由に作成できるのですが、他のユーザーが作ったグループにどうやって参加できるようにするか悩みました。
メールアドレスを入力させて招待メールを送る動線も考えたのですが、アプリの想定用途的にも家族やカップルなど直接顔を合わせるようなメンバーでグループを作ることが考えられるので、この動線ではユーザーにとって少しめんどくさいだろうなと考えました。
LINEの友達登録のようなQRコードでの登録の方が今回のアプリ的にはベターだと考えられるので、今回はこちらを採用しました。
QRコードの読み取りは[jsqr](https://github.com/cozmo/jsQR)、作成・表示は[next-qrcode](https://github.com/Bunlong/next-qrcode)を使用しています。
![](/images/e5be4aeca24fbd/group.png =250x)

### チケットの受け取りをプッシュ通知するようにした
`Katacky`ではチケット受取時のプッシュ通知に対応しています。
これまでWebのプッシュ通知はiOSでは対応していなかったのですが、ついに[iOS 16.4](https://webkit.org/blog/13966/webkit-features-in-safari-16-4/)でSafariでもプッシュ通知が利用できるようになったこともあって実装にチャレンジしました。
プッシュ通知の送信は[Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging?hl=ja)を使用して行っています。
![](/images/e5be4aeca24fbd/notice.jpg =250x)

### PWA対応
プッシュ通知を実現したかったこともあり、`PWA`に対応しています。
ぜひ`ホーム画面に追加`して使ってみてください！
![](/images/e5be4aeca24fbd/icon.jpg =250x)
*ちゃんとアイコンも表示されます*

### チケットをスケジュール発行できるようにした
私と彼女の要件だと月に1回チケットを私は1枚、彼女は2枚発行する必要があるのですが、毎月手作業でやるのは面倒くさいと思ったのでスケジューリング機能をつけて自動化しました。
スケジューリングは`Vercel`を使っていることもあり、[Vercel Cron](https://vercel.com/blog/cron-jobs#limits-of-cron-jobs-and-vercel-functions)を採用してみました。
実行したいGET API(今回は[Route Handler](https://nextjs.org/docs/app/building-your-application/routing/router-handlers)を使用しています)と`vercel.json`を用意し、以下のように記載することで簡単にスケジューリングできました。

```json:vercel.json
{
  "crons": [
    {
      "path": "/api/ticket/schedule",
      "schedule": "0 5 * * *"　// 毎日AM5:00に↑のAPIが呼び出される
    }
  ]
}

```

ただ↓にあるように
https://vercel.com/blog/cron-jobs#limits-of-cron-jobs-and-vercel-functions
>While in beta, Vercel Cron Jobs are free on all plans. However, it'll be a paid feature for general availability.

とのことなので、ベータが外れた暁には無料で使えなくなる可能性が高そうですね。

## 開発してみて感じたこと

## app directory(Next.js)
app directoryについての詳しい説明は他に素晴らしい記事がいくらでもあるので割愛します。
とりあえず[公式ドキュメント](https://nextjs.org/docs)に一通り目を通して、あとはトライアンドエラーの精神で頑張ろうと意気込みました。
正直まだまだちゃんと使いこなしているとはいえず、どういう設計がうまく合うのかはこれからも模索していく必要がありそうです。
サーバーサイドのキャッシュ戦略がまだよくわかっていないところもあるのでちゃんとキャッチアップしたいですね。

### Server Component内で非同期処理をasync-awaitで書けるのは素晴らしい
こんな感じで書けて非常に直観的。個人的には`getServerSideProps`の中で書くよりもぱっと見でやりたいことがわかりやすいです。
```ts
export const Profile = async () => {
  const user = await getUserInfo();
  if (!user) {
    return null;
  }
  ...
```

### 状態管理やデータキャッシュ周りはクライアントからサーバーサイドに寄せるような構成が主流になりそうな雰囲気
基本的にAPIからのデータのfetchもServerComponent内でやることが多く、`ReactQuery`や`SWR`でやっていたようなクライアントでのデータキャッシュみたいな機構がそこまで必要じゃなくなって来るのかなと感じました。
今回ぐらいのシンプルなアプリだと管理したいようなステートもそこまで多くなく、今回は特にクライアント側の状態管理系のライブラリは使用していません。

### バグのような挙動もいくつか見られた
ベータ版(GW中にGAになりましたが)ということもあり、ところどころ動作が不安定になることがあります。
謎にエラーが発生したと思いきや、何もせずにもう一度ページを開くと直ったり、実装が悪いのか何かしらのバグなのかわからない状態で開発を進めるシーンが何回かありました。
ただこれは今後改善されていくはずなので安定するのを期待して待ちたいところです。
いくつか気になった事象は後述しています。

### 周辺ライブラリがまだ対応できていないことが多い
これも前述した通りまだまだ出たばかりなので仕方のないことですが、周辺ライブラリが対応できないことが多く、公式のドキュメント通りに実装してもうまく動かないことが何回かありました。
例えばtRPCは[公式のインテグレーションガイド](https://trpc.io/docs/nextjs/setup)通りに進めると、クライアントからのAPI呼び出しでエラーが起きました。
根本的な原因はわからないですが、今回は`createNextApiHandler`を使うのでなく、以下のように`fetchRequestHandler`を使うことでなんとか動くようになりました。

```ts:app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from "@trpc/server/adapters/fetch";
import { appRouter } from "~/servers";

const handler = (request: Request) => {
  return fetchRequestHandler({
    endpoint: "/api/trpc",
    router: appRouter,
    ...
  });
};

export const GET = handler;
export const POST = handler;
```

## TailwindCSS
開発前には個人的には`class`属性がやたら長くなることと、実際のCSSのプロパティと`TailwindCSS`で設定するクラス名に結構ギャップがあって都度調べないといけないところが気になっていました。
ただ実際に開発で使ってみると見えていなかったいいところもありました。

### 簡単に所感
・`React`のようなコンポーネント単位でスタイルが閉じるような環境では`class`属性の長さはあまり気にならない
・[VSCodeの拡張機能](https://marketplace.visualstudio.com/items?itemName=bradlc.vscode-tailwindcss)である程度補完は効く(なぜかたまに効かないことがある？)
・ダークモード対応が楽！
・今回あまり使いこなせていないが、デザインシステムをかっちり構築したいようなアプリではかなり重宝しそう

## 開発中遭遇したエラーや問題

開発中遭遇したエラーや問題をいくつか書いておきます。
同じ事象に遭遇した方の助けになれば幸いです。

### You're importing a component that needs useState. It only works in a Client Component but none of its parents are marked with "use client", so they're Server Components by default.

何回も見たエラーです。
サーバーコンポーネントでクライアントコンポーネントでしか使えないモジュールをimportした時に発生します。
特に考えさせられたのはutilファイルみたいなものを作成しようとしたとき。
個人的にはhooksと純粋な関数や定数みたいな分け方より、`utils/form.ts`みたいな機能ベースでファイルを作ることが多いんですが、こうするとサーバーコンポーネントでも使えるモジュールとクライアントコンポーネントでしか使えないモジュール(hooksなど)が混在してしまい、よくこのエラーで怒られていました。
共通モジュールはクライアントでしか使えないものとそれ以外でファイルベースで分離することが求められそうです
これに限らず今までのやり方をそのまま踏襲すると問題が起きるので、app dirの気持ちになることが大事ですね。。。

### unhandledRejection: Error [FirebaseError]: Messaging: This browser doesn't support the API's required to use the Firebase SDK. (messaging/unsupported-browser).

Firebase Cloud Messaging関連のモジュール実装時に発生した問題です。
サーバーサイドでのレンダリング時にFirebase Cloud Messagingが要求するAPIがブラウザにないぞと怒られています(当然ですね)。
[isSupported](https://firebase.google.com/docs/reference/js/messaging_.md?hl=ja#issupported)という関数が用意されているのでこれを使用して回避します。
戻り値がPromiseなので注意。

### @supabase/auth-helpersのcreateRouteHandlerSupabaseClientでセッションが取得できない

これはQiitaの方で書きました
https://qiita.com/sky_t/items/e41e46ea071a09c8dce1
が、結局supabaseはやめてfirebaseに切り替えました。

### TypedRouteが正しいパスを指定してもエラーになる
[Next.js@13.2](https://nextjs.org/blog/next-13-2#statically-typed-links)で`Typed Link`がベータ提供されるようになったので嬉々として利用していたんですが、たまに正しいパスを指定しているのにTypeScriptエラーが出ることがありました。
この場合はVSCodeのTypeScriptサーバーを再起動すると直りました。まだベータだからなのか私の環境に問題があるのかは不明です。
機能としては素晴らしいので今後のアップデートに期待です。

## おわりに
`app directory`は個人的には総合的に見れば開発体験上がった感覚ですし、GAになったことで今後さらに重要になっていくことでしょう。
いくつか見えてるバグはあるんですが、ゴールデンウィークの短い期間での実装でなんとか形にできてよかったです(`WakaTime`によると大体40時間くらいで実装できたみたいです)。
この後は時間見つけて[astro](https://astro.build/)使ってLP作ってみたり、機能追加していきたいです。
`app directory`はつい先日GAになったばかりですが、最近のアップデートを見るに力の入れようが尋常じゃないように見えるので今後どんな機能がリリースされるのか楽しみですね。
今回のアプリもなんとかNext.js側のアップデートについていきたいところです。

`Katacky`、よければ使ってみてください！
https://katacky.xyz/
