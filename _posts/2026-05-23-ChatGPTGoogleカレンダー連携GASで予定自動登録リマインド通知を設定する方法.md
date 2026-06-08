---
layout: post
title: "ChatGPT×Googleカレンダー連携｜GASで予定自動登録・リマインド通知を設定する方法"
date: 2026-05-23 09:10:09 +0900
description: "ChatGPT APIとGASを使いGoogleカレンダーに予定を自動登録＆リマインド通知する手順を解説。APIキーの安全な管理からエラー対策まで、コピペで動くコード付きで紹介。"
tags:
  - ChatGPT Google カレンダー 自動登録 GAS
  - GAS OpenAI API カレンダー 連携 やり方
  - Google カレンダー AI 自動化 ノーコード 設定
  - ChatGPT gpt-4o-mini GAS 予定 自動化
---

「会議の日程をカレンダーに手動で登録するのがめんどくさい」——正直、これだけで十分な動機だと思う。

メール本文やSlackのメッセージから予定情報を拾い上げて、カレンダーに手入力する作業。週に10件もあれば、じわじわとストレスが積み上がる。

この記事では、ChatGPT APIとGoogle Apps Script（GAS）を組み合わせて、**自然言語の文章から予定を自動抽出→Googleカレンダーに登録→リマインド通知まで飛ばす**仕組みを作る。コードはコピペで動く最小構成から始めるので、GAS初心者でも手を動かしながら読み進められる。

---

## GASでやる理由｜ZapierやMakeより何がいいのか

ぶっちゃけ、Zapierでも同じようなことはできる。だから最初に「なぜGASなのか」を整理しておく。

**Zapier / Makeのデメリット**
- 無料プランはタスク数の上限がきつい（Zapierは月100タスク）
- 複雑な条件分岐を組むと月額コストが跳ね上がる
- カスタマイズの自由度に限界がある

**GASの優位点**
- Googleアカウントがあれば無料で使える
- GoogleカレンダーやGmailとの連携がAPIなしで書ける
- コードで細かく制御できる（条件分岐、エラー処理、通知先の切り替えなど）

コスト面で言うと、ChatGPT APIの料金は現実的にかなり安い。`gpt-4o-mini`を使えば、入力$0.15・出力$0.60（各100万トークンあたり）。月に予定を100件登録しても**$0.01未満**で収まる計算だ。

一方で、GASには制限があるのも事実なので正直に書いておく。

| 制限項目 | 上限値 |
|---|---|
| スクリプト実行時間（1回） | 最大6分 |
| トリガー実行時間（1日） | 90分（無料） |
| URLフェッチ（1日） | 20,000回 |
| カレンダーイベント作成（1日） | 5,000件 |

個人・小規模チームの用途なら、この制限に引っかかることはほぼない。

**Gemini統合との比較について**

2025年以降、GoogleカレンダーにはGeminiが統合されている。ただし、「外部のテキストを自動で拾ってパースする」「Slackや独自のwebhookに通知を飛ばす」といった処理はGemini単体ではできない。GASはその「橋渡し役」として今でも有効な選択肢だ。

> ノーコードで済ませたい場合は、別記事「Zapier×ChatGPT連携のやり方」も参照してほしい。

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

## 事前準備｜APIキー取得とGASの初期設定

### OpenAI APIキーを取得する

1. [platform.openai.com](https://platform.openai.com) にアクセスしてログイン
2. 右上のアカウントアイコン→「API keys」を選択
3. 「Create new secret key」をクリックし、名前をつけて生成
4. キーは**この画面でしか確認できない**のでメモしておく

### APIキーはコードに直書きしない

これ、意外と知られていない落とし穴だ。GASのコード内にAPIキーをそのまま書くと、スクリプトを共有した瞬間にキーが漏洩するリスクがある。

正しい格納方法は`PropertiesService`を使う方法だ。

**手順：**
GASエディタの上部メニューから「プロジェクトの設定」→「スクリプトプロパティ」→「プロパティを追加」

- プロパティ名：`OPENAI_API_KEY`
- 値：取得したAPIキーを貼り付け

コード内では以下のように呼び出す。

```javascript
const apiKey = PropertiesService.getScriptProperties().getProperty('OPENAI_API_KEY');
```

これだけでコードにキーが露出しなくなる。

### GASプロジェクトを作成する

1. Google Driveを開く
2. 「新規」→「その他」→「Google Apps Script」を選択
3. プロジェクト名を「カレンダー自動登録」などにしておく

### タイムゾーンの確認を忘れずに

GASのデフォルトタイムゾーンとカレンダーのタイムゾーンがズレていると、予定が1時間前後にずれて登録される。これで半日ハマった経験がある。

GASエディタの「プロジェクトの設定」→「タイムゾーン」が`Asia/Tokyo`になっているか確認しよう。コードでも念のため確認できる。

```javascript
function checkTimezone() {
  const cal = CalendarApp.getDefaultCalendar();
  Logger.log(cal.getTimeZone()); // Asia/Tokyo と出ればOK
}
```

---

## 実装ステップ｜自然言語→JSON変換→カレンダー登録

### 全体の処理フロー

```
入力テキスト（自然言語）
    ↓
GAS → OpenAI API（gpt-4o-mini）
    ↓
構造化JSON（title, date, startTime, endTime, description）
    ↓
GAS → Google Calendar API
    ↓
予定登録 ＋ リマインド通知（Gmail or Slack webhook）
```

### Step 1：ChatGPTに予定情報をJSONで返させる

プロンプト設計が精度の9割を決める。曖昧な指示だと出力がブレる。

```javascript
function extractEventFromText(inputText) {
  const apiKey = PropertiesService.getScriptProperties().getProperty('OPENAI_API_KEY');
  
  const prompt = `以下のテキストから予定情報を抽出し、JSON形式のみで返してください。
他の文章は一切含めないこと。

フィールド定義：
- title: 予定のタイトル（文字列）
- date: 日付（YYYY-MM-DD形式）
- startTime: 開始時刻（HH:MM形式）
- endTime: 終了時刻（HH:MM形式）
- description: 詳細メモ（文字列、なければ空文字）

入力テキスト：
${inputText}`;

  const payload = {
    model: 'gpt-4o-mini',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0
  };

  const options = {
    method: 'post',
    contentType: 'application/json',
    headers: { Authorization: `Bearer ${apiKey}` },
    payload: JSON.stringify(payload)
  };

  const response = UrlFetchApp.fetch('https://api.openai.com/v1/chat/completions', options);
  const json = JSON.parse(response.getContentText());
  const content = json.choices[0].message.content;
  
  return JSON.parse(content); // 構造化されたオブジェクトが返る
}
```

`temperature: 0`にするのがポイント。創造性が不要な抽出タスクは0にすることで出力が安定する。

### Step 2：Googleカレンダーに予定を登録する

```javascript
function addEventToCalendar(eventData) {
  const calendar = CalendarApp.getDefaultCalendar();
  
  // 日付と時刻を組み合わせてDateオブジェクトを生成
  const startDateTime = new Date(`${eventData.date}T${eventData.startTime}:00`);
  const endDateTime   = new Date(`${eventData.date}T${eventData.endTime}:00`);
  
  const options = {
    description: eventData.description
  };
  
  const event = calendar.createEvent(
    eventData.title,
    startDateTime,
    endDateTime,
    options
  );
  
  Logger.log(`登録完了：${event.getTitle()} / ${event.getStartTime()}`);
  return event;
}
```

### Step 3：2つの関数をつなぐメイン関数

```javascript
function main() {
  // ここに自然言語テキストを入れる
  const inputText = `来週月曜日の午後3時から4時に、田中さんとの進捗確認MTGがあります。
  場所はZoom、アジェンダは第2四半期レポートの確認です。`;
  
  try {
    const eventData = extractEventFromText(inputText);
    Logger.log('抽出されたデータ：' + JSON.stringify(eventData));
    addEventToCalendar(eventData);
  } catch (e) {
    Logger.log('エラー：' + e.message);
  }
}
```

`main()`を実行すると、テキストから日時・タイトルが自動抽出されてカレンダーに登録される。初めて動いたときは正直驚いた。

---

## リマインド通知の設定｜GmailとSlack webhookに飛ばす

カレンダーへの登録だけでは通知が弱い。GAS側からリマインドを能動的に送る仕組みを追加する。

### パターンA：Gmailでリマインド

```javascript
function sendReminder(eventData) {
  const recipient = Session.getActiveUser().getEmail();
  const subject   = `【リマインド】${eventData.title}`;
  const body      = `予定が近づいています。\n\n`
                  + `タイトル：${eventData.title}\n`
                  + `日時：${eventData.date} ${eventData.startTime}〜${eventData.endTime}\n`
                  + `詳細：${eventData.description}`;
  
  GmailApp.sendEmail(recipient, subject, body);
}
```

### パターンB：Slack webhookに通知

Slackの「Incoming Webhooks」でURL発行後、以下で通知できる。

```javascript
function sendSlackReminder(eventData) {
  const webhookUrl = PropertiesService.getScriptProperties().getProperty('SLACK_WEBHOOK_URL');
  
  const message = {
    text: `*【リマインド】${eventData.title}*\n`
        + `📅 ${eventData.date} ${eventData.startTime}〜${eventData.endTime}\n`
        + `📝 ${eventData.description}`
  };
  
  UrlFetchApp.fetch(webhookUrl, {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify(message)
  });
}
```

webhook URLもAPIキーと同様に、`PropertiesService`に格納しておくこと。

### 時間ベースのトリガーで自動実行する

GASエディタの「トリガー（時計アイコン）」→「トリガーを追加」から設定できる。

- 関数：`main`
- イベントのソース：時間主導型
- 種類：日付ベースのタイマー（毎朝9時など）

これで毎日決まった時間にスクリプトが走る。入力テキストをGmailやスプレッドシートから自動取得するように拡張すれば、完全に手放しで動く。

> **注意**：LINE Notifyは2025年3月にサービス終了している。LINE通知を使いたい場合はLINE Messaging APIへの移行が必要だ。

---

## まとめと次のアクション

ここまでやってきたことを整理する。

- GASを選ぶ理由：無料・柔軟・月$0.01未満で動く
- APIキーは`PropertiesService`に格納してコードから切り離す
- ChatGPTへの指示は「JSON形式のみ返せ」と明示することで出力が安定する
- タイムゾーンのズレは最初に確認しておく
- 通知はGmailかSlack webhookで飛ばす

**今すぐやるべきことは1つ**。GASエディタを開いて、`main()`関数に自分のテキストを貼り付けて実行してみること。動いた瞬間に「あ、これ本物だ」とわかる。そこからカスタマイズは自然と進む。

---

## 📘 もっと深く学びたい方へ

この記事で紹介した内容を、さらに体系的に・実務レベルで習得できる教材を販売中です。

### [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs)

API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

- 動くGASコード・API設定手順・プロンプトをワンセット収録
- スプレッドシート連携／メール／議事録／請求書を実務レベルで自動化
- コピペで即動く実装コード（Python / GAS）付き

👉 [今すぐ購入する](https://aijissenlab.gumroad.com/l/irvscs)

### [n8nノーコード自動化 実践ワークフロー集（¥1,980）](https://aijissenlab.gumroad.com/l/bltzrk)

Gmail/Slack/Notion/Claude連携の動くワークフロー10本を実装込みで完全解説（全35ページ）

- Cloud版前提・2026年最新のn8n v1系UI完全対応
- 動くノード設定値・JSON・OAuth設定まで全部入り
- 10ワークフロー（Gmail/Slack/Notion/Discord/Webhook/RSS）はコピペで即実務投入

👉 [今すぐ購入する](https://aijissenlab.gumroad.com/l/bltzrk)

---

## 関連記事

- [ChatGPT API×スプレッドシート連携｜GAS初心者向け設定と自動化手順](/2026/05/22/ChatGPT-APIスプレッドシート連携GAS初心者向け設定と自動化手順/)
- [ChatGPT APIキー取得と使い方｜料金・無料枠・初心者のつまずき解説](/2026/05/22/ChatGPT-APIキー取得と使い方料金無料枠初心者のつまずき解説/)
- [Notion AI×ChatGPT連携のやり方｜Zapierなし自動化4選](/2026/05/22/Notion-AIChatGPT連携のやり方Zapierなし自動化4選/)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。