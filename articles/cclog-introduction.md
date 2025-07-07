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
published: false
---

## はじめに

まず、このプロジェクトはさめざめさんの[ccresume](https://github.com/sasazame/ccresume)に多大な影響を受けています。ありがとうございます！

## 何をするアプリか

https://www.youtube.com/watch?v=7y5t9dcmjl8

https://github.com/annenpolka/cclog

- `~/.claude/projects`配下にある会話ログを一覧表示、マークダウンとしてプレビューしたりエディタで開いたりできます
  - EDITOR環境変数で設定されたエディタで自動的に開いてくれます（vimでもvscodeでも）
  - 一時ファイルとして開くので、とりあえず見たいだけの時もcwdは汚さずに済みます
- 見やすさのために会話以外のノイズになる内容は省略するフィルター機能がついています
  - （デフォルトでオンですが、sキーでトグルできるようになっています）


- インストールは以下コマンドで

```bash
go install github.com/annenpolka/cclog/cmd/cclog@latest
```

## 作った理由

- Claude Codeに技術的質問をしてそのログをとる習慣があるのですが、当初それをプロンプト＋カスタムコマンドでClaude Code自身にmarkdownとして書いてもらっていました
  - ターミナルの中で完結し、WebUIをコピペするよりも楽なのが嬉しいところです

- が、**ユーザーの回答を捏造し始めたりした**（LLMにやらせているので仕方ない）ので「生のログファイルをちゃんと触ってパースしよう」ということに

## 技術的選択とその感想

- 過去React+InkでTUIを作ったときにパフォーマンス問題で困ったので、GoのTUIライブラリ、[bubbletea](https://github.com/charmbracelet/bubbletea)を採用してみました
  - Claude Code + Sonnetにずっとコードを書かせていましたが、関連ライブラリのbubblesやlipgross、teatestの知識を渡せば問題にぶつかるタイミングは少なく快適でした。サクサクで見た目も可愛くしやすく、おすすめです
  - ただ、レイアウトの組みやすさはYogaベースで柔軟なInkに軍配が上がりそうです



## 既知の問題・やりたいこと

- Claude Code が 回答にmarkdown の見出し(##）を使った時、セマンティックな階層構造が崩壊します
  - コードブロックにする等考えましたが、見た目「いい感じ」に見えることを優先して現状そのままにしてます
- resume 機能をつけようとは思っていますが、躓いています
  - ログから sessionId 取って`ccresume -r <sessionId>`するだけだと`No conversation found with session ID` になることがあるんですけど、なぜ…
- プレビューの縦幅をレスポンシブにしたい（これはできそう）
