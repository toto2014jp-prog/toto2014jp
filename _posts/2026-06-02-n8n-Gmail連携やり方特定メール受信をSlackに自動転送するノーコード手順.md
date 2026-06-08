---
layout: post
title: "n8n Gmail連携やり方｜特定メール受信をSlackに自動転送するノーコード手順"
date: 2026-06-02 09:05:42 +0900
description: "n8nでGmailの特定メールをSlackへ自動転送する手順を解説。OAuth2認証からGmailトリガー・IFノード・Slackノードの組み方まで、v1系UIの最新構成でステップ別に紹介。"
tags:
  - n8n Gmail 連携 やり方
  - n8n メール受信 Slack 自動転送 設定
  - n8n Gmail トリガー ノーコード 自動化
  - n8n OAuth2 Google Cloud Console 設定手順
---

## 「n8nでGmail連携したのに動かない」——その原因、だいたい同じです

「n8nでGmailを受信してSlackに転送したい」と思って設定し始めたものの、認証で詰まる。画面が記事と違う。通知が届いたりこなかったりする。

こういった声を本当によく聞く。原因のほとんどは**2024年以降のGoogle OAuth仕様変更**と**n8n v1系へのUI刷新**だ。古い記事の手順をそのまま試しても動かないのは当然で、読者の設定ミスではない。

この記事では、**v1系UIを前提に、GmailトリガーからSlack通知までを一本通す手順**を解説する。認証の取得から、ノイズを減らすフィルタリング設計まで完結させる。

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

## n8n×Gmail連携を始める前に確認すること

### 誤解①「n8nは無料で使える」

半分正解、半分違う。

セルフホスト版（自分のサーバーで動かす）は無料だが、VPSやRailwayなどのサーバー費用が別途かかる。手軽に始めるなら**n8n Cloud版（Starter: $20/月〜）**が現実的な選択肢だ。

Cloud版は月2,500回の実行が上限（超過は従量課金）。メール→Slack通知くらいなら余裕で収まる。セルフホストの構築手順は別記事に譲る。

### 誤解②「GoogleアカウントでそのままGmailに繋がる」

これが一番の詰まりポイント。

2024年以降、GoogleのOAuthポリシー強化により、**Google Cloud ConsoleでOAuthアプリを作成してClient ID/Secretを取得する手順が必須**になった。旧来の「Googleアカウントでログイン」的な簡易接続はほぼ使えない。

次のH2でこの手順を丁寧に解説する。

### 誤解③「Gmail Triggerはリアルタイムで動く」

正直なところ、これも誤解が多い。

n8nのGmailトリガーは**ポーリング方式**（一定間隔でGmailを確認する仕組み）で、Starterプランだと最短1分間隔になる。リアルタイムに近い通知が必要なら、Google Apps ScriptのWebhookと組み合わせる構成が必要だ。

「数分以内に通知が来ればいい」という用途なら、ポーリングで十分機能する。

> **本記事はn8n v1系（2025年以降のUI）を前提としています。**
> 「Function Node」は「Code Node」に、旧来のノード名も多数変更されているので注意。

---

## 事前準備：OAuth2認証情報とSlack Bot Tokenを取得する

フロー構築の前に、認証情報を2つ用意する。**Gmail用のGoogle OAuth2**と**Slack用のBot Token**だ。

### STEP 1｜Google Cloud ConsoleでOAuth認証情報を作る

**1. プロジェクトを作成する**
[Google Cloud Console](https://console.cloud.google.com/) にアクセスし、上部のプロジェクト選択から「新しいプロジェクト」を作成。名前は「n8n-gmail」など何でもいい。

**2. Gmail APIを有効にする**
左メニュー「APIとサービス」→「ライブラリ」→「Gmail API」を検索して有効化。

**3. OAuth同意画面を設定する**
「APIとサービス」→「OAuth同意画面」を開く。
- User Type：「外部」を選択
- アプリ名・サポートメールを入力
- スコープの追加：`https://www.googleapis.com/auth/gmail.readonly` を追加
- テストユーザーに自分のGmailアドレスを追加

**4. 認証情報（Client ID/Secret）を発行する**
「APIとサービス」→「認証情報」→「認証情報を作成」→「OAuthクライアントID」。
- アプリケーションの種類：「ウェブアプリケーション」
- 承認済みのリダイレクトURI：n8n CloudならURL末尾に `/rest/oauth2-credential/callback` を付けたURLを入力

発行された**クライアントID**と**クライアントシークレット**をメモしておく。

**5. n8n側でCredentialを登録する**
n8nの「Credentials」→「Add Credential」→「Google OAuth2 API」を選択。先ほどのClient IDとSecretを貼り付けて保存。Googleの認証画面が開くので許可して完了。

### STEP 2｜Slack AppのBot Tokenを取得する

**1. Slack Appを作成する**
[api.slack.com/apps](https://api.slack.com/apps) → 「Create New App」→「From scratch」。App名とワークスペースを設定。

**2. Bot Tokenのスコープを設定する**
左メニュー「OAuth & Permissions」→「Bot Token Scopes」に以下を追加。

- `chat:write`（メッセージ送信）
- `channels:read`（チャンネル一覧取得）

**3. ワークスペースにインストール**
同ページ上部の「Install to Workspace」をクリック。完了後に表示される `xoxb-` から始まるトークンをコピー。

**4. チャンネルIDを確認する**
Slackアプリで通知先チャンネルを右クリック→「チャンネル詳細を表示」→一番下にチャンネルIDが表示される（例：`C0XXXXXXXXX`）。

チャンネル名（`#general`など）ではなくIDを使う理由は、**チャンネル名を変更した際にフローが壊れるバグが報告されているため**。最初からIDで設定しておく。

**5. n8n側でCredentialを登録する**
n8nの「Credentials」→「Add Credential」→「Slack API」を選択。先ほどの `xoxb-` トークンを貼り付けて保存。

---

## フロー構築手順：GmailトリガーからSlack通知までの4ステップ

認証情報が揃ったら、いよいよフローを組む。構成は以下の4ノードだ。

```
[Gmail Trigger] → [IF ノード] → [Slack ノード]
                        ↓
                   [条件不一致は何もしない]
```

### STEP 1｜Gmail Triggerノードを設置する

n8nの新規ワークフローを開き、「Add first step」から「Gmail Trigger」を検索して追加。

設定項目：
- **Credential**：先ほど登録したGoogle OAuth2を選択
- **Trigger On**：「New Email」を選択
- **Search Query**（重要）：受信対象を絞り込む検索クエリを入力

Search Queryはそのままメールボックスに流し込むとノイズだらけになる。以下のようにGmailの検索構文がそのまま使える。

```
from:[REDACTED] subject:【緊急】 is:unread
```

ラベルで管理している場合は `label:自動転送対象` のように指定するとさらにスッキリする。

- **Poll Times**：「Every Minute」で1分間隔のポーリング設定

設定後、「Test step」を押して実際のメールデータが取得できるか確認する。`id`, `subject`, `from`, `snippet` などのフィールドが返ってくれば成功。

### STEP 2｜IFノードで多段フィルタリングを設定する

Gmailの検索クエリだけでは拾いきれない条件を、IFノードで追加フィルタリングする。

Gmail Triggerノードの右側「+」から「IF」ノードを追加。

**条件設定例：**

| フィールド | 条件 | 値 |
|---|---|---|
| `{{ $json.subject }}` | Contains | `【緊急】` |
| `{{ $json.from.value[0].address }}` | Equals | `[REDACTED]` |

2つの条件を「AND」で繋ぐと、**件名に「【緊急】」を含む、かつ特定の送信者からのメール**のみを通過させられる。

IFノードの「True」出力側にSlackノードを繋ぐ。「False」側は何もしないか、ログ用のノードを繋いでもいい。

### STEP 3｜Slackノードでメッセージを送信する

IFノードの「True」出力から「Slack」ノードを追加。

設定項目：
- **Credential**：先ほど登録したSlack APIを選択
- **Resource**：Message
- **Operation**：Send
- **Channel**：先ほど確認したチャンネルID（例：`C0XXXXXXXXX`）を入力
- **Text**：送信するメッセージ内容を入力

メッセージには動的な値を埋め込める。以下のように書くと、件名・送信者・本文冒頭が通知される。

```
📧 *新着メール通知*

*件名：* {{ $('Gmail Trigger').item.json.subject }}
*送信者：* {{ $('Gmail Trigger').item.json.from.value[0].address }}
*本文：* {{ $('Gmail Trigger').item.json.snippet }}
```

`snippet` はメール本文の先頭200字程度のプレビューだ。本文全体が必要な場合は `text` フィールドを使う（HTMLが混じる場合もあるので注意）。

### STEP 4｜フローをアクティブ化してテストする

すべてのノードを繋いだら、右上のトグルで「Active」に切り替える。

動作確認は実際に対象のGmailアカウントにテストメールを送って行う。1〜2分待ってSlackに通知が来れば完成だ。

うまく動かない場合のチェックポイント：
- Gmail Triggerの「Test step」でデータが取得できているか
- IFノードの条件式でフィールド名のスペルミスがないか（大文字・小文字も厳密）
- Slackノードにapp-invited済みのチャンネルIDを使っているか

> **補足：** SlackノードはAppを通知先チャンネルに招待していないとエラーになる。チャンネルで `/invite @アプリ名` を実行しておこう。

---

## 運用で役立つ改善ポイント

フローが動くようになったら、実運用を見据えて手を入れておきたい箇所がいくつかある。

**通知メッセージにメール本文リンクを入れる**
`{{ $('Gmail Trigger').item.json.threadId }}` を使うと、Gmailの該当スレッドへの直リンクをSlackメッセージに添付できる。

**エラーハンドリングを設定する**
n8nには「Error Trigger」という専用ノードがある。認証切れや一時的なAPI障害のときに別のSlackチャンネルに通知させると、フロー停止に気づかず放置するリスクを防げる。

**未読フラグを処理済みにする（任意）**
Gmailノードの「Mark as Read」アクションをフローの末尾に追加すると、Slackに転送済みのメールを自動で既読にできる。二重確認のムダが減る。

---

## 今日やること：まずテストメールを1通送ってみる

フローを組んだら、実際に動くかどうかを確かめる以外に確認する方法はない。

設定が終わったら、対象のGmailアドレスに件名「【緊急】テスト」のメールを送ってみる。1〜2分以内にSlackに通知が届けば、このフローは完成だ。

通知が来なければ、Gmail Triggerの「Test step」からデータ取得できているか確認する。そこでつまずいているなら、OAuth認証の設定に戻るのが最短ルートだ。

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

- [Make Gmail連携やり方｜特定メール受信をNotionに自動記録する手順](/2026/06/01/Make-Gmail連携やり方特定メール受信をNotionに自動記録する手順/)
- [Zapier×ChatGPT連携のやり方｜ノーコードで業務通知・メール自動化を設定する手順](/2026/05/23/ZapierChatGPT連携のやり方ノーコードで業務通知メール自動化を設定する手順/)
- [ChatGPT×Googleカレンダー連携｜GASで予定自動登録・リマインド通知を設定する方法](/2026/05/23/ChatGPTGoogleカレンダー連携GASで予定自動登録リマインド通知を設定する方法/)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。