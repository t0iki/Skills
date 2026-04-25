# Skills

Claude Code 用のカスタムスキル集。
各スキルは [agentskills.io spec](https://agentskills.io/specification) に準拠した SKILL.md を持つ。

## 収録スキル

### figurelize

ユーザーと対話しながら考えを整理し、FigJam ボードに図として描画・更新するスキル。
構造把握や仕様設計に使う。明示呼び出し前提。

- 起動: `/figurelize <FigJam URL>`
- 詳細: [figurelize/SKILL.md](./figurelize/SKILL.md)
- 前提: `claude.ai Figma` MCP の接続が必要

## インストール

[`gh skill`](https://dev.classmethod.jp/articles/gh-skill-agent-skills-management/) コマンドを使う。**GitHub CLI v2.90.0+** が必要。

### 1. gh のバージョンを確認・アップグレード

```sh
gh --version              # v2.90.0+ であること
brew upgrade gh           # 古ければアップグレード
```

### 2. スキルをインストール

ユーザースコープ（全プロジェクトで使える）:

```sh
gh skill install t0iki/Skills figurelize --agent claude-code --scope user
```

プロジェクトスコープ（このプロジェクトでのみ使う）:

```sh
gh skill install t0iki/Skills figurelize --agent claude-code --scope project
```

### 3. 更新

```sh
gh skill update
```

### 4. SKILL.md を事前に確認

```sh
gh skill preview t0iki/Skills figurelize
```

## 開発メモ

- 各スキルは独立したディレクトリ（`figurelize/` など）
- ディレクトリ構成は agentskills 仕様準拠:
  ```
  <skill-name>/
  ├── SKILL.md          # 本体（frontmatter + instructions）
  ├── references/       # 必要時ロードする補助ドキュメント
  ├── scripts/          # （任意）実行コード
  └── assets/           # （任意）テンプレ・リソース
  ```
- frontmatter の必須フィールド: `name`, `description`
- 推奨フィールド: `license`, `compatibility`, `metadata`

## ライセンス

[MIT](./LICENSE)
