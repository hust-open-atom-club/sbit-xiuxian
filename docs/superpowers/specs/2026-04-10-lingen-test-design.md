# Lingen Test — Design Spec

**Author:** 华科开放原子开源俱乐部
**Date:** 2026-04-10
**Status:** Draft, pending user review

## 1. Overview

A 16-question personality quiz that maps user answers to a spiritual root (灵根) category inspired by the xianxia novel *A Record of a Mortal's Journey to Immortality* (《凡人修仙传》). The quiz mimics the question/answer flow of SBTI (https://sbti.unun.dev) — short, meme-heavy, shareable — while dressing the form in a Taoist fortune-slip aesthetic. Questions use classical Chinese phrasing (签文体) describing modern situations; results combine the original lore's spiritual root taxonomy with contemporary persona prefixes for comedic contrast.

**Target audience:** Chinese internet users aged 18-30 who enjoy self-deprecating meme tests and have passing familiarity with xianxia tropes.

**Audience expectation:** Open the page, take the test in under three minutes on a phone, get a shareable result.

## 2. Goals & Non-goals

**Goals**

- 16 questions, three options each (上签/中签/下签)
- Dual-layer scoring: five-element dimensions (金木水火土) + persona dimensions (躺平/emo/社牛)
- 25 result entries covering five grades (天灵根/变异灵根/真灵根/伪灵根/隐灵根) plus a fallback
- Pure static site, no backend, no build system, no runtime dependencies
- Zero data exfiltration — all computation happens in the browser
- Fortune-slip visual theme (米黄宣纸 + 楷体 + 朱砂印章)
- Result page includes a programmatically drawn SVG portrait
- Mobile-first responsive layout, with desktop enhancements at ≥ 768px
- Question bank and result dictionary authored as Markdown, parsed at runtime — non-technical contributors can edit the quiz content without touching JavaScript

**Non-goals**

- Back-navigation to previous questions (enforced linear flow, matches SBTI)
- Canvas-generated long-form share images (users screenshot manually)
- User accounts, result history, or any server-side persistence
- i18n (Chinese only)
- Hand-drawn or AI-generated illustrations in v1 (programmatic SVG only; the `portrait` field in the results dictionary is designed to be later replaced by static image URLs without code changes)
- Radar charts, compatibility matching, or other档位 3 features
- marked.js, markdown-it, or any external Markdown parser — we hand-write a tiny parser specific to our DSL

## 3. Architecture

### 3.1 File tree

```
xiuxian/
├── index.html              # four <section>s: home / quiz / result / error
├── style.css               # fortune-slip theme, responsive layout
├── main.js                 # loader, parser, engine, renderer, router
├── test.html               # standalone test runner (loads main.js)
├── data/
│   ├── questions.md        # 16 questions, hand-written Markdown
│   └── results.md          # 25 result entries, hand-written Markdown
├── README.md               # setup, deployment, QA checklist
└── .gitignore
```

Every file in the repo root is directly servable by any static host. No build step, no transpile, no bundler.

### 3.2 Modules inside `main.js`

All modules live in a single file, exposed as a flat namespace (no ES modules, to preserve `file://` compatibility for debugging — though the final runtime still requires an HTTP server due to `fetch` CORS).

| Module | Exports | Responsibility |
|---|---|---|
| loader | `loadMarkdown(path)` | Fetch a `.md` file as text |
| parser | `parseQuestions(text)` → `Question[]` | DSL → array of question objects |
| parser | `parseResults(text)` → `ResultsDict` | DSL → keyed dictionary of result entries |
| engine | `score(answers, questions)` → `Scores` | Accumulate deltas across all answered options |
| engine | `classify(scores)` → `Triple` | Derive `{ 品阶, 属性, 前缀 }` from scores |
| engine | `resolveKey(triple, dict)` → `string` | Walk the fallback chain, return a key guaranteed to exist in `dict` |
| portrait | `renderPortrait(config)` → `SVGElement` | Produce a themed Taoist figure SVG from portrait config |
| render | `renderHome()` / `renderQuiz()` / `renderResult()` / `renderError(msg)` | Populate each section's DOM |
| router | `go(sectionId)` | Toggle `.active` class between sections |
| bootstrap | `main()` | Entry point: parallel fetch both MD files, parse, render home |

### 3.3 Runtime data flow

```
         main()
           │
           ├── fetch data/questions.md ──┐
           └── fetch data/results.md   ──┤
                                         ▼
                                    parse both
                                         │
                                         ▼
                               state.questions
                               state.resultsDict
                                         │
                                         ▼
                                   renderHome()
                                         │
                       user clicks "开卷测灵根"
                                         │
                                         ▼
                            renderQuiz(state.cursor)
                                         │
                 user picks option N → state.answers[cursor] = N
                                         │
                    ┌────────────────────┴────────────────────┐
                    │                                         │
               cursor < 15                              cursor === 15
                    │                                         │
                    ▼                                         ▼
            cursor++, renderQuiz()        state.classification = engine pipeline
                                                               │
                                                               ▼
                                                        renderResult()
```

Errors at any stage call `renderError(msg)` and `router.go('error')`.

### 3.4 Global state (no framework)

```js
const state = {
  questions: [],
  resultsDict: {},
  answers: [],           // length 16, each 0 | 1 | 2
  cursor: 0,             // current question index
  classification: null   // populated after last answer
};
```

All renderers read from `state`. All event handlers write to `state` and call the relevant renderer. No reactive subscriptions, no virtual DOM — just imperative updates on a small, flat object.

### 3.5 Responsive layout strategy

Mobile-first: base styles target 320-767px. One breakpoint at `@media (min-width: 768px)` upgrades for tablet and desktop. An optional second breakpoint at 1200px adds extra whitespace without restructuring.

| Page | < 768px | ≥ 768px |
|---|---|---|
| home | full-viewport centered stack, `max-width: calc(100vw - 32px)` | `max-width: 560px` centered, larger type, more vertical padding |
| quiz | progress bar → prompt → 3 stacked options (min-height 60px for fat-finger tapping) | `max-width: 640px`, prompt 18-20px, options still single-column |
| result | single column: portrait → name → fortune → five-element bars → methods → code → buttons | `max-width: 720px`, fortune text (60%) and five-element bars (40%) side-by-side; methods and code span full width below |

Touch feedback on options uses `:active` with `transform: scale(0.98)` and a brief cinnabar border flash (~150ms). Hover styles are scoped to `@media (hover: hover)` so they don't get stuck on touch devices.

### 3.6 Chinese font fallback

The fortune-slip look depends on 楷体 / 仿宋. Font stack: `"KaiTi", "STKaiti", "FangSong", "楷体", "STFangsong", serif`. macOS and Windows ship the first two; Linux typically degrades to `serif`, which the CSS explicitly accepts rather than shipping custom web fonts (keeps the bundle tiny and avoids hosting issues).

## 4. Data Model

### 4.1 Question DSL

The question bank is a single Markdown file that also renders readably on GitHub.

**Grammar (informal)**

- `# ...` — document title, ignored by parser
- Any line starting with `>` before the first `## Q<n>` — documentation, ignored
- `## Q<n>` — starts a new question; `<n>` is parsed as the question id
- `> <text>` lines inside a question block — accumulate into the prompt (joined with spaces)
- `- <label> \`<expr>\` <text>` lines inside a question block — define an option
  - `<label>` is the prefix shown in the UI (e.g. `上签`, `中签`, `下签`); parser treats anything between `- ` and the first backtick as the label, with leading/trailing whitespace and the separator `·` trimmed
  - Label may be **empty** — in that case the option renders as "甲/乙/丙" using a built-in fallback sequence, so question authors can drop the decoration if they want
  - `<expr>` is the scoring expression: `维度±数字 [ | 维度±数字 ]*`
  - `<text>` is the option body, shown as the option's main line
- Exactly three options per question
- Blank lines and stray whitespace are tolerated

**Dimension whitelist** (locked): `金 木 水 火 土 躺平 emo 社牛`

**Expression grammar**

```
expression := segment ( separator segment )*
segment    := dimension sign digit
dimension  := 金 | 木 | 水 | 火 | 土 | 躺平 | emo | 社牛
sign       := + | -
digit      := 1 | 2 | 3
separator  := " | " | ", " | ","
```

**Sample entry**

```markdown
## Q1
> 熬至子时，宗主遣信曰"辛苦"，尔心所向——

- 上签 · `金+2 | 社牛+1` "此乃分内" 闭户而归
- 中签 · `火+2 | emo+1` 截图入群 戏言"此劫如渡"
- 下签 · `水+1 | 躺平+2` 默不作声 纳手机于匣
```

**Parsed shape**

```js
{
  id: 1,
  prompt: '熬至子时，宗主遣信曰"辛苦"，尔心所向——',
  options: [
    { label: '上签', text: '"此乃分内" 闭户而归',     deltas: { 金: 2, 社牛: 1 } },
    { label: '中签', text: '截图入群 戏言"此劫如渡"', deltas: { 火: 2, emo: 1 } },
    { label: '下签', text: '默不作声 纳手机于匣',     deltas: { 水: 1, 躺平: 2 } }
  ]
}
```

**Parser failure modes** — every failure throws with a location string so the error page can tell the author exactly what to fix:

- Expression does not match the grammar → `"Q3 选项 2 分数表达式不合法: 金+"`
- Dimension not in whitelist → `"Q5 出现未知维度 '暗'"`
- Option count ≠ 3 → `"Q7 共 2 个选项，期望 3"`
- Question has no prompt → `"Q4 缺少题干"`

Soft warnings (logged to console, do not block rendering):

- Total question count ≠ 16

### 4.2 Result DSL

Same Markdown philosophy, richer per-entry structure.

**Grammar (informal)**

- `# ...` — ignored title
- `## <display name>` — starts a result entry; the display name is what appears as the result page heading
- `<key>: <value>` lines immediately below `##` — flat metadata fields (`key`, `品阶`, `属性`, `前缀`, `结果代码`)
- `### 签文` — fortune paragraph (one or more paragraphs of prose)
- `### 推荐功法` — bullet list of 2-4 method names
- `### 画像` — `<key>: <value>` pairs describing the SVG portrait (`袍色`, `配饰`, `背景`, `符文色`)

**Sample entry**

```markdown
## 金水真灵根
key: zhenling_金水_default
品阶: 真灵根
属性: 金水
前缀: 无
结果代码: ZL-金水

### 签文
尔心如刃入于水，锋芒藏于涟漪。遇逆则锐，遇柔则融——金生水涤，水磨金锋，此乃修为两相成之道。然尔常困于"当拔剑乎？当退一步乎？"之纠结，修为虽稳，进境却缓。宗门长老批曰："此子根骨尚可，唯缺一剑决断。"

### 推荐功法
- 《剑气化雾诀》
- 《锋芒自敛心法》
- 《纠结退散篇》

### 画像
袍色: #c4c9d4
配饰: sword-water
背景: mist-waves
符文色: #6fa8dc
```

**Parsed shape**

```js
{
  'zhenling_金水_default': {
    displayName: '金水真灵根',
    key: 'zhenling_金水_default',
    品阶: '真灵根',
    属性: '金水',
    前缀: '无',
    结果代码: 'ZL-金水',
    签文: '尔心如刃入于水，锋芒藏于涟漪...',
    推荐功法: ['《剑气化雾诀》', '《锋芒自敛心法》', '《纠结退散篇》'],
    画像: {
      袍色: '#c4c9d4',
      配饰: 'sword-water',
      背景: 'mist-waves',
      符文色: '#6fa8dc'
    }
  }
  // ...24 more entries
}
```

The `default_default_default` entry MUST exist; the parser throws if it does not.

## 5. Scoring Algorithm

### 5.1 Dimensions

Eight total, always present in every scores object:

```
五行: 金 木 水 火 土
人设: 躺平 emo 社牛
```

Expected per-dimension range after 16 answers: 0-7 (derived from 16 × average 1.5 delta points spread across 8 dimensions).

### 5.2 `score(answers, questions)`

```js
function score(answers, questions) {
  if (answers.length !== questions.length) {
    throw new Error(`测试数据异常：已答 ${answers.length} 题，期望 ${questions.length} 题`);
  }
  const s = { 金:0, 木:0, 水:0, 火:0, 土:0, 躺平:0, emo:0, 社牛:0 };
  answers.forEach((choice, i) => {
    const option = questions[i].options[choice];
    for (const [k, v] of Object.entries(option.deltas)) {
      s[k] += v;
    }
  });
  return s;
}
```

The length guard at the top catches any state corruption (e.g. `再测一次` not clearing `answers` properly) before it silently produces a wrong classification. See §9 for how this bubbles up to the error page.

### 5.3 `classify(scores)` — grade determination

The five elements are sorted descending; `top`, `second`, and `total` refer to values among the five-element dimensions only. Rules are evaluated **in order**; the first match wins.

| Priority | Rule | Grade (品阶) | Attribute (属性) |
|---|---|---|---|
| 1 | `fiveElementTotal ≤ 3` AND `max(躺平, emo, 社牛) ≥ 5` | `yinling` (隐灵根) | `隐` |
| 2 | `top ≥ 5` AND `second ≤ 1` | `tianling` (天灵根) | the top element, e.g. `金` |
| 3 | `top ≥ 4` AND `second ≥ 3` AND the `{top, second}` pair is in `BIAN_COMBO` | `bianling` (变异灵根) | `雷` / `冰` / `风` / `暗` |
| 4 | `top ≥ 3` AND the top two elements each ≥ 2 | `zhenling` (真灵根) | sorted-join of top two, e.g. `金水` |
| 5 | else | `weiling` (伪灵根) | `混杂` |

**BIAN_COMBO table** (lore-accurate):

```js
const BIAN_COMBO = {
  '金_水': '雷',
  '水_土': '冰',
  '木_火': '风',
  '金_火': '暗'
};
// Pair key: two element names sorted into lexicographic order joined with '_'
```

### 5.4 Prefix determination

Look only at the three persona dimensions.

```js
function determinePrefix(scores) {
  const persona = ['躺平', 'emo', '社牛'];
  const sorted = [...persona].sort((a, b) => scores[b] - scores[a]);
  const top = sorted[0];
  return scores[top] >= 3 ? top : '钝感';
}
```

Four possible prefixes: `躺平` / `emo` / `社牛` / `钝感`.

### 5.5 `resolveKey(triple, dict)` — fallback chain

Given `triple = { 品阶, 属性, 前缀 }`, walk this ordered list and return the first key present in `dict`:

```
1. ${品阶}_${属性}_${前缀}
2. ${品阶}_${属性}_default
3. ${品阶}_default_${前缀}
4. ${品阶}_default_default
5. default_default_default
```

Per the catalog in §6:

- 天灵根 / 变异灵根 entries are keyed as `${grade}_${attr}_default` → step 2 matches
- 真灵根 entries are keyed as `zhenling_${combo}_default` → step 2 matches
- 伪灵根 entries are keyed as `weiling_default_${prefix}` → step 3 matches
- 隐灵根 is keyed as `yinling_default_default` → step 4 matches
- `default_default_default` exists as final safety net → step 5 matches when nothing else does

Steps 1 and 4 (for graded catalogs) are rarely hit but kept in the chain so future authors can add exotic精确 entries (e.g. `tianling_金_躺平`) without changing code.

### 5.6 Tunable constants

```js
const PREFIX_THRESHOLD = 3;            // §5.4
const TIANLING_TOP = 5;                // §5.3 rule 2
const TIANLING_SECOND_MAX = 1;         // §5.3 rule 2
const BIANLING_TOP = 4;                // §5.3 rule 3
const BIANLING_SECOND = 3;             // §5.3 rule 3
const YINLING_TOTAL_MAX = 3;           // §5.3 rule 1
const YINLING_PERSONA_MIN = 5;         // §5.3 rule 1
```

These are grouped at the top of `main.js` for easy retuning after running simulation batches (see §10.2).

## 6. Result Catalog

25 entries total. Display name, key, and a one-line persona are listed here; the full 签文 / 推荐功法 / 画像 content is authored in `data/results.md` during implementation.

### 6.1 天灵根 × 5

| # | Display Name | Key | Persona hook |
|---|---|---|---|
| 1 | 金天灵根 | `tianling_金_default` | 锋芒毕露，心如铸剑 |
| 2 | 木天灵根 | `tianling_木_default` | 温润如春，生生不息 |
| 3 | 水天灵根 | `tianling_水_default` | 静而能深，柔以化万物 |
| 4 | 火天灵根 | `tianling_火_default` | 炽烈纯粹，热情不熄 |
| 5 | 土天灵根 | `tianling_土_default` | 沉稳守一，万物之基 |

### 6.2 变异灵根 × 4

| # | Display Name | Key | Persona hook |
|---|---|---|---|
| 6 | 雷灵根 | `bianling_雷_default` | 金水激荡，一念生雷 |
| 7 | 冰灵根 | `bianling_冰_default` | 水土共凝，外冷内定 |
| 8 | 风灵根 | `bianling_风_default` | 木火生风，无迹可寻 |
| 9 | 暗灵根 | `bianling_暗_default` | 金火相杀，阴翳狠辣 |

### 6.3 真灵根 × 10

| # | Display Name | Key | Persona hook |
|---|---|---|---|
| 10 | 金木真灵根 | `zhenling_金木_default` | 刚柔并济，能屈能伸 |
| 11 | 金水真灵根 | `zhenling_金水_default` | 锋藏涟漪，纠结派剑客 |
| 12 | 金火真灵根 | `zhenling_金火_default` | 刀光火影，杀伐果决 |
| 13 | 金土真灵根 | `zhenling_金土_default` | 沉金厚土，老派靠谱人 |
| 14 | 木水真灵根 | `zhenling_木水_default` | 草木得泽，佛系生长 |
| 15 | 木火真灵根 | `zhenling_木火_default` | 烈日焚林，热血理想派 |
| 16 | 木土真灵根 | `zhenling_木土_default` | 扎根立基，大器晚成 |
| 17 | 水火真灵根 | `zhenling_水火_default` | 水火同煎，内耗大师 |
| 18 | 水土真灵根 | `zhenling_水土_default` | 润土无声，老好人 |
| 19 | 火土真灵根 | `zhenling_火土_default` | 炉火凝土，稳而不燥 |

真灵根 entries whose attribute pair also exists in `BIAN_COMBO` (#11 金水, #12 金火, #15 木火, #18 水土) are reachable only when the user's top/second scores are large enough to be in the "真灵根" rule band but not large enough for "变异灵根" — this is intentional: 变异灵根 should feel rare.

### 6.4 伪灵根 × 4

| # | Display Name | Key | Persona hook |
|---|---|---|---|
| 20 | 躺平伪灵根 | `weiling_default_躺平` | 五行杂而无一精，平生所爱唯床笫 |
| 21 | emo 伪灵根 | `weiling_default_emo` | 修为无望道心未成，却已先会发疯 |
| 22 | 社牛伪灵根 | `weiling_default_社牛` | 灵根驳杂，但朋友遍修仙界 |
| 23 | 钝感伪灵根 | `weiling_default_钝感` | 一切无感，苦痛亦如耳旁风 |

### 6.5 隐灵根 × 1

| # | Display Name | Key | Persona hook |
|---|---|---|---|
| 24 | 隐灵根 | `yinling_default_default` | 凡圣相交，时隐时现。恭喜测出稀有彩蛋 |

### 6.6 Fallback × 1

| # | Display Name | Key | Persona hook |
|---|---|---|---|
| 25 | 无根之人 | `default_default_default` | 尔似乎未能感应天地灵气，许是测试偏差——或许该回家了 |

## 7. SVG Portrait Module

### 7.1 Config shape

```js
{
  袍色: '#4a7ab8',        // any hex color
  配饰: 'water-gourd',    // enum
  背景: 'waves',          // enum
  符文色: '#6fa8dc'       // any hex color
}
```

### 7.2 Enum values (v1)

**配饰** (held item or accessory): `sword` · `staff` · `gourd` · `sword-water` · `cauldron` · `seal` · `fan` · `banner` · `broken-sword` · `pillow` · `wine-gourd` · `wooden-fish` · `nothing`

**背景**: `waves` · `mist-waves` · `grass` · `flames` · `rocks` · `lightning` · `ice-flowers` · `wind` · `dark-clouds` · `void` · `quilt` · `night-rain` · `crowd` · `blank`

### 7.3 Implementation approach

A base 240×320 SVG template contains:

- A rice-paper colored background rect (`#f4ead3`)
- A seated Taoist figure silhouette (single compound `<path>`, CSS `fill: var(--pao-color)`)
- All possible accessory `<g>` elements with `display: none` by default, each with `id="acc-<enum>"`
- All possible background `<g>` elements with `display: none` by default, each with `id="bg-<enum>"`
- Decorative rune marks using `stroke: var(--fu-color)`

`renderPortrait(config)` clones the template, sets the two CSS custom properties on the root element, then flips `display: block` on the matching `acc-*` and `bg-*` groups.

The template SVG itself is embedded as a JS template literal at the top of `main.js` to avoid an extra HTTP request.

### 7.4 Forward compatibility

The `portrait` object is deliberately a flat POJO. To swap programmatic SVG for a static image in the future, the results MD can be changed to:

```markdown
### 画像
image: assets/portraits/zhenling-jinshui.png
```

and `renderPortrait` can be extended with a one-line check: `if (config.image) return <img src={config.image}>`. No other code changes required.

## 8. Pages & Interaction

### 8.1 State machine

```
          start                 finish 16
  [home] ───────▶ [quiz] ──────────────▶ [result]
     ▲               │                       │
     │               │ md error              │
     │               ▼                       │
     │           [error]                     │
     │                                       │
     └────────── 再测一次 ──────────────────┘
```

All four sections live in `index.html`. Transitions toggle an `.active` class via `router.go(sectionId)`. No URL hash routing in v1 (keeps things boring and reliable).

### 8.2 Home section

Content:

- H1 `测一测你是什么灵根（凡人修仙传）` — large 楷体, 朱砂 color
- One-line description `16 题测出你在修仙界的资质品阶。数据完全本地计算，不回传。`
- Primary button `开卷测灵根`
- Footer `华科开放原子开源俱乐部 出品`

Error mode: description replaced with the error message, button disabled.

### 8.3 Quiz section

Layout (mobile):

```
┌─────────────────────────────┐
│  第 N 问 / 共 16            │   small label
│  ████████░░░░░░░░  31%      │   cinnabar progress bar
│                             │
│  (prompt, 18-20px 楷体)     │
│                             │
│  ┌──────────────────────┐   │
│  │ 上签                 │   │   cinnabar label
│  │ (option text)        │   │   ink-black body
│  └──────────────────────┘   │
│  ┌──────────────────────┐   │
│  │ 中签                 │   │
│  │ (option text)        │   │
│  └──────────────────────┘   │
│  ┌──────────────────────┐   │
│  │ 下签                 │   │
│  │ (option text)        │   │
│  └──────────────────────┘   │
└─────────────────────────────┘
```

Interactions:

- Click option → write to `state.answers[cursor]` → 300ms fade-out → `cursor++` → fade-in next question
- Keyboard: `1` / `2` / `3` select options, `Enter` confirms focused option (a11y). No back-navigation.
- Touch: press state uses `transform: scale(0.98)` + cinnabar border flash (~150ms)
- After the 16th answer: call `score` → `classify` → `resolveKey`, write `state.classification`, `router.go('result')`

### 8.4 Result section

Mobile single column:

```
┌─────────────────────────────┐
│      [SVG 立像 ~180px]      │
│                             │
│     (灵根名, large 朱砂)    │
│     品阶 · 真灵根            │
│                             │
│  ─── 签文 ───               │
│  (签文 paragraph)           │
│                             │
│  ─── 五行 ───               │
│  金  ████████░░  6          │
│  木  ██░░░░░░░░  1          │
│  水  ███████░░░  5          │
│  火  █░░░░░░░░░  0          │
│  土  ██░░░░░░░░  1          │
│                             │
│  ─── 推荐功法 ───           │
│  《...》                    │
│  《...》                    │
│  《...》                    │
│                             │
│  ─── 结果代码 ZL-金水 ───  │  (tap to copy)
│                             │
│  [ 再测一次 ]               │
│  [ 截图发给好友 → ]         │
└─────────────────────────────┘
```

Desktop (≥ 768px): 签文 and 五行 bars render in a two-column flex row (60% / 40%), methods and code span full width below, portrait floats at the top spanning both columns.

Interactions:

- Tap 结果代码 → `navigator.clipboard.writeText(...)` + toast `已复制，快去炫耀`. Copy text format: `{结果代码} | {灵根名}` (e.g. `ZL-金水 | 金水真灵根`)
- `再测一次` → `state.answers = []`, `state.cursor = 0`, `state.classification = null` → `router.go('home')`
- `截图发给好友` → show a toast with platform-specific screenshot instructions (iOS: `侧键+音量上`; Android: `电源+音量下`; macOS: `Cmd+Shift+4`; Windows: `Win+Shift+S`); detection via `navigator.userAgent` (one-time read at boot, cached)
- 五行 bars: width percentage = `min(scores[dim] / 7, 1) * 100%` clamped to [0, 100]

### 8.5 Error section

Single visual error page reused for all failure modes:

```html
<section id="error">
  <div class="scroll-frame">
    <h2>签文解读失败</h2>
    <p class="error-msg"><!-- set by renderError(msg) --></p>
    <p class="subtitle">宗门弟子请检查卷轴——如果你是直接双击打开的 index.html，请改用 python3 -m http.server 启动本地服务器。</p>
    <button onclick="location.reload()">重试</button>
  </div>
</section>
```

The visual theme stays consistent with the quiz (no jarring "white background stack trace" moment).

## 9. Error Handling

| Scenario | Where detected | User-facing message (set via `renderError`) |
|---|---|---|
| `fetch` fails (CORS / 404 / network) | `loadMarkdown` throws | `题库加载失败：data/questions.md` |
| Question expression malformed | `parseQuestions` throws | `题库格式错误：Q3 选项 2 的分数表达式 "金+" 缺数字` |
| Dimension not in whitelist | `parseQuestions` throws | `题库格式错误：Q5 出现未知维度 "暗"` |
| Option count ≠ 3 | `parseQuestions` throws | `题库格式错误：Q7 共 2 个选项，期望 3` |
| Question has no prompt | `parseQuestions` throws | `题库格式错误：Q4 缺少题干` |
| Missing `default_default_default` in results dict | `parseResults` throws | `结果字典格式错误：缺少 default_default_default 兜底条目` |
| Result entry missing required metadata | `parseResults` throws | `结果字典格式错误：条目 "金水真灵根" 缺少 key 字段` |
| `answers.length ≠ questions.length` at classify time | `score` throws (§5.2 guard) | `测试数据异常，请重新开始` (should never happen; guards against state-reset bugs) |

All errors bubble to the top-level `main()` via `try/catch` → `renderError(msg)` → `router.go('error')`. Nothing fails silently to the console.

Soft warnings (console only, do not trigger error page):

- Question count ≠ 16
- Unused results dictionary entries (logged as a courtesy to authors)

## 10. Testing Strategy

### 10.1 Unit tests — `test.html`

Standalone HTML file that shares `main.js` with production. Open in any browser to run tests; results render as a pass/fail list in the body.

```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <title>Lingen Test · 测试</title>
  <style>body{font-family:monospace;padding:20px} .ok{color:#080} .fail{color:#c00}</style>
</head>
<body>
  <h1>Lingen Test Unit Tests</h1>
  <div id="results"></div>
  <script src="main.js"></script>
  <script>
    const tests = [];
    function test(name, fn) { tests.push({ name, fn }); }
    function assert(cond, msg) { if (!cond) throw new Error(msg || 'assertion failed'); }

    test('parseQuestions: single question', () => {
      const md = `## Q1\n> 题干\n- 上签 \`金+2\` 文本A\n- 中签 \`木+1\` 文本B\n- 下签 \`水+1\` 文本C`;
      const result = window.parseQuestions(md);
      assert(result.length === 1);
      assert(result[0].options[0].deltas['金'] === 2);
      assert(result[0].options[0].label === '上签');
    });

    // ...more tests

    window.onload = () => {
      const root = document.getElementById('results');
      let pass = 0, fail = 0;
      tests.forEach(t => {
        try { t.fn(); pass++; root.innerHTML += `<div class="ok">✓ ${t.name}</div>`; }
        catch (e) { fail++; root.innerHTML += `<div class="fail">✗ ${t.name}: ${e.message}</div>`; }
      });
      root.innerHTML += `<hr><div>Passed: ${pass} / ${pass + fail}</div>`;
    };
  </script>
</body>
</html>
```

**Target coverage:** 15-20 test cases

- `parseQuestions`: happy path, separator variants (`|`, `,`), bad expression, missing option, unknown dimension, missing prompt
- `parseResults`: happy path, metadata extraction, list parsing, missing default fallback
- `score`: accumulation correctness, dimensions absent from deltas stay at 0
- `classify`: all five grade branches explicitly covered; boundary cases (tied top/second, threshold edges); 隐灵根 trigger
- `resolveKey`: each fallback chain step hit at least once

### 10.2 Integration tests — simulated answer sheets

Inside the same `test.html`, run a handful of fixed-answer scenarios end-to-end:

```js
test('integration: all 上签 → expected grade', () => {
  const answers = Array(16).fill(0);
  const scores = window.score(answers, FIXTURE_QUESTIONS);
  const triple = window.classify(scores);
  const key = window.resolveKey(triple, FIXTURE_RESULTS);
  assert(FIXTURE_RESULTS[key], 'resolved key must exist');
});
```

`FIXTURE_QUESTIONS` and `FIXTURE_RESULTS` are small inline fixtures, not the real 16 / 25. Production data gets checked by the QA process, not automated tests (because content, not logic, is what they cover).

### 10.3 Tuning simulation (optional, authoring-time tool)

After the full question bank is written, add a `simulate.html` utility (out of scope for v1 but noted here) that fires 10,000 random answer sheets through the pipeline and prints the distribution across 25 result keys. If any bucket is empty or a single bucket dominates, the `PREFIX_THRESHOLD` / `TIANLING_TOP` constants get retuned.

### 10.4 Manual QA checklist

Lives in `README.md`, gets ticked off before any release:

- [ ] Chrome DevTools iPhone SE 375px: layout correct, font fallback visible, no overflow
- [ ] Chrome DevTools iPad 768px: result page renders side-by-side columns
- [ ] Desktop 1280px: content `max-width` centers, hover styles work
- [ ] Keyboard: `1` / `2` / `3` select, `Enter` confirms, no back nav
- [ ] Intentionally corrupt `data/questions.md` (remove a backtick) → error page shows a readable message
- [ ] Open `index.html` via `file://` → error page tells user to start an HTTP server
- [ ] `python3 -m http.server` → full test flow works end-to-end
- [ ] `navigator.clipboard.writeText` path works in Chrome / Safari / Firefox
- [ ] Linux + serif fallback: page does not collapse
- [ ] One simulated answer sheet per grade (5 total) produces the expected result entry

## 11. Deployment

**Requirements:** none. The repo is a static site; any static host works.

**Recommended targets:** GitHub Pages, Cloudflare Pages, Vercel (static), or a self-hosted `nginx` virtual host.

**Local development:** `python3 -m http.server` in the repo root; visit `http://localhost:8000`. Direct `file://` double-click is **not** supported because `fetch` against the local filesystem is blocked by CORS. The error page explicitly tells users this.

**README.md sections:**

1. 一句话介绍
2. 在线访问链接（部署后填）
3. 本地运行（`python3 -m http.server`）
4. 修改题库 / 结果字典的指南（指向 `data/*.md` 的 DSL 文档）
5. QA 清单（§10.4）
6. 署名与许可证

## 12. Open Questions / Future Work

Not blocking v1 release, noted for posterity:

- **Distribution tuning**: §10.3's `simulate.html`. Needed if real-world analytics reveal skew.
- **AI-generated portraits**: Replace programmatic SVG via the `portrait.image` escape hatch described in §7.4.
- **Additional variant roots**: 光灵根 / 空间灵根 from later volumes of the novel. Requires adding dimensions or a new grade rule.
- **Share cards**: Canvas-generated PNG export for sharing (档位 3 feature, explicitly deferred).
- **Internationalization**: The quiz is culturally specific to Chinese internet humor; an English port would require a full content rewrite, not just translation.
- **Analytics**: Any telemetry must be opt-in and clearly disclosed on the home page, to preserve the "no data exfiltration" promise. v1 ships with none.
