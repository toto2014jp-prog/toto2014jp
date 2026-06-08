---
layout: post
title: "Claude API×Notion連携やり方｜AIライティング結果をDBに自動保存する手順"
date: 2026-05-31 09:06:04 +0900
description: "Claude APIで生成したライティング結果をNotionデータベースに自動保存する手順を解説。コード直書き・MCP・ノーコードの3アプローチを比較し、自分に合う方法をすぐ選べます。"
tags:
  - Claude API Notion 連携 やり方
  - Claude API Notion データベース 自動保存 方法
  - Anthropic API Notion 自動化 設定手順
  - Claude API ライティング結果 Notion 保存 Python
---

「ClaudeでAIライティングしたはいいけど、結果がチャット画面に散らばって管理できない」——これ、けっこう多い悩みだ。

Notionにコピペして保存するのが面倒で、そのまま放置。気づいたら同じテーマで何度もClaude に聞き直している、という無限ループ。

この記事では、**Claude APIのアウトプットをNotionデータベースに自動保存する手順**を3つのアプローチで解説する。コーディングが得意な人も、「コードは触りたくない」という人も、自分に合うルートをそのまま実装できる構成にした。

---

## Claude APIとNotion AIの違い——「わざわざ連携する理由」を先に整理する

ぶっちゃけ、Notionにはすでにノート上でAIが使える「Notion AI」が標準搭載されている。GPT-4ベースで、ページ内でそのまま文章生成できる。

なのに、なぜClaude APIをわざわざ外部連携するのか。実際に両方使ってみて感じた差を正直に書く。

| 比較軸 | Notion AI（標準） | Claude API（外部連携） |
|---|---|---|
| コンテキスト長 | 数千トークン程度 | **最大200Kトークン**（Claude 3.5/3.7 Sonnet） |
| プロンプトの自由度 | テンプレ固定 | **完全カスタム** |
| 出力先 | そのページのみ | **複数DBへ同時書き込み可** |
| 月額コスト | Notion AIオプション月$10〜 | **従量課金（1記事$0.05〜0.15程度）** |
| 自動化（スケジュール実行） | 非対応 | **対応可** |

Notion AIで十分なのは「その場で文章を整えたい」というシーン。一方、Claude API連携が本領を発揮するのは次のようなケースだ。

- 毎日決まったテーマで記事下書きを自動生成し、Notionに蓄積したい
- 競合調査や市場リサーチのサマリーを自動でDBに溜め込みたい
- 出力結果にタグ・日付・ステータスをセットで付けて管理したい

要するに、**「生成→保存→蓄積→検索」というワークフローを自動化したい人向け**の話だ。

### 3アプローチの早見表

この記事では以下の3つの方法を扱う。先に自分がどれに当てはまるか確認してほしい。

| アプローチ | 難易度 | コスト | 向いている人 |
|---|---|---|---|
| **①Pythonコード直書き** | ★★★ | ほぼ無料 | コードが書ける／学びたい人 |
| **②MCP（Model Context Protocol）** | ★★ | ほぼ無料 | Claude Desktopユーザー、設定ファイル編集ができる人 |
| **③ノーコード（Make/n8n）** | ★ | Make無料枠あり | コードを触りたくない人、業務フローに組み込みたい人 |

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

## 前提準備——APIキー取得とNotionデータベースの設計

どのアプローチでも共通の準備が2つある。ここを雑にやると後でハマるので、丁寧に進める。

### Claude APIキーの取得（2025年以降の注意点）

**これ、意外と知られてないんですが**、2025年以降はClaude APIの利用開始に**クレジットカード登録＋初回チャージ（$5〜）が必須**になった。昔は無料枠で試せたが、現在は審査制に変わっている。

手順はこうだ。

1. [Anthropic Console](https://console.anthropic.com/) にアクセスしてアカウント作成
2. 「Billing」メニューからクレジットカードを登録
3. $5以上のクレジットをチャージ
4. 「API Keys」メニューから新しいキーを発行
5. 発行されたキーをコピー（**この画面を閉じると二度と見られない**ので注意）

料金の目安として、Claude 3.5 Sonnetは入力$3・出力$15（1Mトークンあたり）。3,000字の記事1本あたり$0.05〜$0.15程度なので、月100本生成しても$5〜$15の計算だ。

### NotionのDB設計とインテグレーション設定

ここで**最初に型を確定させる**のが最重要ポイントだ。後からプロパティ型を変えると、APIのpayload構造も全部書き直しになる。

推奨するDBのプロパティ構成はこれ。

- **title**（タイトル型）：記事タイトル
- **summary**（テキスト型）：100字サマリー
- **tags**（マルチセレクト型）：カテゴリタグ
- **created_at**（日付型）：生成日時
- **status**（セレクト型）：下書き／確認中／完了

インテグレーションの接続手順はこうなる。

1. [Notion Developers](https://www.notion.so/my-integrations) を開く
2. 「New integration」でインテグレーション作成→Internal typeを選択
3. 発行された**Internal Integration Token**をコピー
4. 保存先のNotionデータベースを開き、右上「…」→「Connections」→作成したインテグレーションを追加
5. DBのURLから**Database ID**（32桁の英数字）を控えておく

---

## 【メイン手順】3アプローチ別の実装方法

### アプローチ①：Pythonコードで直接連携する

コードを書ける人には、これが一番自由度が高い。動作も安定している。

**必要なライブラリをインストール：**

```bash
pip install anthropic requests
```

**実装コードの全体像：**

```python
import anthropic
import requests
import json
from datetime import datetime

CLAUDE_API_KEY = "sk-ant-xxxxxxxx"  # Anthropic ConsoleのAPIキー
NOTION_TOKEN = "secret_xxxxxxxx"    # NotionのIntegration Token
NOTION_DB_ID = "xxxxxxxxxxxxxxxx"   # NotionのDatabase ID

def generate_article_with_claude(theme: str) -> dict:
    client = anthropic.Anthropic(api_key=CLAUDE_API_KEY)

    prompt = f"""
以下のテーマで記事を生成し、必ずJSON形式のみで返してください。
テーマ：{theme}

返すJSONの形式：
{{"title": "記事タイトル", "summary": "100字以内の要約", "tags": ["タグ1", "タグ2"], "body": "本文（マークダウン形式）"}}

JSON以外のテキストは一切含めないでください。
"""

    message = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=4096,
        messages=[{"role": "user", "content": prompt}]
    )

    # JSON部分だけを取り出してパース
    response_text = message.content[0].text
    return json.loads(response_text)

def save_to_notion(article: dict) -> str:
    url = f"https://api.notion.com/v1/pages"
    headers = {
        "Authorization": f"Bearer {NOTION_TOKEN}",
        "Content-Type": "application/json",     # 省略するとエラー
        "Notion-Version": "2022-06-28"           # このヘッダーも必須
    }

    # propertiesとchildrenは分けて書く（混同がエラーの原因）
    payload = {
        "parent": {"database_id": NOTION_DB_ID},
        "properties": {
            "title": {
                "title": [{"type": "text", "text": {"content": article["title"]}}]
            },
            "summary": {
                "rich_text": [{"type": "text", "text": {"content": article["summary"]}}]
            },
            "tags": {
                "multi_select": [{"name": tag} for tag in article["tags"]]
            },
            "created_at": {
                "date": {"start": datetime.now().strftime("%Y-%m-%d")}
            },
            "status": {
                "select": {"name": "下書き"}
            }
        },
        "children": [
            {
                "object": "block",
                "type": "paragraph",
                "paragraph": {
                    "rich_text": [{"type": "text", "text": {"content": article["body"][:2000]}}]
                }
            }
        ]
    }

    response = requests.post(url, headers=headers, json=payload)
    response.raise_for_status()
    return response.json()["url"]

# 実行
article = generate_article_with_claude("Python初心者向けの学習ロードマップ")
page_url = save_to_notion(article)
print(f"Notionに保存完了：{page_url}")
```

ここで**絶対に踏んではいけない地雷**が2つある。

**地雷①：rich_textに文字列を直渡しする**
```python
# ❌ これは400エラーになる
"rich_text": "テキスト内容"

# ✅ オブジェクト配列形式が必須
"rich_text": [{"type": "text", "text": {"content": "テキスト内容"}}]
```

**地雷②：Notion-Versionヘッダーを省略する**
このヘッダーがないと、Notion APIは正常なリクエストでも403を返す。必ず`"Notion-Version": "2022-06-28"`を含めること。

---

### アプローチ②：MCPでClaude Desktopから直接Notionに書き込む

MCP（Model Context Protocol）はAnthropicが2024年末に公開した仕組みで、ClaudeがNotionなど外部ツールを直接操作できるようになる。

正直なところ、「コードレスで使える」と思って始めたら想定より設定が必要だった。ただ、一度動けば最も直感的に使えるアプローチだ。

**前提環境：**
- Claude Desktop（Anthropic公式デスクトップアプリ）がインストール済み
- Node.js 18以上がインストール済み

**設定手順：**

1. Claude Desktopの設定ファイルを開く
   - Mac：`~/Library/Application Support/Claude/claude_desktop_config.json`
   - Windows：`%APPDATA%\Claude\claude_desktop_config.json`

2. 以下の内容を追記する


{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "OPENAPI_MCP_HEADERS": "{\"Authorization\": \"Bearer secret_xxxxxxxx\", \"Notion-Version\": \"2022-06-28\"}"
      }
    }
  }
}
```

3. Claude Desktopを再起動
4. チャット画面にハンマーアイコン（🔨）が表示されればMCPが有効化されている

**実際の使い方：**

Claude Desktopのチャットでこう打つだけで、Notionへの書き込みが完結する。

```
「Python初心者向けの学習ロードマップ」というテーマで
1,500字程度の記事を書いて、
Notionデータベース（ID: xxxxxxxx）に
タイトル・サマリー・本文として保存してください。
タグは「Python」「学習」「初心者」を設定してください。
```

MCPの注意点として、**Node.js環境がない場合は動かない**。`node -v`でバージョンを確認してから進めること。

---

### アプローチ③：Make（旧Integromat）でノーコード連携する

コードを一切触りたくない場合、Makeが現実的な選択肢だ。2025年時点でClaude公式ノードが追加されており、GUIだけで設定できる。

**Makeでのシナリオ構成：**

```
[Schedule（定期実行）]
    ↓
[Anthropic Claude：Generate Message]
    ↓
[JSON：Parse JSON]（構造化出力を分解）
    ↓
[Notion：Create a Database Item]
```

**ステップごとの設定ポイント：**

**Step1：Anthropic Claudeノードの設定**
- Model：`claude-3-5-sonnet-20241022`を選択
- Max Tokens：4096
- Message欄に構造化JSONを要求するプロンプトを入力（前述のプロンプト例と同じ形式）

**Step2：JSONパースノードの設定**
- ClaudeのOutputを受け取り、`title` `summary` `tags` `body`に分解
- Data Structureは事前にスキーマを定義しておく

**Step3：Notionノードの設定**
- Connection：NotionのIntegration Tokenで認証
- Database ID：保存先DBのIDを入力
- 各プロパティフィールドにStep2のパース結果をマッピング

Makeの無料プランは月1,000オペレーション。記事1本の生成→保存で約3〜4オペレーションを消費するので、月250本程度まで無料で動かせる計算だ。

**n8nを使う場合**も基本構成は同じで、「Anthropic Chat Model」ノード→「Code」ノード（JSONパース）→「Notion」ノードの流れになる。n8nはセルフホスト版なら無制限に使えるのが強み。

---

## よくあるエラーと対処法

実装時に詰まりやすいポイントを3つ絞って書く。

**エラー①：`400 Bad Request`（Notion API）**
- 原因の9割は`rich_text`の形式ミス
- 文字列を直渡しせず、必ずオブジェクト配列形式で渡す
- `Notion-Version`ヘッダーの有無も確認する

**エラー②：ClaudeがJSONを返さない**
- プロンプトに「JSON以外のテキストは一切含めないでください」と明示する
- それでも崩れる場合は、レスポンスから`{`〜`}`を正規表現で抽出するフォールバック処理を追加する

**エラー③：Notionへの書き込みが一部失敗する**
- Notion APIのレート制限は**3リクエスト/秒**。複数ページを連続保存する場合は`time.sleep(0.4)`を挟む
- 本文が2,000字を超える場合は複数のparagraphブロックに分割して渡す

---

## まとめ——次にやるべき1つのアクション

3つのアプローチを紹介したが、迷ったら**アプローチ①（Pythonコード）から始める**のを勧める。

理由はシンプルで、動作が一番透明だからだ。エラーが出たときに原因を特定しやすく、後からMakeやMCPに移行するときの理解ベースにもなる。

**今日やるべき1アクションはこれ。**

> AnthropicのコンソールでAPIキーを発行し、Notionにテスト用DBを1つ作る。それだけでいい。コードは後から書ける。

APIキー発行→DB設計→コード実装、この順番で進めれば最短1時間で動作確認まで到達できる。

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

- [Perplexity AI×Notion連携やり方｜競合調査結果をDBに自動保存する手順](/2026/05/28/Perplexity-AINotion連携やり方競合調査結果をDBに自動保存する手順/)
- [Notion AIとChatGPT・Perplexityを併用する方法｜リサーチ→まとめ→DB保存を全自動化する手順](/2026/05/30/Notion-AIとChatGPTPerplexityを併用する方法リサーチまとめDB保存を全自動化する手順/)
- [Claude API×GASでスプレッドシート自動化｜2025年版コピペで動く設定手順](/2026/05/26/Claude-APIGASでスプレッドシート自動化2025年版コピペで動く設定手順/)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。