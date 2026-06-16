---
layout: post
title: "Claude API×GASスプレッドシート連携やり方｜AI要約・自動書き込み手順2026"
date: 2026-06-16 09:05:52 +0900
description: "Claude APIとGASを連携してスプレッドシートのセルをAI自動要約する手順を解説。2026年対応Messages API・コピペ用サンプルコード・コスト試算付き。"
tags:
  - Claude API GAS スプレッドシート 連携 やり方
  - Claude API Google Apps Script 自動要約 書き込み
  - GAS スプレッドシート AI 自動化 手順 2026
  - Claude Messages API GAS サンプルコード
---

## 「コードをコピペしたのに動かない」——その原因、ほぼ特定できます

古い記事のGASサンプルコードを貼ったら`400 Bad Request`が返ってきた。そんな経験がある人に向けて書いた記事です。

原因の9割は**旧Completion API（`/v1/complete`）のコードをそのまま使っていること**。2024年以降、このエンドポイントは非推奨になり、新規実装では使えません。

この記事を読めば、2026年現在で実際に動くMessages API対応のGASコードを手に入れられます。APIキーの安全な管理方法、コスト試算、トリガーによる自動化まで、一通りの流れを完結させます。

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

## Claude API×GASでできること｜全体像を先に把握する

まず「何が実現できるか」を具体的にイメージしてください。

構成はシンプルです。

```
スプレッドシートのA列（入力テキスト）
　　↓
GAS（Google Apps Script）がAPIを呼び出す
　　↓
Claude API（Anthropicのサーバー）
　　↓
GASが返答を受け取ってB列に書き込む
```

用途の例を挙げると：顧客フィードバックの自動要約、長文アンケートの分類・タグ付け、議事録の要点抽出、商品説明文の生成。Googleスプレッドシートにデータが集まる業務なら、ほぼどこにでも応用できます。

### 作業前に必要な準備物

- Anthropic APIキー（[console.anthropic.com](https://console.anthropic.com)で取得）
- Googleアカウント（スプレッドシート・GASが使えればOK）
- GASエディタの起動方法：スプレッドシートを開いて「拡張機能」→「Apps Script」

APIキーの取得自体は5分程度。クレジットカードの登録が必要です。無料トライアルクレジットは$5〜$10程度が付与されます（時期によって変動あり）。

### 2026年時点の重要な変更点

古い実装から乗り換える人は、以下の差分を最初に確認してください。

| 項目 | 旧Completion API（廃止済み） | 新Messages API（現行） |
|---|---|---|
| エンドポイント | `/v1/complete` | `/v1/messages` |
| 必須パラメータ | `prompt` / `max_tokens_to_sample` | `model` / `max_tokens` / `messages` |
| レスポンス取得 | `response.completion` | `response.content[0].text` |
| モデル指定 | `claude-2`, `claude-instant-1` | `claude-3-5-sonnet-20241022` など |
| versionヘッダー | 省略可 | `anthropic-version`の明示が必須 |

`response.completion`でテキストを取り出そうとしてundefinedになる、というエラーの大半はこのレスポンス構造の違いが原因です。

---

## APIキー設定からURLフェッチまで｜最初に書く基本コード

### ステップ1：APIキーをスクリプトプロパティに登録する

APIキーをコードの中にベタ書きするのは絶対にやめてください。GitHubに誤ってプッシュした瞬間にキーが漏洩し、不正利用される事故が実際に起きています。

GASには「スクリプトプロパティ」という安全な格納場所があります。

手順：
1. GASエディタを開く
2. 左サイドバーの「プロジェクトの設定」（歯車アイコン）をクリック
3. 「スクリプト プロパティ」セクションまでスクロール
4. 「プロパティを追加」→ プロパティ名に`ANTHROPIC_API_KEY`、値に実際のキーを入力
5. 「スクリプト プロパティを保存」

コード側からは以下で取得します。

```javascript
const apiKey = PropertiesService.getScriptProperties().getProperty('ANTHROPIC_API_KEY');
```

スプレッドシートを他の人と共有していても、スクリプトプロパティの値は閲覧者には見えません。

### ステップ2：1セルだけ要約する最小構成コード

いきなり全行処理しようとするのはよくある失敗パターンです。まず1セルだけ動かして、レスポンスが正しく返ってくることを確認するのが最速の近道です。

```javascript
function testSingleSummarize() {
  const apiKey = PropertiesService.getScriptProperties().getProperty('ANTHROPIC_API_KEY');
  const inputText = 'ここに要約したいテキストを入れてください。長い文章であるほど効果が出ます。';

  const endpoint = 'https://api.anthropic.com/v1/messages';

  const headers = {
    'x-api-key': apiKey,
    'anthropic-version': '2023-06-01',  // 必須。省略するとエラー
    'content-type': 'application/json'
  };

  const body = {
    model: 'claude-3-5-sonnet-20241022',  // 2026年時点で動作確認済みモデルID
    max_tokens: 512,  // 必須パラメータ。省略不可
    messages: [
      {
        role: 'user',
        content: `以下のテキストを日本語で3行以内に要約してください。\n\n${inputText}`
      }
    ]
  };

  const options = {
    method: 'post',
    headers: headers,
    payload: JSON.stringify(body),
    muteHttpExceptions: true  // エラー時もレスポンスを取得するために必要
  };

  const response = UrlFetchApp.fetch(endpoint, options);
  const json = JSON.parse(response.getContentText());

  // 正しいレスポンス取得方法（response.completionは旧API。使用不可）
  const result = json.content[0].text;
  Logger.log(result);
}
```

実行後、GASエディタの「実行ログ」に要約文が表示されれば成功です。`json.content[0].text`という部分が現行Messages APIのレスポンス構造です。旧APIの`json.completion`と混同しないよう注意。

`muteHttpExceptions: true`を設定すると、APIエラー時にGASのスクリプト自体がクラッシュせず、エラーレスポンスの内容をログで確認できます。デバッグ時に必須のオプションです。

---

## 複数セルの一括要約と自動書き込み｜実装の完成形コード

1セルの動作確認が取れたら、スプレッドシートのA列を全行スキャンしてB列に書き込む処理に拡張します。

### ループ処理の完成形コード

```javascript
function summarizeAllRows() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const lastRow = sheet.getLastRow();
  const apiKey = PropertiesService.getScriptProperties().getProperty('ANTHROPIC_API_KEY');

  for (let i = 2; i <= lastRow; i++) {  // 1行目はヘッダーとして除外
    const inputCell = sheet.getRange(i, 1);   // A列：入力テキスト
    const outputCell = sheet.getRange(i, 2);  // B列：要約結果
    const flagCell = sheet.getRange(i, 3);    // C列：処理済みフラグ

    // すでに処理済みの行はスキップ（重複呼び出し防止）
    if (flagCell.getValue() === '済') continue;

    const inputText = inputCell.getValue();
    if (!inputText) continue;  // 空セルもスキップ

    try {
      const summary = callClaudeAPI(inputText, apiKey);
      outputCell.setValue(summary);
      flagCell.setValue('済');
      SpreadsheetApp.flush();  // 逐次書き込みで進捗を可視化
    } catch (e) {
      flagCell.setValue('エラー: ' + e.message);
      Logger.log(`行${i}でエラー: ${e.message}`);
    }

    Utilities.sleep(1000);  // レート制限対策。1秒待機
  }
}

function callClaudeAPI(text, apiKey) {
  const endpoint = 'https://api.anthropic.com/v1/messages';
  const headers = {
    'x-api-key': apiKey,
    'anthropic-version': '2023-06-01',
    'content-type': 'application/json'
  };
  const body = {
    model: 'claude-3-haiku-20240307',  // 大量処理はHaikuでコスト削減
    max_tokens: 256,
    messages: [{
      role: 'user',
      content: `以下を日本語で3行以内に要約してください。\n\n${text}`
    }]
  };
  const options = {
    method: 'post',
    headers: headers,
    payload: JSON.stringify(body),
    muteHttpExceptions: true
  };
  const response = UrlFetchApp.fetch(endpoint, options);
  const json = JSON.parse(response.getContentText());

  if (json.error) throw new Error(json.error.message);
  return json.content[0].text;
}
```

### コード設計の3つのポイント

**C列にフラグ列を設ける。** これが地味に重要です。処理中にタイムアウトしてスクリプトが途中で止まっても、再実行時に「済」の行を飛ばせます。フラグなしで再実行すると全行をAPIに投げ直すことになり、コストが倍になります。

**Haikuで大量処理、Sonnetで高品質処理と使い分ける。** Claude 3 Haikuは入力1Mトークンあたり$0.25。Sonnetは$3.00。同じ要約タスクなら10分の1以下のコストで処理できます。100行のフィードバック要約ならHaikuで十分です。

**`Utilities.sleep(1000)`は必須。** 連続リクエストを投げると`529 Overloaded`エラーが頻発します。1秒待機を入れるだけでほぼ解消します。処理速度より安定性を優先する設計です。

### トリガーで「編集時に自動要約」する設定

毎回手動で実行するのではなく、スプレッドシートを編集したときに自動で走らせることもできます。

GASエディタの「トリガー」（時計アイコン）から以下を設定：

- 実行する関数：`summarizeAllRows`
- イベントのソース：スプレッドシートから
- イベントの種類：変更時

ただし注意点があります。「変更時」トリガーは編集のたびに発火するため、大量行があると毎回全行スキャンになりコストが増加します。実運用では「B列が空白の行だけ処理する」という条件を組み込むか、ボタンを設置して手動実行する設計の方が現実的です。

---

## コスト試算と運用設計｜動かす前に必ず計算する

「使ってみたら思ったよりAPIコストがかかった」という話はよく聞きます。事前に試算しておけば防げます。

### 具体的なコスト計算

日本語テキスト1セル（300文字程度）の処理で消費するトークンの目安：

- 入力：プロンプト込みで約600〜800トークン
- 出力：要約3行で約150〜200トークン

**Claude 3 Haikuで100行処理した場合：**
- 入力：800トークン × 100行 = 80,000トークン → $0.02
- 出力：200トークン × 100行 = 20,000トークン → $0.006（出力は$0.25/1Mトークン）
- 合計：約$0.026（3円程度）

**Claude 3.5 Sonnetで100行処理した場合：**
- 入力コストだけで $0.24（Haikuの約12倍）
- 高品質な文章生成が必要な場合のみSonnetを使う判断が正解です

**GASの無料枠について。** `UrlFetchApp`の呼び出し上限は1日20,000回（Google Workspace無料版）。100行処理を毎日1回なら問題ありません。ただしGASのスクリプト1実行あたりのタイムアウトは6分。1秒待機を挟むと1分あたり最大60行処理できるため、360行程度が1回あたりの上限になります。1,000行以上のデータを処理するなら、1日複数回に分割するか、Anthropicが提供するBatch APIへの移行を検討してください。Batch APIは通常APIより最大50%コストを削減できます。

---

## よくある失敗とエラーの原因｜詰まったときの確認リスト

### Q1. `400 Bad Request`が返ってきて動かない

最も多いパターンは3つ。

1. `anthropic-version`ヘッダーが抜けている
2. `max_tokens`を省略している（Messages APIでは必須）
3. `messages`配列の`role`に`user`以外の値を入れている（最初のメッセージは必ず`user`）

`muteHttpExceptions: true`を設定した上で`Logger.log(response.getContentText())`でレスポンス全体を出力すると、エラーの詳細メッセージが確認できます。

### Q2. レスポンスからテキストが取れずundefinedになる

```javascript
// ❌ 旧Completion APIの書き方。現在は動作しない
const result = json.completion;

// ✅ 現行Messages APIの正しい書き方
const result = json.content[0].text;
```

`json`の中身をそのまま`Logger.log(JSON.stringify(json))`で出力すると、レスポンスの構造が確認できます。一度これをやっておくと理解が深まります。

### Q3. 途中でスクリプトが止まる

GASのタイムアウト（6分）に引っかかっているか、APIから`529 Overloaded`が返っている可能性があります。

対策は2つ。`Utilities.sleep()`の待機時間を1500〜2000msに増やすこと。そして前述のフラグ列設計で、再実行時に処理済み行をスキップできるようにしておくこと。これがあれば途中停止しても安心して再実行できます。

### 正直なデメリット

- GASはローカル環境がなくブラウザだけで動く反面、デバッグ環境が貧弱です。エラーのスタックトレースが見づらく、複雑なコードのデバッグに時間がかかります
- APIのレスポンス速度はネットワーク状況に依存します。1セルあたり2〜5秒かかるため、1,000行処理は現実的に数時間かかります
- Claude APIは従量課金のため、バグがあってループが暴走すると予期しない課金が発生します。Anthropicのコンソールで利用上限（Monthly Spend Limit）を設定しておくことを強く勧めます

---

## まとめ｜今日やるべき1つのアクション

記事の内容を振り返ると：

- 旧Completion APIは廃止済み。`/v1/messages`への移行が必須
- APIキーはスクリプトプロパティに格納。ベタ書きは厳禁
- `max_tokens`と`anthropic-version`ヘッダーは省略不可
- レスポンスは`json.content[0].text`で取得
- フラグ列と`Utilities.sleep()`で安全なループ処理を設計する

**今日やること、1つだけ選ぶとすれば：** `testSingleSummarize()`関数をGASに貼り付けて、まず1セル動かしてください。ログにAIの要約文が出てきた瞬間、次に何を自動化できるかアイデアが止まらなくなります。その感覚を体験するのが最速の入口です。

複数モデルの使い分けや、Batch API・エラーハンドリングの詳細設計については、実際に運用し始めてからで十分です。まず動かすことが先です。

---

## 関連記事

- [Claude API×GASでスプレッドシート自動化｜2025年版コピペで動く設定手順](/2026/05/26/Claude-APIGASでスプレッドシート自動化2025年版コピペで動く設定手順/)
- [CursorでGAS自動生成｜コード不要でNotion連携を作る手順](/2026/05/30/Cursor-AI-GAS自動生成非エンジニアのNotion連携スクリプト作成手順/)
- [Google Apps Script活用例｜Googleカレンダーにリマインド通知を自動設定する方法](/2026/05/23/ChatGPTGoogleカレンダー連携GASで予定自動登録リマインド通知を設定する方法/)

---

## 📘 もっと深く学びたい方へ

この記事で紹介した内容を、さらに体系的に・実務レベルで習得できる教材を販売中です。

### [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs)

API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

- 動くGASコード・API設定手順・プロンプトをワンセット収録
- スプレッドシート連携／メール／議事録／請求書を実務レベルで自動化
- コピペで即動く実装コード（Python / GAS）付き

👉 [今すぐ購入する](https://aijissenlab.gumroad.com/l/irvscs)

### [ChatGPT＆Claude AIプロンプト集50選（¥980）](https://aijissenlab.gumroad.com/l/yjcdam)

コピペで即使える実践プロンプト50種を全24ページに凝縮

- ビジネスメール・企画書・分析・コーディング等 8カテゴリ網羅
- ChatGPT / Claude / Gemini 全対応
- 変数を埋めるだけで即実務投入

👉 [今すぐ購入する](https://aijissenlab.gumroad.com/l/yjcdam)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。