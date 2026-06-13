---
layout: post
title: "n8n Schedule トリガー Slack 定期通知やり方｜毎朝レポート自動送信の設定手順2026"
date: 2026-06-13 09:06:07 +0900
description: "n8n Schedule Triggerで毎朝SlackにレポートをCron式で自動送信する手順を解説。タイムゾーン9時間ズレ・チャンネルID指定・Catchupオプションまで詰まりポイントを先回りして解説。"
tags:
  - n8n Schedule トリガー Slack 定期通知 やり方
  - n8n 定期実行 毎朝 自動送信 設定方法
  - n8n Cron スケジュール Slack レポート 自動化 手順
  - n8n タイムゾーン Asia/Tokyo 設定 ズレ 対処法
---

{% raw %}
## 「なぜか9時に動かない」——その原因、タイムゾーンです

n8nでSlack通知を定期実行しようとして、設定したはずの時刻に何も届かない経験はないだろうか。ほぼ確実に、Schedule TriggerのTimezone設定がUTCのままになっている。これだけで9時間ズレる。

この記事では、n8n v1.x系の最新UIでSchedule Triggerを設定し、毎朝SlackにレポートをCron式で自動送信するまでの手順を順を追って説明する。タイムゾーン問題・チャンネルID指定・v1.70以降のCatchupオプションまで、詰まりやすいポイントを先回りしてカバーした。**読み終えれば、明日の朝9時からレポートが届く状態にできる。**

---



> 💡 **関連教材**: [n8nノーコード自動化 実践ワークフロー集（¥1,980）](https://aijissenlab.gumroad.com/l/bltzrk) — Gmail/Slack/Notion/Claude連携の動くワークフロー10本を実装込みで完全解説（全35ページ）

## 結論から言う：最小構成はたった3ノード

設定が難しそうに見えるが、基本フローはシンプルだ。

```
Schedule Trigger → Set Node（メッセージ組み立て）→ Slack Node
```

これだけで動く。データ取得やAI要約を挟むのは後から足せばいい。まず「毎朝9時にSlackへ何かが届く」という最小構成を完成させることを優先する。

---

## H2①｜Schedule Triggerとは？旧Cron Nodeとの違いと2026年時点の仕様

### 旧Cron Nodeはもう存在しない

2023年以前のn8n記事を参照している人は要注意。当時の「Cron Node」はすでに廃止済みで、**現在は「Schedule Trigger」として統合・刷新されている。**

旧Cron Nodeとの主な違いをまとめる。

| 項目 | 旧Cron Node | Schedule Trigger（現行） |
|------|------------|------------------------|
| ノード名 | Cron | Schedule Trigger |
| UI | シンプルなCron式入力 | GUI選択 ＋ Cron式の両対応 |
| Timezone設定 | なし（環境変数依存） | ノード個別に設定可能 |
| Catchupオプション | なし | v1.70以降で追加 |
| 実行ログ | 簡易 | 詳細な実行履歴 |

旧記事の手順通りにやろうとしてノードが見つからない——その場合、ほぼ全員がこの変更に引っかかっている。

### Schedule Triggerで設定できる実行パターン

GUIモードでは以下が選べる。

- **Every X Minutes / Hours**：インターバル指定（最短1分）
- **Daily**：毎日の特定時刻
- **Weekly**：曜日＋時刻
- **Monthly**：日付＋時刻
- **Custom (Cron expression)**：Cron式で自由に設定

「毎朝9時・平日のみ」のような複合条件はCron式一択になる。慣れれば難しくない。

### v1.70以降のCatchupオプションについて

Catchupは、**サーバーダウン中にスキップされた実行を復旧後にまとめて処理するかどうか**を制御するオプション。

デフォルトはオフ。毎朝のレポート送信なら基本はオフのままでいい。サーバーが落ちていた翌朝に昨日分も含めて2件送信されても、受け取る側が混乱するだけだ。データ集計系のワークフローで「取りこぼしたくない」場合にのみオンにすることを検討する。

### n8n Cloud vs Self-hosted：定期実行の挙動の差

正直なところ、**Self-hostedは設定が甘いと定期実行が止まる。**

n8n Cloudなら月5,000実行まで無料で使えて（2025年改定値）、毎朝1回なら年365回なので無料枠で余裕。安定性を優先するなら最初はCloudで試すのがいい。

Self-hostedで安定させるには、以下の環境変数が必須。

```bash
# docker-compose.ymlまたは.envファイルに追加
EXECUTIONS_PROCESS=main
N8N_RUNNERS_ENABLED=true
```

これを未設定のまま使っているケースが多い。スケジュールが突然止まったら、まずここを確認する。

---

## H2②｜Schedule Trigger × Slack通知の基本フロー構築手順

### Step 1：Schedule Triggerノードを追加する

n8nのキャンバスを開いて「+」ボタンをクリック。検索欄に「schedule」と入力すると「Schedule Trigger」が出てくる。これを追加。

旧UIに慣れている人は戸感うかもしれないが、2026年時点の新UIでは左上のキャンバスエリアにドラッグ&ドロップでノードが配置される。

### Step 2：Cron式を設定する

Schedule TriggerノードのParametersを開く。**Trigger Rulesセクション**の「Rule」を「Custom (Cron expression)」に切り替える。

平日のみ朝9時に実行する場合：

```text
0 9 * * 1-5
```

各フィールドの意味：

```text
分  時  日  月  曜日
0   9   *   *   1-5

1=月曜、2=火曜、...5=金曜
```

毎日（土日含む）朝9時なら：

```text
0 9 * * *
```

Cron式に慣れていない場合は [crontab.guru](https://crontab.guru) で確認するのが一番早い。入力した式が人間語で何を意味するか表示してくれる。

### Step 3：Timezone設定——ここが最重要

Parametersの下部に「**Timezone**」フィールドがある。**デフォルトはUTC。これを必ず`Asia/Tokyo`に変更する。**

設定箇所の描写：Trigger Rulesの下にある「Settings」を展開すると、Timezoneのドロップダウンが現れる。検索欄に「Tokyo」と入力すると`Asia/Tokyo`が出てくる。これを選択。

これをやらないと、UTCの9時 ＝ 日本時間18時に動く。「なぜ夕方に届くんだ」という問い合わせの9割がこれだ。

### Step 4：Slack Nodeを接続する

Schedule TriggerノードからArrowを引き出して「Slack」ノードを追加。

Slack Nodeを開いたら、まずCredentialの設定。「Create New Credential」から「Slack API」を選び、Bot User OAuth Tokenを入力する。

**Bot TokenのスコープはSlack APIダッシュボードで設定が必要。** 最低限以下が必要：

```
chat:write
```

このスコープがないと、メッセージ送信時に403エラーが返ってくる。2025年以降のSlack APIは権限チェックが厳格になっているため、古いTokenをそのまま使い回していると詰まる。

Bot Tokenの取得手順の詳細は [api.slack.com/apps](https://api.slack.com/apps) から新規Appを作成してBot Token Scopeに`chat:write`を追加し、ワークスペースにインストールすれば取得できる。

### Step 5：チャンネルIDで指定する

Slack Nodeの「Channel」フィールド。ここに`#general`と入力する人が多いが、**チャンネル名は変更されると壊れる。IDで指定するのが鉄則。**

チャンネルIDの取得方法：

1. Slackデスクトップアプリでチャンネル名を右クリック
2. 「チャンネル詳細を表示」を選択
3. 表示されたモーダルの一番下に`C0XXXXXXXX`形式のIDが表示されている
4. これをコピーしてn8nのChannelフィールドに貼り付け

IDは`C`で始まる英数字10文字前後。これを指定しておけばチャンネルのリネーム後も壊れない。

### Step 6：テスト実行からActivateまで

フローを組み終えたら、**必ず「Execute once」でテスト実行してから本番Activateする。**

手順：

1. Schedule Triggerノードを右クリック→「Execute Node」でワークフロー全体を1回だけ走らせる
2. Slackに実際にメッセージが届いたことを確認
3. エラーがなければキャンバス右上の「Inactive」トグルを「Active」に切り替え

Activeにして初めてスケジュール実行が有効になる。Inactive状態ではCronが動かないので、確認後は忘れずにActivateすること。

---

## H2③｜よくある失敗4つと対処法

### 失敗①：設定時刻に動かない → タイムゾーンの確認から

前述の通り、**Timezoneフィールドが`UTC`のままになっているケースが最多。**

Schedule TriggerノードのSettings内にあるTimezoneが`Asia/Tokyo`になっているか必ず確認する。環境変数（`GENERIC_TIMEZONE`）でも設定できるが、ノード個別設定の方が優先されるため、ノード側で明示的に設定する方が確実。

### 失敗②：Slack送信が403エラーになる → Bot Tokenのスコープ不足

エラーメッセージに`missing_scope`と書いてある場合、Bot Tokenに`chat:write`が付与されていない。

[api.slack.com/apps](https://api.slack.com/apps) → 対象App → OAuth & Permissions → Bot Token Scopes → `chat:write`を追加 → 再インストール、の手順で解決する。再インストール後はBot Tokenが更新されるため、n8nのCredential側も更新が必要。

### 失敗③：Self-hostedで定期実行が途中から止まる → 環境変数の設定

数日は動いていたのに急に止まった、というケースの多くはWorkerプロセスの問題。

```bash
# .envまたはdocker-compose.ymlに追記
EXECUTIONS_PROCESS=main
N8N_RUNNERS_ENABLED=true
EXECUTIONS_TIMEOUT=3600
```

`EXECUTIONS_PROCESS=main`はスケジュール実行をメインプロセスで処理する設定。これがないと一部のDocker環境でスケジューラーが機能しない。

### 失敗④：チャンネル名を変えたらSlack通知が届かなくなった → IDで指定していない

これは設定時に防げる失敗。すでにチャンネル名で設定している場合、今すぐIDに変更しておく。取得方法はStep 5を参照。

---

## H2④｜Block Kitでリッチなレポートに仕上げる（応用）

基本の送信が動いたら、メッセージを見やすくする。Slack NodeのMessage Typeを**「Blocks」**に切り替えるだけで、ヘッダー・セクション・区切り線を使ったリッチなレポートになる。

以下はn8n上でBlocksフィールドに入力するJSON例：


[
  {
    "type": "header",
    "text": {
      "type": "plain_text",
      "text": "📊 本日のレポート（{{ $now.format('YYYY/MM/DD') }}）"
    }
  },
  {
    "type": "section",
    "text": {
      "type": "mrkdwn",
      "text": "*売上：* {{ $json.sales }}円\n*前日比：* {{ $json.diff }}%"
    }
  },
  {
    "type": "divider"
  },
  {
    "type": "section",
    "text": {
      "type": "mrkdwn",
      "text": "詳細は <https://your-dashboard.com|ダッシュボード> で確認"
    }
  }
]
```

`{{ $now.format('YYYY/MM/DD') }}`はn8nのExpression記法で今日の日付を動的に入れている。`$json.sales`は前のノードからデータを引き継ぐ変数。

正直、これだけでSlack通知のクオリティが一段上がる。プレーンテキストより視認性が高く、受け取る側のUXが改善する。

---

## H2⑤｜無料版の限界と正直なデメリット

### n8n Cloudの無料プランで詰まるポイント

n8n Cloudの無料プランは**月5,000実行まで**（2025年改定）。毎朝1回のレポートなら年365回で余裕の範囲内。

ただし制約もある。

- **実行タイムアウトが短い**：無料プランは1ワークフロー実行につき最大数十秒程度。外部APIを多数叩くヘビーな処理は有料プランが必要
- **同時実行数の制限**：無料プランでは並列実行数が制限される。単発の朝レポートなら問題ないが、複数ワークフローを同時に動かしたいなら注意
- **実行ログの保持期間**：無料プランはログ保持が短め。デバッグが後から難しくなることがある

### Schedule Triggerの正直なデメリット

- **最短1分間隔**：秒単位の実行はできない。リアルタイム性が必要なユースケースには向かない
- **複雑なスケジュールは1ノードでは無理**：「月末最終営業日のみ実行」のような条件は、Schedule Trigger単体では実現できずIFノードで日付判定を組み合わせる必要がある
- **Self-hostedの初期設定コスト**：環境変数の設定を正しくやらないと不安定。ここはCloudの方が明らかに楽

---

## H2⑥｜よくある質問

### Q1. Schedule Triggerを複数設定することはできますか？

できる。1つのワークフローにSchedule Triggerは複数置ける。「平日は毎朝9時、月曜のみ週次レポートも送る」という場合は、Triggerを2つ用意してそれぞれ別のSlack Nodeに接続する形で実装可能。

ただし、ワークフロー全体をActiveにした時点で両方のTriggerが動き始めるため、設定の意図を混同しないよう各ノードに名前をつけておくことを勧める。

### Q2. Slackに送る内容を前の日のデータにするには？

Schedule Trigger直後にDate & Timeノードを追加し、昨日の日付を計算してそれ以降のノードに引き渡す方法が一般的。

```javascript
// Code Nodeでの前日日付取得例（n8n Expression記法）
const yesterday = $now.minus({days: 1}).toFormat('yyyy-MM-dd');
return [{ json: { date: yesterday } }];
```

これをDBクエリやAPIのパラメータとして渡せば、前日分のデータを取得できる。

### Q3. テスト時は「Execute once」を使えばいいですか？スケジュールを待つ必要はありますか？

テストは「Execute once」で十分。スケジュールが来るのを待つ必要はない。

Schedule Triggerノードを右クリックして「Execute Node」を選ぶか、キャンバス上部の「Test workflow」ボタンを押すと即座に全ノードが実行される。実際のCron設定とは無関係に手動実行できるので、Activateする前に必ずここで動作確認する習慣をつける。

---

## まとめ：明日の朝9時に届くかどうかは、Timezone1行で決まる

設定全体を振り返ると、詰まりポイントの8割はこの2つに集中している。

1. **Schedule TriggerのTimezoneを`Asia/Tokyo`に設定する**（デフォルトUTCのまま放置は厳禁）
2. **SlackのチャンネルをIDで指定する**（`C0XXXXXXXX`形式）

この2点を押さえれば、残りの設定は比較的スムーズに進む。

**今すぐやるべき1つのアクション：** n8nのキャンバスを開いて、Schedule Triggerノードを1つ追加し、Timezoneを`Asia/Tokyo`にしてCron式を`0 9 * * 1-5`に設定する。それだけで平日朝9時の自動実行の土台ができる。Slack送信は後から接続すればいい。

Block Kitを使ったリッチなメッセージの整形や、AI要約を挟んだ朝レポートの作り方については、「n8n Slack Block Kit整形」の記事で詳しく解説している。

---

## 関連記事

- [n8n Gmail連携やり方｜特定メール受信をSlackに自動転送するノーコード手順](/2026/06/02/n8n-Gmail連携やり方特定メール受信をSlackに自動転送するノーコード手順/)
- [Zapier×ChatGPT連携のやり方｜ノーコードで業務通知・メール自動化を設定する手順](/2026/05/23/ZapierChatGPT連携のやり方ノーコードで業務通知メール自動化を設定する手順/)
- [Make Gmail連携やり方｜特定メール受信をNotionに自動記録する手順](/2026/06/01/Make-Gmail連携やり方特定メール受信をNotionに自動記録する手順/)

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

**AIスキルを収益化するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FMMNG2+2PEO+1HLFVM" rel="nofollow">ココナラ</a>でサービスを出品できる。AIライティング・画像生成・データ分析など、AIスキルを活かした案件の需要は増えている。<img border="0" width="1" height="1" src="https://www19.a8.net/0.gif?a8mat=4B3OQW+FMMNG2+2PEO+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。
{% endraw %}
