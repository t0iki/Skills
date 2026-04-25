# Skills

Claude Code 用のカスタムスキル集。

## 収録スキル

### figurelize

ユーザーと対話しながら考えを整理し、FigJam ボードに図として描画・更新するスキル。
構造把握や仕様設計に使う。明示呼び出し前提。

- 起動: `/figurelize <FigJam URL>`
- 詳細: [figurelize/SKILL.md](./figurelize/SKILL.md)
- 前提: `claude.ai Figma` MCP の接続が必要

## インストール

スキルとして使うには、`~/.claude/skills/` にシンボリックリンクを張る。

```sh
ln -s "$PWD/figurelize" ~/.claude/skills/figurelize
```

確認:

```sh
ls -la ~/.claude/skills/figurelize
```

Claude Code を再起動すると `/figurelize` が使えるようになる。

## 開発メモ

- `plans/` には設計メモが入っている
- 各スキルは独立したディレクトリ（`figurelize/` など）
- 各ディレクトリには `SKILL.md`（本体）と必要に応じた補助ファイル
