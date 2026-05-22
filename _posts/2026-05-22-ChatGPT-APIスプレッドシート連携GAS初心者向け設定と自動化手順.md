---
layout: post
title: "ChatGPT API×スプレッドシート連携｜GAS初心者向け設定と自動化手順"
date: 2026-05-22 17:57:54 +0900
description: "GAS×ChatGPT APIでスプレッドシートを自動化する手順を解説。APIキーの安全な設定からエラー対策・コスト節約まで、初心者がつまずく落とし穴をコピペコード付きで紹介。"
tags:
  - ChatGPT API Google スプレッドシート 連携 やり方
  - GAS ChatGPT 自動化 初心者 設定方法
  - Google スプレッドシート AI 関数 使い方
  - GAS ChatGPT APIキー 安全な格納方法
---

スプレッドシートの作業、まだ手動でやってますか。

100行のデータを分類する、メール文面を量産する、顧客コメントを要約する——そういう「頭を使わなくていい繰り返し作業」こそ、ChatGPT APIに丸投げできます。

この記事を読めば、Google スプレッドシート＋GASでChatGPT APIを動かす手順が、今日中に手元で再現できます。

コードのコピペだけで終わらせず、**本番運用で壊れない設計**（セキュリティ・エラー処理・6分制限の回避）まで踏み込んで説明します。2026年現在、古いブログ記事のコードをそのまま使って「動かない」と詰まる人が多いので、その罠も全部先に教えます。

---

## GAS×ChatGPT APIで何ができて、何ができないか

まず正直に書きます。「スプレッドシートにAIを繋げば何でもできる」は過信です。

### 得意なこと

GAS経由でChatGPT APIを叩くと、こういう処理がほぼ自動化できます。

- **データ分類・タグ付け**：商品レビューをポジティブ/ネガティブに仕分け
- **文章生成**：顧客名や条件をセルから読み込んでメール文面を量産
- **要約・抽出**：長い問い合わせ文から要点だけ抜き出す
- **データクレンジング**：表記ゆれを統一（「株式会社」「（株）」を揃えるなど）

実際の導入事例では、タグ付け作業が手動比で**70〜85%の工数削減**、メール文面生成は1件あたり**5分→30秒**程度に縮んでいます。

### ここは無理、またはノーコードツールで代替を検討すべき

**GASの実行時間は1回6分が上限**です。これは有料のWorkspaceでも変わりません。

数百〜数千行に対してAPI呼び出しを一括でかけると、処理途中で強制終了します。大量行処理には「バッチ分割＋時間起動トリガー」という設計が必要で、初心者にはハードルが上がります。

そういうケースでは、Make（旧Integromat）やZapierのChatGPT連携テンプレートを使うほうが早い。GASを書く意味があるのは、「スプレッドシート上のデータと密結合した処理」「社内で誰でも操作できるUIが欲しい」といった場面です。

> **GASの主な実行制限（2026年5月時点）**
> - 1回の実行時間：6分
> - 1日のトリガー実行時間：無料90分、Workspace6時間
> - URLフェッチ呼び出し：20,000回/日

---



> 💡 **関連教材**: [ChatGPT＆Claude AIプロンプト集50選（¥980）](https://aijissenlab.gumroad.com/l/yjcdam) — コピペで即使える実践プロンプト50種を全24ページに凝縮

## セキュアな初期設定｜APIキーの格納からプロジェクト作成まで

### APIキーを「べた書き」するのは絶対にやめてください

古い記事には平気でこういうコードが載っています。

```javascript
const apiKey = 'sk-xxxxxxxxxxxxxxxxxxxxxxxx'; // ← 危険
```

このまま使うと、スクリプトを誰かに見せた瞬間、あるいはGitHubに誤ってプッシュした瞬間にキーが漏洩します。2025年以降、この手の事故が増えています。

**正解は`PropertiesService`への格納**です。

### 手順1｜スクリプトエディタを開く

Google スプレッドシートを開き、上部メニューの「拡張機能」→「Apps Script」をクリック。ブラウザの別タブでスクリプトエディタが開きます。

### 手順2｜APIキーをスクリプトプロパティに保存する

エディタ左側の歯車アイコン（プロジェクトの設定）→「スクリプト プロパティ」→「プロパティを追加」で以下を入力して保存します。

- プロパティ名：`OPENAI_API_KEY`
- 値：`sk-`から始まるあなたのAPIキー

コードからは次のように呼び出します。

```javascript
const apiKey = PropertiesService.getScriptProperties().getProperty('OPENAI_API_KEY');
```

これでスクリプト本体にキーの文字列が一切残りません。

### モデル選定の最新ガイド（2026年5月時点）

古いブログ記事に「`gpt-4`を指定」と書いてあるのを見かけますが、現在`gpt-4`は非推奨・廃止方向です。そのまま書くと動かないか、エラーが返ってきます。

| モデル | 用途の目安 | コスト感 |
|---|---|---|
| `gpt-4o-mini` | 大量処理・分類・タグ付け | 入力$0.15/1Mトークン（激安） |
| `gpt-4o` | 精度重視・複雑な文章生成 | 入力$2.50/1Mトークン |
| `gpt-4.1` | コーディング・長文処理 | 4o miniと同等コスト帯 |

スプレッドシートの自動化用途なら、**まず`gpt-4o-mini`から始める**のが正解です。精度が物足りなければ`gpt-4o`に切り替える、という順番で試してください。

1セルあたりの処理コストの目安：100文字程度のプロンプトと返答なら、`gpt-4o-mini`で**0.01円以下**に収まります。

---

## コピペで動くサンプルコード3選｜基本実装〜エラー対策まで

### サンプル①｜最小構成：セルの値をChatGPTに渡して隣に書く

```javascript
function runChatGPT() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const apiKey = PropertiesService.getScriptProperties().getProperty('OPENAI_API_KEY');
  const prompt = sheet.getRange('A2').getValue(); // A2セルの値をプロンプトに

  const payload = {
    model: 'gpt-4o-mini',
    messages: [{ role: 'user', content: prompt }],
    max_tokens: 500
  };

  const options = {
    method: 'post',
    contentType: 'application/json',
    headers: { Authorization: 'Bearer ' + apiKey },
    payload: JSON.stringify(payload)
  };

  const response = UrlFetchApp.fetch('https://api.openai.com/v1/chat/completions', options);
  const result = JSON.parse(response.getContentText());
  const answer = result.choices[0].message.content;

  sheet.getRange('B2').setValue(answer); // 結果をB2セルに書く
}
```

A2セルに「この商品レビューをポジティブ/ネガティブで分類してください：最高の品質でした」と入れてから実行すると、B2に「ポジティブ」と返ってきます。まずこれで動作確認してください。

### サンプル②｜システムプロンプトをセルから読み込む設計

非エンジニアがプロンプトを自由に変えられる設計です。コードを触らなくてよくなります。

```javascript
function runChatGPTWithSystemPrompt() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const apiKey = PropertiesService.getScriptProperties().getProperty('OPENAI_API_KEY');

  const systemPrompt = sheet.getRange('A1').getValue(); // A1にシステムプロンプトを書いておく
  const userInput = sheet.getRange('A2').getValue();    // A2にユーザー入力

  const payload = {
    model: 'gpt-4o-mini',
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: userInput }
    ],
    max_tokens: 500
  };

  const options = {
    method: 'post',
    contentType: 'application/json',
    headers: { Authorization: 'Bearer ' + apiKey },
    payload: JSON.stringify(payload)
  };

  const response = UrlFetchApp.fetch('https://api.openai.com/v1/chat/completions', options);
  const result = JSON.parse(response.getContentText());
  sheet.getRange('B2').setValue(result.choices[0].message.content);
}
```

A1セルに「あなたはECサイトのカスタマーサポートです。丁寧な日本語で返信を作成してください。」と書いておけば、A2に問い合わせ内容を入れるだけで返信案が生成されます。

### サンプル③｜複数行をバッチ処理＋エラーハンドリング付き

これが本番運用で使える最小構成です。`try-catch`とログ出力を入れています。

```javascript
function runBatchChatGPT() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const apiKey = PropertiesService.getScriptProperties().getProperty('OPENAI_API_KEY');
  const lastRow = sheet.getLastRow();

  for (let i = 2; i <= lastRow; i++) {
    const input = sheet.getRange(i, 1).getValue();
    if (!input) continue; // 空行はスキップ

    try {
      const payload = {
        model: 'gpt-4o-mini',
        messages: [{ role: 'user', content: input }],
        max_tokens: 300
      };

      const options = {
        method: 'post',
        contentType: 'application/json',
        headers: { Authorization: 'Bearer ' + apiKey },
        payload: JSON.stringify(payload),
        muteHttpExceptions: true // エラー時も例外を投げず処理継続
      };

      const response = UrlFetchApp.fetch('https://api.openai.com/v1/chat/completions', options);
      const code = response.getResponseCode();

      if (code === 200) {
        const result = JSON.parse(response.getContentText());
        sheet.getRange(i, 2).setValue(result.choices[0].message.content);
      } else if (code === 429) {
        // レート制限：少し待って再試行
        Logger.log('Row ' + i + ': Rate limited. Waiting...');
        Utilities.sleep(5000);
        i--; // 同じ行を再処理
      } else {
        sheet.getRange(i, 2).setValue('ERROR: ' + code);
        Logger.log('Row ' + i + ': Error ' + code);
      }

      Utilities.sleep(500); // 連続リクエストを避けるインターバル

    } catch (e) {
      sheet.getRange(i, 2).setValue('EXCEPTION: ' + e.message);
      Logger.log('Row ' + i + ': ' + e.message);
    }
  }
}
```

`muteHttpExceptions: true`を入れないと、429エラー（レート制限）が来た瞬間にスクリプト全体が止まります。これ、意外と知られていないんですが、本番運用で最初に詰まるポイントです。

100行を超えるデータを処理する場合は、6分制限を超える可能性があります。その場合は処理済みの行番号をPropertiesServiceに保存し、時間起動トリガーで続きから再開する設計に切り替えてください。

---

## コスト・セキュリティ・運用設計｜本番で壊れないための3つのポイント

### ポイント1｜APIキーには使用量上限を設定する

OpenAIの管理画面（platform.openai.com）でAPIキーごとに月次の使用上限を設定できます。万が一のキー漏洩・バグによる暴走課金を防ぐために、**必ず上限を設定してから本番稼働させてください**。目安は月次想定コストの1.5倍程度。

### ポイント2｜処理コストは`gpt-4o-mini`で試算する

`gpt-4o-mini`は入力$0.15・出力$0.60（1Mトークンあたり）です。スプレッドシートで1000行のタグ付けをするとして、1行あたりプロンプト＋レスポンスを合計200トークンと見積もると、**合計20万トークン≒約$0.09（13円前後）**。ほぼタダです。

精度に問題がなければ、`gpt-4o-mini`で十分なケースがほとんどです。

### ポイント3｜実行ログはスプレッドシート内に残す

GASの`Logger.log()`は実行後しばらくすると消えます。本番運用では、エラー情報や処理件数を専用のログシートに書き込む設計にしておくと、トラブル時の調査が格段に楽になります。

```javascript
// ログシートへの書き込み例
const logSheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('ログ');
logSheet.appendRow([new Date(), '処理完了', i + '行目', answer.substring(0, 50)]);
```

---

## まとめと次のアクション

GAS×ChatGPT APIの連携は、仕組み自体はシンプルです。ただし、「コピペして動いた」と「本番で壊れない」の間には、APIキーの管理・エラーハンドリング・6分制限への対応という3つの壁があります。

今日やるべきことは1つだけです。

**サンプル①の最小構成コードをスプレッドシートに貼り付けて、実際に1セル分の処理を動かしてみてください。**

動作確認できたら、サンプル③のバッチ処理に置き換えて、実際の業務データを10行だけ流してみる。それだけで、自動化の感触がリアルにつかめます。

APIキーの取得がまだの方は、OpenAIの公式サイトでアカウントを作成し、「API keys」メニューからキーを発行してください。取得後すぐにこの記事の手順2（PropertiesServiceへの格納）を実行してください。べた書きはしないように。

---

## 📘 もっと深く学びたい方へ

この記事で紹介した内容を、さらに体系的に・実務レベルで習得できる教材を販売中です。

### [ChatGPT＆Claude AIプロンプト集50選（¥980）](https://aijissenlab.gumroad.com/l/yjcdam)

コピペで即使える実践プロンプト50種を全24ページに凝縮

- ビジネスメール・企画書・分析・コーディング等 8カテゴリ網羅
- ChatGPT / Claude / Gemini 全対応
- 変数を埋めるだけで即実務投入

👉 [今すぐ購入する](https://aijissenlab.gumroad.com/l/yjcdam)

### [AIブログ運営 月1万円達成ステップガイド（¥980）](https://aijissenlab.gumroad.com/l/zxrhbg)

ジャンル選定→記事作成→SEO→収益化の全8ステップを完全解説

- AIプロンプト8種・WordPress設定チェックリスト付き
- ジャンル選定からアフィリエイト・AdSenseまで網羅
- 初心者でも月1万円までの最短ロードマップ

👉 [今すぐ購入する](https://aijissenlab.gumroad.com/l/zxrhbg)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。

---

## 関連記事

- [Notion AI×ChatGPT連携のやり方｜Zapierなし自動化4選](/2026/05/22/Notion-AIChatGPT連携のやり方Zapierなし自動化4選/)
- [ChatGPT APIキー取得と使い方｜料金・無料枠・初心者のつまずき解説](/2026/05/22/ChatGPT-APIキー取得と使い方料金無料枠初心者のつまずき解説/)
- [Claude APIキー取得の手順と料金｜GPT-4oコスト比較・無料枠の実態も解説](/2026/05/22/Claude-APIキー取得の手順と料金GPT-4oコスト比較無料枠の実態も解説/)

