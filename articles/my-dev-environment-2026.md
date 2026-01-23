---
title: "作業環境 2026"
emoji: "🛠️"
type: "tech"
topics: ["開発環境", "mac", "zsh", "cli"]
published: true
publication_name: "atrae"
---

## ハードウェア

- **MacBook Air** (Apple M4)
- **モニター**: BenQ プログラミングモニター RD280UA
- **キーボード**: [Cornix LP](https://note.com/ginbear/n/n832b96bb0d72) + MacBook 内蔵キーボード
- **マウス**: Logicool MX Master 3S

### モニターについて

以前使っていたモニターが壊れてしまい更新した。テキストがとても綺麗。ゲーム (Switch 2) を接続したところアスペクト比の都合上ちょっと縦に伸びたけどすぐに慣れた。マリカーで 3D 酔いしそうなくらい大迫力。

### キーボードについて

Cornix LP はビルドクオリティが高くてかなり気に入って使っている。使い始めて 4 ヶ月、ようやく Layer に慣れてきた気がする。

https://x.com/kensuu/status/2004218708764590425

この投稿を見て Combo の素晴らしさに気づいたが、なぜか Vial で設定してもうまく保存されなかったり設定が変わってしまったりする事象に遭遇し断念している。

### ポインティングデバイスについて

トラックボール（Kensington Expert Mouse TB800 EQ）を検討していたが、不具合の報告が多いので様子見中。Logicool MX Master 4 も検討したが、触ってみたところしっくりこなくてこちらも保留中。[Nape Pro](https://costory.jp/cf-published-sku-groups/1254312814?srsltid=AfmBOorL8bHCmK-zsZaYLvePlgGbaLI1Pro-Hnjcf-5GKV-716e2BDLy) を購入したので正座待機している。

## ターミナル

### Ghostty

https://ghostty.org/

Zig で書かれた高速なターミナルエミュレータ。透過と blur を有効にして見た目を調整、フォントは PlemolJP Console NF を使用。
👻 がかわいい。

```
theme = "Monokai Pro"
font-family = "PlemolJP Console NF"
window-padding-x = 10
background-opacity = 0.85
background-blur-radius = 5
cursor-style = block
cursor-style-blink = false
copy-on-select = clipboard
```

## シェル

### Zsh + Powerlevel10k

プロンプトは Powerlevel10k。instant prompt を有効にして起動を高速化している。接続するクラスタによってカラーを変えている。

### 単語削除の設定

zsh のデフォルトでは `Ctrl+W` で単語を削除すると `/` や `-` などの記号で止まってしまう。bash では空白まで一気に削除されるので、その挙動に寄せる設定を入れている。

```zsh
bindkey '^W'   backward-kill-space   # Ctrl-W: 空白まで削除（bash と同じ）
bindkey '^[^?' backward-kill-punct   # Esc+Backspace: 記号まで削除
```

パスを入力しているときに `Ctrl+W` で一気に消せるので地味に便利。

### お手製コマンド

fzf を多用した自作関数を [.zshrc](https://github.com/ginbear/dotfiles/blob/master/.zshrc) に定義している。

| コマンド | キーバインド | 用途 |
|----------|-------------|------|
| `fzf-ghq-look` | `Ctrl+G` | ghq 管理のリポジトリを選択して cd |
| `fzf-snippets` | `Ctrl+X` | スニペットを選択してクリップボードにコピー |
| `gsw` | - | ブランチを選択して git switch |
| `git-pr` | - | 自分の PR を選択してブラウザで開く |
| `aws-profile` (`ap`) | - | AWS プロファイルを選択して SSO ログイン |
| `kctx` | - | Kubernetes context を選択 |
| `kns` | - | Kubernetes namespace を選択 |
| `k-desc-pod` | - | Pod を選択して describe（プレビュー付き） |
| `k-exec` | - | Pod を選択して shell login |

fzf との組み合わせでインタラクティブに選択できるのが便利。

## CLI ツール

### パッケージ管理

メインのパッケージマネージャーは Homebrew。ランタイムバージョン管理（Node.js, Python など）には mise を使用。以下はよく使ってるやつ。

### Git 関連

| ツール | 用途 |
|--------|------|
| gh | GitHub CLI |
| ghq + fzf | リポジトリ管理 |

### 便利系

| ツール | 用途 |
|--------|------|
| bat | cat の代替（シンタックスハイライト付き） |
| lsd | ls の代替（アイコン付き） |
| ripgrep | 高速 grep |
| fzf | あいまい検索、とにかく汎用性が高い |
| jq / yq | JSON/YAML 処理 |
| atuin | シェル履歴の同期・検索 |
| watch | コマンドの定期実行、稀によく使う |

atuin は最近導入して便利〜となっている。

## エディタ・AI

### Neovim

ターミナル内での作業用。VSCode も入れているが、メモ帳みたいな使い方しかしていない。

### Claude Code

https://docs.anthropic.com/en/docs/claude-code

ターミナルから使える AI アシスタント。コーディングや調査に活用。開発作業は全部 Claude くんがやってくれる。
今一番使っているアプリ。

こんな感じでカスタマイズして使っている。
https://zenn.dev/atrae/articles/claude-code-customization

## GUI アプリ

| アプリ | 用途 |
|--------|------|
| Ghostty | ターミナル |
| ⌘英かな (cmd-eikana) | キーリマップ |
| Raycast | ランチャー |
| Obsidian | ローカルドキュメント管理 |
| keyboardcleantool | キーボード掃除用、綺麗好きなら推奨 |

### ⌘英かな

US キーボードで `左⌘単押し → 英数, 右⌘単押し → かな` をしてくれる専用ツール。Karabiner を使っていたが、この機能（英かな変換）しか使っていなかったので、より軽量なこちらに乗り換えた。

### Raycast

- **GitHub Extension**: リポジトリや issue/PR を開くのに使っている。ないと生きていけない。
- **Quick Links**: ブラウザのブクマではなくここから行きたいサイトを直接開いてる。ないと生きていけない。
- **Window Management**: ショートカットで Window の操作をしている。ないと生きていけない。
  - 以前は Rectangle を使っていたが、同じことが Raycast でできることがわかって統一した
- **Clipboard History**: コピーした内容の履歴から貼り付けることができる。ないと（ry

### Obsidian

とりあえず手軽に保存できるドキュメントの保管先として利用している。よく記事があるように AI に読み込ませて hogehoge みたいなことはしていない。まだ伸び代がある状態。

## dotfiles 管理

https://github.com/ginbear/dotfiles

設定ファイルは全てここで管理している。

### chezmoi

https://www.chezmoi.io/

チームで Claude Code の設定（CLAUDE.md やカスタムコマンド）を共有するために chezmoi を使用している。チーム設定と個人設定をマージする仕組み。
