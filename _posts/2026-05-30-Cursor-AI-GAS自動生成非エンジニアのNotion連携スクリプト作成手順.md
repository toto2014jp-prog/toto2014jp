---
layout: post
title: "Cursor AI GAS自動生成｜非エンジニアのNotion連携スクリプト作成手順"
date: 2026-05-30 09:10:06 +0900
description: "CursorでGASを自動生成しNotionとスプレッドシートを連携する手順を解説。動くコードを出力するプロンプトのコツと、非エンジニアが詰まる3つのポイントを先回りしてカバー。"
tags:
  - Cursor AI GAS 自動生成 やり方
  - Cursor 非エンジニア Google Apps Script 作り方
  - Cursor GAS Notion スプレッドシート 連携 コード生成
  - GAS Notion API 連携 プロンプト 書き方
---

「Cursorを使えばコードが書けると聞いたのに、実際に試したら全然動かなかった」——この記事はそういう人のために書いた。

とくにGASとNotion連携は、2024年以前のサンプルコードが今ほぼ動かない状態になっている。Notion APIの構造が変わったからだ。古い記事を参考にしていると、そこで詰まって終わる。

この記事では、**Cursorへの指示の書き方から、Apps Script IDEへのコピペ、認証の通し方まで**を一通り追う。コードを「読む」必要はない。動かす手順だけ追えばいい。

---

## CursorでGASを作る前に知っておくこと｜3つの「できないこと」

正直なところ、Cursorを初めて使う人の多くがここで誤解する。

**Cursorはローカルのコードエディタだ。**GASのエディタ（script.google.com）とは直接つながっていない。

フローはこうなる。

```
Cursorでコードを生成
　　↓
コードをコピー
　　↓
Apps Script IDEに貼り付け
　　↓
実行・テスト
```

これを前提に置かないと、「生成したのになぜ動かないのか」で永遠に詰まる。

### 生成コードがそのまま動かない3パターン

Cursorが出したコードが動かない理由は、だいたいこの3つに集約される。

**① OAuthスコープが未設定**
GASはスクリプトが使う権限を`appsscript.json`に明記する必要がある。Cursorはコードは生成してくれるが、このファイルの設定までは面倒を見てくれない。SpreadsheetやURLFetchを使うなら、対応するスコープを手動で追加する。

**② Notion APIのバージョン不一致**
2025年以降はNotion API v2系が標準になっている。古い記事のコードをそのままCursorに貼り付けて「修正して」と頼むと、旧構造のまま出力されることがある。**指示には必ず`2022-06-28`とバージョンを明記する**こと。

**③ 認証情報のハードコード**
Cursorはトークンをコード内に直書きするコードを出すことがある。これはそのまま使わない。GASには`PropertiesService`という安全な格納先がある。後述のプロンプトでその指示も含める。

### Notion APIでできないこと

もう一点だけ。NotionのDB操作はAPIでできる範囲が決まっている。

- ページの追加・更新：✅ できる
- 既存ページの読み取り：✅ できる
- DBのカラム（プロパティ）追加：❌ できない

GASを動かす前に、NotionDB側に必要なカラムをUIで作っておく必要がある。これを知らずに「カラムが増えない」と悩む人が多い。

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

## Cursorへの指示テンプレート｜動くGASが出る4要素

使ってみて驚いたのは、プロンプトの「情報量」でコードの質がまるで変わることだ。

「NotionとスプレッドシートをGASで連携して」だと、ほぼ確実に古い構文か抽象的なコードが出る。

動くコードを出すプロンプトには4つの要素が必要だ。

1. **ランタイムの明記**：「Google Apps Script（V8ランタイム）で」
2. **APIバージョンの明記**：「Notion API バージョン2022-06-28を使って」
3. **入出力の具体的な記述**：「スプレッドシートのA列とB列のデータを、NotionDBの『タイトル』と『ステータス』プロパティに追加する」
4. **安全な認証とログ出力の要求**：「APIトークンはPropertiesServiceから取得し、エラーはConsole.logで出力して」

### コピペで使えるプロンプト3パターン

**パターン①：スプレッドシート → Notion追加**

```
Google Apps Script（V8ランタイム）で、以下の処理を行うコードを書いてください。

・Notion API バージョン2022-06-28を使用
・スプレッドシートのアクティブシート、A列（タイトル）とB列（ステータス）のデータを読み取る
・2行目から最終行まで1行ずつNotionデータベースにページとして追加
・NotionのデータベースIDとAPIトークンはPropertiesServiceから取得（キー名：NOTION_TOKEN、NOTION_DB_ID）
・エラー発生時はConsole.logで詳細を出力
・ステータス列はNotionのSelectプロパティとして送信
```

**パターン②：Notion → スプレッドシート取得**

```
Google Apps Script（V8ランタイム）で、以下の処理を行うコードを書いてください。

・Notion API バージョン2022-06-28を使用
・NotionデータベースからページIDと「名前」「担当者」「期日」プロパティを取得
・取得したデータをスプレッドシートの「Notionデータ」シートに1行目からヘッダー付きで書き出す
・APIトークンとDBIDはPropertiesServiceから取得（キー名：NOTION_TOKEN、NOTION_DB_ID）
・ページネーションに対応し100件以上でも全件取得できるようにする
```

**パターン③：双方向同期（応用）**

```
Google Apps Script（V8ランタイム）で、以下の処理を行うコードを書いてください。

・Notion API バージョン2022-06-28を使用
・スプレッドシートのC列にNotionのページIDを保持する
・C列が空の行：Notionに新規ページを作成し、返ってきたページIDをC列に書き込む
・C列にIDがある行：そのページIDでNotionページを更新（PATCHリクエスト）
・更新対象プロパティは「タイトル（title型）」と「ステータス（select型）」
・APIトークンとDBIDはPropertiesServiceから取得
```

これ、意外と知られてないんですが——Cursorにモデルを選べる場合は**Claude 3.5か3.7系**を選ぶとGASの自然言語理解精度が上がる。2026年時点でCursorのPro契約なら選択できる。

---

## Apps Script IDEへの貼り付けと動かすまでの手順

コードが出たら、ここからが本番だ。

### ステップ1：Apps Script IDEを開く

連携したいスプレッドシートを開き、メニューから「拡張機能」→「Apps Script」を選ぶ。script.google.comが新しいタブで開く。

既存の`function myFunction()`の中身を全部消して、Cursorで生成したコードをそのまま貼り付ける。

### ステップ2：OAuthスコープを設定する

エディタ左のファイル一覧に`appsscript.json`が表示されない場合、歯車アイコン（プロジェクトの設定）から「「appsscript.json」マニフェストファイルをエディタで表示する」をオンにする。

`appsscript.json`を開き、`oauthScopes`を追記する。


{
  "timeZone": "Asia/Tokyo",
  "dependencies": {},
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",
  "oauthScopes": [
    "https://www.googleapis.com/auth/spreadsheets",
    "https://www.googleapis.com/auth/script.external_request"
  ]
}
```

NotionへのHTTPリクエストには`script.external_request`が必須だ。これがないと認証エラーで止まる。

### ステップ3：NotionトークンとDBIDを登録する

コード内で`PropertiesService.getScriptProperties().getProperty('NOTION_TOKEN')`としている場合、値を事前に登録しておく必要がある。

Apps Script IDEの左メニューから「プロジェクトの設定」→「スクリプト プロパティ」に進み、以下を追加する。

| プロパティ名 | 値 |
|---|---|
| NOTION_TOKEN | secret_xxxxx（Notionインテグレーショントークン） |
| NOTION_DB_ID | xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx |

Notionのインテグレーショントークンはnotion.so/my-integrationsで発行できる。発行後、**対象のNotionDBを開いて「接続」からそのインテグレーションを招待する手順を忘れずに。** これを飛ばすと403エラーが返ってくる。

### ステップ4：実行してログを確認する

関数名を選択して「実行」を押す。初回は権限の確認ダイアログが出るので「許可」を進める。

実行後は画面下の「実行ログ」でConsole.logの出力が確認できる。エラーが出たらそのエラーメッセージをそのままCursorに貼って「このエラーを修正して」と送る。これだけで大半は直る。

---

## コストと制限｜無料枠で実務に使えるか

ぶっちゃけた話をすると、Cursorの無料枠（月500補完）は**試す段階なら十分、実務で使い続けるには足りない**。

GASコードの生成・修正・デバッグを1セット繰り返すと20〜30補完は使う。複雑な双方向同期を仕上げるまでに無料枠の半分が消えることもある。

実務利用を前提にするなら、Pro（$20/月）に切り替えたほうがストレスが少ない。

GAS側の制限も押さえておく。

- 1回の実行上限：**6分**
- トリガー実行の上限：**90分/日**（無料アカウント）
- Notion APIのレートリミット：**3リクエスト/秒**

1000行を超えるデータを一括同期する場合、Notion APIの3リクエスト/秒制限に引っかかる。Cursorへの指示に「100件ごとにUtilities.sleep(500)を入れて」と追記しておくと、生成コードにスロットリング処理が入る。

---

## まとめ：次にやる1つのアクション

CursorでGASを作る流れを整理するとこうなる。

- 事前にNotionのDBカラムをUI側で作っておく
- インテグレーションを発行してDBに招待する
- Cursorへの指示に「V8ランタイム」「Notion API 2022-06-28」「PropertiesService使用」を必ず含める
- 生成コードはApps Script IDEに貼り付けてOAuthスコープを追記する

今すぐやるなら、**パターン①のプロンプトをCursorにそのまま貼り付けることから始めてほしい。**

スプレッドシートのA列・B列にサンプルデータを3行入れておいて、Notionに届くか確認するだけでいい。その1回を動かせれば、あとは応用できる。

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

## 関連記事

- [Claude API×GASでスプレッドシート自動化｜2025年版コピペで動く設定手順](/2026/05/26/Claude-APIGASでスプレッドシート自動化2025年版コピペで動く設定手順/)
- [Notion GAS連携やり方｜スプレッドシートをNotionDBに自動同期する手順](/2026/05/28/Notion-GAS連携やり方スプレッドシートをNotionDBに自動同期する手順/)
- [ChatGPT API×スプレッドシート連携｜GAS初心者向け設定と自動化手順](/2026/05/22/ChatGPT-APIスプレッドシート連携GAS初心者向け設定と自動化手順/)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。