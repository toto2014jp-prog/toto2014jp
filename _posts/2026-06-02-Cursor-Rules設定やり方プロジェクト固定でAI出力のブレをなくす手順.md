---
layout: post
title: "Cursor Rules設定やり方｜プロジェクト固定でAI出力のブレをなくす手順"
date: 2026-06-02 09:08:01 +0900
description: "Cursor Rulesの.mdc形式での作り方・スコープ設定・ファイル分割まで解説。毎回同じ指示を書く手間をなくし、AI出力のブレを根本から解消する設定手順。"
tags:
  - Cursor Rules 設定 やり方
  - Cursor mdc ファイル プロジェクト 固定
  - Cursor AI 指示 毎回書かない カスタム
  - Cursor Rules スコープ 使い分け 実例
---

- コンポーネントを書くときは、できる限りクラスコンポーネントではなく
  関数コンポーネントを使うようにしてください。なぜなら...

# ✅ 効く書き方
- 関数コンポーネントのみ使用。クラスコンポーネント禁止
```

### Alwaysを過信しない

よくある誤解として「Alwaysに設定すれば必ず守られる」というものがあります。Alwaysはコンテキストに**注入される**だけです。LLMが無視するケースはゼロではありません。

書き方が曖昧だと守られません。「なるべく〜してください」より「〜のみ使用」のような断言表現のほうが遵守率が上がります。

### モノレポでの使い方

サブディレクトリ内にも`.cursor/rules/`を置けます。パッケージごとに異なるルールを管理できるので、モノレポ構成のプロジェクトでも対応可能です。

```
monorepo/
├── .cursor/rules/        # リポジトリ共通ルール
├── packages/
│   ├── frontend/
│   │   └── .cursor/rules/  # フロントエンド専用ルール
│   └── backend/
│       └── .cursor/rules/  # バックエンド専用ルール
```

### テンプレートは「awesome-cursorrules」から始めると速い

ゼロから書くより、GitHubの`awesome-cursorrules`リポジトリ（2025年時点で20,000+スター）からプロジェクトに近いテンプレートを持ってきて改造するほうが早いです。`cursor.directory`というサイトでも用途別にテンプレートが探せます。

---

## まとめ：今日やるべきアクションは1つだけ

設定の全体像を整理します。

- 旧`.cursorrules`は非推奨。`.cursor/rules/*.mdc`形式に移行する
- frontmatterの`globs`と`alwaysApply`でスコープを使い分ける
- ルールは短く断言形式。1ファイル1,000トークンを超えない
- チーム開発ではプロジェクトルールをGitに含めて共有する

**今すぐやること**：`mkdir -p .cursor/rules`を実行して、`base.mdc`を1つ作るだけです。

まず1ファイル作れば、あとは使いながら育てていけます。完璧な設定を最初から目指す必要はありません。「日本語でコメントを書く」「関数は必ずJSDocを付ける」の2行でも、繰り返しの指示入力から解放されます。

---



> 💡 **関連教材**: [ChatGPT業務自動化 実践テンプレート集（¥1,480）](https://aijissenlab.gumroad.com/l/irvscs) — API・スプレッドシート・メール・議事録・請求書をコピペで自動化する実装特化型テンプレート集（全22ページ）

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

## 関連記事

- [Cursor AI Python自動生成｜非エンジニアが業務ファイル処理を丸投げする手順](/2026/05/26/Cursor-AI-Python自動生成非エンジニアが業務ファイル処理を丸投げする手順/)
- [Zapier×ChatGPT連携のやり方｜ノーコードで業務通知・メール自動化を設定する手順](/2026/05/23/ZapierChatGPT連携のやり方ノーコードで業務通知メール自動化を設定する手順/)
- [n8n×Claude API連携のやり方｜Makeより安いセルフホスト自動化構築手順](/2026/05/27/n8nClaude-API連携のやり方Makeより安いセルフホスト自動化構築手順/)

---

## 関連ツール紹介

**ブログ記事を効率的に量産するなら** → <a href="https://px.a8.net/svt/ejp?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" rel="nofollow">Value AI Writer byGMO</a>がSEO記事の自動生成に使える。月額1,650円から利用可能。<img border="0" width="1" height="1" src="https://www13.a8.net/0.gif?a8mat=4B3OQW+FLFS8I+1JUK+1HLFVM" alt="">



おすすめツールの一覧は[こちら](/toto2014jp/recommended/)にまとめている。