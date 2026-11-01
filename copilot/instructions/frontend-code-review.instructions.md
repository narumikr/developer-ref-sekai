---
applyTo: "**/*.{ts,js,jsx,tsx}"
description: "フロントエンド(Next.js/Reactなど)のコードレビューの観点やルールをまとめている。"
---

# フロントエンドコードレビューガイドライン

## 前提事項

- 技術スタック
  - **言語**: TypeScript, JavaScript
  - **フレームワーク**: Next.js, React
  - **スタイリング**: Tailwind CSS
- ルーティング
  - App Routerを使用する
  - ページコンポーネントは`app`ディレクトリ配下に配置する
- ディレクトリ構成
  - **app**: pageファイルやlayoutファイルなど画面ファイルやglobal.cssなどの実装を行う
  - **features**: ページ内機能に特化したコンポーネントの実装を行う
  - **components**: プロジェクト内で再利用可能なコンポーネントの実装を行う。フォルダ内は*AtomicDesign*で構成(atoms/molecules/organisms/templates)する
  - **hooks**: カスタムフックの実装を行う
  - **utils**: 純粋関数・副作用のない処理の実装を行い、API通信や外部サービスアクセスなどは置かない
  - **types**: 共通の型定義や複雑な型定義を記載する。小規模や特定のコンポーネントでしか使わないProps定義はコンポーネント内で完結しても良い
  - **services**: API通信や外部サービスアクセスなどの処理を実装する
  - **dictionaries**: i18nの翻訳ファイル置き場
- ディレクトリ間の参照制約
  - 同一ディレクトリ内の参照は許可
  - appからfeaturesへの参照は許可
  - appからcomponentsへの参照は許可
  - featuresからcomponentsへの参照は許可
- コンポーネント
  - `app`フォルダ配下は基本的にはServer Componentとする
  - `app`フォルダ配下で状態管理や副作用のある処理を行う必要がある場合はClient Componentとしても良い。その際にはコメントを残すこと
- 状態管理
  - コンポーネント内で完結する場合は`useState`を使用する
  - ページを跨いでの状態管理が必要な場合は`Context API`や`zustand`の使用を検討する
- 命名規則
  - **app router**: フォルダ名は`kebab-case`とし、ファイル名はNext.JsのAppRouterファイルの規則に従う(page.tsx, layout.tsx)
  - **コンポーネント**: ファイル名は`PascalCase`とし、コンポーネント名はファイル名と同一にする
  - **utils**: ファイル名は`camelCase`とし、関数名も`camelCase`とする
  - **types**: ファイル名は`PascalCase`で`*.types.ts`とセマンティックネーミングで作成し、型名も`PascalCase`とする
  - **constants**: ファイル名は`kebab-case`で`*.constants.ts`とセマンティックネーミングで作成し、定数名は`UPPER_SNAKE_CASE`とする
- i18n多言語対応
  - 共通文言は`common.json`などの共通ファイルにまとめる(汎用文言やComponentsで使用する文言など)
  - ページ固有の文言はページごとに`{page-name}.json`のようにファイルを分ける
  - 文言キーは意味のある名前にする
  - 文言にまとまりがある場合、ルートキーを機能名や要素名としネストさせ定義する
    - 例: `dialog.title`, `dialog.okButton`, `dialog.cancelButton`
- レビュー実施時はレビューテンプレートを使用すること
  - 重要度
    - **High**: 修正必須
    - **Medium**: 特別な事情がない限り修正推奨
    - **Low**: 任意だが、可能なら修正推奨
    - **Suggestion**: 不具合等ではないが、コード品質向上のための提案

## レビュー指摘テンプレート

```md
## <!-- 指摘内容の要約 -->

- **重要度**: [ 🚨High / ⚠️Medium / ℹ️Low / 💡Suggestion ]

**詳細**

<!-- 指摘内容の詳細を記述する -->

**補足**

<!-- 補足事項があれば記述する。また同様の指摘内容が他の箇所にも存在する場合は、そのファイルパスを列挙する。 -->
```

## レビュー観点

- 前提条件が守られているか
- ディレクトリ参照方向のルールに従っているか
- ページファイルに不必要にClient Componentで実装していないか
  - Client Componentは可能な限り末端になっているか
- page.tsxはルーティングの入口として扱い、複雑な処理はfeaturesに切り出されているか
- any型の使用は禁止とする
- 早期リターンが適切に使用されているか
- 不要なコメントアウトコードが残っていないか
- dictionariesの翻訳ファイルに未使用の文言キーが存在しないか
