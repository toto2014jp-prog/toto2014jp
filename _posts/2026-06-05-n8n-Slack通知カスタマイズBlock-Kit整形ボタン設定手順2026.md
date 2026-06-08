---
layout: post
title: "n8n Slack通知カスタマイズ｜Block Kit整形・ボタン設定手順2026"
date: 2026-06-05 09:06:21 +0900
description: "n8nでSlack通知をBlock Kit整形＋ボタン付きインタラクティブ通知にする手順を解説。JSONの正しい渡し方・Webhook連携・Signing Secret設定まで2026年版で実例付き。"
tags:
  - n8n Slack Block Kit 通知 カスタマイズ
  - n8n Slack インタラクティブ ボタン 設定
  - n8n Slack メッセージ 整形 やり方
  - n8n Webhook Slack 応答 受信 手順
---

## Slack通知、ちゃんと読まれてますか？

n8nでSlack通知を飛ばしているのに「誰も気づかない」「クリックされない」——正直なところ、プレーンテキストのままでは仕方ない話です。

Slack公式の2025年調査によると、ボタン付きBlock Kitメッセージのクリック率はプレーンテキスト比で**約3.2倍**。数字で見ると、通知の「見た目」を整えることがいかに重要か分かります。

この記事では、n8n v1系でSlack通知をBlock Kit形式に整形する方法と、ボタン付きインタラクティブ通知の設定手順を実例コード付きで解説します。「Block KitのJSONを貼ったらエラーになった」「ボタンを押しても何も起きない」という詰まりポイントも合わせて潰せます。

---



> 💡 **関連教材**: [n8nノーコード自動化 実践ワークフロー集（¥1,980）](https://aijissenlab.gumroad.com/l/bltzrk) — Gmail/Slack/Notion/Claude連携の動くワークフロー10本を実装込みで完全解説（全35ページ）

## まず結論：必要なのは「Slackノード単体」ではない

整形済みBlock Kit通知を送るだけなら**Slackノード＋Codeノード**の2ノード構成で十分です。ただし、ボタンクリックを受け取って処理するインタラクティブ通知には**Webhookノードが別途必要**になります。

構成をひと言で言うと：

- **メッセージ整形だけ**：Codeノード → Slackノード
- **ボタン応答まで**：Codeノード → Slackノード ＋ Webhookノード（応答受信用）

これを押さえた上で、順番に設定していきましょう。

---

## H2①：なぜ今、通知のカスタマイズが必要なのか

### プレーンテキストの限界

n8nでGmailの受信をSlackに転送する程度なら、Slackノードのデフォルト設定でも動きます。でも実際に運用してみると、こんな問題が出てきます。

- 本文が長いと途中で切れて読めない
- 「対応する」「無視する」の判断を通知内でできない
- どのシステムからの通知か一目で分からない

特にチームで使う場合、通知が流し読みされるとワークフローの自動化が「意味をなさない」状態になります。

### n8n v1系でUIが変わった問題

2025年以降、n8nはv1系が主流です。旧v0系のチュートリアルをそのまま参照すると、Slackノードのフィールド名や設定項目の場所が違うため、操作が詰まります。

具体的には、旧来の「Additional Fields」の中にあったBlocksフィールドが、v1系では「Message Content」タブ内の独立フィールドに移動しています。スクリーンショット付きの古い記事を参照している方はここで迷う人が多い。

### この記事でできるようになること

1. Block Kit形式でリッチなSlack通知を送る
2. ボタン付きメッセージで「承認/却下」などのアクションを通知内に埋め込む
3. ボタンクリックをn8nのWebhookで受け取って次の処理につなぐ

---

## H2②：メッセージ整形の基本｜Block KitをCodeノードで組み立てる

### よくある失敗：Block Kit BuilderのJSONをそのまま貼るとエラーになる

Slack公式の「Block Kit Builder」（app.slack.com/block-kit-builder）は便利なツールです。ビジュアルで構成を組んで、右側にJSONが生成される。ここまでは問題ない。

問題は、生成されたJSONをn8nのSlackノードにそのまま貼ったときです。

Block Kit Builderが生成するJSONはこういう構造になっています。


{
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*新しいIssueが登録されました*"
      }
    }
  ]
}
```

n8nのSlackノードの「Blocks」フィールドには、**`blocks` キーの外側ラッパーを除いた配列だけ**を渡す必要があります。つまり渡すのはここだけです。


[
  {
    "type": "section",
    "text": {
      "type": "mrkdwn",
      "text": "*新しいIssueが登録されました*"
    }
  }
]
```

外側の `{"blocks": ...}` ラッパーごと貼ると型エラーが出て通知が送れません。ここで詰まる人が本当に多い。

### Slack API v2の仕様変更：textフィールドが必須になった

もう一つ重要な変更点があります。2025年以降のSlack APIでは、`chat.postMessage` でBlocksを使う場合でも `text` フィールドが必須になりました。

`text` はプッシュ通知やスクリーンリーダーのフォールバック用です。指定しないと「text field required」エラーが返ってきます。

n8nのSlackノードでは「Message Text」フィールドに短い説明を入れておきましょう。「新しいIssueが登録されました」程度で十分です。

### Codeノードでブロックを動的に組み立てる

GitHubのIssue登録をトリガーにSlack通知を整形する例を見てみましょう。n8nのCodeノードに以下を記述します。

```javascript
// Codeノード（JavaScript）
const issue = $input.first().json;

const blocks = [
  {
    type: "header",
    text: {
      type: "plain_text",
      text: `🐛 新しいIssue: ${issue.title}`,
      emoji: true
    }
  },
  {
    type: "section",
    fields: [
      {
        type: "mrkdwn",
        text: `*リポジトリ:*\n${issue.repository}`
      },
      {
        type: "mrkdwn",
        text: `*優先度:*\n${issue.priority ?? '未設定'}`
      }
    ]
  },
  {
    type: "section",
    text: {
      type: "mrkdwn",
      text: `*概要:*\n${issue.body?.slice(0, 200) ?? '説明なし'}...`
    }
  },
  {
    type: "actions",
    elements: [
      {
        type: "button",
        text: { type: "plain_text", text: "対応する", emoji: true },
        style: "primary",
        action_id: "issue_accept",
        value: String(issue.id)
      },
      {
        type: "button",
        text: { type: "plain_text", text: "クローズ", emoji: true },
        style: "danger",
        action_id: "issue_close",
        value: String(issue.id)
      }
    ]
  }
];

return [
  {
    json: {
      text: `新しいIssue: ${issue.title}`,  // フォールバック用
      blocks: blocks
    }
  }
];
```

このCodeノードの出力を後続のSlackノードに接続します。Slackノードの設定は次のようにします。

- **Resource**：Message
- **Operation**：Send
- **Channel**：通知先チャンネル（例：`#dev-alerts`）
- **Message Text**：`{{ $json.text }}`
- **Blocks**：`{{ $json.blocks }}`

`Blocks` フィールドに式で配列を渡すとき、`JSON.stringify()` や `JSON.parse()` は不要です。n8nは配列オブジェクトをそのままシリアライズして送信してくれます。

---

## H2③：ボタン付きインタラクティブ通知の設定手順｜Webhookノード連携

### 誤解：Slackノード単体でボタンクリックを受け取れると思っていた

ボタン付きメッセージを送ること自体はSlackノードだけで可能です。ただし、ユーザーがボタンを押したときの**レスポンスを受け取る**には、外部からのHTTPリクエストを受け取れるURLが必要です。これがWebhookノードの役割です。

構成はこうなります。

```
[トリガー] → [Codeノード] → [Slackノード（通知送信）]

[Webhookノード（ボタン応答受信）] → [IF/Switchノード] → [後処理]
```

2つのワークフローが別々に動く形です。Slackはボタンクリック時にWebhook URLへPOSTリクエストを送り、n8nのWebhookノードがそれを受け取ります。

### ステップ1：n8nでWebhookノードを作成する

新しいワークフローを作成し、最初のノードとして「Webhook」を追加します。設定は以下のとおり。

- **HTTP Method**：POST
- **Path**：`slack-interactive`（任意。分かりやすい名前にする）
- **Authentication**：Header Auth（後述のSigning Secret検証のため）

ノードを保存するとWebhook URLが生成されます。クラウド版の場合は `https://your-instance.n8n.cloud/webhook/slack-interactive` の形式です。セルフホスト版なら自分のドメイン配下になります。

**重要**：本番運用ではWebhook URLを「Production URL」で固定化してください。「Test URL」は手動実行時しか有効にならないため、ボタンを押しても反応しない原因になります。

### ステップ2：Slack側でRequest URLを設定する

Slackアプリの管理画面（api.slack.com/apps）を開きます。

1. 対象のアプリを選択
2. 左メニューの「Interactivity & Shortcuts」をクリック
3. 「Interactivity」をONにする
4. 「Request URL」にn8nのWebhook URLを貼り付ける
5. 「Save Changes」をクリック

URLを保存するとき、SlackはそのURLに検証リクエストを送ります。n8nのWebhookノードが200レスポンスを返せれば設定完了です。

### ステップ3：Signing Secretで署名検証する

これが最も見落とされやすい設定です。Signing Secretを検証しないと、第三者が偽のリクエストを送り込んでワークフローを不正起動できてしまいます。

Slackアプリ管理画面の「Basic Information」→「App Credentials」から `Signing Secret` をコピーしておいてください。

n8nでの検証はCodeノードを使って実装します。Webhookノードの直後に以下のCodeノードを挟みます。

```javascript
// Signing Secret検証（Codeノード）
const crypto = require('crypto');

const signingSecret = 'YOUR_SIGNING_SECRET'; // 環境変数化推奨
const headers = $input.first().json.headers;
const body = $input.first().json.body;  // 生のリクエストボディ（文字列）

const timestamp = headers['x-slack-request-timestamp'];
const slackSignature = headers['x-slack-signature'];

// リプレイアタック対策：5分以上古いリクエストは拒否
const currentTime = Math.floor(Date.now() / 1000);
if (Math.abs(currentTime - parseInt(timestamp)) > 300) {
  throw new Error('Request timestamp too old');
}

const sigBaseString = `v0:${timestamp}:${body}`;
const mySignature = 'v0=' + crypto
  .createHmac('sha256', signingSecret)
  .update(sigBaseString, 'utf8')
  .digest('hex');

if (!crypto.timingSafeEqual(
  Buffer.from(mySignature),
  Buffer.from(slackSignature)
)) {
  throw new Error('Invalid signature');
}

// ペイロードをパース
const payload = JSON.parse(
  decodeURIComponent(body).replace('payload=', '')
);

return [{ json: payload }];
```

実務では `signingSecret` をn8nの「Credentials」か環境変数で管理してください。コードにハードコードするのは避けます。

### ステップ4：ボタンの応答を分岐処理する

署名検証が通ったら、`action_id` を見てどのボタンが押されたか判定します。IFノードまたはSwitchノードで分岐させます。

```
{{ $json.actions[0].action_id }} が "issue_accept" → 対応ワークフローへ
{{ $json.actions[0].action_id }} が "issue_close"  → クローズ処理へ
```

それぞれの分岐先でJiraにチケットを更新したり、GitHubのIssueをクローズしたりと、後続処理を組み込めます。

---

## H2④：設定の比較と選択基準

### メッセージ整形の方法比較

| 方法 | メリット | デメリット | 向いているケース |
|------|----------|------------|------------------|
| Slackノードの標準フィールドのみ | 設定が簡単 | 表現力が低い、フォーマット限定 | 簡易な内部通知 |
| Block Kit JSON直貼り | デザインの自由度が高い | n8n式で動的化しにくい | 固定テンプレート通知 |
| Codeノードで動的生成 | データに応じた整形が可能 | コーディング知識が必要 | 本番ワークフロー全般 |
| AI Agent連携（LLM整形） | 要約・翻訳を自動化できる | レイテンシ増・コスト増 | 長文要約・多言語通知 |

### セルフホスト vs クラウド：Webhook URL安定性の比較

| 項目 | セルフホスト（Docker/VPS） | n8nクラウド版 |
|------|---------------------------|---------------|
| Webhook URL固定化 | 独自ドメイン設定で可能 | 2025年後半から標準対応 |
| SSL証明書 | 自己管理（Let's Encrypt等） | 自動管理 |
| 月額コスト | サーバー代のみ（$5〜） | プラン依存（$20〜） |
| Signing Secret設定 | 環境変数で管理 | Credentials UIで管理 |
| 本番稼働の手間 | やや高い | 低い |

n8nユーザーの約60%がセルフホストを選んでいますが（2025年ユーザーアンケート）、インタラクティブ通知のように「Webhook URLが固定されている必要がある」用途では、クラウド版の方がセットアップが圧倒的に楽です。

---

## H2⑤：よくある失敗と対処法

### よくある質問①：ボタンを押しても何も反応しない

原因の90%は以下のどれかです。

- Webhookノードが「Test URL」モードのまま（本番実行になっていない）
- Slack側のRequest URLが古いURLのまま更新されていない
- Signing Secret検証でエラーが出てワークフローが即終了している

確認手順：n8nのWebhookノードを「Production」でアクティブにする → Slackアプリ管理画面のRequest URLを現在のURLに更新 → Slackから検証リクエストが届くか確認。

### よくある質問②：`text field required` エラーが出る

Slack API v2の仕様変更です。Blocksを使うメッセージでも `text` フィールドは必須になりました。Slackノードの「Message Text」フィールドに短いフォールバックテキストを設定してください。空欄のままにしているとこのエラーが出ます。

### よくある質問③：Legacy Tokenが使えなくなった

2025年以降、Slack公式はLegacy Tokenの新規発行を停止しています。既存ワークフローでLegacy Tokenを使っている場合、OAuth2 Bot Token（`xoxb-` で始まる）に移行が必要です。

n8nのCredentials設定で「Slack OAuth2 API」を選択し、Bot Tokenを再設定してください。移行後は旧Credentialsを削除することをおすすめします。不要なTokenが残っているとセキュリティリスクになります。

### よくある質問④：メッセージを更新したい（`chat.update`の使い方）

ボタンを押した後に「✅ 対応済み」とメッセージを書き換えたいケースがあります。この場合、`chat.postMessage` の再送ではなく `chat.update` を使います。

n8nのSlackノードで「Update Message」アクションを選択し、`ts`（タイムスタンプ）パラメータに元のメッセージのtsを指定します。tsはSlackノードの送信レスポンスに含まれているので、最初の送信時に変数として保持しておく必要があります。

```javascript
// 送信時のレスポンスからtsを取得（Codeノード例）
const response = $input.first().json;
const messageTs = response.ts; // 例："1717600000.123456"

return [{ json: { ts: messageTs } }];
```

このtsを後続ワークフローのUpdate Messageノードに渡す形です。

### よくある質問⑤：Block Kitのactionsブロックでボタンが表示されない

`actions` ブロックの `elements` 配列が空になっているか、`type: "button"` の指定が間違っているケースがほとんどです。

Block Kit Builderで確認してから実装するのが確実です。また、1つの `actions` ブロックに入れられるボタンは最大5個までというSlackの制限があります。6個以上入れると最後のボタンが切り捨てられます。

---

## H2⑥：正直なデメリットと無料版の限界

### n8nクラウド無料プランの制限

n8nクラウドの無料プランはワークフロー実行回数に月間上限があります（2026年時点で月500実行）。Slackボタンをよく使う運用では、1クリックで複数のワークフロー実行が発生するため、予想より早く上限に達します。

本番運用ならセルフホストかStarterプランへの移行を前提にした方がいいです。

### Signing Secret検証の実装コストが高い

インタラクティブ通知の設定で最も手間がかかるのがSigning Secret検証です。n8nはビジュアルフローツールなのに、ここだけはコーディングが必要になります。

「ボタンを送るだけでいい、応答は不要」という要件なら、WebhookノードもSigning Secretも不要です。要件を先に固めてから実装に入ることをおすすめします。

### Webhookノードの応答タイムアウト

Slackはボタンクリック後、3秒以内に200 OKが返ってこないと「タイムアウト」エラーをユーザーに表示します。n8nのWebhookノードで受け取った後、重い処理（DB更新、外部API呼び出しなど）を挟むと3秒を超えることがあります。

対策として「応答はWebhookノードですぐに返す → 後処理は別のワークフローにキュー投入」という構成が有効です。n8nのExecute Workflow（非同期モード）を使うと実現できます。

---

## 締め：次にやる1つのアクション

この記事で解説した内容をすべていきなり実装しようとすると混乱します。

**最初の1ステップはBlock Kit整形だけに絞る**のが正解です。

既存のSlack通知ワークフローにCodeノードを1個挟んで、セクションブロック＋textフォールバックを返すだけ。それだけで通知の見た目は劇的に変わります。ボタンとWebhookの設定はその後で追加する流れが、詰まらずに進める一番現実的なルートです。

Slack連携の基本（Gmail受信をSlackへ転送する構成）からおさらいしたい方は、「n8n Gmail連携やり方｜特定メール受信をSlackに自動転送する手順」も合わせてどうぞ。n8n環境のセットアップからやり直したい方は「n8nセルフホストやり方｜Railway・Docker対応」を先に参照してください。

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

- [n8n Gmail連携やり方｜特定メール受信をSlackに自動転送するノーコード手順](/2026/06/02/n8n-Gmail連携やり方特定メール受信をSlackに自動転送するノーコード手順/)
- [Make×Notionデータベース連携やり方｜フォーム回答を自動追加する手順](/2026/05/29/MakeNotionデータベース連携やり方フォーム回答を自動追加する手順/)
- [Make Gmail連携やり方｜特定メール受信をNotionに自動記録する手順](/2026/06/01/Make-Gmail連携やり方特定メール受信をNotionに自動記録する手順/)

---

## 関連ツール紹介

**AIスキルを収益化するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FMMNG2+2PEO+1HLFVM" rel="nofollow">ココナラ</a>でサービスを出品できる。AIライティング・画像生成・データ分析など、AIスキルを活かした案件の需要は増えている。<img border="0" width="1" height="1" src="https://www19.a8.net/0.gif?a8mat=4B3OQW+FMMNG2+2PEO+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。