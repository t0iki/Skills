# FigJam Plugin API 早見表

`use_figma` の `mode: "figjam"` で実行する JavaScript の書き方。
すべて Figma Plugin API（Sandbox 内 JS）として実行される。

> **注**: 公式 SKILL.md の制約として `console.log` は使わず、必ず `return` で値を返す。
> 失敗時はスクリプト全体がアトミックに巻き戻る。

---

## 基本テンプレート

```js
// 1) 描画先ページに切り替え
const page = await figma.getNodeByIdAsync("<pageId>");
await figma.setCurrentPageAsync(page);

// 2) フォントを先にロード
await figma.loadFontAsync({ family: "Inter", style: "Medium" });

// 3) アンカー決定（既存 bbox の右側 +400px）
const items = figma.currentPage.children
  .map(n => ({ x1: n.x, y1: n.y, x2: n.x + (n.width ?? 0), y2: n.y + (n.height ?? 0), w: n.width ?? 0, h: n.height ?? 0 }))
  .filter(it => it.w > 0 && it.h > 0);
let aX = 0, aY = 0;
if (items.length > 0) {
  const maxX = Math.max(...items.map(i => i.x2));
  const minY = Math.min(...items.map(i => i.y1));
  const maxY = Math.max(...items.map(i => i.y2));
  aX = maxX + 400;
  aY = (minY + maxY) / 2;
}

// 4) Section
const section = figma.createSection();
section.name = "figurelize: <topic>（<style>）";

// 5) 各ノードを Sticky/ShapeWithText/Connector で作り、絶対座標は anchor 起点で計算
const created = [];
// ... ここで created に push し、各 node を section.appendChild(node) する ...

// 6) Section を中身にフィットさせる（後述のヘルパー）
fitSectionToChildren(section, 80);

return {
  pageId: page.id,
  sectionId: section.id,
  anchor: { x: aX, y: aY },
  createdNodeIds: created.map(n => n.id),
  mutatedNodeIds: [],
  labels: { "ラベル": created[0].id }
};

// === ヘルパー: Section を子の world bbox に合わせる（詳細は下の Section セクション）===
function fitSectionToChildren(section, margin = 80) {
  const ch = section.children;
  if (ch.length === 0) return;
  const valid = ch.map(c => c.absoluteBoundingBox).filter(Boolean);
  const wMinX = Math.min(...valid.map(b => b.x));
  const wMinY = Math.min(...valid.map(b => b.y));
  const wMaxX = Math.max(...valid.map(b => b.x + b.width));
  const wMaxY = Math.max(...valid.map(b => b.y + b.height));
  const newX = wMinX - margin;
  const newY = wMinY - margin;
  const dx = newX - section.x;
  const dy = newY - section.y;
  for (const c of ch) {
    if (c.type === 'CONNECTOR') continue;
    c.x -= dx; c.y -= dy;
  }
  section.x = newX;
  section.y = newY;
  section.resizeWithoutConstraints((wMaxX - wMinX) + margin * 2, (wMaxY - wMinY) + margin * 2);
}
```

---

## ページ切替とページ作成不可

- ページ切替: `await figma.setCurrentPageAsync(page)` のみ（同期版は throw）
- **`figma.createPage()` は FigJam では未対応**。新規ページが必要ならユーザーに作ってもらい、URL の `node-id=A-B` でその `A:B` を渡してもらう

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

## Section（必ず使う）

描画したノード群は必ず Section に格納する。**Section は中身を完全に覆うようにフィットさせる**。

### 重要な落とし穴

`section.appendChild(node)` すると、子の `x, y` は **section ローカル座標** になる。
さらに `section.x` を変更すると **中身も一緒に world 上を動く**。

つまり以下を順にやる必要がある:

1. 子の **world bbox**（`absoluteBoundingBox`）を集計
2. Section の新しい world 位置/サイズを決定
3. Section を移動するときに発生する子のずれを **逆方向に補正** する
4. Section をリサイズ

### フィット用ヘルパー

```js
function fitSectionToChildren(section, margin = 80) {
  const ch = section.children;
  if (ch.length === 0) return;

  // 1) 子の world bbox（CONNECTOR も含めて OK）
  const valid = ch.map(c => c.absoluteBoundingBox).filter(Boolean);
  const wMinX = Math.min(...valid.map(b => b.x));
  const wMinY = Math.min(...valid.map(b => b.y));
  const wMaxX = Math.max(...valid.map(b => b.x + b.width));
  const wMaxY = Math.max(...valid.map(b => b.y + b.height));

  // 2) Section 新 world 位置
  const newX = wMinX - margin;
  const newY = wMinY - margin;
  const newW = (wMaxX - wMinX) + margin * 2;
  const newH = (wMaxY - wMinY) + margin * 2;

  // 3) Section が dx,dy 動くと中身も動くので、非コネクタの子を逆方向に補正
  const dx = newX - section.x;
  const dy = newY - section.y;
  for (const c of ch) {
    if (c.type === 'CONNECTOR') continue; // コネクタは端点追従なので触らない
    c.x = c.x - dx;
    c.y = c.y - dy;
  }

  // 4) Section を移動・リサイズ
  section.x = newX;
  section.y = newY;
  section.resizeWithoutConstraints(newW, newH);
}
```

### 使い方

```js
const section = figma.createSection();
section.name = "figurelize: <topic>（<style>）";

// ...Sticky/ShapeWithText/Connector を作成し、section.appendChild(node) ...

fitSectionToChildren(section, 80);
```

### 座標系の落とし穴 ★

`section.appendChild(node)` の **後** は `node.x/y` は **section ローカル座標** として
解釈される。「世界座標 (wx, wy) に置きたい」場合は次のヘルパを使う:

```js
function placeWorld(node, section, wx, wy) {
  section.appendChild(node);
  node.x = wx - section.x;  // ローカル = world - section.world
  node.y = wy - section.y;
}
```

**よくあるバグ**:

```js
// ✗ NG: appendChild 前に world 座標で x/y を設定し、その後 appendChild
node.x = wx; node.y = wy;
section.appendChild(node); // この時点で x/y は section ローカルに「再解釈」されて
                           // world 位置がずれる
```

正しくは **必ず appendChild → x/y セット** の順。

### 検証

フィット後、`section.absoluteBoundingBox` が全子要素の world bbox を完全に覆っているか
次の式で確認できる:

```js
const fAbb = section.absoluteBoundingBox;
const inside = section.children
  .map(c => c.absoluteBoundingBox)
  .filter(Boolean)
  .every(b => b.x >= fAbb.x && b.y >= fAbb.y && b.x + b.width <= fAbb.x + fAbb.width && b.y + b.height <= fAbb.y + fAbb.height);
```

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
