---
layout: post
title: "Claude API Notion自動保存やり方｜API応答をDBに書き込む手順2026"
date: 2026-06-18 09:06:07 +0900
description: "Claude APIの応答をNotionデータベースに自動保存する実装手順を解説。2,000文字制限・Markdown変換・インテグレーション招待漏れなど詰まりポイントを先に潰します。"
tags:
  - Claude API Notion 自動保存 やり方
  - Claude API Notion データベース 書き込み 手順
  - Claude API GAS Notion 連携 自動化
  - Notion API リッチテキスト 2000文字 制限 対策
---

## Claude APIの応答、Notionにうまく保存できていますか？

Claude APIを叩いて応答が返ってきた。あとはNotionに保存するだけ——そう思ったら、403エラー、文字化け、途中で切れたテキスト。「なんで動かないんだ」と30分以上溶かした経験がある人、正直に言うと自分もそのひとりだった。

この記事を読むと、**Claude APIの応答をNotionデータベースに確実に書き込むための実装コード**と、詰まるポイントをぜんぶ先に潰せる。PythonとGoogle Apps Script（GAS）両方の実装例を載せているので、自分の環境に合わせて使ってほしい。

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

## 先に結論を出しておく

Claude API→Notion自動保存が失敗する原因は、ほぼ3つに絞られる。

- **リッチテキストの2,000文字上限**を無視して丸ごと渡している
- **MarkdownをそのままNotion APIに渡している**（Notionはプレーンテキストかリッチテキスト形式しか受け付けない）
- **インテグレーションをデータベースに招待していない**（これが一番多い）

この3つを対策するだけで、8割の問題は解決する。順番に手順を説明していく。

---

## H2①：うまくいかない3大原因と対策

### 原因1：リッチテキスト2,000文字上限エラー

Notion APIの仕様として、リッチテキスト1要素あたりの文字数上限は**2,000文字**に固定されている。Claude 3.7は最大200,000 tokensの出力ができる。つまり、長文の応答をそのままNotionに渡せば、必ずエラーになる。

エラーメッセージは `body.children[0].paragraph.rich_text[0].text.content.length should be ≤ 2000` のような形式で返ってくる。初見だと何のことかわからず詰まる。

対策はシンプルで、**書き込む前に1,900文字ずつ分割**する。2,000文字ちょうどではなく1,900文字にしているのは、マルチバイト文字のカウントでズレが生じるときのバッファ。

### 原因2：MarkdownをそのままNotionに渡している

Claude APIの応答はMarkdown形式で返ってくることが多い。`**太字**`、`## 見出し`、`` `コード` `` といった記法がそのまま文字列として含まれる。

NotionのブロックAPIはこれをそのまま受け付けない。`**太字**`は太字として表示されず、アスタリスクが文字通り表示される。

対策は2つ。
- Markdown→Notionブロック変換ライブラリを使う（Pythonなら`md2notion`や`notion-client`の拡張）
- あるいはMarkdown記法を一切使わずプレーンテキストとして保存する割り切り

正直なところ、変換ライブラリは完全に対応できているわけではない。見出しや箇条書きの変換精度は7〜8割程度の印象。完璧な変換を求めるよりも、**段落テキストとしてフラットに保存する設計**にした方が安定する。

### 原因3：インテグレーション招待漏れで403エラー

Notionのインテグレーション（APIキーを発行する仕組み）は、**ワークスペース全体に権限があるわけではない**。対象のデータベースページに、明示的にインテグレーションを招待する操作が必要になる。

手順はこう。
1. 対象のNotionデータベースページを開く
2. 右上の「・・・」→「コネクト」→作成したインテグレーション名を選択
3. 「アクセスを許可」をクリック

これをやっていないと、正しいAPIキーを使っていても`401 Unauthorized`や`404 Not Found`が返ってくる。APIキーを疑う前に、まずここを確認してほしい。

### おまけ：旧エンドポイントはもう使えない

2024年末に`/v1/complete`は廃止済み。今から実装するなら**`/v1/messages`一択**。古いチュートリアルやQiita記事を参考にしているとここで詰まる。

---

## H2②：事前準備｜APIキー取得・DB設計・インテグレーション設定

### Notionインテグレーションの作成手順

1. [https://www.notion.so/my-integrations](https://www.notion.so/my-integrations) にアクセス
2. 「新しいインテグレーション」をクリック
3. 名前（例：`claude-auto-save`）を入力し、ワークスペースを選択
4. 「送信」→「Internal Integration Token」をコピー

このトークンが後述のコードで使う`NOTION_TOKEN`になる。**絶対にGitHubにコミットしないこと**。`.env`ファイルか環境変数で管理する。

### データベースのプロパティ設計（先に決めておく）

あとからプロパティを変えると、既存レコードへの影響や型変換で手間が増える。最初にこの構成で作っておくことを推奨する。

| プロパティ名 | Notionの型 | 用途 |
|---|---|---|
| タイトル | タイトル | Claude応答の冒頭50文字程度 |
| 実行日時 | 日付 | APIを叩いた日時 |
| 使用モデル | セレクト | claude-3-5-sonnet等 |
| プロンプト概要 | テキスト | 送ったプロンプトの要約 |
| 本文 | テキスト | Claude応答全文 |
| ステータス | セレクト | 保存済み・確認待ち等 |

### データベースIDの取得方法

NotionデータベースのURLはこのような形式になっている。

```
https://www.notion.so/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx?v=yyyyyyyy
```

`notion.so/` の直後の32文字（ハイフン区切りの場合もある）が**データベースID**。`?v=`以降は不要。

URLが`https://www.notion.so/myworkspace/xxxxxxxx`のような形式の場合は、`myworkspace/`の後ろがID。

---

## H2③：実装手順｜Python編（基本実装）

### 必要なライブラリのインストール

```bash
pip install anthropic notion-client python-dotenv
```

### .envファイルの準備

```text
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxx
NOTION_TOKEN=secret_xxxxxxxxxxxx
NOTION_DATABASE_ID=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 実際に動くコード（Python）

```python
import os
import anthropic
from notion_client import Client
from datetime import datetime
from dotenv import load_dotenv

load_dotenv()

# クライアントの初期化
claude = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
notion = Client(auth=os.environ["NOTION_TOKEN"])
DATABASE_ID = os.environ["NOTION_DATABASE_ID"]


def split_text(text: str, chunk_size: int = 1900) -> list[str]:
    """Notion 2,000文字制限に対応した分割処理"""
    return [text[i:i+chunk_size] for i in range(0, len(text), chunk_size)]


def text_to_notion_blocks(text: str) -> list[dict]:
    """テキストをNotionの段落ブロックリストに変換"""
    chunks = split_text(text)
    blocks = []
    for chunk in chunks:
        blocks.append({
            "object": "block",
            "type": "paragraph",
            "paragraph": {
                "rich_text": [{
                    "type": "text",
                    "text": {"content": chunk}
                }]
            }
        })
    return blocks


def ask_claude(prompt: str, model: str = "claude-3-5-sonnet-20241022") -> str:
    """Claude APIを叩いて応答テキストを返す"""
    message = claude.messages.create(
        model=model,
        max_tokens=4096,
        messages=[
            {"role": "user", "content": prompt}
        ]
    )
    return message.content[0].text


def save_to_notion(prompt: str, response: str, model: str) -> str:
    """Claude応答をNotionデータベースに保存してページIDを返す"""
    
    # タイトルは応答の冒頭50文字
    title = response[:50].replace("\n", " ")
    if len(response) > 50:
        title += "..."
    
    # ページ（レコード）を作成
    new_page = notion.pages.create(
        parent={"database_id": DATABASE_ID},
        properties={
            "タイトル": {
                "title": [{
                    "text": {"content": title}
                }]
            },
            "実行日時": {
                "date": {"start": datetime.now().isoformat()}
            },
            "使用モデル": {
                "select": {"name": model}
            },
            "プロンプト概要": {
                "rich_text": [{
                    "text": {"content": prompt[:200]}  # 200文字に絞る
                }]
            }
        }
    )
    
    page_id = new_page["id"]
    
    # 本文ブロックを追記（100ブロック上限に注意）
    blocks = text_to_notion_blocks(response)
    
    # 100ブロックずつに分けて追記
    for i in range(0, len(blocks), 100):
        chunk_blocks = blocks[i:i+100]
        notion.blocks.children.append(
            block_id=page_id,
            children=chunk_blocks
        )
    
    return page_id


def main():
    prompt = "Pythonでシングルトンパターンを実装する方法を教えてください"
    model = "claude-3-5-sonnet-20241022"
    
    print("Claude APIに問い合わせ中...")
    response = ask_claude(prompt, model)
    print(f"応答取得完了（{len(response)}文字）")
    
    print("Notionに保存中...")
    page_id = save_to_notion(prompt, response, model)
    print(f"保存完了！ページID: {page_id}")


if __name__ == "__main__":
    main()
```

このコードで気をつけているのは2点。`split_text`で1,900文字ずつ分割していること、そして100ブロックずつまとめて`append`していること。Notion APIは1リクエストあたり最大100ブロックの制限がある（2025年以降は統一されて安定している）。

---

## H2④：GAS（Google Apps Script）で実装する場合

サーバーを立てたくない、Googleアカウントだけで完結させたい——そういう要件ならGASで実装できる。定期実行（時間ドリブン）との組み合わせも簡単。

### GASの実装コード

```javascript
// Google Apps Script（GAS）版
// スクリプトのプロパティにAPIキーを設定しておくこと

const ANTHROPIC_API_KEY = PropertiesService.getScriptProperties().getProperty('ANTHROPIC_API_KEY');
const NOTION_TOKEN = PropertiesService.getScriptProperties().getProperty('NOTION_TOKEN');
const NOTION_DATABASE_ID = PropertiesService.getScriptProperties().getProperty('NOTION_DATABASE_ID');

/**
 * テキストを1900文字ずつ分割してNotionブロック配列を生成
 */
function textToNotionBlocks(text) {
  const chunkSize = 1900;
  const blocks = [];
  
  for (let i = 0; i < text.length; i += chunkSize) {
    const chunk = text.substring(i, i + chunkSize);
    blocks.push({
      object: 'block',
      type: 'paragraph',
      paragraph: {
        rich_text: [{
          type: 'text',
          text: { content: chunk }
        }]
      }
    });
  }
  
  return blocks;
}

/**
 * Claude APIを叩いて応答テキストを取得
 */
function askClaude(prompt) {
  const url = 'https://api.anthropic.com/v1/messages';
  
  const payload = {
    model: 'claude-3-5-sonnet-20241022',
    max_tokens: 4096,
    messages: [
      { role: 'user', content: prompt }
    ]
  };
  
  const options = {
    method: 'POST',
    headers: {
      'x-api-key': ANTHROPIC_API_KEY,
      'anthropic-version': '2023-06-01',
      'content-type': 'application/json'
    },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };
  
  const response = UrlFetchApp.fetch(url, options);
  const data = JSON.parse(response.getContentText());
  
  if (response.getResponseCode() !== 200) {
    throw new Error(`Claude APIエラー: ${JSON.stringify(data)}`);
  }
  
  return data.content[0].text;
}

/**
 * NotionデータベースにページとしてClaude応答を保存
 */
function saveToNotion(prompt, responseText) {
  const title = responseText.substring(0, 50).replace(/\n/g, ' ') + 
                (responseText.length > 50 ? '...' : '');
  
  // ページ作成
  const createUrl = 'https://api.notion.com/v1/pages';
  const createPayload = {
    parent: { database_id: NOTION_DATABASE_ID },
    properties: {
      'タイトル': {
        title: [{ text: { content: title } }]
      },
      '実行日時': {
        date: { start: new Date().toISOString() }
      },
      'プロンプト概要': {
        rich_text: [{ text: { content: prompt.substring(0, 200) } }]
      }
    }
  };
  
  const createOptions = {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${NOTION_TOKEN}`,
      'Notion-Version': '2022-06-28',
      'Content-Type': 'application/json'
    },
    payload: JSON.stringify(createPayload),
    muteHttpExceptions: true
  };
  
  const createResponse = UrlFetchApp.fetch(createUrl, createOptions);
  const pageData = JSON.parse(createResponse.getContentText());
  const pageId = pageData.id;
  
  // 本文ブロックを追記（100ブロック単位で分割）
  const blocks = textToNotionBlocks(responseText);
  const appendUrl = `https://api.notion.com/v1/blocks/${pageId}/children`;
  
  for (let i = 0; i < blocks.length; i += 100) {
    const chunkBlocks = blocks.slice(i, i + 100);
    const appendOptions = {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${NOTION_TOKEN}`,
        'Notion-Version': '2022-06-28',
        'Content-Type': 'application/json'
      },
      payload: JSON.stringify({ children: chunkBlocks }),
      muteHttpExceptions: true
    };
    UrlFetchApp.fetch(appendUrl, appendOptions);
    
    // レート制限対策：100msウェイト
    if (i + 100 < blocks.length) {
      Utilities.sleep(100);
    }
  }
  
  return pageId;
}

/**
 * メイン実行関数（時間ドリブントリガーに設定可能）
 */
function main() {
  const prompt = 'JavaScriptのasync/awaitを初心者向けに説明してください';
  
  Logger.log('Claude APIに問い合わせ中...');
  const response = askClaude(prompt);
  Logger.log(`応答取得: ${response.length}文字`);
  
  const pageId = saveToNotion(prompt, response);
  Logger.log(`Notion保存完了: ${pageId}`);
}
```

GASで注意が必要なのは`UrlFetchApp.fetch`のタイムアウト。デフォルトは60秒で、Claude APIの長文生成では超えることがある。長い出力が予想される場合は`max_tokens`を絞るか、応答を短くするよう指示するプロンプト設計が現実的な対応になる。

---

## H2⑤：Python vs GAS 比較と使い分け

| 観点 | Python実装 | GAS実装 |
|---|---|---|
| 実行環境 | ローカルorサーバー必要 | Googleアカウントのみ |
| タイムアウト | 自分で設定可能（推奨120秒） | 最大6分（Apps Scriptの制限） |
| 定期実行 | cronやCloud Schedulerが必要 | トリガー設定で完結 |
| 無料運用 | VPS費用が発生 | 無料（一定量まで） |
| デバッグのしやすさ | ローカルで容易 | Logger.logで確認、やや不便 |
| Markdown変換 | ライブラリが豊富 | 自前実装が必要 |
| 向いているケース | バッチ処理・大量保存 | 軽量な定期実行・個人利用 |

正直なところ、**個人利用や小規模な自動化はGASで十分**。サーバー管理が不要で、Googleカレンダーや他のGoogleサービスとの連携も簡単。一方、1日に100件以上の保存が必要だったり、Notionへのレスポンスを加工して別のサービスにも流したりするならPythonの方が柔軟性が高い。

---

## よくある質問

### Q1：Notion API v2（2025年〜）に対応させる必要がある？

Notion API v2は2025年中頃から段階リリースされた。`Notion-Version`ヘッダーに`2022-06-28`を指定している場合は既存の動作が継続される。新規実装なら最新バージョンを指定した方が今後のサポート面で安心。ただし、リッチテキストの仕様変更があるため、既存コードをそのまま移行すると動かないプロパティが出ることがある。移行前に公式Changelogを確認することを推奨する。

### Q2：ストリーミング応答をリアルタイムでNotionに保存できる？

できなくはないが、設計がやや複雑になる。ストリーミングで受け取りながら逐次ブロックをappendしていく方法が技術的には可能。ただし、Notionのレート制限（最大10 req/秒）に引っかかりやすい。現実的には、**ストリーミングで全文受信してから一括書き込み**する設計の方がシンプルで安定する。

### Q3：無料プランのNotion APIで使う場合の制限は？

無料プランでもNotion APIは使える。ただしレート制限は有料プランより厳しく、**3 req/秒**が目安。大量に書き込む場合は`time.sleep(0.5)`などのウェイトを入れないとスロットリングエラーが出る。個人利用の範囲なら無料プランで十分な場面がほとんど。

### Q4：既存のNotionページを上書きしたい場合はどうする？

Notion APIはブロックの上書きができない仕様になっている。既存ブロックを削除して再作成するか、ブロックをappendして追記するかの2択。実務では「削除→再作成」よりも、**ステータスプロパティで管理して新規ページとして追加する**設計が扱いやすい。

---

## 注意点：コスト感覚を持っておく

Claude 3.5 Sonnetの料金目安（2025年時点）はinput $3/MTok、output $15/MTokになっている。1,000文字（≒750トークン）の応答を1日100回生成すると、output側だけで月に約$3.4程度。少額に見えるが、バグでループしてしまうと一気に積み上がる。

実装時は必ず`max_tokens`を適切に絞り、テスト中は短い応答を返すプロンプト（「一行で答えてください」など）を使うこと。これ、意外と見落とされるポイント。

---

## まとめ：今日やるべき1つのアクション

Claude API×Notion自動保存の落とし穴は「2,000文字分割」「Markdown変換」「インテグレーション招待」の3点に集約される。この記事のコードはそのまま動く状態で書いているので、まず`.env`ファイルを作ってAPIキーを設定し、Pythonコードの`main()`を実行してみてほしい。

最初の保存が成功したら、次はプロンプトをスプレッドシートから読み込んで複数件を一括処理する仕組みに発展させると、実用的な自動化ループが完成する。GASとスプレッドシートを組み合わせた応用については、Claude API GASスプレッドシート連携の記事も参考にしてほしい。

---

## 関連記事

- [Notion GAS連携やり方｜スプレッドシートをNotionDBに自動同期する手順](/2026/05/28/Notion-GAS連携やり方スプレッドシートをNotionDBに自動同期する手順/)
- [Perplexity AI×Notion連携やり方｜競合調査結果をDBに自動保存する手順](/2026/05/28/Perplexity-AINotion連携やり方競合調査結果をDBに自動保存する手順/)
- [Claude API×Notion連携｜無料で始めるAI文章自動保存の3つのやり方](/2026/05/31/Claude-APINotion連携やり方AIライティング結果をDBに自動保存する手順/)

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