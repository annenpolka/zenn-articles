---
title: 'CursorのDebug ModeをClaude Codeなどで使えるAgent Skillとして再現してみた'
emoji: '🐞'
type: 'tech'
topics:
  - AI
  - Claude Code
  - Cursor
  - Agent Skills
  - debug
published: true
---

[debug-mode](https://github.com/annenpolka/skills/tree/main/debug-mode)というAgent Skillを作ったので紹介します。CursorのDebug Modeに着想を得た、「仮説駆動のprintfデバッグ」をAIエージェントに叩き込むためのスキルです。

AIコーディングエージェントが当たり前になってきた今、バグ修正のアプローチにもまだ改善の余地があります。多くのエージェントは、バグを報告されるとコードを読んで「たぶんこれが原因だろう」と推測し、修正を試みます。これが上手くいくこともあるけれど、厄介なバグに対しては的外れな修正を何度も繰り返す地獄にハマりがちです。

debug-modeスキルは、その「推測で直す」アプローチを禁止し、「まず観察して、証拠に基づいて直す」ワークフローをエージェントに強制します。

## CursorのDebug Mode——何が画期的だったのか

2025年12月、CursorがDebug Modeを発表しました。Cursor 2.2で導入された新しいエージェントモードです。

CursorのDebug Modeのワークフローはこうなっています。

1. ユーザーがバグを報告する
2. エージェントがコードベースを読み、バグの原因について複数の仮説を生成する
3. 各仮説を検証するためのデバッグログをコードに仕込む
4. ユーザーにバグの再現を依頼する
5. 収集したランタイムログを分析して根本原因を特定する
6. 修正を実装し、再度ユーザーに検証を依頼する
7. 修正が確認されたらデバッグログを除去してクリーンアップする

やっていることの本質は、エージェントにランタイムの実際の挙動を観察させること。しかもHTTPベースのテキストログという極めてシンプルな手法で、LSPやクラシカルなデバッガのような「高度な」仕組みは一切使っていない。テキストログの解析はLLMが最も得意とするところで、だからこそあらゆる言語・環境で動作します。

## 仮説検証ループ——debug-modeの全体フロー

debug-modeのワークフローは10ステップで構成されています。

### Step 1-2: 問題の把握とログ基盤の準備

「〇〇が正しく画面に反映されない。debug-mode」みたいな感じで呼び出すと、エージェントがユーザーから情報を収集します。症状（期待される動作 vs 実際の動作）、再現手順、最近の変更。

そしてログの出力先を準備します。環境に応じてファイル直接書き込み（サーバーサイド）か、HTTPコレクターサーバー経由（ブラウザ等）かが選択され、`debug.log` にログが集約されます。

### Step 3: 仮説を「複数」立てる

ここが最も重要なステップです。エージェントは最低3つ、理想的には5つの仮説を生成します。

```markdown
## Hypotheses

### H1: [タイトル]
- Cause: ...
- Verify: XをYの前後で確認

### H2: [タイトル]
- Cause: ...
- Verify: ...

### H3: [タイトル]
- Cause: ...
- Verify: ...
```

なぜ複数の仮説が重要なのか。AIが最初に思いついた原因で「たぶんこれだろう」で修正してみて、直らなくて、また別の推測…というサイクルは時間の浪費です。最初から複数の仮説を立て、それぞれを検証するログを同時に仕込むことで、1回の再現で複数の可能性を同時に絞り込めるわけですね。

### Step 4: ログの仕込み（構造化されたprintfデバッグ）

各仮説に対応するログを3〜8箇所に挿入します。ログのフォーマットはNDJSON（1行1JSON）で統一されています：

```jsonl
{"h":"H1","l":"state_before","v":{"userId":"123"},"ts":1702567890123}
```

| フィールド | 意味 |
| --- | --- |
| h | 仮説ID（H1, H2, ...） |
| l | ラベル（このログが何を表すか） |
| v | 値（任意のJSONシリアライズ可能なデータ） |
| ts | タイムスタンプ（ミリ秒） |

ログの仕込み箇所として推奨されているのは、関数の入口（引数）、関数の出口（戻り値）、クリティカルな操作の前後、分岐パス（どのif/elseを通ったか）、状態変更の前後。

重要なルールとして、ログは必ず `#region debug:{hypothesisId}` で囲みます。

```javascript
//#region debug:H1
fetch('http://localhost:4567', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    h: 'H1',
    l: 'user_state',
    v: { userId, cart, timestamp: new Date().toISOString() },
    ts: Date.now()
  })
}).catch(() => {});
//#endregion
```

このregionタグがあることで、後からクリーンアップ時に `grep -rn "#region debug:" src/` で一発で仕込み箇所を特定できます。「仮のデバッグコードが本番に紛れ込む」事故を防ぐための仕組みです。

### Step 5-6: ログクリアと再現

古いログを削除し、ユーザーにバグの再現を依頼します。

### Step 7: ログ分析——証拠に基づく判定

```bash
cat debug.log | jq .                              # 全件表示
jq 'select(.h == "H1")' debug.log                 # 仮説ごとにフィルタ
cat debug.log | jq -s 'group_by(.h)'              # 仮説ごとにグループ化
tail -f debug.log | jq .                           # リアルタイム監視
```

各仮説に対して判定を下します：

| 判定 | 意味 |
| --- | --- |
| CONFIRMED | ログがこの仮説を明確に支持 |
| REJECTED | ログがこの仮説に矛盾 |
| INCONCLUSIVE | データ不十分 |

### Step 8-10: 修正、検証、クリーンアップ

修正は仮説がCONFIRMEDの場合のみ行います。全仮説がREJECTEDなら、新たな仮説を立ててStep 3に戻ります。

修正後もログはそのまま残し、再度再現テストを行って修正を検証。ユーザーが修正を確認した後、ようやくログを除去します。

## 本質は「printfデバッグの効率化」

先にも書いていますが、debug-modeがやっていることの本質は「構造化されたprintfデバッグ」です。

ただし、AIエージェントにこれを「何も考えずに」やらせると、フォーマットがバラバラだったり、仮説もなく闇雲にログを追加し続けたりします。

debug-modeは、printfデバッグに構造を持ち込んでくれるわけです。

| 通常のprintfデバッグ | debug-modeの構造化デバッグ |
| --- | --- |
| 場当たり的にログ挿入 | 仮説に基づいて計画的に挿入 |
| フォーマットがバラバラ | NDJSON統一フォーマット |
| 後から探すのが困難 | 仮説IDで即座にフィルタリング |
| クリーンアップを忘れがち | regionタグで確実に除去 |
| 1つの原因を仮定 | 3-5個の仮説を同時検証 |
| 推測→修正→確認 | 仕込み→観察→判定→修正→検証 |

## エージェントに強制するルール——「禁止事項」がミソ

debug-modeのSKILL.mdには明確な禁止事項が定義されています。

- ログを分析する前に修正を提案すること
- 仮説が1つだけ
- 検証前にログを除去すること
- 「おそらくこれだろう」式の推測
- setTimeout/sleepを「修正」として使うこと

debug-modeは「毎回デバッグのたびに仮説を立てろ、ログを見ろ、推測するな、と言い聞かせる」手間を省くためのスキルです。一度入れてしまえば、エージェントは常にこのプロトコルに従います。

## 余談：ブラウザ上の動作をデバッグする手法が面白い

debug-mode、もとい元ネタのCursorのDebug Modeで個人的に最も「おっ」と思ったのは、ブラウザ上のJavaScript動作をデバッグする手法です。

サーバーサイド（Node.js, Python等）であれば、ファイルに直接ログを書き込めば済みます。しかしブラウザのJavaScriptにはファイルシステムへのアクセスがない。かといって `console.log` をいくら仕込んでも、AIエージェントがブラウザのDevToolsを見ることは（通常は）できません。

Debug Modeの解法はログコレクターサーバーです。

```bash
node -e "require('http').createServer((q,s)=>{
  s.setHeader('Access-Control-Allow-Origin','*');
  s.setHeader('Access-Control-Allow-Methods','POST,OPTIONS');
  s.setHeader('Access-Control-Allow-Headers','Content-Type');
  if(q.method==='OPTIONS'){s.writeHead(204).end();return}
  let b='';
  q.on('data',c=>b+=c);
  q.on('end',()=>{
    require('fs').appendFileSync('debug.log',b+'\n');
    s.writeHead(204).end()
  })
}).listen(4567,()=>console.log('Collector: http://localhost:4567'))"
```

（実際のスキルではワンライナーで提供されています）

ブラウザ側のコードからは `fetch('http://localhost:4567', ...)` でログをPOSTし、コレクターサーバーがそれを `debug.log` に追記する。CORSヘッダーも設定済みなので、開発環境ならそのまま動きます。

つまり、ブラウザのランタイム情報がテキストファイルに書き出され、エージェントはそのファイルを読むだけで良い。DevToolsもブラウザ操作MCPも不要。HTTPとテキストファイルという最もローテクな手法で、ブラウザの壁を突破しています。

CursorではHTTPサーバーをプロダクト内部で自動的に立ち上げるためユーザーが意識する必要がない。debug-modeスキルではこれを明示的にセットアップする形になっていますが、原理は同じです。

実際、フロントエンドとバックエンドの両方にログを仕込んでイベントの順序を追うような使い方もできます。このコレクターサーバー手法があれば、フロントエンドもバックエンドも同じ `debug.log` にログが集約されるため、クライアント-サーバー間のタイミング問題を分析する際に特に威力を発揮します。

React NativeやFlutterのような「ブラウザではないがファイル直接書き込みが難しい」環境でも、同じHTTP方式が使えます（物理デバイスの場合は `localhost` をホストマシンのIPに変えるだけ）。

## まとめ

debug-modeは、CursorのDebug Modeが実現した「構造化printfデバッグ」のエッセンスを、Agent Skillとしてポータブルにしたものです。ぜひ使ってみてください。

https://github.com/annenpolka/skills/tree/main/debug-mode

## 参考

- [Introducing Debug Mode: Agents with runtime logs](https://cursor.com/blog/debug-mode) - Cursor公式ブログ
- [Cursor's Debug Mode Is Arguably Its Best Feature](https://davidgomes.com/cursor-debug-mode/) - David Gomes
