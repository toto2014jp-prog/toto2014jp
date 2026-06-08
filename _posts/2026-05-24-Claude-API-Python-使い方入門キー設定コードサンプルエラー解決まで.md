---
layout: post
title: "Claude API Python 使い方入門｜キー設定・コードサンプル・エラー解決まで"
date: 2026-05-24 09:09:56 +0900
description: "Claude APIをPythonで使う方法を初心者向けに解説。APIキーの安全な設定から動くコードサンプル、よくあるエラーの解決策まで2025年最新の実装パターンで紹介。"
tags:
  - Claude API Python 使い方 初心者
  - Anthropic Python ライブラリ 設定 やり方
  - Claude API Python コード サンプル
  - Claude API エラー 解決 初心者
---

「APIキーは取得できたけど、Pythonでどう書けばいいかわからない」——正直、ここで詰まる人が一番多い。

ネットで検索すると出てくるコードの多くが古い。旧モデル名、廃止済みエンドポイント、そして**APIキーをコードに直書きする危険な書き方**まで普通に上位表示されている現状がある。

この記事を読めば、2025年時点で「実際に動く・安全な」Claude API実装の最短ルートをつかめる。

---

## 環境を整える｜インストールとAPIキーの安全な設定

### 2行のインストールで準備は終わる

ターミナルで以下を実行するだけだ。

```bash
pip install anthropic python-dotenv
```

`anthropic`が公式SDK。`python-dotenv`はAPIキーを安全に管理するためのライブラリで、この2つはセットで入れるのが鉄則だ。

### APIキーは絶対にコードに直書きしない

これ、意外と知られていないんだが、GitHubへのAPIキー流出事故は今も毎日起きている。原因のほぼ全てが「`.py`ファイルへの直書き」だ。

**❌ やってはいけない書き方**

```python
# 絶対にダメ。Gitにpushした瞬間に漏れる
client = anthropic.Anthropic(api_key="sk-ant-xxxxxxxxxx")
```

**✅ 正しい管理方法**

プロジェクトのルートに`.env`ファイルを作り、キーをそこに書く。

```bash
# .envファイル（このファイルはGitに含めない）
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxx
```

コード側はこう書く。

```python
from dotenv import load_dotenv
import os
import anthropic

load_dotenv()  # .envファイルを読み込む
client = anthropic.Anthropic(api_key=os.environ.get("ANTHROPIC_API_KEY"))
```

さらに、`.gitignore`ファイルに以下を追記するのを忘れずに。

```
.env
```

この3ステップがセキュリティの最低ライン。面倒でも最初に必ずやっておく。

> APIキー自体の取得手順については「[Claude APIキーの取得手順と料金]()」の記事を参照してほしい。本記事は「取得後の実装」に絞って解説する。

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

## 最初に動かす｜テキスト生成の基本コードと必須パラメータ

### そのままコピペして動く最小構成コード

```python
from dotenv import load_dotenv
import os
import anthropic

load_dotenv()

# モデル名は定数化して管理するのがおすすめ
MODEL_NAME = "claude-3-5-sonnet-20241022"

client = anthropic.Anthropic(api_key=os.environ.get("ANTHROPIC_API_KEY"))

response = client.messages.create(
    model=MODEL_NAME,
    max_tokens=1024,  # 必須パラメータ。省略するとエラーになる
    system="あなたは丁寧な日本語アシスタントです。",  # systemはここに書く
    messages=[
        {"role": "user", "content": "Pythonの辞書型を3行で説明してください"}
    ]
)

print(response.content[0].text)

# トークン消費量の確認（コスト管理に必須）
print(f"入力トークン: {response.usage.input_tokens}")
print(f"出力トークン: {response.usage.output_tokens}")
```

実行すると、`response.content[0].text`にClaudeの回答が入っている。使ってみて驚いたのは、初回実行から本当にこれだけで動くことだ。

### 初心者がハマる3つのパラメータの罠

**① `max_tokens`は省略できない**

OpenAI APIに慣れている人が最初にやらかすポイント。Claude APIでは`max_tokens`が**必須パラメータ**になっている。書かないとバリデーションエラーで即死する。

**② `system`プロンプトの置き場所がOpenAIと違う**

OpenAIでは`messages`配列の中に`{"role": "system", ...}`として書くが、ClaudeはAPIのトップレベルパラメータとして`system=`に書く。混同すると意図通りに動かない。

**③ モデル名には日付が入る**

`claude-3-5-sonnet`と書いてもエラーになる。正解は`claude-3-5-sonnet-20241022`のように**日付付きの完全なモデル名**だ。ネット上に`claude-2`や`claude-v1`を使うコードが大量に残っているが、現在は非推奨または廃止済みなので使わないこと。

**モデル選択の目安**

| モデル名 | 用途 | 入力コスト目安 |
|---|---|---|
| claude-3-5-sonnet-20241022 | バランス重視・メイン用途 | $3 / 1Mトークン |
| claude-3-haiku-20240307 | 速度・コスト重視 | $0.25 / 1Mトークン |

※料金は変動するため公式ページで必ず確認すること。

### モデル名を定数化する理由

```python
MODEL_NAME = "claude-3-5-sonnet-20241022"  # ここだけ変えれば全ファイルに反映
```

プロジェクトが大きくなると、モデル名を複数ファイルにハードコードしていると更新が地獄になる。最初から定数化しておくのが正解だ。

---

## よくある5つのエラーと即解決コード

正直なところ、初心者が詰まるポイントはほぼ決まっている。エラーメッセージ→原因→修正コードの順で解説する。

### エラー① `max_tokens`省略によるバリデーションエラー

```
anthropic.BadRequestError: {'type': 'error', 'error': {'type': 'invalid_request_error', 'message': 'max_tokens: Field required'}}
```

**原因**：`max_tokens`を書き忘れている。

**修正**：`max_tokens=1024`を必ず追加する。目安として、短い回答なら512〜1024、長文生成なら2048〜4096を指定する。

### エラー② 旧モデル名による404エラー

```
anthropicl.NotFoundError: Error code: 404 - model not found
```

**原因**：`claude-2`や`claude-v1`など廃止済みのモデル名を使っている。

**修正**：`model="claude-3-5-sonnet-20241022"`のように日付付きの正式名を使う。現在有効なモデル名は[Anthropic公式ドキュメント](https://docs.anthropic.com/en/docs/about-claude/models)で確認できる。

### エラー③ `RateLimitError`の対処

```
anthropicl.RateLimitError: 429 Too Many Requests
```

**原因**：Tier1のレートリミット（約50,000トークン/分）を超えた。

**修正**：`tenacity`ライブラリを使ったリトライ処理を実装する。

```python
import time
import anthropic

def call_with_retry(client, **kwargs):
    for attempt in range(3):  # 最大3回リトライ
        try:
            return client.messages.create(**kwargs)
        except anthropic.RateLimitError:
            wait_time = 2 ** attempt  # 1秒→2秒→4秒と指数的に待機
            print(f"レートリミット。{wait_time}秒後にリトライ...")
            time.sleep(wait_time)
    raise Exception("リトライ上限に達しました")
```

### エラー④ APIキーの認証エラー

```
anthropicl.AuthenticationError: 401 Unauthorized
```

**原因**は3つのどれかだ。

- `.env`ファイルの保存場所がPythonスクリプトと別のディレクトリにある
- `load_dotenv()`を呼び出す前に`Anthropic()`を初期化している
- APIキー自体が無効または期限切れ

**修正**：`load_dotenv()`は必ず`Anthropic()`より前に書く。パスの問題なら`load_dotenv(dotenv_path="/絶対パス/.env")`で明示的に指定する。

### エラー⑤ `systemプロンプトをmessages配列に入れてしまう`

```python
# ❌ OpenAI流の書き方。Claudeでは動かない
messages=[
    {"role": "system", "content": "丁寧に答えてください"},
    {"role": "user", "content": "質問です"}
]
```

これはエラーにならずに動いてしまうケースもあるが、systemプロンプトが正しく機能しない。

```python
# ✅ Claudeの正しい書き方
client.messages.create(
    model=MODEL_NAME,
    max_tokens=1024,
    system="丁寧に答えてください",  # トップレベルに置く
    messages=[
        {"role": "user", "content": "質問です"}
    ]
)
```

---

## 実装の次のステップ｜ストリーミングと会話履歴の管理

基本実装が動いたら、次の2つを押さえると一気に実用的になる。

### ストリーミングで応答をリアルタイム表示する

チャットアプリ的に使うなら、ストリーミングはほぼ必須だ。長い回答でも文字が順番に流れてくるので体験が全然違う。

```python
# ストリーミング実装
with client.messages.stream(
    model=MODEL_NAME,
    max_tokens=1024,
    messages=[{"role": "user", "content": "Pythonの非同期処理を教えてください"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)  # リアルタイムで出力
print()  # 最後に改行
```

`client.messages.create()`を`client.messages.stream()`に変えるだけ。差分はこれだけだ。

### 会話履歴を保持してマルチターン会話を実現する

複数のやり取りを続けるには、`messages`リストに会話履歴を積み上げていく。

```python
conversation_history = []  # 会話履歴を保持するリスト

def chat(user_input):
    conversation_history.append({"role": "user", "content": user_input})
    
    response = client.messages.create(
        model=MODEL_NAME,
        max_tokens=1024,
        system="あなたは親切なアシスタントです。",
        messages=conversation_history  # 全履歴を渡す
    )
    
    assistant_reply = response.content[0].text
    conversation_history.append({"role": "assistant", "content": assistant_reply})
    
    return assistant_reply

# 使用例
print(chat("Pythonの変数とは何ですか？"))
print(chat("では、さっきの例をリストでも教えてください"))  # 前の文脈が引き継がれる
```

Context Windowは最大200,000トークン（約15万語相当）あるので、よほど長い会話でない限り切り捨て処理は不要だ。

---

## まとめ｜今日やるべき1つのアクション

この記事で押さえたポイントを整理する。

- インストールは`pip install anthropic python-dotenv`の2行
- APIキーは必ず`.env`ファイルで管理し、`.gitignore`に追記する
- `max_tokens`は必須、`system`はトップレベルに書く
- モデル名は日付付きの正式名（`claude-3-5-sonnet-20241022`など）を使う

**今日やるべきアクションは1つだけ。**

この記事の「最小構成コード」をそのままコピーして、まず1回動かしてみること。設定が正しければ30秒で動く。動いた瞬間に、次に何を作るかのアイデアが自然と浮かんでくるはずだ。

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

## 関連記事

- [ChatGPT API×スプレッドシート連携｜GAS初心者向け設定と自動化手順](/2026/05/22/ChatGPT-APIスプレッドシート連携GAS初心者向け設定と自動化手順/)
- [ChatGPT PythonコードをコピペOK！初心者向けプロンプト集と頼み方のコツ](/2026/05/22/ChatGPT-PythonコードをコピペOK初心者向けプロンプト集と頼み方のコツ/)
- [Claude APIキー取得の手順と料金｜GPT-4oコスト比較・無料枠の実態も解説](/2026/05/22/Claude-APIキー取得の手順と料金GPT-4oコスト比較無料枠の実態も解説/)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。