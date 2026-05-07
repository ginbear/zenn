---
title: "[小ネタ] Claude Code の GitHub MCP をやめて gh CLI に統一した"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "mcp", "github", "cli"]
published: false
publication_name: "atrae"
---

## きっかけ

Claude Code で GitHub の操作をするとき、MCP ツールがエラーになって `gh` CLI にフォールバックしている様子をよく見かけていた。調べてみたところ、GitHub MCP に設定している Personal Access Token (PAT) が期限切れになっていた。
`claude mcp list` のヘルスチェックでは `✓ Connected` と表示されるため、使えていなかったことに気づくのが遅れてしまった。
PAT を再発行すれば直るが、そもそも `gh` CLI で問題なく動いていたので、GitHub MCP が必要なのか調べてみた。

## gh CLI と比べてみた

MCP と CLI のトークン消費差について、[ScaleKit のベンチマーク](https://www.scalekit.com/blog/mcp-vs-cli-use)が参考になった。GitHub Copilot MCP Server を対象にした計測では、最も単純なタスク（リポジトリの言語判定）で CLI が 1,365 トークンに対し MCP は 44,026 トークンと、約 32 倍の差が出ている (そんなに差が出る？という感想)。GitHub MCP は 43 のツール定義を毎回コンテキストに含める必要があり、このスキーマオーバーヘッドが主因とされている。ただし同記事はマルチテナント環境では MCP の認証・監査機能に価値があるとも述べており、一概に CLI が優位とは結論づけていない。

[Mario Zechner のベンチマーク](https://mariozechner.at/posts/2025-08-15-mcp-vs-cli/)でも MCP と CLI を比較しているが、こちらの結論は「プロトコルの違いよりツール設計の質が重要」というもので、MCP 自体を否定する内容ではない。

運用面としては、GitHub MCP は PAT を手動管理する必要があり、期限切れに気づきにくい（冒頭の体験がまさにこれ）。一方 `gh` CLI は OAuth ベースで `gh auth login` するだけで済む。便利。

## 結論

自分の環境では `gh` CLI に統一することにした。`gh api` で全 GitHub API にアクセスできるのでカバー範囲も MCP より広い。[Claude Code の Best Practices](https://code.claude.com/docs/en/best-practices) でも、外部サービスとの連携には CLI が最もコンテキスト効率が良いとされている。

削除は 1 行で済む。

```bash
claude mcp remove github -s user
```
