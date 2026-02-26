# Copilot Instructions

## ドキュメント言語

このリポジトリでは、**日本語を主言語としてドキュメントを作成してください。**

- 記事、コメント、ドキュメントは日本語で記述します
- コードのコメントも日本語で記述します
- コミットメッセージは日本語または英語のどちらでも構いません

## Documentation Language

Please use **Japanese as the primary language** for documentation in this repository.

- Articles, comments, and documentation should be written in Japanese
- Code comments should also be written in Japanese
- Commit messages can be in either Japanese or English

## このプロジェクトでの手順メモ（2026-02-26）

- 依存関係がない状態だと `npx zenn ...` が失敗するため、最初に `npm install` を実行する
- 記事の認識確認は `npx zenn list:articles` を使う
- 記事ファイルは front matter の必須項目（`title` / `emoji` / `type` / `topics` / `published`）を埋める
- `title` が空だと保存・検証でエラーになるため、空の下書きは削除するかタイトルを入れる
