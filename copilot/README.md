# Copilot Instructions Template

このディレクトリは、GitHub Copilot（Copilot Chat / Copilot copilots 等）で利用する指示書（instructions）ファイルのテンプレート置き場です。

テンプレートを一元管理し、新規プロジェクト時に再利用できるように準備します。

## テンプレート作成、利用手順

1. `instructions/` に新しい `*.instructions.md` ファイルを作成します。
2. 後述する[テンプレート作成の基本ルール](#テンプレート作成の基本ルール)や[テンプレートの書き方](#テンプレートの書き方)を参考にInstructionsファイルの内容を記述する。
3. 既存/新規プロジェクトに作成したテンプレートをコピペして配置します。
   - `.github/copilot-instructions.md`として利用する場合はフロントマターを削除し、テンプレートの内容のみコピペしてください。
   - `.github/instructions/*.instructions.md`として利用する場合はテンプレートファイルをそのままコピーしてください。必要に応じてファイル名を変更しても大丈夫です。
4. 実際に利用し、改善点があった際には修正しテンプレートはメンテナンスを行うこと。

## テンプレート作成の基本ルール
- ファイル名: `<purpose>.instructions.md` のように目的が分かる名前にします。
- 保存場所: このディレクトリ直下の `instructions/` に配置します。

## テンプレートの書き方

詳細は公式のドキュメント [Creating path-specific custom instructions (GitHub Docs)](https://docs.github.com/ja/copilot/how-tos/copilot-on-github/customize-copilot/add-custom-instructions/add-repository-instructions?tool=vscode#creating-path-specific-custom-instructions) を確認してください

以下の記述欄をコピーし、プロジェクトの構成に合わせて書き換えてください。

```markdown
---
applyTo: "**/*.ts,**/*.tsx"     # 【必須】適用するファイルをGlob構文で指定（例: 所有する主要言語の拡張子など）
# excludeAgent: "code-review"   # 【任意】特定の機能（"code-review" または "cloud-agent"）で読み込ませない場合にコメントアウトを解除
---

# GitHub Copilot Custom Instructions for [プロジェクト名/機能名]
<!-- Instructionsファイルのタイトル、用途などわかるような記載をする -->

Copilotにふるまってほしい動作や指示文を記載する
```