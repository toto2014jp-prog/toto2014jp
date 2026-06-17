---
layout: post
title: "Claude API Python入門｜キー取得からテキスト生成・レスポンス受け取り手順2026"
date: 2026-06-17 09:05:54 +0900
description: "Claude APIをPythonで使う2026年最新手順。キー取得・SDK導入・Messages APIでのテキスト生成まで、コピペ動作コード付きで解説。旧コードの落とし穴も明示。"
tags:
  - Claude API Python 使い方 入門
  - Claude API キー取得 Python 連携 手順
  - Claude API Python テキスト生成 始め方 2026
  - anthropic SDK Messages API 実装方法
---

## Claude APIをPythonで使おうとして、古いコードに引っかかっていないか

ネットで「Claude API Python」を検索すると、今でも`anthropic.Client()`や`client.completions.create()`を使った記事がヒットする。正直なところ、これらは2026年時点では動かないか、非推奨になっているコードだ。

この記事では、**APIキー取得 → SDK導入 → テキスト生成 → レスポンス受け取り**までを、今すぐ動くコードで解説する。OpenAI APIから乗り換えようとして詰まった人にも、ハマりポイントを先に伝えておく。

---



> 💡 **関連教材**: [ChatGPT＆Claude AIプロンプト集50選（¥980）](https://aijissenlab.gumroad.com/l/yjcdam) — コピペで即使える実践プロンプト50種を全24ページに凝縮

## 結論：2026年のClaude API Python連携はこの3点を押さえるだけ

- クライアントは`anthropic.Anthropic()`（旧`Client()`は廃止）
- テキスト生成は`client.messages.create()`（旧`completions`は廃止）
- APIキーは`.env`ファイルで管理（コードに直書きは絶対NG）

この3点さえ知っていれば、最短15分でテキスト生成まで動かせる。

---

## H2① Claude APIのアカウント登録とAPIキー取得手順

### Anthropic Consoleにアクセスする

まず[console.anthropic.com](https://console.anthropic.com)にアクセスする。Googleアカウントでのサインアップが最も早い。メールアドレス＋パスワードでの登録も選べる。

### 無料プランはない。でも最初の壁は低い

はっきり言っておく。**AnthropicのAPIに無料プランは存在しない**（2026年時点）。ただし、クレジットカードを登録すると**初回約$5分の無料クレジット**が付与される。入門レベルの使い方なら、これで数百〜数千回のAPI呼び出しができる。

支払いが怖い人向けに計算しておくと、`claude-haiku-3.5`で100トークン程度の短文生成を1,000回やっても、費用はほぼ数十セント程度。試用段階では事実上ほぼ無料で動かせる。

### APIキーを発行する

コンソールにログイン後、左メニューの「API Keys」を開く。「Create Key」ボタンをクリックし、名前をつけて発行する。

⚠️ **ここが重要**：発行直後にしか全文表示されない。画面を閉じると二度と見られないので、必ずコピーしてどこかに控えておく。

### WorkspacesとProjectsの概念（チーム利用時）

2025年以降、Anthropic ConsoleにはWorkspaces/Projectsという概念が導入された。個人利用なら「Default Workspace」のAPIキーをそのまま使えばいい。チームで開発する場合は、プロジェクトごとにAPIキーを分けてコスト管理できる。今は気にしなくていいが、頭の片隅に置いておくといい。

---

## H2② Python環境の準備とSDKの正しいインストール

### SDKのインストール

```bash
pip install anthropic>=0.30.0
```

インストールできたか確認する。

```bash
pip show anthropic
```

`Version: 0.3x.x`以上が表示されれば問題ない。

### APIキーを`.env`で安全に管理する

APIキーをコードに直書きしているサンプルが今でもネットに溢れているが、**GitHubに上げた瞬間にキーが漏洩する**。最低限、`.env`ファイルで管理しよう。

まず`python-dotenv`を追加インストールする。

```bash
pip install python-dotenv
```

プロジェクトのルートに`.env`ファイルを作成する。

```text
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxxxxxxxxxxxxxxxxxxxx
```

`.gitignore`に`.env`を必ず追記しておく。

```text
.env
```

これを忘れてGitHubにプッシュすると、数時間以内にbotにキーをスキャンされる。実際に経験した話だ。

### クライアントの初期化コード

```python
import anthropic
from dotenv import load_dotenv
import os

# .envファイルを読み込む
load_dotenv()

# クライアント初期化（環境変数ANTHROPIC_API_KEYを自動参照）
client = anthropic.Anthropic()

# api_keyを明示する書き方も可
# client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
```

**旧記法との違いをはっきりさせておく。**

| 項目 | 旧記法（動かない） | 現在の正しい書き方 |
|---|---|---|
| クライアント生成 | `anthropic.Client()` | `anthropic.Anthropic()` |
| テキスト生成 | `client.completions.create()` | `client.messages.create()` |
| モデル名 | `claude-2`, `claude-instant-1` | `claude-sonnet-4-20250514`など |
| systemプロンプト | messagesの中に含める | 別の`system`パラメータで渡す |

### モデル名は定数化して管理する

モデル名をコード内に文字列で何度も書くと、更新時に修正漏れが起きる。定数化がベストプラクティスだ。

```python
# モデル名を定数として一箇所で管理
MODEL = "claude-sonnet-4-20250514"
```

日付付きのバージョン名（`-20250514`の部分）を固定することで、Anthropicがモデルをアップデートしても**自分のコードの動作が変わらない**。再現性のある開発に必須の習慣だ。

---

## H2③ Messages APIでテキスト生成・レスポンスを受け取る基本コード

### まずコピペで動く最小構成

```python
import anthropic
from dotenv import load_dotenv

load_dotenv()

client = anthropic.Anthropic()
MODEL = "claude-sonnet-4-20250514"

message = client.messages.create(
    model=MODEL,
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Pythonでフィボナッチ数列を出力するコードを書いてください"}
    ]
)

print(message.content[0].text)
```

これを`main.py`として保存して`python main.py`で実行すると、Claudeの回答がターミナルに表示される。実際に動かしてみると、レスポンスが返るまで1〜3秒程度かかる感覚だ。

### 各パラメータの役割

**`model`**：呼び出すモデル名。2026年時点の主要モデルと料金感は次のとおり。

| モデル名 | 特徴 | 入力コスト | 出力コスト |
|---|---|---|---|
| `claude-opus-4` | 最高精度・複雑なタスク向け | 約$15/MTok | 約$75/MTok |
| `claude-sonnet-4-20250514` | 精度とコストのバランス型 | 約$3/MTok | 約$15/MTok |
| `claude-haiku-3.5` | 高速・低コスト。大量処理向け | 約$0.8/MTok | 約$4/MTok |

入門段階では`claude-haiku-3.5`で十分な場面が多い。コストを気にするなら積極的に使うべきモデルだ。

**`max_tokens`**：生成するテキストの最大トークン数。**これを省略したり小さくしすぎると、文章が途中でブツ切れになる**。最初は`1024`〜`2048`あたりに設定しておくのが無難だ。

**`messages`**：会話履歴をリスト形式で渡す。`role`は`user`か`assistant`のどちらかを指定する。

### systemプロンプトはOpenAIと渡し方が違う

OpenAI APIを使ったことがある人は要注意だ。ChatGPT APIでは`messages`の中に`{"role": "system", "content": "..."}`を入れるが、**Claude APIではsystemは別パラメータ**で渡す。

```python
message = client.messages.create(
    model=MODEL,
    max_tokens=1024,
    system="あなたはPythonの専門家です。コードには必ずコメントを入れてください。",  # ← ここ
    messages=[
        {"role": "user", "content": "辞書のソートを教えてください"}
    ]
)

print(message.content[0].text)
```

ここを間違えると`role`エラーが出る。OpenAI経験者が最初に詰まるポイントだ。

### レスポンスオブジェクトの構造

APIから返ってくる`message`オブジェクトには、テキスト以外にも情報が含まれている。

```python
# テキスト本文
print(message.content[0].text)

# トークン使用量の確認（コスト管理に使う）
print(f"入力トークン: {message.usage.input_tokens}")
print(f"出力トークン: {message.usage.output_tokens}")

# 停止理由（end_turn / max_tokens / stop_sequenceのどれか）
print(f"停止理由: {message.stop_reason}")
```

`stop_reason`が`max_tokens`になっている場合、文章が途中で切れている。`max_tokens`の値を増やすサインだ。

---

## H2④ エラーハンドリングと実用的なコード

### 本番運用を意識した実装

入門を終えたら、次はエラーハンドリングを入れた実用コードに移行する。

```python
import anthropic
from dotenv import load_dotenv
import os

load_dotenv()

client = anthropic.Anthropic()
MODEL = "claude-sonnet-4-20250514"

def generate_text(user_prompt: str, system_prompt: str = "") -> str:
    """
    Claude APIでテキストを生成して返す関数
    """
    try:
        kwargs = {
            "model": MODEL,
            "max_tokens": 1024,
            "messages": [
                {"role": "user", "content": user_prompt}
            ]
        }
        # systemプロンプトが指定されている場合のみ追加
        if system_prompt:
            kwargs["system"] = system_prompt

        message = client.messages.create(**kwargs)

        # 途中で切れていないか確認
        if message.stop_reason == "max_tokens":
            print("警告: max_tokensに達しました。出力が途中で切れている可能性があります")

        return message.content[0].text

    except anthropic.AuthenticationError:
        print("APIキーが無効です。.envファイルを確認してください")
        raise
    except anthropic.RateLimitError:
        print("レートリミットに達しました。しばらく待ってから再試行してください")
        raise
    except anthropic.APIError as e:
        print(f"APIエラーが発生しました: {e}")
        raise

# 実際に呼び出す
if __name__ == "__main__":
    result = generate_text(
        user_prompt="Pythonのリスト内包表記を3行で説明してください",
        system_prompt="エンジニア向けに簡潔に答えてください"
    )
    print(result)
```

### ストリーミングで応答を逐次表示する

長い文章を生成するとき、全部生成し終わるまで画面に何も表示されないのはUXが悪い。ストリーミングを使えば、ChatGPTのようにリアルタイムで文字が出てくる。

```python
with client.messages.stream(
    model=MODEL,
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Pythonの非同期処理について詳しく説明してください"}
    ]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
print()  # 最後に改行
```

`.stream()`コンテキストマネージャを使うだけで実装できる。「ストリーミングは難しそう」と思っていた人も、これを見ると拍子抜けするはずだ。

---

## H2⑤ よくある失敗・ハマりポイント集

### 失敗① 古い記事のコードをコピーして動かない

これが一番多い。`anthropic.Client()`や`client.completions.create()`はもう動かない。エラーメッセージは`AttributeError: 'Anthropic' object has no attribute 'completions'`のように出る。見た瞬間に「古いSDKの書き方だ」と判断できるようになっておくといい。

### 失敗② モデル名が間違っている

`claude-2`や`claude-v1`を指定すると、`NotFoundError: 404 Not Found`が返ってくる。必ず[Anthropic公式ドキュメントのモデル一覧](https://docs.anthropic.com/en/docs/about-claude/models)で現行のモデル名を確認する。モデル名は`claude-sonnet-4-20250514`のように日付が入るので、スペルミスに注意。

### 失敗③ `max_tokens`を省略して文章が途中で切れる

デフォルト値が低い場合があり、長い文章を要求すると途中で終わる。`stop_reason`を確認する習慣をつけると原因がすぐわかる。

### 失敗④ OpenAIの書き方でsystemプロンプトを渡す

```python
# ❌ これはClaudeでは動かない（またはエラーになる）
messages=[
    {"role": "system", "content": "あなたは専門家です"},
    {"role": "user", "content": "質問"}
]

# ✅ 正しい書き方
system="あなたは専門家です",
messages=[
    {"role": "user", "content": "質問"}
]
```

Claude APIは`messages`内に`role: system`を入れるとエラーになる（またはバリデーションで弾かれる）。

---

## よくある質問

### Q1. 無料で試す方法はないか？

厳密な無料プランはないが、クレジットカード登録後の初回クレジット（約$5相当）で十分試せる。`claude-haiku-3.5`で短文生成を繰り返すなら、数千回は動かせる計算になる。使いすぎが心配なら、Anthropic Consoleの「Usage Limits」で月額上限を設定しておくといい。

### Q2. OpenAI APIとどっちがいいか？

用途次第だ。長文の要約・分析・コード生成はClaudeが得意な場面が多い。一方、エコシステムの広さ・プラグインの豊富さではOpenAIに分がある。両方試して使い分けるのが現実的な答えだ。Pythonコードの書き方は似ているので、一方を覚えればもう一方にも移行しやすい。

### Q3. レートリミットに引っかかった場合はどうする？

Tier1（初期状態）では1分あたり約50リクエスト・約4万トークンが上限だ。大量処理を行う場合は、`time.sleep(1)`で1秒間隔を入れるか、`tenacity`ライブラリでリトライロジックを組む。利用実績が積み上がると自動でTierが上がり、制限が緩和される。

### Q4. Prompt Cachingとは何か？

長いsystemプロンプトを毎回APIに送ると、その分のトークン料金がかかる。Prompt Cachingを使うと、一度処理したsystemプロンプトをキャッシュして**最大90%のコスト削減**ができる機能だ（2025年〜一般提供）。同じsystemプロンプトで大量のリクエストを投げる用途では効果が大きい。入門段階では気にしなくていいが、本番運用では検討する価値がある。

---

## 次にやるべき1つのアクション

まずこの記事のコードをコピーして、自分のターミナルで動かすこと。それだけでいい。

読んだだけで終わると何も残らない。`pip install anthropic python-dotenv`から始めて、「Pythonで今日の天気の挨拶文を作って」と一行投げるだけでも、Claude APIが何者かは体でわかる。

動かせたら次は、自分が毎日やっている作業の中で「テキストを生成できたら楽になる部分」を一つ見つけてみる。そこからが本当の使い方の始まりだ。

---

## 関連記事

- [Claude APIキー取得の手順と料金｜GPT-4oコスト比較・無料枠の実態も解説](/2026/05/22/Claude-APIキー取得の手順と料金GPT-4oコスト比較無料枠の実態も解説/)
- [Claude API Python 使い方入門｜キー設定・コードサンプル・エラー解決まで](/2026/05/24/Claude-API-Python-使い方入門キー設定コードサンプルエラー解決まで/)
- [Claude API×Notion連携｜AI生成文をDBに自動保存する3つの方法](/2026/05/31/Claude-APINotion連携やり方AIライティング結果をDBに自動保存する手順/)

---

## 📘 もっと深く学びたい方へ

この記事で紹介した内容を、さらに体系的に・実務レベルで習得できる教材を販売中です。

### [ChatGPT＆Claude AIプロンプト集50選（¥980）](https://aijissenlab.gumroad.com/l/yjcdam)

コピペで即使える実践プロンプト50種を全24ページに凝縮

- ビジネスメール・企画書・分析・コーディング等 8カテゴリ網羅
- ChatGPT / Claude / Gemini 全対応
- 変数を埋めるだけで即実務投入

👉 [今すぐ購入する](https://aijissenlab.gumroad.com/l/yjcdam)

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