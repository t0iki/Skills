---
name: figurelize
description: ユーザーと対話しながら考えを整理し、FigJamボードに図として描画・更新するスキル。物事の解像度を上げる構造把握や、エンジニアリングの仕様設計に使う。明示呼び出し前提。引数として FigJam URL を受け取る。
license: MIT
compatibility: Designed for Claude Code. Requires the `claude.ai Figma` MCP server connection (`use_figma` tool).
metadata:
  author: t0iki
  version: "0.1.0"
---

# figurelize

ユーザーと対話で考えを発散・整理し、その結果を FigJam ボード上の図として描画する。
議論が進むたびに同じボードを差分更新していく。

## 起動

`/figurelize <FigJam URL>`

- URL は必須引数。渡されていなければ最初に質問して取得する
- URL の形式例: `https://www.figma.com/board/<fileKey>/<title>`
- `<fileKey>` を抽出して状態管理に使う

## 前提

- `claude.ai Figma` MCP が接続されていて `use_figma` ツールが使える状態であること
  - 未接続なら **ユーザーに `/mcp` で再接続するよう一言伝えて中断する**（自分で再接続は試みない）
- 状態ファイル: `~/.claude/figjam-state/<fileKey>.json`
  - ディレクトリが無ければ作成する

## ワークフロー

### Step 1. URL 解釈と状態ロード

1. 引数 URL から `<fileKey>` を抽出（`figma.com/(board|file)/<fileKey>/...`）
2. `~/.claude/figjam-state/<fileKey>.json` を確認
   - **存在する** → 読み込んで「前回は『○○』について整理していました。続けますか? 別テーマに切り替えますか?」と確認
   - **存在しない** → 新規セッションとして進む

### Step 2. 意図ヒアリング（短く）

最初に確認するのは 2 点だけ。深掘りはしない。

1. 「何について整理 / 設計したいか?（一言で）」
2. 「ざっくり構造を把握したい? それとも仕様レベルまで詰めたい?」

詳細は描きながら詰めていく方針。ヒアリングだけで時間を使わない。

### Step 3. 描画スタイルの自動選択

ユーザーの返答内容から以下のいずれかを **自動で選択**する。
描画前に「**◯◯のスタイルで描きます**」と一言だけ宣言する（ユーザーは止めて変更可能）。

| 判定キーワード / 文脈 | スタイル | 使うノード |
|---|---|---|
| テーマ → 要素群を発散 / ブレスト | **マインドマップ** | Sticky + Connector |
| 処理の流れ / 手順 / 分岐 / フロー | **フローチャート** | ShapeWithText (RECT/DIAMOND/ELLIPSE) + Connector |
| 親子 / 階層 / 分類体系 | **階層ツリー** | ShapeWithText を縦/横並べ + Connector |
| 要素を「種類」でまとめたい | **カードソート** | 色分け Sticky のクラスタ |
| データモデル / ER / テーブル設計 | **ER図風** | ShapeWithText (ENG_DATABASE 等) + Connector |

迷ったら **マインドマップ** をデフォルトにする（最も汎用）。

### Step 4. 初版描画

`use_figma` を以下の指針で呼び出す（詳細 API は `references/figjam-api.md` を参照）:

- **mode**: `"figjam"`
- **1 コール最大 10 操作**（多すぎると失敗するので分割）
- スクリプトは必ず以下の形で値を返す:

```js
return {
  createdNodeIds: [...],
  mutatedNodeIds: [...],
  labels: { "<人間可読ラベル>": "<node id>" }  // ★ これを state に保存する
};
```

- 配置は座標を明示指定する（`auto-arrange` だけに頼らない）
  - マインドマップ: 中心 + 放射状
  - フローチャート: 縦に流す（y を 200 ずつ増やす）
  - ツリー: 親子の x オフセット 200, 子の y +150

### Step 5. 状態保存

描画後、`~/.claude/figjam-state/<fileKey>.json` を更新する:

```json
{
  "url": "<元の URL>",
  "fileKey": "<fileKey>",
  "lastUpdated": "<ISO 8601>",
  "topic": "<ユーザーが取り組んでいるテーマ>",
  "style": "mindmap | flowchart | tree | card-sort | er",
  "nodes": {
    "<人間可読ラベル>": {
      "id": "<Figma node id>",
      "type": "STICKY | CONNECTOR | SHAPE_WITH_TEXT | CODE_BLOCK",
      "meta": { "x": 0, "y": 0, "color": "YELLOW" }
    }
  },
  "history": [
    { "ts": "<ISO>", "summary": "<何をしたか>" }
  ]
}
```

- ラベルは人間可読なキー（例: `認証要件`, `ログインフロー → MFA分岐`）
- スキル内ではラベルで参照し、書き込み時に ID に解決する
- 既存ファイルがあればマージして書き込む（潰さない）

### Step 6. 更新ループ

ユーザーフィードバックを受けて差分更新する:

- **既存の書き換え**: ラベルから ID を引いて、その ID のノードを mutate
- **新規追加**: 既存配置を考慮して座標を決め、追加。state にも追記
- **削除**: **削除前に必ずユーザーに確認**（「○○ を消していい?」）
- **大きな構造変更**: 「再描画してよいか?」と確認してから

各更新後に出力する内容:

1. 何を変えたかを 1〜2 行のサマリで
2. 次の選択肢を 2〜3 個提示（深掘りする要素 / 別の観点 / 完了）

## 重要ルール

- **ノード ID は必ず保存し、再生成しない**。同じノードを update する
- **スクリプトはアトミック**（失敗時は何も変わらない）→ fail-fast でよい
- **1 コールあたりの操作数を絞る**（多くて 10 個。超えそうなら分割）
- **色は 0-1 の範囲**（0-255 ではない）。`{r: 1, g: 0.9, b: 0.4}` の形
- **テキスト操作前に `await figma.loadFontAsync(...)`** が必要
- **ページ切替は `await figma.setCurrentPageAsync(...)`** のみ（同期版は throw）
- **fills/strokes は読み取り専用配列** → clone, modify, reassign

## 参考ファイル（必要時にロード）

- [`references/figjam-api.md`](./references/figjam-api.md): FigJam で使える Plugin API のサンプル
- [`references/prompting.md`](./references/prompting.md): ヒアリング用の質問テンプレとスタイル判定のヒント
