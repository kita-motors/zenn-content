# zenn-drafts

Zenn（zenn.dev/kitamotors）の記事リポジトリのローカル。GitHub: `kita-motors/zenn-content`（Public）と双方向同期。push すると Zenn が自動デプロイする。

## 著者ペルソナ

- 一人称「私」
- 中小企業の事務・管理部門に所属する**非エンジニア**
- Claude Code を AI 共同作業者として使う実践者
- 口調: です・ます調、丁寧だが堅すぎず、技術ブログとして読みやすく
- 二本柱: ① Claude Code / AI エージェント運用、② 業務自動化（GAS / API 連携 / ブラウザ拡張）

## 記事フォーマット

frontmatter のテンプレート:

    ---
    title: "（30〜45文字目安、検索流入と気持ちに刺さるバランス）"
    emoji: "🛠️"
    type: "tech" または "idea"
    topics: ["claudecode", "..."]
    published: false
    ---

- ファイル名（slug）: 12〜50字、英数字 + ハイフン
- topics は最大5個
- published は **必ず false** で書き始め、レビュー後に true へ

## 公開フロー

1. `articles/<slug>.md` を作成・編集
2. `npx zenn preview` でローカル確認（http://localhost:8000）
3. `published: true` に切り替えてコミット
4. `git push origin main`（user が手元で実行）
5. Zenn 自動デプロイで数分以内に反映
6. X 投稿用テンプレを Claude Code 側で生成して、user が手動投稿

## マスキング（このリポジトリは Public）

記事・README・コミットメッセージにも以下は出さない。判断に迷ったら**伏字側に倒す**。

- 業務系の社内ツール名・コードネーム
- 顧客名・社内ドメイン・社内メアド
- API キー・トークン・パスワード様の文字列
- 個人名・社内チャットのハンドル
- 業務で使っている特定 SaaS の固有名は、文脈で個人特定に繋がりそうなら一般化（例: ジョブカン → クラウド勤怠 SaaS）

## 既出記事

- `prologue-claude-code-business-automation.md` — プロローグ宣言記事（type=idea、published）
- `kita-memory-mcp-claude-code-long-term-memory.md` — 第1弾、MCP サーバー体験談（type=idea、published、Zennfes Spring 2026 応募）

## zenn-drafter エージェント

`~/.claude/agents/zenn-drafter.md` に配置したサブエージェント。kita-memory MCP の work カテゴリ記録から技術記事下書きを生成する。明示呼び出しのみ起動。重めの作業が一区切りついたら「回しますか？」と user に提案する運用。
