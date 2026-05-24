---
layout: post
title: "Make×ChatGPT連携のやり方｜Zapierより安い自動化設定手順"
date: 2026-05-24 09:05:34 +0900
description: "MakeとChatGPTをOpenAI APIで連携する設定手順を解説。Zapierとのコスト比較・公式モジュールの使い方・実務シナリオ3選まで、月$10以上節約できる自動化の始め方。"
tags:
  - Make ChatGPT 連携 やり方
  - Make Integromat OpenAI 自動化 設定
  - Zapier Make コスト比較 どっちがいい
  - Make OpenAI APIキー 登録手順
---

「Zapierで ChatGPT連携を動かしてるけど、月額がじわじわ痛い」——そう感じている人、けっこう多いはず。

実際、月500回程度のChatGPT自動化でZapierを使うと$19.99/月かかる。同じことをMakeでやると$9/月で収まる。年間で$120以上の差だ。

この記事では、MakeとOpenAI APIを繋ぐ具体的な設定手順と、Zapierからの乗り換えを判断するためのコスト比較を解説する。「Makeって複雑そう」という印象をお持ちなら、読み終わるころには変わっているはず。

---

## MakeとZapier、ChatGPT自動化のコスト差はいくら？

まず正直に言う。**Zapierが悪いツールなわけじゃない。**シンプルな自動化ならZapierのほうが直感的に動く。ただ、ChatGPTと絡めて使う回数が増えると、コスト構造の違いが効いてくる。

### 課金単位がそもそも違う

Zapierは「Zap1回実行＝1タスク消費」とシンプルだ。一方、Makeは「モジュール1個実行＝1 Operations（ops）消費」という単位になる。

つまり、3つのモジュールで構成されたシナリオを1回動かすと3 ops消費する。一見不利に見えるが、無料プランの上限が**Makeは月10,000 ops**に対し、**Zapierは月100タスク**。桁がふたつ違う。

### 月500回のChatGPT連携で比べると

| ツール | プラン | 月額 | 500回の自動化に必要な消費 |
|---|---|---|---|
| Zapier | Starter | $19.99 | 500タスク（上限ちょうど） |
| Make | Core | $9.00 | 約1,500 ops（余裕あり） |
| Make | 無料 | $0 | 月2,000ops以内なら無料で動く |

シナリオが3モジュール構成の場合、500回×3＝1,500 ops。Makeの無料プランでも10,000 opsまで使えるので、**月500回程度なら無料で回せてしまう。**

年間に換算するとZapierのStarterプランだけで$239.88。Makeのゼロ円運用との差額は、そのままOpenAI APIへの投資に回せる。

### Zapierを使い続けるべきケース

乗り換えを一辺倒に推奨するつもりはない。以下に当てはまるなら、Zapierのままでいい。

- 月50回以下の低頻度自動化で複雑な条件分岐が不要
- Salesforce・HubSpotなど、Makeが対応していないアプリを中心に使っている
- エラー時の通知設定やサポートに$20払う価値を感じている

ZapierとChatGPTの連携設定については別記事で詳しく解説しているので、比較検討の参考にしてほしい。

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

## Make×ChatGPT連携の初期設定｜APIキー登録からシナリオ作成まで

### まず誤解を一つ潰しておく

「ChatGPT Plusを契約してるからAPIも使える」——これ、間違い。ChatGPT PlusとOpenAI APIは**別の課金体系**だ。APIを使うには[platform.openai.com](https://platform.openai.com)でAPIキーを別途発行する必要がある。

APIキーの発行手順はシンプルだ。

1. platform.openai.comにログイン
2. 左メニューの「API keys」→「Create new secret key」
3. 生成されたキー（`sk-...`で始まる文字列）をコピーして安全な場所に保存

キーは**発行時の一度しか表示されない。**見逃したら再発行するしかないので注意。

### MakeでのConnection登録

ここが地味に重要なポイントだ。APIキーをモジュールに直接貼り付けるやり方をしている人がいるが、Makeでは**Connection（接続情報）として登録する**のが正解。

Connection登録のメリットは明快で、キーを一か所で管理できる。キーをローテーションするときも、Connection側を更新するだけで全シナリオに反映される。

**登録手順：**

1. Makeのシナリオ編集画面を開く
2. 「＋」でモジュールを追加→「OpenAI (ChatGPT, Whisper, DALL-E)」を検索
3. モジュールの設定画面で「Connection」→「Add」をクリック
4. 「API Key」欄に先ほどのキーを貼り付けて保存

これで登録完了。次回以降は同じConnectionを選ぶだけで使い回せる。

### モデルはGPT-4o miniを選ぶ

モジュールの「Model」欄でモデルを選択する場面が来る。**gpt-3.5-turboは選ばないほうがいい。**

2025年以降、gpt-3.5-turboは実質的にレガシー扱いになっている。GPT-4o miniのほうが安くて精度も高い。Input $0.15/1Mトークン、Output $0.60/1Mトークンというコストで、500トークンの処理なら1回あたり約0.06円だ。月1,000回回しても60円程度のAPIコストしかかからない。

### シナリオのトリガーを選ぶ

トリガーとはシナリオを起動するきっかけのこと。よく使われる選択肢を挙げる。

- **Webhook**：外部からHTTPリクエストが来たら起動
- **Gmail**：特定のメールを受信したら起動
- **Googleスプレッドシート**：新しい行が追加されたら起動
- **スケジュール**：毎日9時など時間指定で起動

Googleスプレッドシートをトリガーにして文章を一括生成するユースケースは特に応用範囲が広い。別記事「ChatGPT API×スプレッドシート連携」も参考にしてほしい。

---

## 実務で使える連携シナリオ3選

設定の雰囲気が掴めたところで、すぐ使えるシナリオを3つ紹介する。

### シナリオA：Gmailの受信メール→ChatGPTで要約→Slackに通知

**モジュール構成：**
`Gmail（Watch Emails）` → `OpenAI（Create a Completion）` → `Slack（Create a Message）`

- モジュール数：3個
- 1回あたりのops消費：3 ops
- 月100回動かした場合：300 ops（無料プランの3%）

**Prompt例：**
```
以下のメール本文を3行以内で要約してください。
要点と必要なアクションがあれば最後に1行で添えてください。

{{メール本文}}
```

Gmailモジュールの「Subject」や「From」も変数として渡せるので、Slackの通知文に差出人名を含めると実用性が上がる。

### シナリオB：Googleスプレッドシートの行追加→ChatGPTで文章生成→同シートに書き戻し

**モジュール構成：**
`Googleスプレッドシート（Watch Rows）` → `OpenAI（Create a Completion）` → `Googleスプレッドシート（Update a Row）`

- モジュール数：3個
- 1回あたりのops消費：3 ops

A列にキーワードを入れると、B列にブログ導入文が自動生成されるシートが5分で作れる。コンテンツ制作の下書き量産に使っている人が周りに多い。

**Structured Outputsを使うとさらに便利**になる。JSONで出力させることでMake側のパース処理が不要になり、タイトル・本文・タグを別々のセルに書き分けることも簡単だ。

### シナリオC：Webhookで受け取ったデータ→ChatGPTで分類→Notionデータベースに振り分け

**モジュール構成：**
`Webhooks（Custom Webhook）` → `OpenAI（Create a Completion）` → `Router` → `Notion（Create a Database Item）` × カテゴリ数

- モジュール数：5〜7個（ルート数による）
- 1回あたりのops消費：5〜7 ops

これはMakeの条件分岐「Router」が活きる構成だ。Zapierでは同様の分岐処理にフィルター設定を複数Zapに分けて書く必要があり、タスク消費が倍以上になりやすい。

フォームの回答やサポート問い合わせをChatGPTにカテゴリ分類させ、Notionの担当者別データベースに自動振り分けするといった使い方が実務に近い。

---

## よくあるエラーと対処法

設定してすぐ詰まりやすいポイントを絞って書く。

**「401 Unauthorized」が出る**
APIキーが間違っているか、Connectionに登録したキーが失効している。platform.openai.comで新しいキーを発行し直してConnectionを更新する。

**「429 Too Many Requests」が出る**
OpenAI APIのレートリミットに引っかかっている。Makeのシナリオ設定で「Rate limiting」のウェイトを入れるか、OpenAIのUsage limitsページで上限を確認する。

**シナリオが動いているのにSlackに通知が来ない**
SlackのConnectionとチャンネルIDを確認。Slackモジュールのテスト実行時にMakeが「実際には送信しない」モードになっている場合がある。右上の「Run once」で本番実行して確認する。

---

## 次にやること、1つだけ

この記事を読んで「試してみようかな」と思ったなら、今日やることは一つだけでいい。

**platform.openai.comでAPIキーを発行する。**

Makeのアカウントは無料で作れるし、APIの初回クレジットも少額ながら付与されている。設定に1時間もかからず、シナリオAのGmail→ChatGPT→Slack通知は動く。

Zapierと並行して使い始めて、1か月後にコストを見比べてみるのが一番確かめやすい。

---

## 関連記事

- [ChatGPT API×スプレッドシート連携｜GAS初心者向け設定と自動化手順](/2026/05/22/ChatGPT-APIスプレッドシート連携GAS初心者向け設定と自動化手順/)
- [Notion AI×ChatGPT連携のやり方｜Zapierなし自動化4選](/2026/05/22/Notion-AIChatGPT連携のやり方Zapierなし自動化4選/)
- [ChatGPT×Googleカレンダー連携｜GASで予定自動登録・リマインド通知を設定する方法](/2026/05/23/ChatGPTGoogleカレンダー連携GASで予定自動登録リマインド通知を設定する方法/)

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