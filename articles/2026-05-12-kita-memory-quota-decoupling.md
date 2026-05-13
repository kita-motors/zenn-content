---
title: "Claude Code の会話を Supabase に保存していたら Gemini 無料枠で詰んだ話"
emoji: "🪫"
type: "tech"
topics: ["claudecode", "supabase", "gemini", "pgvector", "pgcron"]
published: false
---

Claude Code の会話を Supabase + pgvector で長期記憶化する CLI（`kita-memory`）を運用しています。Stop hook で会話を取り、Gemini Embedding API でベクトル化して `memories` テーブルに INSERT する、という素朴な構成でした。

ある日から保存がぽろぽろ落ち始め、原因は Gemini Embedding API の無料枠 1,000 req/日 でした。設計を直したので記録しておきます。

## 何が壊れていたか

hook が走るたびに、以下を同期実行していました。

1. 直前の Q&A を Gemini Embedding API でベクトル化
2. ベクトル + 本文を `memories` に INSERT

長時間のセッションを回した日に、後半の会話が `429 RESOURCE_EXHAUSTED` で軒並み失敗。検索 RPC からも見えなくなりました。記憶 CLI で記憶が抜ける状態です。

## 設計上の問題

**embedding を「保存」の必須経路に置いていた** のが筋が悪い。クォータが尽きた瞬間に保存自体が止まります。

責務を 3 つに分けました。

- **save**: hook から呼ばれる即時 INSERT。`embedding = NULL`, `consolidation_status = 'raw'`。外部 API 呼び出しゼロ
- **consolidate**: セッション単位で raw を Gemini Flash に要約させ、要約文だけを embed して `consolidated` に昇格、raw は DELETE
- **検索 RPC**: `consolidation_status = 'consolidated'` のみを対象

これで hook 側はネットワーク I/O が消え、クォータ問題も「夜間にまとめてやる」だけになりました。

## 自動化: Edge Function + pg_cron

consolidate を手元 PC の cron で回すと、PC を閉じている日に溜まります。Supabase 側に寄せました。

- consolidate ロジックを Supabase Edge Function 化
- `pg_cron` + `pg_net` で日次 HTTP POST
- クォータが回復する JST 03:00 に発火

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

デプロイ時は `--no-verify-jwt` を付けないと、`pg_cron` から呼ぶときに JWT 検証で 401 になります。

```bash
npx supabase functions deploy consolidate --no-verify-jwt
```

## ハマりどころ

- **`pg_cron` は UTC**。JST 03:00 で動かしたいなら `0 18 * * *`。一度ミスりました
- **`pg_net` は `extensions` スキーマで明示有効化**しないと `net.http_post` が見えない
- **`--no-verify-jwt` の付け忘れ**で `cron.job_run_details` に 401 が並ぶ。`select status, return_message from cron.job_run_details order by start_time desc limit 5;` で確認

## 副次効果

要約済みデータだけを検索対象にしたら、ヒット品質が上がりました。raw のままだと「えーと」「ちょっと待って」のようなノイズまでベクトル化されていて、類似検索の上位を埋めることがあったのですが、要約を一段噛ませると検索対象がセッションのサマリに揃います。

raw は consolidate 時点で削除しています。元の jsonl は `~/.claude/projects/` に残るので、再 save → 再 consolidate でやり直せます。

## まとめ

無料枠の外部 API は「基本動作の必須経路」に置かない。保存と embedding を分離し、重い処理は pg_cron で夜間に逃がす。これだけで Free tier でも安定して回ります。
