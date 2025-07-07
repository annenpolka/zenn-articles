---
title: 'Claude Codeの会話ログをいい感じのマークダウンとしてプレビュー・書き出しできるTUIをGoで書いた'
emoji: '📝'
type: 'tech'
topics:
  - AI
  - Claude Code
  - Go
  - terminal
  - bubbletea
published: true
published_at: 2050-07-07 19:00
---

## はじめに

まず、このプロジェクトはさめざめさんの[ccresume](https://github.com/sasazame/ccresume)に多大な影響を受けています。ありがとうございます！

Claude Code に技術的質問をしてそのログをとる習慣があるのですが、当初それをプロンプト＋カスタムコマンドで Claude Code 自身に markdown として書いてもらっていました。ターミナルの中で完結し、WebUI をコピペするよりも楽なのが嬉しいところです。

しかし、**ユーザーの回答を捏造し始めたりした**（LLM にやらせているので仕方ない）ので「生のログファイルをちゃんと触ってパースしよう」ということになり、[cclog](https://github.com/annenpolka/cclog)を作りました。

## 何をするアプリか

https://www.youtube.com/watch?v=7y5t9dcmjl8

https://github.com/annenpolka/cclog

- `~/.claude/projects`配下にある会話ログを一覧表示、マークダウンとしてプレビューしたりエディタで開いたりできます
  - EDITOR 環境変数で設定されたエディタで自動的に開いてくれます（vim でも vscode でも）
  - 一時ファイルとして開くので、とりあえず見たいだけの時も cwd は汚さずに済みます
- 見やすさのために会話以外のノイズになる内容（システムメッセージなど）は省略するフィルター機能がついています

  - （デフォルトでオンですが、s キーでトグルできるようになっています）

- インストールは以下コマンドで

```bash
go install github.com/annenpolka/cclog/cmd/cclog@latest
```

## 技術的選択とその感想

- 過去 React+Ink で TUI を作ったときにパフォーマンス問題で困ったので、Go の TUI ライブラリ、[bubbletea](https://github.com/charmbracelet/bubbletea)を採用してみました
  - Claude Code + Sonnet にずっとコードを書かせていましたが、関連ライブラリの bubbles や lipgross、teatest の知識を渡せば問題にぶつかるタイミングは少なく快適でした。サクサクで見た目も可愛くしやすく、おすすめです
  - ただ、レイアウトの組みやすさは Yoga ベースで柔軟な Ink に軍配が上がりそうです

## 既知の問題・やりたいこと

- Claude Code が 回答に markdown の見出し(##）を使った時、セマンティックな階層構造が崩壊します
  - コードブロックにする等考えましたが、見た目「いい感じ」に見えることを優先して現状そのままにしてます
- プレビューの縦幅をレスポンシブにしたい（これはできそう）
