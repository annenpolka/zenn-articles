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
---

## はじめに

まず、このプロジェクトはさめざめさんの[ccresume](https://github.com/sasazame/ccresume)に多大な影響を受けています。ありがとうございます！

Claude Codeに技術的質問をしてそのログをとる習慣があるのですが、当初それをプロンプト＋カスタムコマンドでClaude Code自身にmarkdownとして書いてもらっていました。ターミナルの中で完結し、WebUIをコピペするよりも楽なのが嬉しいところです。

しかし、**ユーザーの回答を捏造し始めたりした**（LLMにやらせているので仕方ない）ので、「生のログファイル（jsonlファイルなので加工の必要あり）をちゃんと触ってパースしよう」ということになり、[cclog](https://github.com/annenpolka/cclog)を作りました。

## 何をするアプリか

https://www.youtube.com/watch?v=iexFYGWVpbc

https://github.com/annenpolka/cclog

- `~/.claude/projects`配下にある会話ログを一覧表示、マークダウンとしてプレビューしたりエディタで開いたりできます
  - EDITOR 環境変数で設定されたエディタで自動的に開いてくれます（vimでもvscodeでも）
  - 一時ファイルとして開くので、とりあえず見たいだけの時も cwdは汚さずに済みます
- 見やすさのために会話以外のノイズになる内容（システムメッセージなど）は省略するフィルター機能がついています

  - （デフォルトでオンですが、sキーでトグルできるようになっています）

- インストールは以下コマンドで

```bash
go install github.com/annenpolka/cclog/cmd/cclog@latest
```

## 技術的選択とその感想

- 過去 React+InkでTUIを作ったときにパフォーマンス問題で困ったので、Go の TUI ライブラリ、[bubbletea](https://github.com/charmbracelet/bubbletea)を採用してみました
  - Claude Code + Sonnet 4にずっとコードを書かせていましたが、関連ライブラリのbubblesやlipgross、teatestの知識を渡せば問題にぶつかるタイミングは少なく快適でした。サクサクで見た目も可愛くしやすく、おすすめです
  - ただ、レイアウトの組みやすさはYogaベースで柔軟なInkに軍配が上がりそうです

## 既知の問題・やりたいこと

- Claude Codeが回答にmarkdownの見出し(##）を使った時、見出しの階層構造が崩れます
  - コードブロックにする等考えましたが、見た目「いい感じ」に見えることを優先して現状そのままにしてます

## 追記（7/9）

セッションのresume機能やセッションIDのコピー機能を追加しました！ぜひお試しください！
