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
- URL の形式例: `https://www.figma.com/board/<fileKey>/<title>?node-id=<pageId>`
- `<fileKey>` と URL の `node-id` を両方抽出する
  - `node-id` は描画先ページの ID（FigJam では `node-id=0-1` がデフォルトページ "Page 1"）
  - URL 中の `node-id=A-B` は Plugin API 上 `A:B` に変換する

## 前提

- `claude.ai Figma` MCP が接続されていて `use_figma` ツールが使える状態であること
  - 未接続なら **ユーザーに `/mcp` で再接続するよう一言伝えて中断する**（自分で再接続は試みない）
- 状態ファイル: `~/.claude/figjam-state/<fileKey>.json`
  - ディレクトリが無ければ作成する

## ワークフロー

### Step 1. URL 解釈と状態ロード

1. 引数 URL から `<fileKey>` と `node-id` を抽出
   - `node-id=A-B` → ページ ID `A:B`
   - `node-id` が無い場合はデフォルト `0:1` (Page 1)
2. **指定された `pageId` がページ (PAGE) かどうか確認** する
   - PAGE 以外なら親をたどって所属ページを取得する
3. `~/.claude/figjam-state/<fileKey>.json` を確認
   - **存在する** → そのまま続きとして読み込む（前回テーマと同じ文脈で進む）
   - **存在しない** → 新規セッションとして進む

> **重要**: FigJam Plugin API には `figma.createPage()` が存在しない。
> 新しいページが必要な場合は **ユーザー自身に手動で作成してもらい**、
> その新ページを開いた状態の URL（`node-id=` 付き）を再度渡してもらう。

### Step 2. 意図の最小ヒアリング

ユーザーの最初の発話 + ボードのタイトル/中身から判断できれば **質問せずに描き始める**。
判断材料が無い場合のみ、1 問だけ「何について整理したいですか?」と聞く。

質問の連投・「解像度の確認」「目的の確認」のような確認文は出さない。

### Step 3. 描画スタイルの自動選択

ユーザーの発話内容から以下のいずれかを **自動で選択**する。スタイルの選択肢を
ユーザーに提示しない（描画後に変更要望が来たら対応する）。

| 判定キーワード / 文脈 | スタイル | 使うノード |
|---|---|---|
| テーマ → 要素群を発散 / ブレスト | **マインドマップ** | Sticky + Connector |
| 処理の流れ / 手順 / 分岐 / フロー | **フローチャート** | ShapeWithText (RECT/DIAMOND/ELLIPSE) + Connector |
| 親子 / 階層 / 分類体系 | **階層ツリー** | ShapeWithText を縦/横並べ + Connector |
| 要素を「種類」でまとめたい | **カードソート** | 色分け Sticky のクラスタ |
| データモデル / ER / テーブル設計 | **ER図風** | ShapeWithText (ENG_DATABASE 等) + Connector |

迷ったら **マインドマップ** をデフォルトにする（最も汎用）。

### Step 4. 初版描画

#### Step 4a. 描画先ページに切り替え

```js
const page = await figma.getNodeByIdAsync("<pageId>");
await figma.setCurrentPageAsync(page);
```

- `setCurrentPageAsync` は **必ず await**（同期版は throw）
- 取得したノードが PAGE でない場合は親をたどる、または中断してユーザーに確認

#### Step 4b. アンカー（描画起点）を決める ★衝突回避

**描画前に必ず既存ノードを scan して、空きエリアにアンカーを置く。**
原点 (0, 0) に何も考えず描くと既存コンテンツに被る。

state ファイルに `anchor: { x, y }` がない場合、以下を実施する:

1. `figma.currentPage.children` を全部読み、bbox を取得
2. 既存の **maxX + 400px** を新規描画のアンカー x にする（東に空けて配置）
3. アンカー y は既存の中心付近（`(minY + maxY) / 2`）にする
4. ページが空（要素ゼロ）なら anchor = (0, 0)
5. state ファイルに `anchor` を保存し、以降のすべての座標は `anchor` 起点の相対オフセットで指定

state にすでに `anchor` がある場合（更新ループ時）はそれをそのまま使う。

詳細実装は [`references/figjam-api.md` のアンカー計算テンプレ](./references/figjam-api.md) を参照。

#### Step 4c. Section で必ず囲む ★

**描画されるノード群は必ず Section 内に格納する。** Section が無いとボードが散らかる。

```js
const section = figma.createSection();
section.name = `figurelize: <topic>（<style>）`;
// ...nodes 作成...
for (const n of created) section.appendChild(n);
```

- Section の名前は `figurelize: <topic>（<style>）` の形（識別しやすく）
- Section は appendChild した時点で子ノードの絶対座標を保ったまま内包する（要検証）
- state に `sectionId` と `sectionName` を保存する

複数の論理グループに分かれる大きな図の場合は、グループごとに Section を分ける。

#### Step 4d. 描画

`use_figma` を以下の指針で呼び出す（詳細 API は `references/figjam-api.md` を参照）:

- **mode**: `"figjam"`
- **1 コール最大 10 操作**（多すぎると失敗するので分割）
- 配置は **anchor からの相対オフセット** で指定（直接 x = 0 などとしない）
  - マインドマップ: 中心 = anchor、放射状に展開
  - フローチャート: anchor から縦に流す（y を 200 ずつ増やす）
  - ツリー: anchor を root に、子の x オフセット 200、子の y +150
- スクリプトは必ず以下の形で値を返す:

```js
return {
  pageId: "...",
  sectionId: "...",
  anchor: { x, y },
  createdNodeIds: [...],
  mutatedNodeIds: [...],
  labels: { "<人間可読ラベル>": "<node id>" }  // ★ これを state に保存する
};
```

### Step 5. 状態保存

描画後、`~/.claude/figjam-state/<fileKey>.json` を更新する:

```json
{
  "url": "<元の URL>",
  "fileKey": "<fileKey>",
  "lastUpdated": "<ISO 8601>",
  "topic": "<ユーザーが取り組んでいるテーマ>",
  "style": "mindmap | flowchart | tree | card-sort | er",
  "pageId": "<page node id, 例 934:1184>",
  "pageName": "<ページ名>",
  "sectionId": "<section node id>",
  "sectionName": "<section 名>",
  "anchor": { "x": 0, "y": 0 },
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
- **削除**: 明示的な指示がある時のみ実行（「○○ 消して」など）
- **大きな構造変更**: ユーザーから明示要求があれば再描画

各更新後の出力は **何を変えたかの 1 行サマリのみ**。
次の選択肢の提案、確認質問、誘導は出さない。

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
