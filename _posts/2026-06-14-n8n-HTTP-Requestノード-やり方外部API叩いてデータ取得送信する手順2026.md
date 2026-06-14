---
layout: post
title: "n8n HTTP Requestノード やり方｜外部API叩いてデータ取得・送信する手順2026"
date: 2026-06-14 09:06:22 +0900
description: "n8n HTTP RequestノードでGET/POST・認証設定・エラー対処まで手順を解説。2026年v1.x系の最新UI対応。コピペで動くサンプル付き。"
tags:
  - n8n HTTP Requestノード やり方
  - n8n 外部API連携 データ取得 設定方法
  - n8n APIリクエスト GET POST 手順
  - n8n 認証設定 Credential Bearer APIキー
---

{% raw %}
n8nで「外部APIを叩きたい」と思って調べると、古いv0.x系の記事ばかり出てくる。コピペしたら`$node["xxx"].json`が動かない、認証の設定欄がどこにあるかわからない——そんな経験、ないだろうか。

この記事では、**2026年現在のv1.x系UIに完全対応した手順**で、HTTP RequestノードのGET・POST・認証・エラー対処・Paginationまで一気に解説する。読み終わったら、たいていのAPIはこの記事だけで叩けるようになる。

---

## 結論：HTTP Requestノードで最初に押さえる3点

細かい設定は後で説明するが、まず答えから出す。

1. **Method・URL・Response Format**の3つだけ設定すれば最初のGETは動く
2. 認証は**Credentialタブ**で設定する。Headerに手書きは絶対にやめる
3. v1.x系の変数記法は`{{ $json.key }}`か`{{ $('ノード名').item.json.key }}`。古い記法と混ぜると壊れる

これだけ頭に入れておけば、以下の手順がスムーズに追える。

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

## H2①：GETリクエストの基本設定｜最小構成で動かす手順

### HTTP Requestノードを追加してUIを確認する

n8nのキャンバスで「＋」ボタンを押し、検索欄に「HTTP Request」と入力する。2026年v1.x系のUIでは、ノードを開いたときに以下のタブが並んでいる。

- **Parameters**（Method・URL・Headers・Body等の主設定）
- **Authentication**（Credentialの紐付け）
- **Options**（タイムアウト・リダイレクト・Paginationなど）

旧バージョンでは「Settings」タブに認証があったが、今は**Authenticationタブに分離**されている。古い記事の手順と画面が合わない場合、大抵ここが原因だ。

### 最小構成でGETを動かす

まず手元で試せる無料のテストAPIとして、`jsonplaceholder.typicode.com`を使う。設定はこれだけ。

```text
Method: GET
URL: https://jsonplaceholder.typicode.com/todos/1
Response Format: JSON（デフォルトのまま）
```

「Test Step」を押すと、以下のようなレスポンスが返ってくる。


{
  "userId": 1,
  "id": 1,
  "title": "delectus aut autem",
  "completed": false
}
```

取得したデータを次のノードで使うときの記法は、**v1.x系では`{{ $json.title }}`**と書く。`$json`はそのノードの出力データを指す。

複数アイテムがある場合や別ノードのデータを参照する場合は`{{ $('ノード名').item.json.title }}`を使う。

### よくある失敗①：古い記法をコピペして動かない

QiitaやZennの2023年以前の記事には、こう書いてあることが多い。

```text
# ❌ v0.x系の古い記法（v1.xでは動かない）
{{$node["HTTP Request"].json["title"]}}
```

v1.x系で実行すると「Cannot read properties of undefined」系のエラーが出る。見た目は似ているので気づきにくい。

```text
# ✅ v1.x系の正しい記法
{{ $json.title }}
{{ $('HTTP Request').item.json.title }}
```

Stack Overflowやn8nコミュニティフォーラムでも、この混在ミスの質問が2025年以降急増している。古い記事を参考にするときは記法だけ必ず確認してほしい。

### GETでクエリパラメータを渡す方法

URLに直接書く方法と、「Query Parameters」セクションで設定する方法がある。後者が可読性・メンテナンス性で圧倒的に優れている。

Parameters → 「Add Query Parameter」で以下のように追加する。

```text
Name: userId
Value: {{ $json.userId }}   ← 動的に渡す場合
```

これで`?userId=1`が自動的にURLに付与される。URLに直接`?userId={{ $json.userId }}`と書いても動くが、値にスペースや日本語が入ったときのエンコード漏れのリスクがある。パラメータは必ずフィールドで設定する習慣をつけておく方が無難だ。

---

## H2②：認証設定（Bearer・APIキー・Basic）｜Credentialタブの正しい使い方

### なぜHeaderに手書きしてはいけないのか

正直なところ、自分も最初はHeaderに`Authorization: Bearer sk-xxxxx`と直書きしていた。これの何が問題かというと、**ワークフローをJSON形式でエクスポートした瞬間にAPIキーが平文で含まれる**。

n8nのワークフローはチームで共有・GitHubで管理することが多い。そこにAPIキーが混入するのはセキュリティ上アウトだ。Credentialを使えばエクスポートJSONには`credentialId`の参照IDしか含まれない。

### 3パターンの認証設定手順

**AuthenticationタブでCredentialを紐付ける流れは共通**。「Credential for connecting to」のドロップダウンから新規作成する。

| 認証タイプ | 代表的なAPI | Credentialの種類 |
|---|---|---|
| Bearer Token | OpenAI、GitHub、Notion | Header Auth（Bearer形式）|
| APIキー（Header型） | SendGrid、各種SaaS | Generic API Key |
| APIキー（Query型） | 一部の地図・天気API | Generic API Key |
| Basic認証 | 社内システム、WordPress REST | HTTP Basic Auth |
| OAuth2 | Google、Salesforce等 | OAuth2専用（別途設定）|

#### Bearer Token（OpenAI等）の場合

```text
Credential Type: Header Auth
Name: Authorization
Value: Bearer sk-proj-xxxxxxxxxx
```

Valueに`Bearer `（スペース含む）を忘れるミスが多い。OpenAI APIで401が返ってきたら、まずここを疑う。

#### APIキー（Header型）の場合

```text
Credential Type: Generic API Key
Header Name: X-API-Key  ← APIによって異なる
API Key: your-api-key-here
```

Header名はAPI仕様書を必ず確認する。`X-API-Key`・`api-key`・`apikey`とサービスによってバラバラだ。

### POSTリクエストのBody設定｜JSON・form-urlencoded・Rawの使い分け

よくある失敗②として、**Body TypeをJSON一択だと思い込んでいる**パターンがある。実際はAPIによって受け付けるBody形式が異なる。

```text
# Content-Typeで判断する
application/json        → JSON Body
application/x-www-form-urlencoded → Form Urlencoded
multipart/form-data     → Form Data（ファイル含む場合）
text/plain など         → Raw
```

SlackのIncoming WebhookはデフォルトでJSONだが、Stripe APIの多くは`application/x-www-form-urlencoded`を要求する。APIドキュメントの「Request Body」欄のContent-Typeを確認してから設定する。

### OpenAI Chat Completion APIをHTTP Requestノードで叩く設定例

専用のOpenAIノードがあるが、最新エンドポイント（Responses APIなど）はHTTP Requestノードで直接叩く方が柔軟なことが多い。

```text
Method: POST
URL: https://api.openai.com/v1/chat/completions
Authentication: Header Auth（Bearer sk-proj-xxxx）
Body Content Type: JSON
```

BodyのJSONはこのように設定する。


{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "user",
      "content": "{{ $json.userMessage }}"
    }
  ],
  "max_tokens": 1000
}
```

`{{ $json.userMessage }}`の部分が動的に前のノードのデータで埋まる。ここにv1.x記法を使うのがポイント。

レスポンスから回答テキストを取り出すときは`{{ $json.choices[0].message.content }}`。

---

## H2③：エラー別対処法｜401・403・429・500が出たときの原因と手順

HTTP Requestノードを使っていて最初に詰まるのが、だいたいこの4種類のエラーだ。それぞれの原因と対処を整理しておく。

### エラー別の原因と対処一覧

| ステータスコード | よくある原因 | 対処手順 |
|---|---|---|
| **401 Unauthorized** | ①Bearerの「Bearer 」スペース漏れ ②APIキー期限切れ ③Credentialの設定ミス | Credentialを再確認。Headerに`Authorization`が正しく乗っているかHTTPログで確認 |
| **403 Forbidden** | ①権限スコープ不足 ②IPホワイトリスト制限 ③プランによる機能制限 | API管理画面でスコープ・IP制限を確認。エラーレスポンスのmessageフィールドを読む |
| **429 Too Many Requests** | レート制限に引っかかっている | WaitノードかRetry設定を追加（後述） |
| **500 / 502 / 503** | API側のサーバーエラー | Retry On Failを設定。自分のリクエストに問題がないか確認してから待つ |

### 429対策：WaitノードとRetry On Failの組み合わせ

n8nコミュニティフォーラムの質問の約20〜25%がレート制限関連というデータがある。それだけ詰まる人が多いということだ。

**パターン①：SplitInBatches + Waitノード**

大量データを処理する場合はこの構成が安定する。

```text
SplitInBatches（batchSize: 1）
  ↓
HTTP Request
  ↓
Wait（1秒 or APIのレート制限に応じて調整）
  ↓
（SplitInBatchesに戻る）
```

**パターン②：Retry On Fail設定**

HTTP RequestノードのOptionsタブで設定できる。

```text
Retry On Fail: ON
Max Tries: 3
Wait Between Tries: 2000ms（2秒）
```

一時的なレート制限ならこれで自動リトライしてくれる。ただし、APIキー自体が無効な401エラーはリトライしても意味がないので、エラーコードで分岐する設計が理想的だ。

### Continue On Failでワークフロー全停止を防ぐ

デフォルトのままだと、HTTP Requestがエラーを返した時点でワークフロー全体が止まる。大量データの一括処理では致命的だ。

Optionsタブの「Continue On Fail」をONにすると、エラーが出たアイテムだけスキップして処理を継続できる。

続けて「IF」ノードで`{{ $json.error }}`の有無を判定し、エラーがあった分だけSlack通知する——という設計がプロダクション環境での定番構成になっている。

---

## H2④：Paginationと動的URL設定｜大量データを自動ループ取得する

### PaginationタブでGUIだけで複数ページ取得

2025年のアップデートでPaginationタブが追加され、手動でSplitInBatches＋ループを組む必要がなくなった。

Optionsタブ内の「Pagination」をONにすると、以下の設定が現れる。

```text
Pagination Mode:
  ① Next Page URL（レスポンス内のnext_urlを使う方式）
  ② Page Number（page=1, page=2...とインクリメント）
  ③ Offset（offset=0, offset=100...とオフセット増加）

Complete When:
  - レスポンス内の特定フィールドが空になったとき
  - 取得件数が0になったとき
  - 最大ページ数に達したとき（安全装置として設定推奨）
```

GitHub APIの例で設定するとこうなる。

```text
URL: https://api.github.com/repos/n8n-io/n8n/issues
Pagination Mode: Next Page URL
Next Page URL Path: {{ $response.headers.link }}  ← Link headerから解析
Max Pages: 10（無限ループ防止）
```

### 動的URLと動的Headerの組み立て

URLに前のノードのデータを埋め込む方法。これが使えるとワークフローの幅が一気に広がる。

```text
# URLに動的なIDを埋め込む
https://api.example.com/users/{{ $json.userId }}/orders

# 複数の値を組み合わせる
https://api.example.com/{{ $json.version }}/{{ $json.endpoint }}
```

Headerも同様に動的にできる。認証トークンを別のAPIで取得してから使うケースがある。

```text
# Authenticationノードで取得したトークンを使う
Header: Authorization
Value: Bearer {{ $('Auth HTTP Request').item.json.access_token }}
```

### よくある失敗③：タイムアウトのデフォルト値を知らない

n8nのHTTP Requestノードのデフォルトタイムアウトは**300秒（5分）**。大抵のAPIには十分だが、LLMの長い生成タスクや、大容量ファイルのアップロードでは足りなくなることがある。

Optionsタブの「Timeout」を用途に応じて調整する。逆に言えば、5分経っても応答がないAPIリクエストは大抵何かがおかしい——タイムアウト値を短くしてフェイルファストな設計にする方が健全だ。

---

## H2⑤：正直なデメリットと無料版の限界

HTTP Requestノードは便利だが、いくつか知っておくべき制限がある。

### セルフホスト版とCloud版の差（2026年時点）

2025年末時点でほぼ同等機能になったとはいえ、細かい差は残っている。

| 項目 | Cloud版 | セルフホスト版（無料） |
|---|---|---|
| 実行回数 | プランによる制限あり | 実質無制限 |
| Credentialの保管 | n8nのクラウドに保管 | 自前DBに保管 |
| SSLのHTTPS叩き放題 | ◎ | 自前証明書が必要なケースあり |
| カスタムCA証明書対応 | △ | ◎ |
| 最新機能の先行リリース | ◎（先行適用される） | 自分でアップデート要 |

Cloud版無料プランは月5,000ワークフロー実行まで。APIを高頻度で叩くユースケースだと意外と早く上限に達する。本番運用ならセルフホスト版か有料プランを検討した方がいい。

### HTTP Requestノードで正直きつい場面

- **WebSocket・Server-Sent Eventsには非対応**：リアルタイムストリームAPIはHTTP Requestノードでは扱えない。別途カスタムノードが必要
- **OAuth2の自動トークンリフレッシュはCredential依存**：Credentialで管理していれば自動更新されるが、手動実装は煩雑
- **レスポンスが10MB超えると不安定**：大容量ファイルのダウンロードは別途工夫が必要。Binary Dataとして扱うか、署名付きURLで直接ダウンロードさせる設計が現実的

---

## よくある質問

### Q1：n8n HTTP RequestノードとCode（Function）ノードのaxios、どちらを使うべきか？

基本はHTTP Requestノードを使う。理由は3つ。

1. Credential管理がGUIでできる
2. Pagination・Retry・タイムアウトがオプションで設定できる
3. ノードの出力が自動的に後続ノードに渡せる

Codeノード（axios）を使う場面は、**条件分岐を含む複雑なリクエストロジックをJavaScriptで書きたい場合**と、**動的に生成したリクエストを複数並列で投げる場合**くらいだ。

### Q2：レスポンスが配列で返ってきたとき、次のノードでどう処理するか？

HTTP RequestノードのOptionsにある「Split Into Items」をONにする。これをONにすると、レスポンスの配列が自動的にn8nのアイテムに展開される。

OFFのままだと、配列全体が1アイテムとして次のノードに渡されるため、後続のノードでループ処理ができない。大抵の場合はONにしておく方が扱いやすい。

### Q3：Self-hostedでHTTPSのAPIが叩けない（証明書エラーが出る）

社内の自己署名証明書を使っているAPIを叩くとSSLエラーになる。緊急回避として、OptionsタブのSSL certificateを「Allow Unauthorized Certs」にする方法がある。

ただし本番環境での使用は推奨しない。正しくはn8nのDockerコンテナにカスタムCA証明書をマウントする方法か、SSL終端をリバースプロキシ側で処理する設計を取る。

### Q4：HTTP Requestノードでファイルをアップロードしたりダウンロードしたりできるか？

できる。ダウンロードは「Response Format」を「File」に変更するだけ。アップロードはBody TypeをForm Dataにして、Binaryタイプのフィールドを追加する。

2025年のアップデートでBinary Data（ファイル）の送受信がノード内で完結しやすくなった。PDFや画像をAPIに送る場合もHTTP Requestノードだけで完結する。

---

## まとめ：次にやる1つのアクション

HTTP Requestノードの設定を整理するとこうなる。

- **GETの最小設定**：Method・URL・Response Formatの3つ
- **認証はCredentialタブ**：Headerに直書きしない
- **記法はv1.x系に統一**：`{{ $json.key }}`か`{{ $('ノード名').item.json.key }}`
- **429対策**：WaitノードかRetry On Failを必ず設定
- **大量データ**：PaginationタブをONにして手動ループを排除

まず**`jsonplaceholder.typicode.com/todos/1`でGETを1回動かす**ことから始めてほしい。設定欄の見え方と、レスポンスデータの参照方法が実感できる。それができたら、自分が使いたいAPIのドキュメントを開いて認証設定に進む——この順番が一番詰まらない。

取得したデータをNotionやGoogleスプレッドシートに自動保存する流れを作りたい場合は、次のステップとして「n8n Webhookでデータを受け取ってNotionに自動登録する手順」の記事が参考になる。

---

## 関連記事

- [n8n Webhook Notion 自動登録やり方｜外部フォーム送信をDBに保存する手順2026](/2026/06/09/n8n-Webhook-Notion-自動登録やり方外部フォーム送信をDBに保存する手順2026/)
- [ChatGPT API×スプレッドシート連携｜GAS初心者向け設定と自動化手順](/2026/05/22/ChatGPT-APIスプレッドシート連携GAS初心者向け設定と自動化手順/)
- [Claude APIキー取得の手順と料金｜GPT-4oコスト比較・無料枠の実態も解説](/2026/05/22/Claude-APIキー取得の手順と料金GPT-4oコスト比較無料枠の実態も解説/)

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

## 関連ツール紹介

**AIスキルを収益化するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FMMNG2+2PEO+1HLFVM" rel="nofollow">ココナラ</a>でサービスを出品できる。AIライティング・画像生成・データ分析など、AIスキルを活かした案件の需要は増えている。<img border="0" width="1" height="1" src="https://www19.a8.net/0.gif?a8mat=4B3OQW+FMMNG2+2PEO+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。
{% endraw %}
