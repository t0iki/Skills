# FigJam Plugin API 早見表

`use_figma` の `mode: "figjam"` で実行する JavaScript の書き方。
すべて Figma Plugin API（Sandbox 内 JS）として実行される。

> **注**: 公式 SKILL.md の制約として `console.log` は使わず、必ず `return` で値を返す。
> 失敗時はスクリプト全体がアトミックに巻き戻る。

---

## 基本テンプレート

```js
// 必要なフォントを先にロード
await figma.loadFontAsync({ family: "Inter", style: "Medium" });

const created = [];

// === ここでノードを作る ===

return {
  createdNodeIds: created.map(n => n.id),
  mutatedNodeIds: [],
  labels: {
    "ラベル": created[0].id
  }
};
```

---

## Sticky（付箋）

```js
const sticky = figma.createSticky();
await figma.loadFontAsync(sticky.text.fontName);
sticky.text.characters = "認証要件";

// 色（FigJam 標準色プリセットがある）
sticky.fills = [{
  type: 'SOLID',
  color: { r: 1, g: 0.9, b: 0.4 }  // 0-1 の範囲
}];

sticky.x = 100;
sticky.y = 100;

figma.currentPage.appendChild(sticky);
```

### Sticky の標準色（参考値）

| 色 | r, g, b |
|---|---|
| YELLOW | 1.0, 0.90, 0.40 |
| GREEN | 0.55, 0.85, 0.55 |
| BLUE | 0.55, 0.75, 1.0 |
| PINK | 1.0, 0.65, 0.80 |
| ORANGE | 1.0, 0.70, 0.30 |
| PURPLE | 0.75, 0.60, 1.0 |
| RED | 1.0, 0.45, 0.45 |
| LIGHT_GRAY | 0.90, 0.90, 0.90 |
| GRAY | 0.65, 0.65, 0.65 |

---

## ShapeWithText（ラベル付き図形 / フローチャート用）

```js
const shape = figma.createShapeWithText();
shape.shapeType = 'DIAMOND';   // 後述の一覧から選ぶ
await figma.loadFontAsync(shape.text.fontName);
shape.text.characters = "認証済み?";
shape.x = 200;
shape.y = 200;
figma.currentPage.appendChild(shape);
```

### shapeType の主な値

| 値 | 用途 |
|---|---|
| `ROUNDED_RECTANGLE` | 通常の処理ステップ |
| `DIAMOND` | 条件分岐 |
| `ELLIPSE` | 開始 / 終了 |
| `TRIANGLE_UP` / `TRIANGLE_DOWN` | 三角 |
| `PARALLELOGRAM_RIGHT` / `PARALLELOGRAM_LEFT` | 入出力 |
| `ENG_DATABASE` | DB（ER図） |
| `ENG_QUEUE` | キュー |
| `ENG_FILE` | ファイル |
| `ENG_FOLDER` | フォルダ |

---

## Connector（接続線）

```js
const conn = figma.createConnector();
conn.connectorStart = { endpointNodeId: a.id, magnet: 'AUTO' };
conn.connectorEnd   = { endpointNodeId: b.id, magnet: 'AUTO' };

// ラベル（任意）
await figma.loadFontAsync(conn.text.fontName);
conn.text.characters = "Yes";

// 線の種類
conn.connectorLineType = 'ELBOWED';  // or 'STRAIGHT'
conn.connectorEndStrokeCap = 'ARROW_LINES';  // 矢印の形
```

`magnet` は `'AUTO' | 'TOP' | 'BOTTOM' | 'LEFT' | 'RIGHT' | 'CENTER'`。

---

## CodeBlock（コードスニペット）

```js
const code = figma.createCodeBlock();
code.code = "function login(email, password) {\n  // ...\n}";
code.codeLanguage = 'JAVASCRIPT';
code.x = 300;
code.y = 300;
figma.currentPage.appendChild(code);
```

`codeLanguage` の例: `JAVASCRIPT`, `TYPESCRIPT`, `PYTHON`, `GO`, `RUST`, `JSON`, `BASH`, `SQL`, `CPP`, `CSS`, `HTML` など。

---

## ノードを ID で取得して mutate

```js
const node = await figma.getNodeByIdAsync("12:34");
if (node && node.type === 'STICKY') {
  await figma.loadFontAsync(node.text.fontName);
  node.text.characters = "更新後のテキスト";
}
```

`figma.getNodeByIdAsync` は **必ず await**。同期版もあるが将来非推奨。

---

## ノード削除

```js
const node = await figma.getNodeByIdAsync("12:34");
if (node) node.remove();
```

---

## 複数ノードのバッチ作成 + 接続例（マインドマップ）

```js
await figma.loadFontAsync({ family: "Inter", style: "Medium" });

const center = figma.createSticky();
center.text.characters = "中心テーマ";
center.x = 0; center.y = 0;
figma.currentPage.appendChild(center);

const branches = ["要素A", "要素B", "要素C", "要素D"];
const created = [center];
const labels = { "中心テーマ": center.id };

for (let i = 0; i < branches.length; i++) {
  const angle = (i / branches.length) * Math.PI * 2;
  const r = 400;
  const s = figma.createSticky();
  s.text.characters = branches[i];
  s.x = Math.cos(angle) * r;
  s.y = Math.sin(angle) * r;
  figma.currentPage.appendChild(s);
  created.push(s);
  labels[branches[i]] = s.id;

  const c = figma.createConnector();
  c.connectorStart = { endpointNodeId: center.id, magnet: 'AUTO' };
  c.connectorEnd   = { endpointNodeId: s.id, magnet: 'AUTO' };
  figma.currentPage.appendChild(c);
  created.push(c);
}

return {
  createdNodeIds: created.map(n => n.id),
  mutatedNodeIds: [],
  labels
};
```

---

## 縦フローチャート例

```js
await figma.loadFontAsync({ family: "Inter", style: "Medium" });

const steps = [
  { type: 'ELLIPSE', text: '開始' },
  { type: 'ROUNDED_RECTANGLE', text: 'ログイン画面表示' },
  { type: 'DIAMOND', text: '認証済み?' },
  { type: 'ROUNDED_RECTANGLE', text: 'ホーム画面' },
  { type: 'ELLIPSE', text: '終了' }
];

const created = [];
const labels = {};
let prev = null;

for (let i = 0; i < steps.length; i++) {
  const s = figma.createShapeWithText();
  s.shapeType = steps[i].type;
  s.text.characters = steps[i].text;
  s.x = 0;
  s.y = i * 200;
  figma.currentPage.appendChild(s);
  created.push(s);
  labels[steps[i].text] = s.id;

  if (prev) {
    const c = figma.createConnector();
    c.connectorStart = { endpointNodeId: prev.id, magnet: 'BOTTOM' };
    c.connectorEnd   = { endpointNodeId: s.id, magnet: 'TOP' };
    figma.currentPage.appendChild(c);
    created.push(c);
  }
  prev = s;
}

return {
  createdNodeIds: created.map(n => n.id),
  mutatedNodeIds: [],
  labels
};
```

---

## 罠（gotchas）

- `console.log` ではなく `return` で値を返す
- すべての Promise を `await`（race condition の温床）
- ページ切替は `await figma.setCurrentPageAsync(page)` のみ。同期セッターは throw
- 色は **0-1 の範囲**
- `fills` / `strokes` は読み取り専用配列。clone → modify → 再代入
- テキスト系は `await figma.loadFontAsync(node.text.fontName)` してから書く
- 1 コール最大 10 操作。超えそうなら分割呼び出し
