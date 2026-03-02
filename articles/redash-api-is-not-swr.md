---
title: "RedashのAPIを見て「SWRみたいだ」と思ったけど違った話"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["redash", "swr", "cache"]
published: false
---

## はじめに

ハコベルではデータ分析ツールとして Redash を利用しています。
Redash はオープンソースの BI ツールで、SQL を実行・保存してダッシュボードの作成などができるツールです。

![Redash Dashboard](https://redash.io/assets/images/elements/query-editor-focus.png)
*こういうやつです。*

外部 API も提供されており、 リクエストを送るとクエリ結果などを取得することができます。

最近業務で Redash の API を触る機会があったのですが、あるエンドポイントの挙動を見てこう思いました。

> これ、SWR っぽくね？

* リクエストすると、すぐに何かしら返ってくる
* でも、その結果が必ずしも最新とは限らない
* そして裏でクエリが再実行されている

このような特徴が、フロントエンドのデータフェッチライブラリである `SWR` の挙動と似ていたからです。

ただ、ちゃんと仕様を追っていくと、それは少し違うということが分かってきました。

この記事では、

* stale-while-revalidate とは何か
* どこが決定的に違うのか
* Redash の API がなぜ SWR っぽく見えたのか

を整理して書いていきます。

## そもそも stale-while-revalidate とは

stale-while-revalidate とは、「古い（stale）データを一旦返しつつ、裏で再検証（revalidate）する」というキャッシュ戦略を指します。

これはフロントエンドのライブラリである SWR を指すものではなく、HTTP のキャッシュ制御として RFC 5861 [^1] で定義されている概念です。

RFC 5861 によると、`stale-while-revalidate` が指定されたレスポンスは、

* キャッシュが stale になっても一定時間は即座に古いレスポンスを返してよい
* その間にバックグラウンドで再検証を行う

という挙動を取ります。

重要なのは、「stale でも返す」と「裏で再検証する」がセットになっている点です。

例えばレスポンスに `Cache-Control: max-age=600, stale-while-revalidate=300` とあった場合、10分間は新鮮なレスポンスを返し、その後5分間は古いレスポンスを即座に返しつつ、裏で再検証を行うことができます。

![stale-while-revalidate](/images/stale-while-revalidate.png)

## フロントエンドの stale-while-revalidate

私が stale-while-revalidate を知ったのは、フロントエンドで Vercel 製のデータフェッチライブラリである `SWR` を使い始めたのがきっかけでした。
ハコベル配車管理システムのフロントエンドでも利用しています。

このライブラリでは、

* まずキャッシュされたデータを即座に返す
* その裏でデータを再取得する
* UI 側では「裏で再取得している」ことを表現できる（例: isValidating でスピナーを出す など）

といった挙動を、とても分かりやすい形で体験できます。

![Fetch and Revalidate](https://swr.vercel.app/_next/static/media/fetch-and-revalidate.ed3b70e4.svg)
*ポイントは図の2つ目のfetching中に再検証（isValidating）しつつキャッシュされたデータ（value）を返す点です [^2]*

この「とりあえず表示できるものを返す」「再検証は裏でやる」という感覚が身についていたので、Redash の API を見たときに無意識に SWR と結びつけてしまいました。

## Redash の API を触って「SWR っぽい」と思った瞬間

Redash でクエリ結果を取得する場合、次のような API を使います。

```
POST /api/queries/<query_id>/results
```

この API は、単に「最新の結果を返す」ものではありません。

* キャッシュされた結果があれば、それを返す
* キャッシュが古い場合は、クエリの再実行を開始する
* 再実行中であれば、ジョブ情報を返す

という挙動を取ります。

実際のレスポンスはこんな感じです。

キャッシュされた結果の場合
```json
[
    {
        "query_result": {
            "data": {
                "columns": [...],
                "rows": [...]
            },
            "retrieved_at": "2026-02-28T12:00:00.000+09:00"
        }
    }
]
```

ジョブが再実行中の場合
```json
[
    {
        "job": {
            "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
            "updated_at": 0,
            "status": 1,
            "error": "",
            "result": null,
            "query_result_id": null
        }
    }
]
```

図で示すとこのようになります。

![redash query api](/images/redash-query-api.png)

このレスポンスを見たとき、

* 古いかもしれない `query_result` が即座に返ってくる
* 同時に、裏でジョブが走っていることが分かる

という点が、SWR の体験と重なりました。

さらに、 max_age があると、より一層 SWR に見えます。
Redash の API では、`max_age` というパラメータを指定できるのです。

これは「この秒数より新しいキャッシュがあれば、それを使ってよい」という意味になります。

この `max_age` を見たとき、HTTP の `Cache-Control: max-age` と非常によく似ていると感じました。

* HTTP キャッシュ

  * `max-age` + `stale-while-revalidate`
* Redash API

  * `max_age` + バックグラウンド実行

構造だけを見ると、かなり近いことをやっているように見えます。

## 何が決定的に違うのか

### そもそもレイヤーが違う

これはお気づきかと思いますが、Redash の API は HTTP キャッシュの話ではありません。

Redash の API は、アプリケーションレベルでキャッシュをどう扱うか、クエリをいつ実行するかを定義しているものです。

これを前提として、レイヤーの違いを無視しても、それぞれが似て非なるものであることをここから説明します。

### Redash の API は stale なキャッシュを返さない

RFC 5861 にはこう書いてあります。

> caches MAY serve the response after it becomes stale

つまり、

> stale なキャッシュを返してよい

ということです。

一方 Redash は、キャッシュが max_age より古いと即座に再実行を開始し job を返すだけです。

stale なキャッシュを返すことはありません。

もう少し補足すると、Redash の API は「stale とみなす基準をクライアント側が max_age で指定する」という設計になっています。

つまり Redash の API はキャッシュと再検証の仕組みを取り入れているだけで、stale-while-revalidate とは似て非なるものだと言えます。

体験は似ているが、「stale なキャッシュを返す」わけではない点が違うところです。

### 解いている課題が違う

さらに言えば、両者が解いている問題自体も少し異なります。

stale-while-revalidate や SWR の主目的は、「ネットワークやサーバー遅延をユーザー体験から隠すこと」にあります。

多少古くてもいいから、とにかくブロックしないという戦略です。

一方 Redash は、「重いクエリ実行を非同期化すること」が主目的です。

クエリは数秒〜数十秒かかることもあるため、
同期的に待たせるのではなく、job を返して後から結果を取得します。

つまり、

SWR は「レイテンシ隠蔽の戦略」
Redash は「長時間処理の実行制御」

という目的の違いがあります。

## それでも似ている理由

それでも私が「SWR だ」と感じたのは、両者が同じ方向を向いて設計されているからだと思います。

* できるだけ早く何かを返す
* バックグラウンドで再検証する

この思想自体は、HTTP キャッシュでも、フロントエンドのデータフェッチでも、Redash のような BI ツールでも共通しています。

だからこそ、最初に見たときに「SWR っぽい」と感じたのだと思います。

そして今回いちばん面白かったのは、

> この思想を、REST API として実装する方法もあるのか

という気づきでした。

非同期 API が job を返すような設計自体は珍しくありませんし、キャッシュがあればそれを返すというのもよくあるパターンです。

ただ Redash は、

* cache があれば query_result を返す
* cache が古いなら job を返す

これらを API レスポンスの構造として表現しています。
つまり、「結果」だけでなく「状態」もリソースとして返しているという設計になっています。

私はこれを見て

> あ、そういう設計もありなのか

と思った次第です。

## まとめ

* Redash の API は、SWR のような体験を提供する
* しかし、それは RFC で定義された SWR そのものではない
* 正確には、アプリケーションレベルでのキャッシュと非同期処理の設計である
* ただし、「即座にフィードバックを返す」という設計思想は非常に近い

SWR だと思って違った、で終わるのではなく、
「なぜそう見えたのか」を考えると、設計の意図が見えてきて面白かったです。

## 参考

https://redash.io/help/user-guide/integrations-and-api/api/

[^1]: https://datatracker.ietf.org/doc/html/rfc5861
[^2]: https://swr.vercel.app/docs/advanced/understanding
