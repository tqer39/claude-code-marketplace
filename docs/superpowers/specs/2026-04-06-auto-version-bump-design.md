# プラグイン自動バージョンバンプ設計

## Context

claude-code-marketplace の各プラグインは `plugin.json` にバージョン（semver）を持つが、現在すべて `0.1.0` で固定されており手動管理。プラグインの内容を更新した際にバージョンを自動で上げる仕組みが必要。

## 概要

`git commit` 時に pre-commit フレームワーク（`commit-msg` ステージ）を使い、コミットメッセージのプレフィックスとステージされたファイルパスから、対象プラグインの `plugin.json` バージョンを自動更新する。

## バージョン種別の判定ルール

| コミットメッセージパターン | バージョン種別 |
|---------------------------|---------------|
| `fix`, `bugfix` | patch |
| `feat`, `refactor`, `style`, `perf`, `docs`, `test`, `chore` | minor |
| `BREAKING CHANGE` (本文), `feat!`, `fix!` 等の `!` 付き | major |

コミットメッセージの1行目からプレフィックスを抽出する。形式: `prefix(scope): message` または `prefix: message`。

## 処理フロー

1. `commit-msg` フックが起動（引数: `.git/COMMIT_EDITMSG`）
2. コミットメッセージの1行目を読み取り
3. マージコミット（`Merge` で始まる）ならスキップ
4. プレフィックスからバージョン種別（patch/minor/major）を判定
5. `git diff --cached --name-only` でステージされたファイル一覧を取得
6. `plugins/<plugin-name>/` に一致するファイルを抽出し、対象プラグイン名を特定
7. `plugin.json` のみの変更ならスキップ（無限ループ防止）
8. 各プラグインの `plugins/<name>/.claude-plugin/plugin.json` を `jq` で更新
9. 更新した `plugin.json` を `git add` してステージに追加

## ファイル構成

### 新規ファイル

- `scripts/bump-plugin-version.sh` — メインスクリプト

### 変更ファイル

- `.pre-commit-config.yaml` — `commit-msg` ステージにローカルフックを追加
- `Brewfile`（存在する場合）— `jq` を追加

## スクリプト仕様

### 入力

- `$1`: コミットメッセージファイルパス（`.git/COMMIT_EDITMSG`）

### 依存

- `jq`（必須）: JSON の読み取り・書き込み
- `git`: ステージされたファイルの取得、ファイルのステージング

### エッジケース

| ケース | 挙動 |
|--------|------|
| `plugins/` 配下以外の変更のみ | スキップ（正常終了） |
| `plugin.json` のみの変更 | スキップ（無限ループ防止） |
| マージコミット | スキップ |
| 複数プラグインの同時変更 | 各プラグインを個別にバージョン更新 |
| 認識できないプレフィックス | patch として扱う |
| jq 未インストール | エラーメッセージを出して中断 |

### pre-commit 設定

```yaml
- repo: local
  hooks:
    - id: bump-plugin-version
      name: bump plugin version
      entry: scripts/bump-plugin-version.sh
      language: script
      stages: [commit-msg]
      always_run: true
```

## 検証方法

1. `jq` がインストールされていることを確認
2. テスト用プラグインファイルを変更してステージ
3. `fix:` プレフィックスでコミット → patch バージョンが上がることを確認
4. `feat:` プレフィックスでコミット → minor バージョンが上がることを確認
5. `feat!:` プレフィックスでコミット → major バージョンが上がることを確認
6. `plugins/` 外のファイルのみ変更してコミット → バージョンが変わらないことを確認
7. マージコミット → バージョンが変わらないことを確認
