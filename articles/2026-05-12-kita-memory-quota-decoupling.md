---
title: "Claude Code の会話を Supabase に長期記憶していたら Gemini 無料枠で詰んだ"
emoji: "🪫"
type: "tech"
topics: ["claudecode", "supabase", "gemini", "pgvector", "pgcron"]
published: true
---

Claude Code の会話を Supabase + pgvector で長期記憶化する CLI（kita-memory）を自作して運用しています。Stop hook で Q&A をベクトル化して保存する素朴な構成だったのですが、ある日から会話の保存がぽろぽろ落ち始めました。原因は Gemini Embedding API の無料枠 1,000 req/日にぶつかっていたこと。「保存」と「embedding」を hook の中で密結合させていた設計の素朴さが祟った形です。本記事では、これをどう設計し直したかを、非エンジニアの私が試行錯誤した目線で書きます。同じく Claude Code の会話保存を自作している方の参考になれば幸いです。

## 何が起きていたか

私の構成では、Claude Code の Stop hook が走るたびに以下が同期実行されていました。

1. 直前の Q&A を Gemini Embedding API に投げてベクトル化
2. 返ってきたベクトルと本文を `memories` テーブルに INSERT

普段は問題なかったのですが、Claude Code を長時間回した日に、後半の会話が `429 RESOURCE_EXHAUSTED` で軒並み保存失敗していました。検索 RPC からも当然見えません。記憶の CLI なのに記憶が抜ける、というシャレにならない状態です。

ちなみにこの記事を書いている最中にも、自分で記憶を検索しようとして 429 を踏みました。題材としては説得力がありますが、笑えない。

## 設計上の間違い

ふり返ってみると、hook の中で embedding まで完結させていたのが筋が悪かったです。embedding は外部 API への呼び出しでクォータも遅延もあるのに、それを「保存」という基本動作の必須経路に置いていました。クォータ尽きる = 保存できない、と直結している以上、無料枠で運用する限り遅かれ早かれ詰む構造です。

ここから「**保存と embedding は分離すべき**」というところに行き着き、3 つの責務に切り直しました。

- **save**: hook から呼ばれる即時 INSERT。embedding は NULL のまま。クォータ消費ゼロ
- **consolidate**: セッション単位で raw 群を Gemini Flash に要約させ、要約文だけを embed して `consolidated` 状態に昇格
- **検索 RPC**: `consolidation_status = 'consolidated'` のみを対象にする（要約前の生ログでノイズを増やさない）

これだけで hook 側はネットワーク I/O が一気に減り、体感でも目に見えて軽くなりました。

## 自動化: ローカル cron から Edge Function + pg_cron へ

最初は consolidate コマンドを手元の PC で日次実行するつもりでした（Phase A と呼んでいたもの）。実際これだけでもクォータ問題自体は解消します。ただ、PC を閉じている日は処理が走らず溜まる一方になるので、結局 Supabase 側に寄せました。

consolidate ロジックを Edge Function 化し、`pg_cron` + `pg_net` で日次 HTTP POST を打つ構成です。クォータが回復する JST 03:00 ちょうどに走らせています。

```sql
select cron.schedule(
  'consolidate-daily',
  '0 18 * * *',  -- UTC 18:00 = JST 03:00
  $$
  select net.http_post(
    url     := 'https://<project-ref>.supabase.co/functions/v1/consolidate',
    headers := jsonb_build_object(
      'Authorization', 'Bearer <cron-shared-secret>',
      'Content-Type',  'application/json'
    ),
    body    := '{}'::jsonb
  );
  $$
);
```

`pg_cron` は UTC で動くので、JST に直すときの時差換算で一度ミスりました。あと `pg_net` は拡張を `extensions` スキーマで明示有効化しないと関数が見えません。

これで PC の電源状態と無関係に毎晩消化されるようになり、hook 側の責務も「INSERT するだけ」に痩せたままです。

## ハマりどころ

副次効果として、検索 RPC が要約済みデータだけを見るようになったので、ヒット品質が上がりました。raw のままだと「えーと」「ちょっと待って」みたいな会話のノイズまでベクトル化されてしまっていて、類似検索の上位を埋めることがあったのです。要約を一段噛ませると、検索対象が「そのセッションで何が起きたか」のサマリに揃うので、想像していたより検索精度の改善幅が大きかったです。

逆に潔く割り切ったのは、要約に昇格した時点で raw を削除する設計にした点です。元のセッション jsonl は `.claude/projects/` 配下にローカルで残っているので、生発言を辿りたいときはそちらを見ればよく、DB に二重持ちする意味は薄いと判断しました。要約が気に入らなければ jsonl を再 save → 再 consolidate でやり直せます。

## まとめ

無料枠で外部 API を使うときは、その API 呼び出しを「基本動作の必須経路」に置かない、という当たり前のことを身をもって学びました。保存と embedding を分離し、重い処理を pg_cron で夜間に逃がす。これだけで、非エンジニアの個人プロジェクトでもそれっぽい構成になります。Claude Code の会話保存を自作している方は、最初から二段階で設計しておくとあとが楽だと思います。
