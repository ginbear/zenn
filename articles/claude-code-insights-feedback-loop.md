---
title: "/insight で CLAUDE.md を育てるフィードバックループを作ってみた"
emoji: "🔄"
type: "tech"
topics: ["claudecode", "cli", "ai"]
published: true
publication_name: "atrae"
---

## きっかけ

Claude Code に `/insight` というのがあると同僚から教えてもらい、実行してみたところ想像以上に色々出力されて驚きました。もしまだ出力したことがないなら一度出してみることをおすすめします。使い方を褒めてくれたりもして自己肯定感も上がる。
最後にこんなコメントもあったりしてユーモアもあります。

> "Claude didn't recognize its own mascot's birthday hat and had to be corrected by the user"
> When a user asked why Claude Code's mascot was wearing a hat, Claude failed to recognize it was a birthday hat — the user had to explain Claude's own branding to it.
> 訳: 「クロードは自身のマスコットの誕生日帽子を認識できず、ユーザーに修正してもらう必要がありました」
> ユーザーがクロード・コードのマスコットがなぜ帽子をかぶっているのかと尋ねたところ、クロードはそれが誕生日の帽子だとは気づかなかった。ユーザーはクロード自身のブランドを説明する必要があった。

この insight の結果には Claude Code をより良く使うための提案も入っています。この分析結果をそのまま CLAUDE.md の改善に使えばいいのでは？と考えました。
公式の [Best Practices](https://code.claude.com/docs/en/best-practices) にも「CLAUDE.md はコードと同じように扱え。定期的にレビューして刈り込め」と書いてあります。

> Treat CLAUDE.md like code: review it when things go wrong, prune it regularly, and test changes by observing whether Claude's behavior actually shifts.

CLAUDE.md の更新を常に意識するのはしんどく、後回しにしがちでした。
`/insight` → CLAUDE.md 改善のループを仕組み化すれば、この脳内負荷を減らせるのではと思い試して見ました。

## やっていること

シンプルで、以下のサイクルを回しているだけです。

1. `/insight` でセッションを振り返る
2. 繰り返しているミスや非効率なパターンを特定する
3. CLAUDE.md か `.claude/rules/` のどちらが適切かを判断して反映する

これをカスタムスキルとして定義して `/insights-feedback` で呼べるようにしました。手動トリガーですが、「そろそろ振り返るか」と思ったときにすぐ実行できます。

`/insight` 自体は週1くらいの実行が目安のようですが、自分はもっとゆるく、気が向いたときにやっています。

## 具体例

実際に `/insight` を元に改善したときの diff です。

```diff
# CLAUDE_PERSONAL.md

+## 回答の正確性
+
+- ツールの機能・設定項目について確信がない場合は「未確認」と明示し、公式ドキュメントや `--help` で確認してから回答する
+- 「できない」「存在しない」と断言する前に、実際にコマンドやドキュメントで検証する

+### リソース状態の報告ルール
+- kubectl の出力を部分的に見て「正常」と断言しない。STATUS/READY カラムを必ず確認する
+- 「動いている」と報告する前に、Pod の STATUS が Running かつ READY が期待値であることを確認する
+- 不確かな場合は「未確認」と明示し、確認コマンドを提示する
```

insight で「ECR pull が正常と断言したが実際は全 Pod が Failed だった」「Argo Workflows の RBAC モデルを誤って説明した」といったフリクションが見えたので、確認ルールとして明記しました。

なお、insight の提案には **「やってほしいこと」より「やりがちだけどやめてほしいこと」を書く方が効果が高い** とありました。確かに Claude は賢いので正解は推測できますが、禁止事項は明示しないと繰り返す印象があります。

## スキルの概要

`~/.claude/skills/insights-feedback/SKILL.md` に以下のワークフローを定義しています。

1. `/insight` でセッション分析を取得（※ `/insight` は CLI 組み込みコマンドなのでスキルから直接実行できない。事前に実行してもらう前提）
2. 現在の CLAUDE.md 系ファイルを読み込む
3. 「追加 / 修正 / 削除 / 構造改善」の4観点で分析し、CLAUDE.md か rules かの配置先を判断
4. レポートを出力してユーザーに提示
5. 承認された提案のみ適用

個人設定（CLAUDE_PERSONAL.md）は直接編集、チーム設定（CLAUDE_TEAM.md）は PR を作成する、という分け方にしています。

スキルの全文は [dotfiles リポジトリ](https://github.com/ginbear/dotfiles/tree/master/dot_claude/skills/insights-feedback) に置いています。ただし chezmoi のパスやチーム設定の PR フローなど独自の事情が多いので、そのまま使うより「こういうスキルを作りたい」と Claude Code に伝えて自分の環境に合ったものを書かせるのが良さそうです。

## CLAUDE.md から .claude/rules/ へ

[Claude Codeで「CLAUDE.mdに何を書くべきか」の最適解を探る](https://zenn.dev/cureapp/articles/65b9a99d22ce2b) という記事で `.claude/rules/` の存在を知り、取り入れてみました。

| 配置先 | 注入タイミング | 適した内容 |
|--------|-------------|-----------|
| `CLAUDE.md` | セッション開始時 | プロジェクト概要、環境情報 |
| `.claude/rules/*.md` | 該当ファイル初出時 | コーディング規約、禁止事項 |

YAML frontmatter の `paths` で適用条件を絞れます。

```yaml
---
paths:
  - "**/*.tf"
  - "**/terragrunt.hcl"
---
# Terraform 運用ルール
- apply の前に必ず plan を実行
- `-target` の使用は極力避ける
```

公式ドキュメント: [Organize rules with .claude/rules/](https://code.claude.com/docs/en/memory#organize-rules-with-clauderules)

これを受けて insights-feedback スキルも rules 対応にアップデートし、分析結果の配置先を CLAUDE.md と rules のどちらが適切かも判断するようにしました。実際にチームの CLAUDE_TEAM.md から運用ルールを rules/ に移行し、CLAUDE_TEAM.md は環境情報のみに整理しています。

なお、dotfiles が公開リポジトリの場合は組織固有ルールを直接置けないので、シンボリックリンクで非公開リポジトリの rules を参照する形にしています。

## 今後やりたいこと

まだ始めたばかりなので、しばらく回してみて効果を確認したいと思っています。

- 何回かサイクルを回した上での before/after の比較
- スキルのプロンプト自体の改善（分析精度を上げる）
- rules 移行後のルール遵守率の変化を `/insight` で追跡

「CLAUDE.md を育てる」を脳内リソースに頼らず仕組みで回せるようになると、だいぶ楽になりそうです。
