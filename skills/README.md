# Skills

このフォルダーは、開発作業を支援する AI エージェント向けの **スキル置き場**です。
各スキルは専用フォルダーにまとめ、`SKILL.md` と必要な参照資料・スクリプトなどを配置します。

## 配置と利用方法

ここにあるスキルはプロジェクト内で管理するための共通の保管場所です。`skills/` 配下に置いただけでは、各エージェントから自動で読み込まれるとは限りません。
使用するエージェントに応じて、スキルのフォルダー一式を、そのエージェントが認識する場所へ配置してください。

| エージェント | プロジェクト内の配置先 |
| --- | --- |
| Claude Code | `.claude/skills/<スキル名>/` |
| OpenAI Codex | `.agents/skills/<スキル名>/` |
| GitHub Copilot | `.github/skills/<スキル名>/` |

たとえば `skills/glue-pyspark-review/` を利用する場合は、フォルダー全体を `.claude/skills/glue-pyspark-review/` または `.github/skills/glue-pyspark-review/` に配置します。`SKILL.md` だけでなく、`references/` などの関連ファイルも一緒に配置してください。

## スキル一覧

- [`glue-pyspark-review/`](./glue-pyspark-review/): AWS Glue の Python / PySpark ジョブをレビューするスキル
- [`glue-pyshell-review/`](./glue-pyshell-review/): AWS Glue の Python Shell ジョブ（小規模な pandas / boto3 処理）をレビューするスキル

新しいスキルを追加するときは、スキル名のフォルダーを作成して `SKILL.md` を置き、この一覧にも追記してください。