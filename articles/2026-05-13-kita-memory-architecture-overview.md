---
title: "Claude Code に長期記憶を持たせる自作ツール kita-memory の仕組み"
emoji: "🧠"
type: "tech"
topics: ["claudecode", "supabase", "pgvector", "mcp", "rag"]
published: true
---

Claude Code は会話を跨いだ記憶を持ちません。同じ説明を何度もする手間を減らすため、過去の会話を Supabase に貯めて検索できるようにする CLI を自作しています。名前は `kita-memory`。この記事は「中で何をしているか」を一通り解説する回です。

## 全体像

3 つのコンポーネントでできています。

- **save (CLI)**: Claude Code の会話ログを Supabase の `memories` テーブルに書き込む
- **consolidate (Edge Function)**: 貯まった生ログを Gemini Flash でセッション要約に変換し、ベクトル化する
- **MCP サーバー**: Claude Code から検索を呼べるようにする窓口

データの流れはこうなっています。

```
Claude Code (会話)
  └─ jsonl ファイル (~/.claude/projects/)
       └─ save: raw として INSERT (embedding は NULL)
            └─ consolidate (夜間): 要約 + embed → consolidated に昇格
                 └─ MCP search_memory: 次のセッションから検索可能
```

## なぜ「保存」と「ベクトル化」を分けたか

最初は「保存と同時に Gemini Embedding API を呼ぶ」素朴な作りでした。これが Gemini の無料枠（1,000 req/日）に当たると、保存自体が落ちます。記憶ツールの保存が落ちるのは致命的なので、責務を切りました。

- save 側は **外部 API を呼ばない**。`embedding = NULL`, `consolidation_status = 'raw'` で即 INSERT
- consolidate 側で **セッション単位**でまとめて Gemini に投げ、要約文だけを embed する

保存は常に成功し、ベクトル化は失敗しても翌日リトライすればいい、という非対称な信頼性設計になりました。

## consolidate の中身

`consolidate` は Supabase Edge Function として動いています。日次の `pg_cron` で叩かれます。

中で何をしているか:

1. `consolidation_status = 'raw'` の行を `session_id` 単位で取得
2. その会話列を Gemini Flash に渡し、「このセッションで決めたこと・分かったこと」だけを抽出させる
3. 抽出された memory それぞれを Gemini Embedding でベクトル化し、`consolidated` として INSERT
4. 元の raw 行は DELETE

ノイズ（「えーと」「ちょっと待って」のような会話の繋ぎ）が消えるので、検索ヒットの質が体感で上がります。

## 検索: ベクトル + 全文のハイブリッド

検索はベクトル類似度 1 本にしませんでした。日本語のクエリだと、固有名詞や型名のような「単語そのものが効く」検索が弱くなります。

そこで Supabase 上の RPC で、

- pgvector のコサイン類似度
- `pg_trgm` の trigram 一致

を別々にランキングし、**Reciprocal Rank Fusion (RRF, k=60)** で合流させています。

```
final_score = 1 / (60 + rank_vector) + 1 / (60 + rank_trigram)
```

ベクトル検索が拾えなかった「型名そのまま一致」を trigram が救い、trigram が拾えなかった「意味的に近い別の言い方」をベクトルが救う、という関係になります。日本語とコードが混ざるログには相性が良かったです。

## MCP として Claude Code に繋ぐ

Claude Code から呼ぶ口は MCP サーバーとして実装しました。バイナリ名は `kita-memory-mcp`、stdio 通信です。

公開しているツールは 2 つ:

- `search_memory(query, category?, project?, limit?)`: 上のハイブリッド検索を叩く
- `get_timeline(period, category?)`: 直近 24h / 前日 24h の会話を新しい順で返す

カテゴリは `work` / `personal` を CWD から自動判定しています。仕事のリポジトリで開いたセッションからは仕事の記憶だけ引ける、というシンプルな分離です。

**MCP は読み専用に倒しました**。保存系を MCP に持たせると、セッション中に Claude 自身が書き込めてしまい記憶を汚せます。書き込みは CLI / Edge Function 側に隔離しています。

## ハマりどころ

- **session_id を NULL にした旧データ**を引きずると、検索品質が落ちる。バックアップテーブルに退避してから削除しました
- **CWD ベースのカテゴリ判定**は、タスクスケジューラなど外側から save を呼ぶときに崩れやすい。`Push-Location` で CWD を切ってから渡します
- **embedding モデルの次元数**を後から変えるのは重い（既存ベクトルとの混在が壊れる）。最初に 768 次元で揃えて固定しました

## まとめ

- 保存と embedding を分けると、無料枠の外部 API に依存しても落ちなくなる
- ベクトル + 全文の RRF ハイブリッドで、日本語+コードのクエリに耐える
- Claude Code への口は MCP、書き込みは別バイナリ、と読み書きを分離する

このあと、Gemini 無料枠で詰んだ話と、Stop hook が不発になった話を別記事で書きます。
