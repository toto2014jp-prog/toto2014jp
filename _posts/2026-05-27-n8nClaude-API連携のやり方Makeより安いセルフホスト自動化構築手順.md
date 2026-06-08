---
layout: post
title: "n8n×Claude API連携のやり方｜Makeより安いセルフホスト自動化構築手順"
date: 2026-05-27 09:10:19 +0900
description: "n8nとClaude APIを連携してMakeより安く自動化を構築する手順を解説。セルフホスト月$5〜10の費用感・Anthropicノード設定・ワークフロー実例まで紹介。"
tags:
  - n8n Claude API 連携 やり方
  - n8n セルフホスト 自動化 Make 比較 コスト
  - n8n Anthropicノード 設定 方法
  - n8n Docker Compose セルフホスト 手順
---

Makeの請求額を見るたびにため息をついていないだろうか。「月10,000オペレーションで$16、でも毎月普通に超える……」という状況、正直かなりしんどい。

そこで本格的に移行してみたのがn8nのセルフホスト＋Claude API構成だ。結論から言うと、**月のツール費用がVPS代の$5〜10だけになった**。APIコストは使った分だけで、実行回数制限もない。

この記事では、セルフホスト環境の立ち上げからAnthropicノードの正しい設定、実際に動くワークフローの組み方まで、順を追って説明する。

---

## n8nとMakeのコスト比較｜セルフホストが圧倒的に安い理由

まず正直な数字を並べる。

| サービス | 月額費用 | 実行回数 |
|---|---|---|
| Make（Basic） | $9〜/月 | 10,000オペレーション |
| Make（Pro） | $16〜/月 | 10,000オペレーション |
| n8n Cloud（Starter） | $20/月 | 2,500実行 |
| **n8nセルフホスト** | **$5〜10/月（VPS代のみ）** | **実質無制限** |

Makeの「オペレーション」という課金単位がやっかいで、1つのワークフローでも複数ステップあれば一気に消費する。Claude APIで長文処理をガシガシ回すと、あっという間に上限に達する。

n8nセルフホストの場合、**n8n本体への課金はゼロ**。費用はVPS代だけ。Hetzner CX22（2vCPU/4GB RAM）なら月€3.79、日本円で600〜700円程度だ。

Claude APIのコスト感も補足しておく。claude-3-5-sonnet-20241022を使って1日100回・平均500トークンの処理をすると、**月$2〜5程度**に収まる。重い処理でもなければ、ツール費用全体で月1,000円台に落ち着くことが多い。

ちなみにn8nのGitHubスター数は2025年末時点で50,000超。「聞いたことないツール」ではもうない。

**「Makeのほうが機能が豊富」という誤解についても触れておく。** n8nのノード数は現在400超。JavaScriptコードノードを使えばカスタム処理は無制限に書ける。できることの幅はMakeを超えている部分も多い。

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

## n8nセルフホスト環境の構築手順｜VPS月$5〜10で動かすまで

「セルフホスト＝エンジニアじゃないと無理」という思い込みは、2025年時点でもう古い。Coolifyというセルフホスト管理パネルを使えば、**ノンエンジニアでも30分以内に立ち上がる**。

### 使うスタックの全体像

```
Hetzner VPS
  └── Coolify（Docker管理パネル）
        └── n8n（Docker Compose）
              └── PostgreSQL（ワークフロー・実行履歴の永続化）
Cloudflare（ドメイン・アクセス保護）
```

SQLiteではなくPostgreSQLを選ぶ理由がある。SQLiteはデフォルトで使えるが、実行履歴が増えると重くなり、データ消失リスクもある。PostgreSQLにしておけば**ワークフロー数・実行履歴が無制限**で安定する。

### 手順①｜HetznerでVPSを立ち上げる

1. [Hetzner Cloud](https://www.hetzner.com/cloud)でアカウント作成
2. 「Create Server」→ロケーションは`Nuremberg`か`Falkenstein`（安い）
3. OSは**Ubuntu 22.04 LTS**を選択
4. タイプは`CX22`（2vCPU / 4GB RAM）→ 月€3.79
5. SSHキーを登録してサーバー作成

### 手順②｜CoolifyをVPSにインストール

SSHでVPSに接続後、以下の1行を実行する。

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

終わったらブラウザで`http://[VPSのIPアドレス]:8000`を開く。初回セットアップ画面が出たら成功だ。

### 手順③｜n8nをCoolify上でデプロイ

Coolifyのダッシュボードから「New Resource」→「Docker Compose」を選び、以下の構成を貼り付ける。

```yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=yourpassword
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=yourpassword
      - WEBHOOK_URL=https://your-domain.com
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    restart: always
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=yourpassword
      - POSTGRES_DB=n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  n8n_data:
  postgres_data:
```

`yourpassword`と`your-domain.com`は自分の環境に合わせて変更する。

### 手順④｜Cloudflareでドメインを保護

独自ドメインをCloudflareのDNSで管理し、VPSのIPにAレコードを向ける。Cloudflare Zero TrustのTunnelを使えば、**VPSのポート開放なしに安全なHTTPS接続**が作れる。設定自体は無料プランで完結する。

ここまでで、ブラウザから`https://your-domain.com`にアクセスしてn8nのログイン画面が出れば環境構築は完了だ。

---

## Claude APIとn8nの連携設定手順｜Anthropicノードの正しい使い方

ここが一番ハマりやすい部分なので、よくある間違いを先に潰しておく。

**❌ よくある間違い①：HTTP RequestノードにAPIキーを直書き**

古い記事にはこういう設定が多い。セキュリティリスクがある上に、n8n v1系ではCredentials管理機能が整備されているので、直書きは今すぐやめていい。

**❌ よくある間違い②：エンドポイントに`/v1/complete`を使う**

旧Completions APIは2025年に完全非推奨となった。現行は`/v1/messages`一択だ。

**❌ よくある間違い③：モデル名を`claude-3-opus`と書く**

バージョンサフィックスまで正確に書かないと400エラーが返る。正しくは`claude-opus-4-5`や`claude-3-5-sonnet-20241022`のように末尾まで含める。

### 手順①｜AnthropicのAPIキーを取得する

[Anthropic Console](https://console.anthropic.com/)にアクセスし、「API Keys」から新しいキーを発行する。キーは一度しか表示されないのでメモ必須。

### 手順②｜n8nにAnthropicのCredentialを登録

n8nのダッシュボード左メニューから「Credentials」→「Add Credential」を開く。

検索窓に`Anthropic`と入力すると「Anthropic API」が出てくる。APIキーを貼り付けて保存。これだけだ。

### 手順③｜ワークフローにAnthropicノードを追加

ワークフロー編集画面で「+」ボタンを押し、ノード検索に`Anthropic`と入力する。「Anthropic Chat Model」または「Anthropic」ノードが表示されるので追加する。

ノードの設定項目はシンプルだ。

- **Credential**：手順②で登録したものを選択
- **Model**：`claude-3-5-sonnet-20241022`（コスパ重視）または`claude-opus-4-5`（高精度）
- **Max Tokens**：1024〜4096の範囲で用途に合わせて設定
- **System Prompt**：AIへの指示をここに書く

### 手順④｜実際に動くワークフロー例

「GmailでメールをトリガーしてClaudeに要約させ、Slackに投稿する」構成を例に取る。

```
[Gmail Trigger]
  ↓ 新着メール受信
[Anthropic ノード]
  ↓ System: "以下のメールを3行で要約してください"
  ↓ User: {{ $json.body }}
[Slack ノード]
  ↓ 要約をチャンネルに投稿
```

Anthropicノードの「User Message」欄に`{{ $json.body }}`と入力すると、前のノードから受け取ったデータをそのままClaudeに渡せる。n8nの式エディタはMakeより直感的で、慣れると速い。

---

## 本番運用で詰まるポイントと対処法

動いた、で終わりにすると後で痛い目を見る。運用段階で当たりやすい問題を3つ挙げておく。

**レート制限エラー（429）への対処**

Claude APIには1分あたりのリクエスト数制限がある。大量処理をするワークフローでは、n8nの「Wait」ノードを挟んで1〜2秒の間隔を入れるだけで解消することが多い。

**モデル名エラー（400）の確認方法**

エラーログに`model_not_found`が出たらモデル名のスペルミスを疑う。Anthropic公式の[モデル一覧ページ](https://docs.anthropic.com/en/docs/about-claude/models)で現行モデル名を確認してそのままコピーする習慣をつける。

**n8nのワークフロー実行履歴の肥大化対策**

セルフホストはデータが溜まり続ける。`Settings` → `Log Level`を`warn`に下げ、不要な実行ログは定期的に削除するか、`EXECUTIONS_DATA_MAX_AGE`環境変数で自動削除期間を設定する（例：`30`日）。

```yaml
# docker-compose.ymlのn8nサービスに追加
environment:
  - EXECUTIONS_DATA_MAX_AGE=30
  - EXECUTIONS_DATA_PRUNE=true
```

この2行を追加するだけで、30日以上前の実行履歴が自動で消える。

---

## まとめ：次にやるべき1つのアクション

n8n×Claude API構成の魅力を一言で言うと、「**月1,000円台で、実行回数を気にせず自動化できる環境**」だ。Makeの月額を払い続けるより、1〜2ヶ月で乗り換えコストは回収できる。

今すぐHetznerのアカウントを作り、CX22サーバーを1台立ててほしい。Coolifyのインストールまで含めて30分もかからない。環境さえ立ち上がれば、あとはノードを繋ぐだけだ。

---

*著者：AI実践ラボ編集部*

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

- [Make×ChatGPT連携のやり方｜Zapierより安い自動化設定手順](/2026/05/24/MakeChatGPT連携のやり方Zapierより安い自動化設定手順/)
- [Claude API料金を最大90%削減する方法｜トークン節約・モデル選択・キャッシュ設定](/2026/05/25/Claude-API料金を最大90削減する方法トークン節約モデル選択キャッシュ設定/)
- [Claude API×GASでスプレッドシート自動化｜2025年版コピペで動く設定手順](/2026/05/26/Claude-APIGASでスプレッドシート自動化2025年版コピペで動く設定手順/)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。