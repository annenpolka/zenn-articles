---
title: 'Claude Codeの会話ログをいい感じのマークダウンとしてプレビュー・書き出しできるTUIをGoで書いた'
emoji: '📝'
type: 'tech'
topics:
  - AI
  - Claude Code
published: false
---

## はじめに

- まず、このプロジェクトはさめざめさんのccresumeに多大な影響を受けています。ありがとうございます
- この記事はAIの助力を受けながら書いていますが、アプリはちゃんと存在します（大体バイブコーディング）

## 何をするアプリか

(画像を入れる)

https://github.com/annenpolka/cclog

- `~/.claude/projects`配下にある会話ログを一覧表示、マークダウンとしてプレビューしたりエディタで開いたりできます





## 既知の問題・やりたいこと

- Claude Codeがmarkdownの見出し(##）などを使った時、ちゃんとした階層構造じゃなくなります
  - コードブロックにしたりするのも考えましたが、「いい感じ」に見えることを優先して現状そのままにしてます
- resume機能をつけようとは思っていますが、躓いています
  - ログからsessionId取って`ccresume -r <sessionId>`するだけだと`No conversation found with session ID` になることがあるんですけど、なぜ…