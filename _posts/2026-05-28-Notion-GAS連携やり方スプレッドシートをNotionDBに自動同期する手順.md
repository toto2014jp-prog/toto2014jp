---
layout: post
title: "Notion GAS連携やり方｜スプレッドシートをNotionDBに自動同期する手順"
date: 2026-05-28 09:07:56 +0900
description: "スプレッドシートのデータをGASでNotionデータベースに自動同期する手順を解説。APIキー設定・コピペ用サンプルコード・重複防止・レート制限対策まで実務レベルで紹介。"
tags:
  - Notion GAS スプレッドシート自動同期
  - Notion API Google スプレッドシート連携
  - GAS Notion データベース自動化
  - Notion スプレッドシート 重複防止 PageID
---

## スプレッドシートのデータ、毎回手でNotionに転記していませんか

Excelやスプレッドシートで管理していたデータをNotionにも反映させる。この作業、地味にしんどい。

コピペミスが怖くて何度も見直す。更新のたびに同じ作業を繰り返す。気づいたら片方が古いままになっている。

正直なところ、この「二重管理の地獄」から抜け出したくてGASを調べる人がほとんどだと思う。この記事では、Google Apps Script（GAS）を使ってスプレッドシートのデータをNotionデータベースに自動同期する手順を、コピペで使えるサンプルコード付きで解説する。

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

## GAS・Zapier・公式連携の比較｜どれを使うべきか先に判断する

「とりあえずGASで」と始める前に、一度立ち止まってほしい。ツールの選択を間違えると、構築後に後悔することになる。

### 選択肢は3つある

**Zapier・Make（旧Integromat）などのノーコードツール**は、コードを書かずに連携を作れる。ただし月額コストがかかる。Makeの場合、月1,000オペレーション超から有料プランが必要になり、データ量によっては月数千〜数万円になることも珍しくない。

**Notion公式のAI Connector**（2025年から順次展開中）は、Googleワークスペースとの接続をNotion内で完結させようとする機能だ。ただし現時点では参照・要約が中心で、スプレッドシートのデータをNotionDBに「書き込む」用途にはまだ向いていない。

**GAS**は無料で動き、条件分岐や差分同期などの細かいロジックを自分で組める。初期構築に数時間かかるが、一度動けばランニングコストはゼロだ。

### 判断の目安はこう考える

- 月100件以下の同期で条件分岐が不要 → MakeやZapierで十分
- 月数百件以上、または「更新行だけ同期したい」などの細かい制御が必要 → GASが現実的
- コードを書きたくない・メンテできる人がいない → ノーコードを選ぶ

この記事を読んでいる人の多くは後者のケースだと思うので、以降はGASでの実装を進める。

---

## 事前準備｜Notion APIキー取得とGASプロジェクトの初期設定

### Notionインテグレーションの作成

まずNotionの[インテグレーション管理画面](https://www.notion.so/my-integrations)にアクセスする。

1. 「新しいインテグレーション」をクリック
2. 名前（例：`GAS Sync`）とワークスペースを選択
3. 「コンテンツを読む」「コンテンツを挿入する」「コンテンツを更新する」の3つにチェック
4. 「送信」でトークンが発行される

トークンは `secret_xxxxxxxx` という形式。これがAPIキーになる。

### ⚠️ここを忘れると403エラーになる

インテグレーションを作っただけでは使えない。**同期先のNotionデータベースにインテグレーションを「コネクト」する操作が別途必要**だ。

データベースのページ右上「…」→「コネクト先」→ 作成したインテグレーション名を選択。これをやらないと、APIリクエストのたびに403が返ってくる。ハマりやすい落とし穴なので最初に確認しておく。

### データベースIDの取得

NotionデータベースのURLを見ると、以下のような形式になっている。

```
https://www.notion.so/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx?v=yyyy
```

`?v=` より前の32文字がデータベースIDだ。`8f3b...` のような英数字の羅列をメモしておく。

### GAS側の初期設定

スプレッドシートを開いて「拡張機能」→「Apps Script」を選択。

APIキーやデータベースIDをコードに直書き（ハードコーディング）するのは危険だ。スクリプトを共有した瞬間に流出する。必ずスクリプトプロパティに格納する。

```javascript
// スクリプトプロパティへの保存手順
// 「プロジェクトの設定」→「スクリプトプロパティ」→ 以下を追加
// NOTION_TOKEN : secret_xxxxxxxx
// DATABASE_ID  : xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

コード側では以下のように呼び出す。

```javascript
const props = PropertiesService.getScriptProperties();
const NOTION_TOKEN = props.getProperty('NOTION_TOKEN');
const DATABASE_ID = props.getProperty('DATABASE_ID');
```

### スプレッドシートの列設計とNotionプロパティ型の対応

Notionのプロパティ型によって、GASから渡すデータの形式が変わる。事前に対応表を整理しておくと実装がスムーズだ。

| Notionの型 | GASでの値の渡し方（抜粋） |
|---|---|
| title | `{title: [{text: {content: "値"}}]}` |
| rich_text | `{rich_text: [{text: {content: "値"}}]}` |
| number | `{number: 数値}` |
| select | `{select: {name: "選択肢名"}}` |
| date | `{date: {start: "2025-01-01"}}` |
| checkbox | `{checkbox: true}` |

**注意**：RollupとFormulaはAPIからの書き込み不可。読み取りのみに対応している。これはどうにもならない仕様なので、そもそも設計段階で書き込み対象から外す必要がある。

---

## コピペ用サンプルコード｜新規作成・更新・重複防止を一括実装

3段階に分けて紹介する。Step1から順に追加していく構成なので、自分のユースケースに合わせて止める段階を選んでほしい。

### Step1｜新規行をNotionDBにPOSTする最小構成

まず「スプレッドシートの全行をNotionに新規作成する」だけのシンプルなコードから始める。

```javascript
function syncToNotion() {
  const props = PropertiesService.getScriptProperties();
  const NOTION_TOKEN = props.getProperty('NOTION_TOKEN');
  const DATABASE_ID = props.getProperty('DATABASE_ID');

  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const data = sheet.getDataRange().getValues();
  const headers = data[0]; // 1行目をヘッダーとして使用

  for (let i = 1; i < data.length; i++) {
    const row = data[i];

    // Notionに送るプロパティを組み立てる
    const properties = {
      "タスク名": {
        title: [{ text: { content: row[0] } }]
      },
      "担当者": {
        rich_text: [{ text: { content: row[1] } }]
      },
      "ステータス": {
        select: { name: row[2] }
      }
    };

    const payload = {
      parent: { database_id: DATABASE_ID },
      properties: properties
    };

    const options = {
      method: 'post',
      headers: {
        'Authorization': 'Bearer ' + NOTION_TOKEN,
        'Content-Type': 'application/json',
        'Notion-Version': '2022-06-28'
      },
      payload: JSON.stringify(payload),
      muteHttpExceptions: true
    };

    UrlFetchApp.fetch('https://api.notion.com/v1/pages', options);

    Utilities.sleep(400); // レート制限対策（3req/秒）
  }
}
```

このままでは実行のたびに全行が重複して作成される。次のステップで解決する。

### Step2｜PageIDをスプレッドシートに書き戻して重複防止

NotionはページIDでデータを管理している。スプレッドシート側にPageIDの列を用意して、作成済みの行をスキップするロジックを追加する。

スプレッドシートの最終列（例：E列 = 5列目）を「NotionPageID」列として確保しておく。

```javascript
function syncToNotion() {
  const props = PropertiesService.getScriptProperties();
  const NOTION_TOKEN = props.getProperty('NOTION_TOKEN');
  const DATABASE_ID = props.getProperty('DATABASE_ID');

  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const data = sheet.getDataRange().getValues();
  const ID_COLUMN = 5; // E列（1始まり）

  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    const existingPageId = row[ID_COLUMN - 1];

    const properties = {
      "タスク名": {
        title: [{ text: { content: String(row[0]) } }]
      },
      "担当者": {
        rich_text: [{ text: { content: String(row[1]) } }]
      },
      "ステータス": {
        select: { name: String(row[2]) }
      }
    };

    let response;

    if (!existingPageId) {
      // PageIDなし → 新規作成（POST）
      const payload = {
        parent: { database_id: DATABASE_ID },
        properties: properties
      };
      const options = {
        method: 'post',
        headers: {
          'Authorization': 'Bearer ' + NOTION_TOKEN,
          'Content-Type': 'application/json',
          'Notion-Version': '2022-06-28'
        },
        payload: JSON.stringify(payload),
        muteHttpExceptions: true
      };
      response = UrlFetchApp.fetch('https://api.notion.com/v1/pages', options);
      const pageData = JSON.parse(response.getContentText());

      // 作成されたPageIDをスプレッドシートに書き戻す
      sheet.getRange(i + 1, ID_COLUMN).setValue(pageData.id);

    } else {
      // PageIDあり → 更新（PATCH）
      const options = {
        method: 'patch',
        headers: {
          'Authorization': 'Bearer ' + NOTION_TOKEN,
          'Content-Type': 'application/json',
          'Notion-Version': '2022-06-28'
        },
        payload: JSON.stringify({ properties: properties }),
        muteHttpExceptions: true
      };
      UrlFetchApp.fetch(
        'https://api.notion.com/v1/pages/' + existingPageId,
        options
      );
    }

    Utilities.sleep(400);
  }
}
```

これで「2回目以降は更新のみ、新しい行だけ追加」という動作になる。

### Step3｜差分のみ同期する完成版

全行を毎回処理すると、1,000行超のデータではGASの6分制限に引っかかる。最終同期日時をスクリプトプロパティに保存して、それ以降に更新された行だけを処理するようにする。

スプレッドシートに「最終更新日時」列（例：F列 = 6列目）を追加しておく前提だ。

```javascript
function syncToNotionDiff() {
  const props = PropertiesService.getScriptProperties();
  const NOTION_TOKEN = props.getProperty('NOTION_TOKEN');
  const DATABASE_ID = props.getProperty('DATABASE_ID');
  const lastSyncStr = props.getProperty('LAST_SYNC_TIME');
  const lastSync = lastSyncStr ? new Date(lastSyncStr) : new Date(0);

  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const data = sheet.getDataRange().getValues();
  const ID_COLUMN = 5;       // E列：NotionPageID
  const UPDATED_COLUMN = 6;  // F列：最終更新日時

  const syncStart = new Date(); // 今回の同期開始時刻

  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    const updatedAt = new Date(row[UPDATED_COLUMN - 1]);

    // 前回同期以降に更新された行のみ処理
    if (updatedAt <= lastSync) continue;

    const existingPageId = row[ID_COLUMN - 1];
    const properties = {
      "タスク名": {
        title: [{ text: { content: String(row[0]) } }]
      },
      "担当者": {
        rich_text: [{ text: { content: String(row[1]) } }]
      },
      "ステータス": {
        select: { name: String(row[2]) }
      }
    };

    if (!existingPageId) {
      const options = {
        method: 'post',
        headers: {
          'Authorization': 'Bearer ' + NOTION_TOKEN,
          'Content-Type': 'application/json',
          'Notion-Version': '2022-06-28'
        },
        payload: JSON.stringify({
          parent: { database_id: DATABASE_ID },
          properties: properties
        }),
        muteHttpExceptions: true
      };
      const response = UrlFetchApp.fetch('https://api.notion.com/v1/pages', options);
      const pageData = JSON.parse(response.getContentText());
      sheet.getRange(i + 1, ID_COLUMN).setValue(pageData.id);
    } else {
      const options = {
        method: 'patch',
        headers: {
          'Authorization': 'Bearer ' + NOTION_TOKEN,
          'Content-Type': 'application/json',
          'Notion-Version': '2022-06-28'
        },
        payload: JSON.stringify({ properties: properties }),
        muteHttpExceptions: true
      };
      UrlFetchApp.fetch(
        'https://api.notion.com/v1/pages/' + existingPageId,
        options
      );
    }

    Utilities.sleep(400);
  }

  // 同期完了後に最終同期時刻を更新
  props.setProperty('LAST_SYNC_TIME', syncStart.toISOString());
}
```

### 自動実行のトリガー設定

GASの「トリガー」機能を使えば、定期実行できる。設定は「トリガーを追加」から以下のように設定する。

- 実行する関数：`syncToNotionDiff`
- イベントのソース：時間主導型
- 時間ベースのトリガーのタイプ：分ベースのタイマー
- 時間の間隔：5分ごと（最小1分）

GASはWebhookの受信ができないため、リアルタイム同期は不可能だ。5〜15分間隔のポーリングが現実的な落としどころになる。

### 1,000行を超えるデータの扱い

6分の実行制限に引っかかりそうな場合は、処理する行数に上限を設けて複数回に分割する。`LAST_SYNC_TIME`の代わりに「最後に処理した行番号」をスクリプトプロパティに保存すると安定する。データ量が多い場合はこの設計を検討してほしい。

---

## 次にやること：まずStep2のコードを1つのスプレッドシートで動かしてみる

Step3の差分同期まで一気に実装しようとすると、どこかでつまずいたときに原因が特定しにくくなる。

まずStep2（重複防止付き新規作成・更新）をテスト用のスプレッドシートとNotionDBで動かしてみること。PageIDがE列に書き戻されることを確認できれば、その時点で連携の土台は完成している。

そこから差分同期やエラーハンドリングを追加していくのが、失敗しない進め方だ。

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
- [Claude API×GASでスプレッドシート自動化｜2025年版コピペで動く設定手順](/2026/05/26/Claude-APIGASでスプレッドシート自動化2025年版コピペで動く設定手順/)
- [ChatGPT×Googleカレンダー連携｜GASで予定自動登録・リマインド通知を設定する方法](/2026/05/23/ChatGPTGoogleカレンダー連携GASで予定自動登録リマインド通知を設定する方法/)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。