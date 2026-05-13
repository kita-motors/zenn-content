---
title: "Claude Code の Stop hook が不発になったので、タスクスケジューラから外側で保存させた話"
emoji: "🛡"
type: "tech"
topics: ["claudecode", "windows", "powershell", "taskscheduler", "mcp"]
published: false
---

Claude Code の会話を Supabase に保存する自作 CLI（`kita-memory`）を Stop hook 経由で動かしていたら、ある日から hook が発火しなくなりました。半日 hook 側を追ったあと、「hook を直すより外側から定期保存する」方針に切り替えたので記録しておきます。

## 観測できた事実

- `~/.claude/settings.local.json` に Stop hook 設定は入っている
- Claude Code 内の `/hooks` ダイアログには登録された hook として表示される
- 新しいセッションを開始しても save が走らない（保存先 DB が増えない）
- SessionStart hook も同様に発火しない

設定が消えたわけでも JSON が壊れているわけでもなく、**UI 上は登録されているのに実行されない** という見た目と実態のズレが厄介でした。

## 切り分けは詰んだ

save コマンドの入口に「呼ばれたらタイムスタンプをファイルに書く」を仕込んで再現してみたところ、**ファイルが一行も増えませんでした**。hook の入口にすら到達していない。

ここで Claude Code 本体側を追うか、外側で殴るかの判断になりました。会話 jsonl は `~/.claude/projects/` に書き出されているので、後から拾えば済みます。hook 本体の修正は諦めました。

## 外側から殴る

タスクスケジューラに PowerShell スクリプトを登録して、

- **10 分おき** に実行
- **At LogOn** で実行

の 2 トリガーで `~/.claude/projects/` 配下の jsonl を拾わせます。

```powershell
# scripts/autosave.ps1（抜粋）
$projectsRoot = Join-Path $env:USERPROFILE ".claude\projects"
$cutoff = (Get-Date).AddHours(-24)

Get-ChildItem $projectsRoot -Recurse -Filter *.jsonl |
  Where-Object { $_.LastWriteTime -gt $cutoff } |
  ForEach-Object {
    $first = Get-Content $_.FullName -TotalCount 1 | ConvertFrom-Json
    if ($first.cwd -and (Test-Path $first.cwd)) {
      Push-Location $first.cwd
      try {
        kita-memory save $_.FullName
      } finally {
        Pop-Location
      }
    }
  }
```

ポイントは jsonl の先頭行から `cwd` を読み出して `Push-Location` で切り替えること。`kita-memory` は CWD を見て work / personal カテゴリを判定するので、`C:\Windows\System32` のまま実行すると分類が崩れます。

## 重複保存を恐れなくていい設計

複数経路から同じ jsonl を投げると、当然同じ Q&A が二重 INSERT されかけます。これは `memories` テーブルに **本文 hash の UNIQUE 制約** を張ってあるので、二度目以降は静かに弾かれます。

```sql
alter table memories add column content_hash text;
create unique index memories_content_hash_uniq on memories(content_hash);
```

冪等性が保証されているので、hook が復活した日にタスクスケジューラを止め忘れても DB は壊れません。「何重に保険を張ってもよい」という構造が、今回の方針転換を支えてくれました。

## ハマりどころ

- **`/hooks` の表示は信用しない**。「登録されている = 実行されている」ではない。入口に必ずログを仕込む
- **CWD ベース判定の罠**。タスクスケジューラから呼ぶときは作業ディレクトリを意識しないと、分類が全部 personal や work の片方に寄る
- **stdin から transcript_path を受ける hook の書き方**。Claude Code の hook は引数ではなく stdin に JSON を渡してくる。PowerShell なら以下のようになる

```powershell
powershell -Command "$input | ConvertFrom-Json | Select-Object -ExpandProperty transcript_path | xargs kita-memory save"
```

## まとめ

- Stop hook は「動けば便利」の位置づけにして、永続化を一元化しない
- jsonl は `~/.claude/projects/` に残るので、外側から拾えば hook に依存しなくていい
- 本文 hash UNIQUE を張っておくと、複数経路の重複保存が無害化される
