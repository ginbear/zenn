---
title: "Claude Code をカスタマイズして快適に使う3つの設定"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claudecode", "cli", "mac", "obsidian"]
published: true
---

## はじめに

Claude Code をより便利に使いたくなり、いくつかのカスタマイズを取り入れました。本記事では、参考にした記事と、私が加えた変更点を紹介します。

1. **ステータスラインのカスタマイズ** - 現在のディレクトリやコンテキスト使用量を可視化
2. **作業完了時の通知** - Claude が作業を終えたらポップアップで通知
3. **会話履歴の Obsidian 連携** - 過去のやり取りをナレッジベースとして活用

## 1. ステータスラインのカスタマイズ

Claude がどのディレクトリで作業しているかわからず、指示の出し方を工夫したことがありました。ステータスラインに現在のディレクトリを表示することで解決しました。

また、コンテキスト使用量を可視化することで、100% になる前に `/compact` で情報整理するようになりました。

![ステータスラインの表示例](/images/statusline-example.png)

### 参考にした記事

https://zenn.dev/kawarimidoll/articles/00cfa200c12c5f

### Nerd Fonts アイコンの出力方法

私の環境では直接 Unicode 文字を記述するとステータスラインが表示されなかったため、`printf` で UTF-8 バイト列を出力するようにしています。

```bash
# 参考元
ICON_MODEL="󰚩"

# 私の設定
ICON_MODEL=$(printf "\xf3\xb0\x9a\xa9")    # U+F06A9 nf-md-assistant
ICON_DIR=$(printf "\xef\x90\x93")          # U+F413 nf-oct-file_directory
ICON_GIT=$(printf "\xee\x82\xa0")          # U+E0A0 nf-dev-git_branch
ICON_CONTEXT=$(printf "\xef\x83\xa4")      # U+F0E4 nf-fa-tachometer
ICON_COST=$(printf "\xef\x85\x95")         # U+F155 nf-fa-dollar
```

### コスト表示の追加

参考元にはなかった API 使用料の表示を追加しました。ステータスラインの入力 JSON から `cost.total_cost_usd` を取得しています。

```bash
COST=$(echo "$input" | jq -r '.cost.total_cost_usd // 0')
```

スクリーンショットの `$0.30` の部分がこれに該当します。セッション中の累計コストが表示されるので、使いすぎに気づきやすくなります。

## 2. 作業完了時の通知

Claude Code で長い作業を依頼した後、別の作業をしていると完了に気づかないことがあります。Hooks 機能を使って、作業完了時にポップアップ通知を出すようにしました。

### 参考にした記事

https://qiita.com/take8/items/28bae27208580f0a2e44

参考記事をベースに、以下の方針で通知設定をカスタマイズしました。

- **応答完了（Stop）**: 基本的に自分のタイミングで見に行きたいので、バナーは出さない。ただし他のアプリを使っているときは音だけ鳴らして気づけるようにする
- **許可確認（Notification）**: インタラクティブにやり取りしている間は通知不要だが、目を離した隙に許可待ちになっていると気づけない。1分経っても反応がない場合に通知する

| イベント | トリガー | ターミナルにフォーカス中 | 他のアプリにフォーカス中 |
|---------|---------|------------------------|------------------------|
| **Stop** | Claude の応答完了 | 何もしない | 🔔 Glass |
| **Notification** | 許可確認（1分後） | 🔔 Hero + バナー | 🔔 Hero + バナー |
| **UserPromptSubmit** | ユーザーが入力 | Notification をキャンセル | Notification をキャンセル |

:::message
Notification には `permission_prompt`（許可確認）と `idle_prompt`（60秒アイドル後）の2種類があります。

`idle_prompt` は「ユーザーが60秒間何も入力しなかった」ときに発火しますが、Claude が処理中であっても**ユーザー視点では入力待ち（idle）状態**とみなされます。そのため、Claude の処理が60秒以上かかると `idle_prompt` が発火し、Stop イベントと重複してしまいます。

```
00:00  ユーザーが入力 → Claude 処理開始
         ↓ （この間、ユーザーは入力できない = idle 状態）
01:00  60秒経過 → idle_prompt 発火 → 通知タイマー開始
01:30  Claude 処理完了 → Stop 発火
02:00  タイマーの60秒が経過 → 通知が届く（Stop と重複！）
```

本設定では `permission_prompt` のみを対象にしています。

参考: [Hooks reference - Claude Code Docs](https://code.claude.com/docs/en/hooks)
:::

以下、実装上のポイントを説明します。

### sender オプションの追加

私の環境（macOS Tahoe）では `terminal-notifier` の通知がデスクトップに表示されない問題がありました。`-sender com.apple.Terminal` オプションを追加することで解決しました。

```bash
# これだと通知センターにしか表示されない
terminal-notifier -title 'Claude Code' -message '完了'

# -sender を指定するとデスクトップ通知が出る
terminal-notifier -title 'Claude Code' -message '完了' -sender com.apple.Terminal
```

### subtitle にウィンドウタイトルを表示

複数のターミナルで Claude Code を使っていると、どのセッションからの通知かわからなくなります。`-subtitle` オプションでウィンドウタイトルを表示するようにしました。

```bash
# 呼び出し時点でウィンドウタイトルを取得（1分後にはフォーカスが変わっている可能性があるため）
WINDOW_TITLE=$(osascript -e 'tell application "System Events" to get name of first window of (first process whose frontmost is true)')

terminal-notifier -title 'Claude Code' -subtitle "$WINDOW_TITLE" -message '確認が必要です'
```

### バックグラウンドプロセスの完全切り離し

単純に `&` でバックグラウンド実行するだけでは、Claude Code が hooks の完了を待ってしまい、遅延時間の間応答がブロックされる問題がありました。

標準入出力を `/dev/null` にリダイレクトし、`disown` でジョブテーブルから削除することで解決しました。

```bash
# NG: Claude Code が子プロセスの終了を待ってしまう
( sleep 60; terminal-notifier ... ) &

# OK: 完全に切り離す
( sleep 60; terminal-notifier ... ) </dev/null >/dev/null 2>&1 &
disown
```

### ターミナルにフォーカス中は音を鳴らさない

ターミナルにフォーカスしているときは Claude の応答を確認しているケースがほとんどなので、音で通知する必要がありません。Ghostty（私が使っているターミナル）がフォーカス中は音を鳴らさないようにしました。

AppleScript でフォーカス中のアプリ名を取得し、条件分岐しています。

```bash
#!/bin/bash
# Ghostty がフォーカス中でなければ音を鳴らす
FRONTMOST=$(osascript -e 'tell application "System Events" to get name of first process whose frontmost is true')
if [[ "$FRONTMOST" != "ghostty" ]]; then
  afplay /System/Library/Sounds/Glass.aiff
fi
```

:::message
アプリ名は大文字小文字が区別されます。私の環境では `Ghostty` ではなく `ghostty` でした。以下のコマンドで確認できます。

```bash
osascript -e 'tell application "System Events" to get name of first process whose frontmost is true'
```
:::

## 3. 会話履歴の Obsidian 連携

Claude Code の会話履歴を Obsidian の Vault に Markdown として書き出すことで、過去のやり取りをナレッジベースとして活用できます。

:::message
ちなみに、Claude Code には `/resume` コマンドがあり、過去のセッションを再開できます。単にセッションを復元したいだけなら、こちらの機能を使う方が良さそうです。
:::

### 参考にした記事

https://zenn.dev/pepabo/articles/ffb79b5279f6ee

### プロジェクト・セッション単位でファイルを分割

参考記事では日付単位で1ファイルにまとめていますが、私の設定ではプロジェクトごと・セッションごとにファイルを分けています。

複数プロジェクトを並行して作業することが多いため、プロジェクトごとに分かれている方が後から探しやすいです。また、1ファイルが長くなりすぎると人間にも AI にも優しくないので、セッション単位で分けるようにしました。

### タイムスタンプの JST 変換

Claude Code のセッションファイルはタイムスタンプが UTC で記録されています。日本時間（JST）で日付を判定するには変換が必要です。同様に、会話内の時間表示（`## User (01:30:00)` など）も JST に変換しています。

```bash
# タイムスタンプ例: 2026-01-14T16:28:22.064Z (UTC)
# これは JST では 2026-01-15T01:28:22 になる

ts_clean="${timestamp%.???Z}"
epoch=$(/bin/date -j -u -f "%Y-%m-%dT%H:%M:%S" "$ts_clean" "+%s")
msg_date=$(TZ=Asia/Tokyo /bin/date -r "$epoch" "+%Y-%m-%d")
```

### YAML フロントマターの追加

Obsidian で検索や整理をしやすくするため、ファイル作成時にメタ情報を追加しています。

```yaml
---
created: 2026-01-15T01:31:29+0900
project: my-project
repository: /Users/username/path/to/my-project
session_id: c9919dec-3538-4abb-ab69-d69747854ec1
tags:
  - claude-code
---
```

## まとめ

Claude Code はカスタマイズ性が高く、自分のワークフローに合わせた工夫ができます。参考記事の著者の方々に感謝します。

本記事で紹介した設定は [GitHub の dotfiles](https://github.com/ginbear/dotfiles/tree/master/.claude) で管理しています。常に更新しているため、記事の内容と異なる状態になっている可能性があります。最新の設定はリポジトリをご確認ください。
