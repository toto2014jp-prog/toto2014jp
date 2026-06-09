---
layout: post
title: "n8n Webhook Notion 自動登録やり方｜外部フォーム送信をDBに保存する手順2026"
date: 2026-06-09 09:06:34 +0900
description: "n8n WebhookノードとNotionを連携し、外部フォームのデータをDBに自動登録する手順を解説。Test URL/本番URL切り替えやプロパティ名の落とし穴まで対策済み。"
tags:
  - n8n Webhook Notion 自動登録 やり方
  - n8n Webhookノード 外部フォーム データベース 保存
  - n8n Notion Webhook トリガー 設定方法 2026
  - n8n Tally Typeform Notion 連携 手順
---

ngrok http 5678
# 表示されたhttps://xxxx.ngrok.ioをベースURLとして使う
```

セルフホスト環境の構築自体が必要な場合は、別記事「n8nセルフホスト やり方｜Railway編」を先に読んでほしい。

### 必要なもの（5分で揃う）

- n8nアカウント（CloudまたはセルフホストでURL確認済みのもの）
- Notionアカウント＋インテグレーション作成済み
- 送信テスト用の外部フォーム（TallyかTypeformが一番手軽）
- NotionデータベースのID（URLから取得する。後述）

---

## H2②：Webhookノードの設定——URLの取得と「本番化」の手順

### Webhookノードを追加する

n8nで新規ワークフローを作成し、「+」からWebhookノードを追加する。設定画面で確認すべき項目は以下の4つ。

```
HTTP Method  : POST（フォームはほぼPOST）
Path         : 任意の文字列（例：form-to-notion）
Authentication: None（まず動かすためにAuthなしで始める）
Response Mode: Immediately ← 必ずここを変える
```

**Response Modeは`Immediately`に設定する**。デフォルトの`Last Node`のままにすると、n8nがフロー全体の処理完了を待ってからフォームにレスポンスを返す。Typeformのタイムアウトは30秒。Notionへの書き込みが重なると超えることがある。`Immediately`にすれば受信した瞬間にフォーム側へ「200 OK」を返せる。

### Test URLと本番URLの違い——ここが最大の罠

Webhookノードを開くと、2種類のURLが表示されている。

```
Test URL       : https://xxxx.n8n.cloud/webhook-test/form-to-notion
Production URL : https://xxxx.n8n.cloud/webhook/form-to-notion
```

**Test URLはワークフローエディタが開いているときだけ動く**。ブラウザを閉じたり、エディタから離れると受信しなくなる。

**Production URLはワークフローを「Active」にして初めて動き出す**。右上のActivateトグルをオンにしないと、本番フォームからのデータは一切受け取れない。

これを知らずに「テスト中は動いたのに本番で動かない」と半日溶かす人が後を絶たない。正直なところ、自分も最初にここで1時間近く無駄にした。

動作の違いをまとめると：

| 状態 | Test URL | Production URL |
|---|---|---|
| エディタ表示中・Inactive | ✅ 受信可 | ❌ 受信不可 |
| エディタ表示中・Active | ✅ 受信可 | ✅ 受信可 |
| エディタを閉じた・Active | ❌ 受信不可 | ✅ 受信可 |
| エディタを閉じた・Inactive | ❌ 受信不可 | ❌ 受信不可 |

**実フォームに設定するURLは必ずProduction URLを使い、ワークフローをActiveにしておく**。これが鉄則。

### フォームサービス別：Webhook設定箇所

各フォームサービスでの設定場所は以下の通り。

- **Tally**：フォーム編集画面 → 「Integrations」タブ → 「Webhooks」→ Production URLを貼り付け
- **Typeform**：フォーム編集画面 → 「Connect」→ 「Webhooks」→ 「Add a webhook」
- **Googleフォーム**：Googleフォームにはネイティブなwebhook機能がない。Google Apps Script（GAS）でフォーム送信をトリガーにしてn8nへPOSTする必要がある（後述のコード例を参照）

### テスト送信でペイロードを確認する

フォームサービスにWebhook URLを設定したら、**実際にフォームを1件送信してn8n側でデータ構造を確認する**。これをやっておかないと次のSetノード設定で詰まる。

n8nのWebhookノードを選択した状態でテスト送信すると、受け取ったデータが右パネルに表示される。Tallyの場合、受信データの構造は概ね以下のようになる。


{
  "eventId": "xxxx",
  "eventType": "FORM_RESPONSE",
  "data": {
    "responseId": "yyyy",
    "fields": [
      {
        "key": "question_abc123",
        "label": "お名前",
        "value": "田中太郎"
      },
      {
        "key": "question_def456",
        "label": "メールアドレス",
        "value": "[REDACTED]"
      }
    ]
  }
}
```

Typeformはフラットな構造（`form_response.answers`の配列）、Googleフォーム経由のGASは自分でペイロードを設計できる。どのフォームを使うかで次のSetノードの書き方が変わる。

---



> 💡 **関連教材**: [n8nノーコード自動化 実践ワークフロー集（¥1,980）](https://aijissenlab.gumroad.com/l/bltzrk) — Gmail/Slack/Notion/Claude連携の動くワークフロー10本を実装込みで完全解説（全35ページ）

## H2③：SetノードでNotionに渡すデータを整形する

### なぜSetノードが必要か——3大エラーの原因

Webhookで受け取った生データをそのままNotionノードに渡そうとすると、高確率で以下のどれかで落ちる。

**エラー1：型ミスマッチ**
フォームから来る数値は文字列（`"42"`）として届くことが多い。NotionのNumber型プロパティに文字列を渡すとエラーになる。

**エラー2：プロパティ名の大文字小文字不一致**
Notionのプロパティ名は完全一致が必要。DBのプロパティが「Name」なのにn8nで「name」と指定するとデータが入らない。スペースも含めて一字一句合わせる必要がある。

**エラー3：TitleプロパティをTextとして扱ってしまう**
Notionデータベースには必ず1つ「Title」タイプのプロパティが存在する（デフォルトは「名前」という名前）。これはNotionノードの設定画面で**他のプロパティとは別の専用フィールド**に入力しないといけない。同じProperties欄にTextとして追加しようとするとエラーになる。

### Setノード（Edit Fields）の設定手順

Webhookノードの後ろにEdit Fields（Set）ノードを追加する。設定画面で「Add Field」して必要なフィールドを手動で追加していく。

例として、TallyフォームのデータをNotionに渡す場合の設定：

```
Field名: name
型: String
値: {{ $json.data.fields[0].value }}

Field名: email  
型: String
値: {{ $json.data.fields[1].value }}

Field名: submitted_at
型: String
値: {{ $now.toISO() }}
```

日付はISO8601形式（`2026-01-15T10:30:00.000Z`）で渡すのが安全。n8nの`$now.toISO()`を使えば自動でこの形式になる。

Typeformの場合、answersの配列番号でアクセスする。

```
値: {{ $json.form_response.answers[0].text }}
```

### GoogleフォームをGAS経由でWebhookに繋ぐコード例

GoogleフォームにはWebhook機能がないので、GASでn8nへPOSTする。スクリプトエディタに以下を貼り付けて、フォームのonSubmitトリガーに設定する。

```javascript
function onFormSubmit(e) {
  const webhookUrl = 'https://xxxx.n8n.cloud/webhook/form-to-notion'; // Production URLに変更
  
  const responses = e.response.getItemResponses();
  const payload = {};
  
  responses.forEach(function(response) {
    const questionTitle = response.getItem().getTitle();
    const answer = response.getResponse();
    payload[questionTitle] = answer;
  });
  
  payload['submitted_at'] = new Date().toISOString();
  
  const options = {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };
  
  UrlFetchApp.fetch(webhookUrl, options);
}
```

このスクリプトを使うと、Googleフォームの質問タイトルがそのままJSONのキー名になる。Notionのプロパティ名と完全一致させるか、Setノードでマッピングする。

---

## H2④：NotionノードでデータベースにページをCreate Pageする

### NotionのIntegration設定（済でない場合）

先にNotionインテグレーションを作成してAPIキーを取得し、対象データベースをそのインテグレーションと「接続」しておく。これをやり忘れると「object not found」エラーになる。

手順：
1. [notion.so/my-integrations](https://www.notion.so/my-integrations) でNew integrationを作成
2. 発行された「Internal Integration Secret」をコピー
3. Notionデータベースを開き、右上「...」→「接続先」→ 作成したインテグレーションを選択

### Database IDの正しい取得方法

これも間違える人が多い箇所。2025年以降のNotionでは、データベースのURLは以下のような形式になっている。

```
https://www.notion.so/ワークスペース名/[Database-ID]?v=[View-ID]
```

`?v=`の手前にある32桁の文字列がDatabase ID。ハイフンで区切られていることもあるが、n8nに入力する際はハイフンあり・なしどちらでも動く。

実際のURL例：
```
https://www.notion.so/myworkspace/1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d?v=xxx
                                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                  これがDatabase ID（32桁）
```

### Notionノードの設定

Notionノードを追加して以下を設定する。

```
Resource : Page
Operation: Create
Database : [取得したDatabase IDを貼り付け]
```

**Titleプロパティの設定**（ここが重要）:
Notionノードの設定画面には「Title」という専用フィールドがある。DBのTitleプロパティ（デフォルトは「名前」）に入れる値はここに設定する。

```
Title: {{ $json.name }}
```

**その他のプロパティの設定**:
「Add Property」でプロパティを追加していく。**Notionのプロパティ名は一字一句DBと一致させる**。

```
プロパティ名 : Email（NotionDBの列名と完全一致）
型          : Email
値          : {{ $json.email }}

プロパティ名 : 送信日時
型          : Date
値          : {{ $json.submitted_at }}
```

### 動作確認の手順

1. ワークフローをSaveする
2. ワークフローをActivateする（右上のトグル）
3. **本番フォームから1件送信する**（Test URLではなくProduction URLを使っていることを再確認）
4. n8nの実行履歴（Executions）でSuccessになっているか確認
5. NotionDBに新しい行が追加されているか確認

SuccessになっているのにNotionに入っていない場合、99%の確率でプロパティ名の不一致。n8nのNotionノードの実行ログに`could not find property`のような文言が出ているはず。

---

## H2⑤：よくある失敗と正直なデメリット

### よくある失敗3選

**失敗1：本番ワークフローをActivateし忘れる**
前述の通り、最多の失敗。フォームに設定するURLはProduction URLで、ワークフローはActiveにしておく。エディタを閉じた状態でも動くことをテストしてから本番に使うこと。

**失敗2：Notionインテグレーションをデータベースに「接続」し忘れる**
 APIキーだけ取得して満足してしまい、データベース側の接続設定を忘れるケースがある。Notion側で「接続先」にインテグレーションを追加しないと、正しいAPIキーでも「object not found」になる。

**失敗3：大量フォーム受信時のRate Limit**
NotionのAPIレート制限は1インテグレーションあたり3リクエスト/秒。同時にフォームが大量送信された場合、n8n側でリトライ処理を組んでいないと書き込みが抜けることがある。小規模（1日50件以下）なら問題ないが、キャンペーン等で大量送信が予想される場合はSplit In Batchesノードと組み合わせて流量制御を検討する。

### 正直なデメリット

**n8n Cloudの無料枠は月5,000実行まで**。1回のフォーム送信でWebhook→Set→Notionの3ノードを通るとカウントは3消費。月1,666件のフォーム送信が上限の目安になる。それ以上になる場合は有料プランかセルフホストへの移行が必要。

**Notionノードv2になっても、複雑なリレーション設定は手間がかかる**。リレーション先のページIDを別途取得してSetノードで渡す必要があり、「フォーム入力の値でリレーション先を動的に検索して紐付ける」ようなフローは中級以上のn8n知識が必要になる。

**セルフホスト環境はWebhookの安定稼働に気を遣う**。ngrokの無料プランはセッションが8時間で切れる。本番運用するならCloudflare Tunnelかn8n Cloudを使った方がいい。

---

## H2⑥：よくある質問

### Q1：「n8nでWebhookを受け取るとき認証は必要ですか？」

必須ではないが、本番運用では設定した方がいい。Webhookノードの`Authentication`で`Header Auth`を選び、フォームサービス側でカスタムヘッダーを送信するよう設定する。これがないと、URLが漏れた場合に誰でもNotionにデータを書き込める状態になる。Tally・TypeformともにカスタムヘッダーをWebhook送信時に付与できる機能がある。

### Q2：「テスト環境（Test URL）と本番環境（Production URL）で同じデータをNotionに書いてしまわないようにするには？」

テスト用と本番用で別のNotionデータベースを用意するのがシンプルな解決策。もしくは、Notionの「環境」列（Select型）を追加して、Setノードで`test`/`production`の値を条件分岐で設定する方法もある。IFノードでTest URLからのリクエストかどうかを判定して書き込み先を分けることも可能。

### Q3：「フォームに入力必須でないフィールドがあり、空の場合にNotionのノードがエラーになります」

Setノードで空値チェックを入れる。n8nの式エディタでは三項演算子が使える。

```
{{ $json.email ? $json.email : '' }}
```

もしくは空文字の場合はプロパティ自体をNotionに送らない設計にする。NotionはプロパティをPOSTのbodyに含めなければ空欄のまま保存してくれる（エラーにならない）。

### Q4：「Tallyのフォームで複数選択（チェックボックス）の値をNotionのMulti-select型に入れるにはどうすればいいですか？」

Tallyの複数選択フィールドは配列として届く。Notionのmulti_selectに渡す際は、n8nの式で配列を変換する必要がある。

```javascript
// Setノードの値として
{{ $json.tags.map(t => ({name: t})) }}
```

NotionのMulti-select型は`[{name: "タグ1"}, {name: "タグ2"}]`という形式を要求する。これに合わせて変換すれば複数選択も問題なく入る。

---

## まとめ：次にやるべき1つのアクション

この記事で解説した手順を整理すると：

1. Webhookノードを追加 → Production URLを取得 → Response ModeをImmediatelyに設定
2. フォームサービスにProduction URLを登録 → テスト送信でペイロード確認
3. Edit Fields（Set）ノードで型とプロパティ名を整形
4. Notionノードでインテグレーション設定・DatabaseID入力・Titleプロパティを専用フィールドに設定
5. ワークフローをActivate → 本番フォームから1件送信して確認

今すぐやるべき1つのことを挙げるなら、**TallyかTypeformで無料フォームを1つ作り、n8nのTest URLに繋いでテスト送信してみること**。ペイロードの中身を一度目で見ると、SetノードとNotionノードの設定が格段に理解しやすくなる。

このフローが動くようになったら、次は「Webhookで受け取ったフォームデータをAIノードで自動分類してNotionにタグ付き登録する」構成に発展させるのが面白い。n8nのAI Agentノードとの組み合わせについては、別記事で解説する予定なのでそちらも参考にしてほしい。

---

## 関連記事

- [Make×Notionデータベース連携やり方｜フォーム回答を自動追加する手順](/2026/05/29/MakeNotionデータベース連携やり方フォーム回答を自動追加する手順/)
- [n8n×Notionデータベース連携やり方｜タスク自動登録フローをノーコードで作る手順](/2026/05/29/n8nNotionデータベース連携やり方タスク自動登録フローをノーコードで作る手順/)
- [Claude API×Notion連携やり方｜AIライティング結果をDBに自動保存する手順](/2026/05/31/Claude-APINotion連携やり方AIライティング結果をDBに自動保存する手順/)

---

## 📘 もっと深く学びたい方へ

この記事で紹介した内容を、さらに体系的に・実務レベルで習得できる教材を販売中です。

### [n8nノーコード自動化 実践ワークフロー集（¥1,980）](https://aijissenlab.gumroad.com/l/bltzrk)

Gmail/Slack/Notion/Claude連携の動くワークフロー10本を実装込みで完全解説（全35ページ）

- Cloud版前提・2026年最新のn8n v1系UI完全対応
- 動くノード設定値・JSON・OAuth設定まで全部入り
- 10ワークフロー（Gmail/Slack/Notion/Discord/Webhook/RSS）はコピペで即実務投入

👉 [今すぐ購入する](https://aijissenlab.gumroad.com/l/bltzrk)

### [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs)

API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

- 動くGASコード・API設定手順・プロンプトをワンセット収録
- スプレッドシート連携／メール／議事録／請求書を実務レベルで自動化
- コピペで即動く実装コード（Python / GAS）付き

👉 [今すぐ購入する](https://aijissenlab.gumroad.com/l/irvscs)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。