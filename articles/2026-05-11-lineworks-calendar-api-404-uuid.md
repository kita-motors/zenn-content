---
title: "LINE WORKS Calendar API 2.0 で 404 Api not exists が出るときに見直す3つの仕様差"
emoji: "🛠"
type: "tech"
topics: ["lineworks", "gas", "googleappsscript", "api", "業務自動化"]
published: false
---

## はじめに

Google Workspace（GWS）のカレンダー予定を LINE WORKS（以下 LW）のカレンダーに片方向同期する Google Apps Script（GAS）を書いていて、JWT 認証は通っているのに予定 POST だけが `404 NOT_FOUND: Api not exists` で蹴られ続ける、という現象に丸一日ハマりました。

公式リファレンスを読んでもサンプルコードがあるわけでもなく、検索しても旧 API（Bot 用の Calendar API）の話ばかり出てきて、API 2.0 の Calendar 系については情報が薄かったので、原因と直し方を記録に残します。

## ハマっていた状況

構成はざっくり以下のとおりです。

- GAS から GWS の Service Account を JWT + DWD（ドメインワイド委任）で叩いて `events.list` を取得
- 取得した予定を LW Calendar API 2.0 へ `POST` して、ユーザーごとの LW カレンダーに転記
- LW 側の認証は Service Account（Client ID + Service Account ID + Private Key）で JWT を作って `access_token` を取る方式

JWT 署名のエラー（`Invalid argument: key`）はクリア済みで、`access_token` も問題なく取得できている状態。それでも予定 POST だけが 404 で返ってくる、というのが今回の起点でした。

返ってきていたエラーはこれです。

```json
{ "code": "NOT_FOUND", "description": "Api not exists" }
```

`Api not exists` という文言が地味に厄介で、「URL が間違っている」ではなく「そんな API は存在しません」と言われているように見えてしまうため、エンドポイントそのものを疑い始めるまでに時間がかかりました。

## 原因は3つ重なっていた

最終的に動いた状態と比較したら、間違っていたポイントが3つありました。1つではなく **3つ重なっていた** のがハマった原因です。

### 1. userId はメールアドレスではなく UUID

ここが本丸でした。LW の Calendar API 2.0 では、URL の `{userId}` 部分に **UUID 形式の userId** を入れる必要があります。LW の管理画面で見えるメールアドレス形式の ID（`user@example.co.jp`）を入れていると、たとえアプリのスコープと認可が正しくても 404 になります。

UUID は管理画面では普段見えないので、`/users` エンドポイントを叩いて一覧から取ってきて、自前で Spreadsheet にダンプして突き合わせる、という運用にしました。注意点として `/users` を呼ぶには `user.read` スコープがアプリに追加されている必要があります。`calendar` スコープだけでは取れません。

### 2. エンドポイントは `/calendar/events` で `defaultCalendar` は不要

私が最初に書いていた URL は `…/calendar/defaultCalendar/events` のような形でした。これも 404 の一因です。

実際に動いたのはこちらです。

```
POST https://www.worksapis.com/v1.0/users/{userId}/calendar/events
```

`defaultCalendar` をパスに含める必要はありません。

### 3. ボディは `eventComponents` 配列でラップする

POST するイベント本体も、単独のオブジェクトを渡すのではなく `eventComponents` という配列にラップする形式でした。

```json
{
  "eventComponents": [
    {
      "eventId": "gws-xxxxxxxxxxxx",
      "summary": "テスト予定",
      "start": { "dateTime": "2026-05-11T10:00:00+09:00" },
      "end":   { "dateTime": "2026-05-11T11:00:00+09:00" }
    }
  ]
}
```

URL とボディ形式の両方が違っていたので、片方だけ直してもエラー文言は変わらず、原因の切り分けがしづらかったというのが正直なところです。

## 動いたときのログ

3点を直して再実行したところ、ようやく `201 Created` が返ってきました。

```
[LW POST] url=https://www.worksapis.com/v1.0/users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/calendar/events
[LW POST] body={"eventComponents":[{"eventId":"gws-xxxxxxxxxxxx", ... }]}
[LW POST] status=201
```

`Invalid argument: key` → `Api not exists` → `201` と段階的に進んでいったのが、結果的には切り分けの助けになりました。

## ハマりどころ・注意点

実装中に踏んだ罠で、検索でなかなか出てこなかったものを残しておきます。

- **エラー文言で原因を断定しない**。`Invalid argument: key` は GAS の `Utilities.computeRsaSha256Signature()` が PEM をパースできないときに出る固有のメッセージなので、ここで詰まっているうちはまだ POST にすら到達していません。userId の形式違いだと、もっと先で `401` や `404` の別文言になります。順番をひっくり返さないことが大事です。
- **PEM を Script Properties に貼ると改行が落ちる**。これも JWT 署名失敗の常連原因なので、私は事前に PEM を `\n` リテラルに置換して 1 行化し、GAS 側で改行へ戻すユーティリティを噛ませることにしました。
- **`updatedMin` を使う差分同期は、失敗時にも `LAST_RUN` を更新しないこと**。これは Calendar API そのものの話ではないですが、初回失敗 → 2回目「変更なしで 0 件」となって「動いてるように見えるけど何もしていない」状態を作り出して、デバッグが余計にこじれました。失敗時はタイムスタンプを進めない実装にしてあります。
- **API 2.0 で `/users` を呼ぶには `user.read` スコープ**。Calendar 同期目的だけだと `calendar` スコープしか入れない設計にしがちですが、UUID を取り出すために結局 `user.read` も要ります。

## まとめ

LW Calendar API 2.0 の `Api not exists` は、(1) userId が UUID 形式でない、(2) URL に余計な `defaultCalendar` が入っている、(3) ボディが `eventComponents` でラップされていない、の **どれかが欠けても出る** という印象です。私の場合は3つ全部に当たっていて、エラー文言は同じまま中身の原因が入れ替わっていたので、切り分けが余計に長引きました。

同じ箇所で詰まっている人の手がかりになれば幸いです。
