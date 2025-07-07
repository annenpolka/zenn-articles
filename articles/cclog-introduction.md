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

- まず、このプロジェクトはさめざめさんの ccresume に多大な影響を受けています。ありがとうございます！
- この記事は AI の助力を受けながら書いていますが、アプリはちゃんと存在します（大体バイブコーディング）

## 何をするアプリか

https://www.youtube.com/watch?v=7y5t9dcmjl8

https://github.com/annenpolka/cclog

- `~/.claude/projects`配下にある会話ログを一覧表示、マークダウンとしてプレビューしたりエディタで開いたりできます

## 既知の問題・やりたいこと

- Claude Code が markdown の見出し(##）などを使った時、ちゃんとした階層構造じゃなくなります
  - コードブロックにしたりするのも考えましたが、「いい感じ」に見えることを優先して現状そのままにしてます
- resume 機能をつけようとは思っていますが、躓いています
  - ログから sessionId 取って`ccresume -r <sessionId>`するだけだと`No conversation found with session ID` になることがあるんですけど、なぜ…
