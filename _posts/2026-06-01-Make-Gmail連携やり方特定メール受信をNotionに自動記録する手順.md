---
layout: post
title: "Make Gmail連携やり方｜特定メール受信をNotionに自動記録する手順"
date: 2026-06-01 09:10:15 +0900
description: "MakeとGmailを連携し、特定条件のメールをNotionに自動保存する手順を解説。Watch Emailsトリガーの設定・Pub/Sub構成・30日期限切れ対策まで実務レベルで紹介。"
tags:
  - Make Gmail 連携 自動化 やり方
  - Make Gmail Notion 自動保存 設定方法
  - Make メール受信 トリガー Notion 登録
  - Gmail Watch Emails Pub/Sub 設定手順
---

「Makeでメール自動化を設定したのに、気づいたら止まってた」——そういう経験、一度はありませんか。

Gmailトリガーの落とし穴は「作るのは簡単、維持するのが難しい」という点にあります。本記事では、**特定メールをNotionに自動記録するシナリオの構築手順**と、実運用で詰まりやすいポイントをまとめて解説します。

これを読めば、「動いたけど2週間後に止まった」という状況を未然に防ぎながら安定稼働させる設計が身につきます。

---

## Make×Gmail連携で何ができる？完成形と全体フローを先に確認する

### 今回作るシナリオの完成形

まず作った後のイメージを共有します。完成するフローはこちらです。

```
Gmail Watch Emails（Pub/Sub）
  └─ Router（条件分岐）
       ├─ [条件A] 件名に「請求書」含む → Notionに新規ページ作成
       ├─ [条件B] 送信者が特定ドメイン → 別のNotionDBに記録
       └─ [条件C] それ以外 → 何もしない
```

Notionには1メール＝1ページとして以下の情報が自動で入ります。

- **件名**（Title プロパティ）
- **送信者メールアドレス**（Text）
- **受信日時**（Date・ISO 8601形式）
- **本文テキスト**（Text・HTML除去済み）
- **Message-ID**（重複防止用・Text）

### 「New Email」と「Watch Emails」、どちらを使うか

これ、意外と混同されているんですが、**2つのGmailトリガーは動作がまったく違います。**

| トリガー | 方式 | 最短間隔 | 向いているケース |
|---|---|---|---|
| New Email | ポーリング | **15分** | 即時性が不要な用途 |
| Watch Emails | Pub/Sub（プッシュ） | **ほぼリアルタイム** | 請求書・アラート等 |

本記事では**Watch Emailsを推奨**します。理由はシンプルで、15分遅延だと「メール受信→即座にNotionで確認」というワークフローが成立しないからです。

### 無料プランで月何通まで処理できるか

正直なところ、2025年10月のプラン改定でMake無料枠はかなり厳しくなりました。

| プラン | 月間オペレーション | メール処理可能数（目安） |
|---|---|---|
| 無料 | **500回** | 約100〜166通 |
| Core（$9〜） | 10,400回 | 約2,080〜3,466通 |

メール1通の処理で3〜5オペレーション消費します。月200通以上のメールを処理したいなら、有料プランを最初から想定した方が現実的です。

---



> 💡 **関連教材**: [n8nノーコード自動化 実践ワークフロー集（¥1,980）](https://aijissenlab.gumroad.com/l/bltzrk) — Gmail/Slack/Notion/Claude連携の動くワークフロー10本を実装込みで完全解説（全35ページ）

## 事前準備｜Google Cloud・Make・Notionの3点セット設定

### ステップ1：Notionデータベースの設計

ここを雑にすると後でエラーが頻発します。先に型を正確に設計してください。

| プロパティ名 | 型 | 備考 |
|---|---|---|
| 件名 | Title | ページタイトルに使用 |
| 送信者 | Text | メールアドレスをそのまま格納 |
| 受信日時 | **Date** | ISO 8601形式必須（例：2026-06-01T09:00:00+09:00） |
| 本文 | Text | HTML除去後のプレーンテキスト |
| Message-ID | Text | 重複防止用。最初から設けておく |
| ステータス | Select | 「未対応／対応中／完了」など |

**DateプロパティはISO 8601形式でないとエラーになります。** MakeのDateフォーマット変換モジュールで`YYYY-MM-DDTHH:mm:ssZ`に変換してから渡してください。これを忘れると「Invalid date」エラーが延々と出ます。

Message-IDプロパティは地味に重要です。Gmailのメッセージには固有のMessage-IDヘッダーがあり、これをNotionに保存しておくことで**同じメールが二重登録されるのを防げます。**

### ステップ2：Google Cloud ConsoleでPub/Sub設定

Watch Emailsトリガーを使うにはGoogle Cloud側の設定が必要です。初回だけ30分ほどかかりますが、これをやるかやらないかで安定性が大きく変わります。

**① Google Cloud Consoleにアクセス**
`console.cloud.google.com`にGmailと同じGoogleアカウントでログインします。

**② プロジェクトを作成または選択**
既存プロジェクトがあればそれを使用。新規の場合は「新しいプロジェクト」から作成してください。

**③ Gmail APIを有効化**
「APIとサービス」→「ライブラリ」→「Gmail API」を検索して有効化。

**④ Pub/Subトピックを作成**
「Pub/Sub」→「トピック」→「トピックを作成」をクリック。トピックIDは`gmail-make-notify`など分かりやすい名前に。

**⑤ サブスクリプションを作成**
作成したトピックを選択→「サブスクリプションを作成」。配信タイプは「プッシュ」を選び、エンドポイントURLにMakeが発行するWebhook URLを入力します（次のステップ3で取得）。

**⑥ GmailにPub/Subの発行権限を付与**
トピックの「権限を管理」から`[REDACTED]`を追加し、ロールを「Pub/Sub パブリッシャー」に設定。これを忘れると通知が届きません。

### ステップ3：MakeでGmailを接続する（OAuth2.0認証）

2025年Q2にGoogleのセキュリティポリシーが更新され、**旧接続のままだとシナリオが突然停止するケースが増えています。** 以前に設定した接続がある場合は、一度削除して再認証することを強く勧めます。

**Makeでの接続手順：**

1. Makeにログインし、「Connections」→「Add connection」
2. 「Gmail」を検索して選択
3. 「Sign in with Google」でGoogleアカウントを認証
4. 権限確認画面で「Gmail の読み取り」にチェックが入っていることを確認
5. 接続名を分かりやすいものに変更（例：`Gmail_本番用`）して保存

接続後、テスト送信を1通自分宛てに送ってMakeのシナリオ実行ログに届いているか確認してください。ここで動作確認が取れてから次のステップに進むのが鉄則です。

---

## シナリオ構築｜Watch Emails → Router → Notion登録の設定手順

### Watch Emailsトリガーの設定

Makeで新規シナリオを作成し、最初のモジュールに「Gmail > Watch Emails」を追加します。

設定画面で入力する項目はこちらです。

- **Connection**：先ほど作成したGmail接続を選択
- **Folder**：`INBOX`（受信トレイ）を指定
- **Criteria**：`All email`または`Unread email`
- **Sender email address**：特定ドメインのみ受け取る場合に入力
- **Subject**：件名フィルターが必要な場合に入力

Pub/Sub連携の場合、このモジュールの設定画面下部に「Webhook URL」が表示されます。このURLをステップ2⑤で設定したPub/Subサブスクリプションのエンドポイントに貼り付けてください。

### RouterとFilterで条件分岐を作る

Watch Emailsの後に「Router」モジュールを追加します。Routerは分岐点で、それぞれのルートに「Filter」を設定して条件を絞ります。

フィルター設定の例：

**ルートA（請求書メール）**
- Field：`Subject`
- Operator：`Contains`
- Value：`請求書`

**ルートB（特定ドメイン）**
- Field：`From`
- Operator：`Contains`
- Value：`@example.com`

**ルートC（それ以外は無視）**
- フィルターなし＆接続先モジュールなしでOK

Gmail側のラベルフィルターだけで管理しようとするのは危険です。**Make側のRouterで明示的に分岐させないと、意図しないメールがNotionに大量登録されます。**

### Text Parserで本文をクリーンにする

Gmailの本文はHTML形式で届くことが多く、そのままNotionに渡すとタグが混入します。

Routerの後、Notionモジュールの前に**「Text Parser > HTML to Text」**モジュールを挟みます。

- Input：`{{1.body.html}}`（Watch Emailsのhtml本文）
- Output：プレーンテキストに変換された本文

これを挟むだけで可読性が大きく変わります。

### Notionへの書き込み設定

「Notion > Create a Database Item」モジュールを追加します。

各フィールドのマッピング例：

- **件名（Title）**：`{{1.subject}}`
- **送信者（Text）**：`{{1.from}}`
- **受信日時（Date）**：`{{formatDate(1.date; "YYYY-MM-DDTHH:mm:ssZ")}}`
- **本文（Text）**：`{{3.text}}`（Text Parserの出力）
- **Message-ID（Text）**：`{{1.headers["message-id"]}}`

Dateのフォーマット変換を忘れずに。`formatDate()`関数を使ってISO 8601形式に変換してから渡してください。

---

## 「動いたけど止まった」を防ぐ｜Watch Email期限切れ対策と運用Tips

### 30日期限切れ問題の解決策

Watch Emailsのトリガーは**最大30日で期限切れ**になります。これを放置すると「設定したはずなのに動いていない」という状態が発生します。

対策は**「更新用シナリオを月1回スケジュール実行する」**こと。

別シナリオを作成し、以下のように設定します。

```
[スケジュール：毎月1日 午前0時]
  └─ Gmail > Renew a Watch Email Trigger
```

「Gmail > Renew a Watch Email Trigger」モジュールに元のWatch Emailsシナリオのシナリオ名を指定するだけです。これで自動的にWatch期限がリフレッシュされます。2分で作れる割に効果は絶大です。

### 重複登録を防ぐフロー

Message-IDを使った重複チェックを入れると、より堅牢になります。

```
Gmail Watch
  └─ Notion > Search Objects（Message-IDで検索）
       ├─ 結果あり → スキップ（何もしない）
       └─ 結果なし → Notion > Create Database Item
```

このフローを入れるとオペレーション消費が増えますが（1通あたり+1〜2回）、再送メールや誤トリガーによる重複登録を完全に防げます。

### 無料プランで運用するなら知っておくべきこと

月500オペレーションで運用するなら、**処理対象メールを徹底的に絞る**ことが前提です。

Gmail側でラベルを事前に設定し、Watch EmailsのFolder指定でそのラベルフォルダのみを監視する。これだけで無駄なオペレーション消費を大幅に削減できます。月100通以上の処理が見込まれるなら、最初からCoreプラン（月$9）を選んだ方が精神的に楽です。

---

## まとめ｜今すぐやること1つだけ

本記事で作ったシナリオのポイントをおさらいします。

- トリガーはNew EmailではなくWatch Emails（Pub/Sub）を使う
- Notionのプロパティ型、特にDateのISO 8601形式は必須
- RouterとFilterで「記録する/しない」を明示的に分岐させる
- Watch Emails更新シナリオを月1回実行する設定を忘れずに

**今日やるべきアクションは1つです。** Google Cloud ConsoleでPub/Subトピックを作成してください。ここさえ終われば、残りのMake側の設定は30分以内に完了します。Pub/Sub設定を後回しにしたまま「なんとなく動かない」で詰まるケースが一番多いので、まずここから手をつけてください。

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

## 関連記事

- [Perplexity AI×Notion連携やり方｜競合調査結果をDBに自動保存する手順](/2026/05/28/Perplexity-AINotion連携やり方競合調査結果をDBに自動保存する手順/)
- [Claude API×Notion連携やり方｜AIライティング結果をDBに自動保存する手順](/2026/05/31/Claude-APINotion連携やり方AIライティング結果をDBに自動保存する手順/)
- [Zapier×ChatGPT連携のやり方｜ノーコードで業務通知・メール自動化を設定する手順](/2026/05/23/ZapierChatGPT連携のやり方ノーコードで業務通知メール自動化を設定する手順/)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。
