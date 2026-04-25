# FigJam思考整理スキル設計

## 目的

ユーザーと一緒に物事の構造を整理したり、エンジニアリングの仕様を設計したりするとき、
議論をテキストだけで進めると解像度が上がりにくい。
このスキルは **対話で発散したアイデアを FigJam ボード上の図として可視化し、
議論が進むたびに同じボードを更新していく** ためのもの。

## ユースケース

1. **構造把握**: ぼんやりしたテーマを要素・関係・制約に分解して可視化する
2. **仕様設計**: 機能のフロー、API設計、データモデルなどを図に起こして共通理解を作る

## 起動方法

明示呼び出し: `/figurelize <FigJam URL>`

- 引数の URL は必須（毎回ユーザーから受け取る方針）
- URL を渡さず呼び出された場合は最初に質問して取得する

## 使う MCP

`claude.ai Figma` MCP の `use_figma` ツール（mode: `"figjam"`）

- Plugin API 経由で JS を実行する仕組み
- FigJam ノード型: `Sticky`, `Connector`, `ShapeWithText`, `CodeBlock`
- 1 コール最大 10 操作、ノード ID を毎回返す（インクリメンタル前提）
- スクリプトはアトミック失敗

> **前提**: 現在 `claude.ai Figma` MCP は `Failed to connect` 状態。
> スキルを使う前に `/mcp` で再認証する必要がある。SKILL.md の冒頭で明記。

## ワークフロー

```
1. 引数パース・URL確認
   └ FigJam URL から fileKey を抽出。state ファイルの存在確認。

2. 意図ヒアリング（1〜2 ターン）
   └ 「何の解像度を上げたいか」「どこまで詳細化したいか」を確認

3. 発散の対話（数ターン）
   └ 要素・関係・制約・前提を引き出す
   └ 必要に応じて掘り下げ質問（grill-me ほど厳しくなくてよい）

4. 初版描画（use_figma で1コール）
   └ Sticky / ShapeWithText / Connector を配置
   └ レイアウトは ShapeWithText + Connector のフローチャート系か
     Sticky のグルーピング系を、内容に応じて選択
   └ 戻り値の createdNodeIds を取得

5. 状態保存
   └ ~/.claude/figjam-state/<fileKey>.json に
     ラベル → ノード ID のマッピングを保存

6. 更新ループ
   └ ユーザーフィードバックを受けて差分更新
   └ 既存 ID を使って書き換え/削除/追加
   └ 状態ファイルも同期
   └ 各更新後に短いサマリと次の選択肢を提示
```

## 状態ファイルの形式

`~/.claude/figjam-state/<fileKey>.json`

```json
{
  "url": "https://www.figma.com/board/<fileKey>/...",
  "fileKey": "<fileKey>",
  "lastUpdated": "2026-04-25T12:34:56+09:00",
  "topic": "ユーザーが取り組んでいるテーマの短い説明",
  "nodes": {
    "ログインフロー": { "id": "12:34", "type": "FRAME" },
    "認証要件": { "id": "12:35", "type": "STICKY", "color": "YELLOW" },
    "認証要件→ログインフロー": { "id": "12:99", "type": "CONNECTOR" }
  },
  "history": [
    { "ts": "2026-04-25T12:34:56+09:00", "summary": "初版作成: 認証フロー骨組み" }
  ]
}
```

ラベルは人間可読なキー、ID は Figma 側の実 ID。
スキル内ではラベルで参照し、書き込み時に ID に解決する。

## 描画スタイル

ユーザーの目的に応じて切り替え:

| 目的 | 推奨スタイル |
|------|------------|
| 構造把握（ブレスト寄り） | Sticky を色でグルーピング、必要に応じて Connector |
| 仕様設計（フロー） | ShapeWithText（DIAMOND/RECT/ELLIPSE）+ Connector |
| 仕様設計（データモデル） | ShapeWithText（ENG_DATABASE 等）+ Connector |
| 階層構造 | Sticky のネスト or ShapeWithText を縦並べ |

スキル内では、初回の意図ヒアリング結果からスタイルを **自動選択** する（確定）。
描画前に「○○のスタイルで描きます」と一言宣言してから実行する。

## ファイル配置

```
Skills/
├── README.md
├── plans/
│   └── FigJam思考整理スキル設計.md   ← 本ファイル
└── figurelize/
    ├── SKILL.md                      # スキル本体（trigger / workflow）
    ├── figjam-api.md                 # FigJam Plugin API 早見表（必要時 load）
    └── prompting.md                  # ヒアリングのフレーズ集（必要時 load）
```

開発はこのレポジトリで行い、利用時は `~/.claude/skills/figurelize` に
シンボリックリンクで配置する想定（インストール手順は README に記載）。

## SKILL.md の概要（実装イメージ）

- フロントマター
  - `name: figurelize`
  - `description`: 「ユーザーと対話しながら構造や仕様を整理し、FigJam ボードに図として描画・更新するスキル。明示呼び出し前提。」
- 本文
  1. 前提条件（Figma MCP 接続必須）
  2. 起動時の引数解釈（URL 必須）
  3. ヒアリング指針（深掘りすぎず、目的を 1〜2 質問で確認）
  4. 描画指針（スタイル選択ロジック、1 コール 10 操作上限）
  5. 状態管理（state ファイル仕様）
  6. 更新時のルール（既存 ID 優先、削除は確認）

## 確認したい事項（実装前にユーザーへ）

1. ファイル配置先はこのレポジトリ (`Skills/skills/<name>/`) で OK か
2. ~~スキル名~~ → `figurelize` に確定
3. 状態ファイル `~/.claude/figjam-state/` の場所で OK か
4. 描画スタイルの自動選択 vs 都度確認、どちら寄りで始めるか
5. Figma MCP の再接続は別途ユーザーが対応するか、スキル冒頭でガイダンスを出すだけでよいか

## 実装ステップ（合意後）

1. `skills/figurelize/` ディレクトリ作成
2. `SKILL.md` を上記方針で記述
3. `figjam-api.md` に Plugin API の FigJam 部分を整理
4. `prompting.md` にヒアリング用テンプレ会話を整理
5. README に install 手順（symlink コマンド）を追記
6. 軽い手動テスト（モック state で更新シミュレーション）
