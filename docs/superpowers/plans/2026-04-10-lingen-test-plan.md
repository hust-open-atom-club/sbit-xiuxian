# Lingen Test — Implementation Plan

## Goal Description

Implement a zero-dependency static web quiz that faithfully realizes the 744-line design spec at `docs/superpowers/specs/2026-04-10-lingen-test-design.md`. The site presents 16 fortune-slip questions, scores answers across five elemental dimensions (金木水火土) and three persona dimensions (躺平/emo/社牛), classifies the user into one of 25 spiritual root categories using a deterministic rule chain with fallback resolution, and displays a themed result page with a programmatically rendered SVG portrait, signed fortune text, recommended methods, and a shareable result code. The site is authored by the 华科开放原子开源俱乐部, ships as pure HTML/CSS/JS with content files in Markdown, requires no build step, no runtime dependencies, and runs under any static host via `python3 -m http.server`.

## Acceptance Criteria

Following TDD philosophy, each criterion includes positive and negative tests for deterministic verification.

- AC-1: Deterministic scoring and classification pipeline
  - Positive Tests (expected to PASS):
    - Given the same fixture questions, results dictionary, and answer sequence, three consecutive invocations of `score()` → `classify()` → `resolveKey()` return the same triple and key
    - A five-grade fixture set (one answer sheet per grade: tianling / bianling / zhenling / weiling / yinling) each resolves to a key whose entry exists in the dictionary
    - Tie-break sort preserves the declared dimension order (金木水火土 for five-elements; 躺平 emo 社牛 for persona) when two dimensions have equal scores, and this is verified by a fixture with a deliberate tie
  - Negative Tests (expected to FAIL):
    - Calling `score()` with an `answers` array whose length differs from `questions.length` throws an error containing both numbers
    - Two fixtures that differ only in the answer ordering (same multiset of answers, different sequence) must still resolve to the same key; reordering is rejected as a bug trigger

- AC-2: Question DSL parser safety
  - AC-2.1: Option-level diagnostic errors
    - Positive: A valid option `` - 上签 · `金+2 | 社牛+1` 文本 `` is parsed into `{ label: '上签', deltas: { 金: 2, 社牛: 1 }, text: '文本' }`
    - Positive: Expression separators ` | `, `, `, and `,` are all accepted
    - Negative: `` `金+` `` throws an error whose message names the offending question id, option index, and the raw expression snippet
    - Negative: `` `暗+2` `` (未知维度) throws an error naming question id, option index, and the unknown dimension
    - Negative: An option line missing its backtick pair throws an error naming question id and option index
  - AC-2.2: Question-level diagnostic errors
    - Positive: A question block with `## Q1`, one `>` prompt, and exactly three `-` options parses without error
    - Negative: A question block with zero `>` prompt lines throws an error naming the question id only (no option index)
    - Negative: A question block with two or four options throws an error naming the question id and the actual count
    - Negative: A question whose `## Q<n>` header is malformed (missing n) throws a header parse error
  - AC-2.3: Production question data
    - Positive: Loading the real `data/questions.md` yields exactly 16 Question objects with well-formed prompts, labels, deltas, and text
    - Negative: Intentionally removing one backtick from any production question causes the test runner to report the specific question id

- AC-3: Result dictionary DSL parser safety
  - AC-3.1: Entry-level diagnostic errors
    - Positive: A valid entry with `##` display name, `key` / `品阶` / `属性` / `前缀` / `结果代码` metadata, `### 签文`, `### 推荐功法`, and `### 画像` sub-sections parses into a complete entry object
    - Negative: A duplicate `key` across two entries throws an error naming both display names
    - Negative: A duplicate display name across two entries throws an error
    - Negative: A duplicate `结果代码` across two entries throws an error
    - Negative: Missing `default_default_default` entry in the parsed dictionary throws a fatal error
    - Negative: An entry missing required `key` metadata throws an error naming the display name
    - Negative: A `画像.袍色` field containing an invalid hex color (e.g. `#notacolor`) throws an error
    - Negative: A `画像.配饰` or `画像.背景` value not in the whitelisted enum throws an error naming the illegal value and the entry
  - AC-3.2: Production result data
    - Positive: Loading the real `data/results.md` yields exactly 25 entries including `default_default_default`, with `displayName`, `key`, `品阶`, `属性`, `前缀`, `结果代码`, `签文`, `推荐功法`, and `画像` fields all populated

- AC-4: Authoring-time content lint in test runner
  - Positive Tests:
    - Opening `test.html` in a browser loads production `data/questions.md` and `data/results.md` and asserts: exactly 16 questions parse, `default_default_default` exists, every fixture answer sheet for the five grades resolves successfully, and every `画像.配饰` and `画像.背景` value across all 25 entries is in the whitelisted enum
    - All assertions surface as `✓` pass lines in the test runner output
  - Negative Tests:
    - If an author sets the question count to 15, `test.html` shows `✗ question count 15, expected 16` with the file path
    - If an author references an accessory not in the whitelist, `test.html` shows the entry display name and the illegal enum value

- AC-5: Test isolation between production and test entry points
  - Positive Tests:
    - Opening `test.html` does not trigger production `main()` bootstrap; no `fetch` for production data files fires unless a test case explicitly calls it
    - Production entry (index.html) and test entry (test.html) each work independently when opened in separate tabs
    - `main.js` exposes `parseQuestions`, `parseResults`, `score`, `classify`, `resolveKey`, `renderPortrait`, and other public functions on a named namespace that tests can import
  - Negative Tests:
    - If `main.js` is accidentally refactored to auto-invoke `main()` at module top level, the test runner reports unexpected network activity or DOM mutation and fails a dedicated isolation test

- AC-6: Responsive flow across mobile and desktop
  - Positive Tests:
    - At 320px viewport width, the full 16-question flow can be completed without horizontal overflow, clipped text, or inaccessible controls
    - At 375px (iPhone SE) and 393px (iPhone 14 Pro), the quiz page renders correctly with three vertically stacked options, each with `min-height: 60px`
    - At 768px (iPad) viewport, the result page renders the 签文 paragraph and the five-element bars in a side-by-side two-column layout
    - At 1280px desktop, mouse hover feedback on options is visible and does not stick to touch devices
  - Negative Tests:
    - Touch input on a mobile emulator does not leave a stuck `:hover` state after tap release
    - Landscape 320px tall by 568px wide still permits scrolling without overflow

- AC-7: Fortune-slip visual theme consistency
  - Positive Tests:
    - Home, quiz, result, and error sections all use the rice-paper yellow background, KaiTi font (via the declared fallback chain), and cinnabar accents for emphasis elements
    - The declared fallback chain `"KaiTi", "STKaiti", "FangSong", "楷体", "STFangsong", serif` appears in the CSS font stack
  - Negative Tests:
    - On a Linux system without KaiTi or STKaiti installed, the page degrades gracefully to `serif` without broken layout, missing glyphs, or overflow
    - No web font is loaded from any remote origin

- AC-8: SVG portrait module coverage
  - Positive Tests:
    - `renderPortrait(config)` returns a valid `SVGElement` (or SVG string) for every `配饰` enum value listed in the spec
    - It returns a valid element for every `背景` enum value
    - CSS custom properties `--pao-color` and `--fu-color` are set on the returned root element from `config.袍色` and `config.符文色`
    - Forward-compatibility: if `config.image` is set, `renderPortrait` returns an `<img>` element pointing at the image instead of the SVG template
  - Negative Tests:
    - Passing a `配饰` value not in the enum throws a clear error naming the illegal value
    - Passing a malformed hex color throws an error

- AC-9: Unified error handling through the dedicated error section
  - Positive Tests:
    - Any failure during `fetch` (CORS, 404, network error) routes to the `error` section with a readable Chinese message identifying the failing file
    - Any parser error produces an error page whose message pinpoints the failing question id or entry display name
    - Missing `default_default_default` produces a fatal error routed to the error section
    - `answers.length` mismatch in `score()` produces a fatal error routed to the error section
  - Negative Tests:
    - The home section does NOT carry a separate "disabled error mode"; the self-contradiction in the draft is resolved by eliminating the home-level error state
    - No failure path silently reaches a half-loaded quiz or half-loaded result

- AC-10: Score display semantics for five-element bars
  - Positive Tests:
    - Bar width for `score = 0` is 0%
    - Bar width for `score = 7` is 100%
    - Bar width for `score = 5` is approximately 71.4% (= 5/7 * 100%)
    - Negative scores display as 0% (clamped)
    - Scores above 7 display as 100% (clamped)
    - The formula `max(0, min(score, 7)) / 7 * 100%` is documented in a comment near the render site
  - Negative Tests:
    - The code does NOT attempt to normalize bars against the user's own max score of the day (i.e. no relative rescaling)
    - The code does NOT render negative-going bars or dual-direction bipolar bars

- AC-11: Distribution sanity as a soft gate
  - Positive Tests:
    - Five hand-authored fixture answer sheets, one per grade, cover all five grade branches of `classify()`
    - Every fixture's resolved key points to an entry that exists in the production results dictionary
    - Each fixture's grade assignment matches an expected value declared in the fixture itself
  - Negative Tests:
    - A 10,000-sample random simulate run is NOT required for v1 release (per DEC-4 resolution "C")
    - Distribution calibration via `simulate.html` and threshold retuning (`task10` / `task11`) remain optional backlog items; they do not block v1 release

- AC-12: Zero-dependency deploy-ready package
  - Positive Tests:
    - Running `python3 -m http.server` in the repo root and visiting `http://localhost:8000` loads the full app and allows completing the quiz
    - No `<script src="http...">` or other remote resource reference exists in any HTML, CSS, or JS file
    - No `package.json`, `node_modules`, or build artifact exists in the deployed tree
    - The project deploys unchanged to GitHub Pages, Cloudflare Pages, Vercel, or any nginx virtual host
  - Negative Tests:
    - Opening `index.html` directly via `file://` protocol (double-click) does NOT silently crash; it shows the documented error page explaining how to start a local HTTP server
    - No external Markdown parser (marked.js, markdown-it) appears anywhere in the tree

- AC-13: Manual QA checklist present and actionable
  - Positive Tests:
    - `README.md` contains the QA checklist as enumerated in the spec §10.4 (at least ten items covering iPhone SE layout, iPad two-column, desktop hover, keyboard, corruption handling, file:// open, HTTP server run, clipboard across browsers, font fallback, per-grade fixture verification)
    - Every item is phrased as a check action (`- [ ]` style or equivalent) with a clear pass criterion
  - Negative Tests:
    - The checklist does NOT contain vague items like "ensure it works" without an observable outcome

- AC-14: Keyboard interaction on the quiz page
  - Positive Tests:
    - Pressing `1`, `2`, or `3` on the quiz section selects the corresponding option and advances to the next question exactly as a click would
    - Pressing `Enter` confirms the currently focused option (whichever option has keyboard focus at that moment), not a default option
    - Tab-navigable focus ring is visible on the focused option (via `:focus-visible`) so keyboard users can see where they are
  - Negative Tests:
    - Pressing `←` does NOT navigate back to the previous question (back-navigation is a spec non-goal)
    - Pressing keys other than `1`/`2`/`3`/`Enter`/`Tab` does not accidentally select an option

- AC-15: Result page actions (copy, screenshot hint, retry)
  - Positive Tests:
    - Tapping (click or touch) the 结果代码 element invokes `navigator.clipboard.writeText()` with `"{结果代码} | {灵根名}"` (e.g. `"ZL-金水 | 金水真灵根"`), and a toast shows "已复制，快去炫耀"
    - The `截图发给好友` button shows platform-specific screenshot instructions (iOS: 侧键+音量上; Android: 电源+音量下; macOS: Cmd+Shift+4; Windows: Win+Shift+S) when `navigator.userAgent` resolves to a known platform
    - Clicking `再测一次` clears `state.answers`, resets `state.cursor` to 0, clears `state.classification`, and routes to the home section
  - Negative Tests:
    - If `navigator.clipboard` is unavailable (legacy browser, insecure context), the toast falls back to "请手动选中并复制" and selects the result code text node so the user can copy manually; the app does not crash
    - If `navigator.userAgent` resolves to an unrecognized platform, the screenshot button shows a generic instruction ("请用系统截图快捷键截图") rather than throwing

## Path Boundaries

Path boundaries define the acceptable range of implementation quality and choices. Because the draft is a highly deterministic design, upper and lower bounds sit close together; most choices are fixed by the spec.

### Upper Bound (Maximum Acceptable Scope)

The implementation contains all 16 questions and all 25 result entries in Markdown-authored content files, a zero-dependency hand-written Markdown parser implementing the full DSL, the scoring engine with locked declared-order tie-break, a programmatic SVG portrait module covering every accessory and background enum with forward-compatible `config.image` escape hatch, responsive layouts covering 320 / 375 / 768 / 1280 viewport widths, complete keyboard interaction (1/2/3 + Enter + focus-visible), clipboard copy with toast and graceful fallback, platform-aware screenshot hints with a generic fallback, runtime-fatal unified error page for every load/parse failure, authoring-time content lint in `test.html` that loads real production data and enforces dimension whitelist / enum whitelist / exact count / default entry presence, a per-entry persona seal icon overlay on the result page (per DEC-5 resolution B), and a `README.md` containing deployment instructions, the manual QA checklist, and guidance for editing `data/*.md`.

### Lower Bound (Minimum Acceptable Scope)

The implementation contains all 16 questions and all 25 result entries, a parser that correctly parses the production data files without errors, a scoring engine that hits all five grade branches under fixture testing with locked tie-break, a minimal programmatic SVG portrait that renders a consistent silhouette with distinguishable accessory and background, mobile responsive layout without overflow at 320px, desktop layout that is readable (single-column acceptable at the minimum), keyboard support for `1`/`2`/`3` selection (Enter confirm acceptable optional at the minimum), clipboard copy with a toast and fallback selection for legacy browsers, platform-generic screenshot hint (per-platform detection acceptable optional), unified error section covering fetch / parse / missing fallback failures, `test.html` with unit tests that run without bootstrapping production, a per-entry persona seal icon on the result page (still required), and a `README.md` with deployment instructions and the QA checklist.

### Allowed Choices

Can use:
- Vanilla JS (ES2019 or newer baseline; optional chaining, `Object.fromEntries`, async/await), HTML5, CSS3, inline SVG template literal embedded in JS
- Native Web APIs: `fetch`, `navigator.clipboard`, `navigator.userAgent`, `document.querySelector`, `classList`, CSS custom properties, `@media` queries
- Hand-written tiny Markdown DSL parser (≤ 150 LOC total for both parsers combined)
- Either a monolithic `main.js` exposing functions on a `window.LingenTest` namespace, or a 2-4 file split under `js/` (e.g. `parser.js` / `engine.js` / `render.js` / `main.js`) at the implementer's discretion
- `<script src="js/...">` with plain script tags, not ES modules (to avoid the `file://` CORS module loading pitfall called out in the draft)

Cannot use:
- marked.js, markdown-it, or any external Markdown parser
- React, Vue, Svelte, Angular, Solid, or any component framework
- Build steps (webpack, vite, rollup, esbuild, parcel), transpilers (Babel, TypeScript), or bundlers
- External CSS frameworks (Bootstrap, Tailwind, Bulma) or CSS-in-JS
- Remote runtime data fetching or remote fonts (the only `fetch` calls are to own `data/*.md`)
- Canvas-generated shareable long images (draft档位 3 feature, explicitly deferred)
- Back-navigation between questions (draft non-goal)
- i18n infrastructure or language switching

> **Note on Deterministic Design**: The draft specifies file tree, module boundaries, DSL grammar, algorithm steps, fallback chain, 25-entry catalog names, SVG enum values, error messages, and QA checklist items explicitly. The implementer's freedom is confined to the choices listed above (file split, namespace style) and to expressive choices in the fortune text content of `data/results.md` and the question prompts of `data/questions.md`.

## Feasibility Hints and Suggestions

> **Note**: This section is for reference and understanding only. These are conceptual suggestions, not prescriptive requirements.

### Conceptual Approach

Skeleton of the runtime flow:

```
async function bootstrap() {
  try {
    const [qText, rText] = await Promise.all([
      loadMarkdown('data/questions.md'),
      loadMarkdown('data/results.md')
    ]);
    state.questions   = parseQuestions(qText);
    state.resultsDict = parseResults(rText);
    renderHome();
  } catch (e) {
    renderError(e.message);
    router.go('error');
  }
}

function score(answers, questions) {
  if (answers.length !== questions.length) {
    throw new Error(`测试数据异常：已答 ${answers.length} 题，期望 ${questions.length} 题`);
  }
  const s = { 金:0, 木:0, 水:0, 火:0, 土:0, 躺平:0, emo:0, 社牛:0 };
  answers.forEach((choice, i) => {
    for (const [k, v] of Object.entries(questions[i].options[choice].deltas)) {
      s[k] += v;
    }
  });
  return s;
}

function classify(scores) {
  const WUXING  = ['金', '木', '水', '火', '土'];              // declared order for tie-break
  const PERSONA = ['躺平', 'emo', '社牛'];                       // declared order for tie-break
  const sorted  = [...WUXING].sort((a, b) => {
    const d = scores[b] - scores[a];
    return d !== 0 ? d : WUXING.indexOf(a) - WUXING.indexOf(b);
  });
  const top = sorted[0], second = sorted[1];
  // Apply rules 1-5 per spec §5.3 ...
}

function resolveKey(triple, dict) {
  const candidates = [
    `${triple.品阶}_${triple.属性}_${triple.前缀}`,
    `${triple.品阶}_${triple.属性}_default`,
    `${triple.品阶}_default_${triple.前缀}`,
    `${triple.品阶}_default_default`,
    'default_default_default'
  ];
  for (const k of candidates) if (dict[k]) return k;
  throw new Error('结果字典缺少 default_default_default 兜底条目');
}
```

Parser strategy: line-by-line state machine with a small set of states (`title`, `intro`, `question-block`, `options-list`, `entry-block`, `section-签文`, `section-推荐功法`, `section-画像`). On state transitions, flush accumulated content into the current question or entry object. Extract option parts via one regex per line: `^- ([^`]*?)\s*\`([^`]+)\`\s*(.*)$`. Validate expressions via one regex per segment: `^(金|木|水|火|土|躺平|emo|社牛)([+-])(\d)$`. Reject anything else with a locator-rich error message.

Portrait strategy: embed a 240×320 SVG template as a JS template literal. Each accessory is a `<g id="acc-<name>" style="display:none">…</g>`; each background is a `<g id="bg-<name>" style="display:none">…</g>`. `renderPortrait` parses the literal (via `DOMParser` or a cached fragment), clones it, sets `element.style.setProperty('--pao-color', config.袍色)`, does the same for `--fu-color`, then flips `display: block` on `acc-<config.配饰>` and `bg-<config.背景>`.

Persona seal overlay: for every non-weiling entry, also render a small cinnabar seal next to the 灵根 name showing 躺/emo/牛/钝 based on `determinePrefix(scores)`. This is the visible manifestation of DEC-5 resolution B and lets the persona dimension matter for every user regardless of grade.

### Relevant References

- `docs/superpowers/specs/2026-04-10-lingen-test-design.md` — complete design spec (source of truth); every section below directly defers to this file
  - §3 — module layout, data flow, responsive strategy, font fallback
  - §4.1 — question DSL grammar (authoritative)
  - §4.2 — result dictionary DSL grammar (authoritative)
  - §5 — scoring algorithm, classify rules, fallback chain, tunable constants
  - §6 — 25-entry catalog (authoritative names, keys, hooks, portrait hints)
  - §7 — SVG portrait module contract and enum whitelist
  - §8 — page state machine, mobile and desktop layouts, interactions
  - §9 — error handling table (authoritative messages)
  - §10 — testing strategy (unit, integration, content lint, manual QA)
- `https://sbti.unun.dev` — external reference UX for the question/answer flow
- `https://blog.axiaoxin.com/post/fanren-linggen/` — lore reference for spiritual root system
- This plan file — derivative planning artifact that refines the draft into testable criteria and tasks

## Dependencies and Sequence

### Milestones

1. **Foundation** — scaffold project files, apply visual theme, set up routing shell
   - Phase A: Create `index.html` with four `<section>` skeletons (home, quiz, result, error), `.gitignore` ignoring `.humanize/`, `.superpowers/`, `.DS_Store`, `.cache/`, and `data/` directory
   - Phase B: Author `style.css` with fortune-slip theme (rice-paper color, KaiTi fallback chain, cinnabar accents), base typography, mobile-first layout, `@media (min-width: 768px)` desktop upgrades, `@media (hover: hover)` hover isolation
   - Phase C: Implement `router.go(sectionId)` and the global `state` object

2. **Parser** — implement both DSL parsers with rich diagnostics
   - Phase A: `parseQuestions(text)` — state machine, option regex, expression regex, question-level and option-level error messages
   - Phase B: `parseResults(text)` — entry parser, metadata parser, sub-section parser, uniqueness checks, enum whitelist checks, hex color validation, default fallback enforcement

3. **Engine** — implement scoring, classification, and key resolution
   - Phase A: `score(answers, questions)` with length guard
   - Phase B: `classify(scores)` with 5-rule ordered evaluation and locked tie-break (declared dimension order per DEC-2 resolution A)
   - Phase C: `resolveKey(triple, dict)` with the 5-step fallback chain
   - Phase D: Tunable constants grouped at the top of the file

4. **Portrait** — implement programmatic SVG template and render function
   - Phase A: Author the inline SVG template literal with all accessory and background groups
   - Phase B: Implement `renderPortrait(config)` with CSS var application, enum dispatch, and `config.image` escape hatch

5. **Pages** — implement the renderers and wire events
   - Phase A: `renderHome()` — title, description, start button, footer
   - Phase B: `renderQuiz(index)` — progress bar, prompt, three options with labels, keyboard binding (1/2/3/Enter), focus-visible, touch feedback
   - Phase C: `renderResult(classification)` — portrait, 灵根 name, persona seal overlay (per DEC-5 resolution B), fortune text, five-element bars (per DEC-6 resolution A clamping), recommended methods, result code copy handler with clipboard fallback, screenshot hint with platform dispatch and generic fallback, re-test button
   - Phase D: `renderError(msg)` — shared error scroll frame with dynamic message and retry button

6. **Content** — author the Markdown content files
   - Phase A: `data/questions.md` — 16 questions in 签文体, calibrated to spread across all eight dimensions, reviewed by user before milestone closes
   - Phase B: `data/results.md` — 25 entries matching the §6 catalog names and keys, each with fortune text, three recommended methods, and a portrait config whose accessory and background values are in the enum whitelist

7. **Tests** — standalone test runner with isolation and content lint
   - Phase A: `test.html` scaffold that imports `main.js` without invoking `main()`; production entry must only auto-bootstrap in production HTML via `DOMContentLoaded`
   - Phase B: Unit tests — 15 to 20 cases covering happy paths and negative paths for `parseQuestions`, `parseResults`, `score`, `classify`, `resolveKey`
   - Phase C: Integration tests — one fixture answer sheet per grade (5 total) running the full pipeline and asserting resolved key existence
   - Phase D: Content lint — load real `data/*.md` via `fetch` and assert: question count 16, default fallback entry, all five grades reachable under fixtures, all portrait enums whitelisted, all hex colors valid

8. **Calibration** (conditional) — optional distribution analysis
   - Phase A: `simulate.html` — only built if DEC-4 escalates to hard-gate (currently C, so skipped)
   - Phase B: Threshold retuning — only if Phase A runs and surfaces skew

9. **Responsive polish** — verify layouts across breakpoints
   - Phase A: Chrome DevTools at 320 / 375 / 393 / 768 / 1280 px widths
   - Phase B: Verify KaiTi font fallback chain on Linux (serif degrade path)
   - Phase C: Verify touch feedback, hover isolation, keyboard focus ring
   - Phase D: Verify clipboard and screenshot fallbacks in Chrome / Safari / Firefox

10. **Release** — README, QA, and sign-off
    - Phase A: Author `README.md` with project intro, one-line access, local dev instructions, content editing guide (pointing at the DSL sections in the spec), the manual QA checklist, and signing attribution
    - Phase B: Run the manual QA checklist end-to-end and tick all items
    - Phase C: Commit with `Signed-off-by: Chao Liu <chao.liu.zevorn@gmail.com>` per project convention

Dependencies:
- Milestones 2, 3, 4 can proceed in any order once Milestone 1 is complete
- Milestone 5 depends on 2, 3, 4 (renderers need parsers, engine, and portrait)
- Milestone 6 depends on 2 and 3 (content is validated by parsers as it is authored)
- Milestone 7 depends on 2, 3, 4, 5, 6 (content lint needs production data to exist)
- Milestone 8 is conditional on DEC-4 escalation; currently not in v1 scope
- Milestone 9 depends on 5 and 6
- Milestone 10 depends on 7 and 9

## Task Breakdown

Each task is tagged `coding` (implemented by Claude) or `analyze` (executed via Codex). Tasks map back to acceptance criteria.

| Task ID | Description | Target AC | Tag | Depends On |
|---------|-------------|-----------|-----|------------|
| task1 | Scaffold project files (index.html with 4 sections, style.css with fortune-slip theme and responsive breakpoints, font fallback chain, .gitignore, empty data/ directory); implement router.go() and global state object | AC-7, AC-12 | coding | - |
| task2 | Implement parseQuestions DSL parser with split question-level and option-level diagnostic errors and expression regex validation against the dimension whitelist | AC-2.1, AC-2.2 | coding | task1 |
| task3 | Implement parseResults DSL parser with metadata, uniqueness checks (key/displayName/结果代码), portrait enum whitelist, hex color validation, and default_default_default enforcement | AC-3.1 | coding | task1 |
| task4 | Implement score/classify/resolveKey with answers length guard, declared-order tie-break (per DEC-2 A), 5-rule classify chain, and 5-step fallback key resolution | AC-1 | coding | - |
| task5 | Implement inline SVG base template (240×320 rice-paper + seated figure + accessory/background groups) and renderPortrait with CSS var application, enum dispatch, and config.image escape hatch | AC-8 | coding | task1 |
| task6 | Implement renderHome / renderQuiz / renderResult / renderError, event handlers for click and keyboard (1/2/3/Enter), touch feedback, clipboard copy with selection fallback, screenshot hint with platform dispatch and generic fallback, persona seal overlay on result page (per DEC-5 B), and unified error routing (eliminating home disabled-error-mode) | AC-6, AC-9, AC-14, AC-15 | coding | task1, task4 |
| task7 | Author data/questions.md — 16 questions in 签文体, dimension distribution calibrated so all five grade branches can be reached under fixture testing; content reviewed with user before closing the task | AC-2.3 | coding | task2 |
| task8 | Author data/results.md — 25 entries matching the §6 catalog (5 tianling + 4 bianling + 10 zhenling + 4 weiling + 1 yinling + 1 default) with 签文 paragraphs, 推荐功法 lists, and 画像 configs using whitelisted enum values; clamped five-element bar semantics (per DEC-6 A) documented in comments | AC-3.2, AC-10 | coding | task3, task5 |
| task9 | Write test.html standalone runner that imports main.js without bootstrapping, includes 15-20 unit tests covering happy and negative paths for all parsers and engine functions, 5 per-grade integration fixtures, and content lint against real data/*.md enforcing AC-4 assertions | AC-4, AC-5, AC-11 | coding | task2, task3, task4 |
| task10 | (Optional, conditional on DEC-4 escalation) Build simulate.html that runs 10,000 random answer sheets through the pipeline and reports per-key distribution | AC-11 | analyze | task7, task8, task9 |
| task11 | (Optional, conditional on task10 outcome) Retune classify and prefix threshold constants based on simulate distribution analysis | AC-11 | coding | task10 |
| task12 | Responsive QA pass at 320 / 375 / 768 / 1280 px, Linux serif fallback, touch feedback isolation, keyboard focus ring verification, clipboard and screenshot fallback verification across Chrome / Safari / Firefox | AC-6, AC-7 | coding | task6, task8 |
| task13 | Author README.md with project intro, online link placeholder, local dev instructions (`python3 -m http.server`), content editing guide, manual QA checklist matching AC-13 expectations, and attribution footer | AC-13 | coding | task12 |
| task14 | Execute the manual QA checklist end-to-end, tick all items, record any follow-up issues, and confirm v1 readiness | AC-13 | coding | task13 |

## Claude-Codex Deliberation

### Agreements

- Pure static site with no build step is the right architecture for this spec's scope and the authoring team's context
- Two Markdown files (questions.md, results.md) plus a hand-written parser is the correct content model — keeps authors out of JavaScript and enables direct GitHub reading and PR review
- Error messages must pinpoint the failing question id or entry display name, not just line numbers or generic "parse error" strings
- `default_default_default` as the terminal fallback entry is mandatory; the parser must enforce its existence
- The four-section SPA with `.active` class toggling is simpler than hash routing and sufficient for v1
- Eliminating the "home disabled error mode" and routing all failures to the dedicated error section resolves the draft's only self-contradiction in §8.2 vs §8.5
- `config.image` escape hatch in the portrait module preserves forward-compatibility for hand-drawn or AI-generated art without a future code rewrite
- `test.html` must be isolated from production `main()` bootstrap to prevent tests from issuing real `fetch` calls against production data
- Tie-break behavior must be locked and testable; implicit JS sort stability is not acceptable
- Runtime-fatal errors, authoring-time lint warnings, and integration-test-only assertions should be explicitly classified — blurring them leads to either noisy user-facing crashes or silent content bugs
- Keyboard interaction and result page actions belong in acceptance criteria, not "polish" — the draft lists them as explicit behaviors
- `file://` direct-open support is NOT a goal; the error page must tell users to run `python3 -m http.server`, and this is the full extent of the compatibility story

### Resolved Disagreements

- **Distribution calibration as a HARD gate**
  - Claude v1 position: Elevate `simulate.html` 10,000-sample calibration from the draft's "optional" status to a mandatory pre-release gate
  - Codex second-pass position: Scope creep — the draft explicitly labels this as optional / out of scope for v1
  - Resolution: Downgrade AC-11 to a soft gate covering only the 5 per-grade fixtures. Hard-gate escalation is gated on DEC-4. The user resolved DEC-4 to C (premium entries reachable, fallback entries allowed unreachable), so AC-11 remains soft and task10/task11 stay optional backlog items.

- **Persona visibility scope in the upper bound**
  - Claude v1 position: Upper bound should promise "persona surfaces in all grades" to honor the 8-dimension scoring claim
  - Codex second-pass position: That pre-commits DEC-5 before the user decides; remove it from the boundary statement
  - Resolution: Removed from Path Boundaries. User then resolved DEC-5 to B (seal icon overlay in all grades), which is now captured as a requirement inside task6 (rendering), task8 (per-entry seal choice authoring), and explicitly listed in the Upper Bound description.

- **AC-2 diagnostic granularity**
  - Claude v1 position: All parser errors include question id + option index
  - Codex second-pass position: Question-level errors (missing prompt, wrong option count) have no option index; forcing one creates an unsatisfiable verification criterion
  - Resolution: AC-2 split into AC-2.1 (option-level), AC-2.2 (question-level), and AC-2.3 (production data). Same split mirrored for AC-3 entry-level errors.

- **Keyboard and result action specs missing from AC**
  - Claude v1 position: Treat keyboard / clipboard / screenshot as polish
  - Codex second-pass position: The draft explicitly specifies these behaviors; they must be acceptance criteria or the draft is being silently de-scoped
  - Resolution: Added AC-14 (keyboard interaction: 1/2/3 + Enter + focus-visible + no back-nav) and AC-15 (result page actions: clipboard with fallback, platform-aware screenshot hint, re-test).

- **Classify behavior under negative deltas (Codex UNRESOLVED from round 2)**
  - Claude position: DEC-9 added as explicit dependency on DEC-1
  - Codex third-pass position: Adequate, but recommend annotating DEC-9 as "only active if DEC-1 allows negatives"
  - Resolution: Captured in DEC-9 body text below.

### Convergence Status

- Final Status: `converged`
- Rounds executed: 3 (Codex first-pass critique → Claude candidate v1 → Codex second-pass review → Claude v2 revision → Codex third-pass verdict `CONVERGED`)
- Codex v3 verdict: "基于 v2 变更，上一轮阻塞性的 required changes 和未决承接项都已被合理吸收，现阶段可以接受。"

## Pending User Decisions

- DEC-1: Allow negative `delta` values in scoring expressions (e.g. `` `社牛-2` ``)?
  - Claude Position: Disallow in v1 (simpler parser regex, pairs cleanly with DEC-6=A clamped display, leaves the expression grammar closer to spec §4.1 default reading)
  - Codex Position: Either is defensible; requires explicit decision for the parser regex and AC-10 edge handling
  - Tradeoff Summary: Allowing negatives adds expressive range for "strongly anti-X" options but complicates classify rules and display clamping; disallowing simplifies everything and loses little creative flexibility (weights can shift between positive dimensions instead)
  - Decision Status: PENDING — default-to-disallow during implementation unless the user overrides during plan review. Affects AC-2 regex, AC-10 edge cases, and makes DEC-9 inactive.

- DEC-2: Tie-break rule for `classify()` sort stability when two dimensions have equal scores
  - Claude Position: Use declared dimension order (金木水火土 for five-elements; 躺平 emo 社牛 for persona)
  - Codex Position: Explicit locking required; any deterministic scheme is acceptable
  - Tradeoff Summary: Declared order is readable and testable but introduces an implicit preference; Unicode order is mechanical but counter-intuitive; hash-seeded random is deterministic but unpredictable
  - Decision Status: **Resolved to A (declared order)**. Applied in task4 and the classify pseudocode snippet in Feasibility Hints.

- DEC-3: Expected trigger rate for 隐灵根 (hidden root) Easter egg
  - Claude Position: Less than 1% (rare event aligned with lore)
  - Codex Position: Needs explicit target to tune `YINLING_TOTAL_MAX` and `YINLING_PERSONA_MIN` constants
  - Tradeoff Summary: Too high dilutes the "rare egg" feel; too low means no one ever sees it; the actual rate depends on threshold values which can be retuned later
  - Decision Status: PENDING — default to less than 1%, revisit if task10 simulation ever runs and reveals skew. Non-blocking for v1.

- DEC-4: Reachability requirement for the 25 result entries
  - Claude Position: Middle ground — premium entries reachable, fallback entries may remain unreachable but must still exist in the dictionary
  - Codex Position: Any of the three options is defensible if explicit
  - Tradeoff Summary: Loose (5 grades reachable) is fastest; strict (25/25 reachable) forces full simulate.html pipeline; middle ground is the v1 sweet spot
  - Decision Status: **Resolved to C (middle ground)**. task10/task11 stay in backlog; AC-11 stays a soft gate; fallback entries like `default_default_default` are authored but not asserted to be reachable under production answers.

- DEC-5: Persona dimension visibility in non-weiling grades
  - Claude Position: Add a small cinnabar persona seal icon next to the 灵根 name on the result page
  - Codex Position: Must resolve before locking upper-bound scope; middle-ground solution is desirable
  - Tradeoff Summary: A (persona only in weiling) is minimal work but makes the 8-dimension claim feel cosmetic; C (full prefix text for all 25 entries) quadruples content; B (seal icon) is the middle path
  - Decision Status: **Resolved to B (seal icon overlay in all grades)**. task6 renders the seal; task8 authors the seal-icon choice per entry; icon set has four values (躺 / emo / 牛 / 钝) mapped from `determinePrefix(scores)`.

- DEC-6: Five-element bar display formula
  - Claude Position: Clamped `max(0, min(score, 7)) / 7 * 100%`
  - Codex Position: Must be defined with explicit edge-case handling
  - Tradeoff Summary: Clamped is simplest and deterministic; absolute-normalized is more expressive but confuses low-score users; signed bipolar is the most visual but requires resolving DEC-1 and adds CSS complexity
  - Decision Status: **Resolved to A (clamped 0-7)**. Applied in task6 and AC-10.

- DEC-7: Retain or eliminate the "disabled error mode" on the home section (§8.2 of draft)
  - Claude Position: Eliminate (unify all error handling in the dedicated error section, per AC-9)
  - Codex Position: Eliminating resolves the self-contradiction in the draft cleanly
  - Tradeoff Summary: Keeping it is marginally more informative when the user navigates back to home after a failure but duplicates the error rendering logic
  - Decision Status: PENDING — default to eliminate during implementation unless the user overrides. Affects AC-9 final wording and task6 (one fewer branch to implement).

- DEC-8: Test isolation strictness for test.html
  - Claude Position: Strict — `main.js` exposes a named `main()` function that is only invoked from `index.html` on `DOMContentLoaded`, not from module top level
  - Codex Position: Strict isolation is the only sane default
  - Tradeoff Summary: Strict means test.html can safely import main.js and exercise public functions without triggering production fetch; relaxed allows main.js to self-initialize but then tests cannot load it without side effects
  - Decision Status: PENDING — default to strict during implementation unless the user overrides. Affects AC-5 wording and the entry-point conventions in task2, task4, and task9.

- DEC-9: When `classify()` runs, should negative dimension scores be clamped to 0 before sorting and threshold evaluation?
  - Claude Position: N/A if DEC-1 disallows negatives; clamp-to-zero if DEC-1 allows negatives (keeps threshold rules intuitive)
  - Codex Position: Explicit resolution required; dependency on DEC-1 must be documented
  - Tradeoff Summary: Clamping loses information about "strongly anti-X" signals but keeps classify rules 1-5 straightforward; using raw values lets anti-weight count against threshold comparisons but makes the 5-rule branches harder to reason about
  - Decision Status: PENDING — **active only if DEC-1 is resolved to "allow negative deltas"**. Default fallback if DEC-1 stays default-disallow: DEC-9 becomes not applicable.

## Implementation Notes

### Code Style Requirements

- Implementation code and comments must NOT contain plan-specific terminology such as "AC-", "Milestone", "Step", "Phase", or similar workflow markers
- These terms are for plan documentation only, not for the resulting codebase
- Use descriptive, domain-appropriate naming in code instead (e.g. `parseQuestions`, `classifyGrade`, `resolveResultKey`, not `step3_classify`)
- JavaScript code and JSDoc comments in English; only the content files `data/questions.md`, `data/results.md`, user-visible HTML text, CSS comments, and README user-facing sections are in Chinese
- Indentation: 4 spaces (per project AGENTS constraint)
- UTF-8 without BOM for all source files
- Prefer small focused functions over monoliths; prefer explicit error `throw` over silent fallbacks
- No `console.log` in production code paths; the only acceptable console output is authoring-time soft warnings such as "question count 15, expected 16" or "3 result entries are never reached", and those warnings live inside `test.html` or the lint path, not in the production bootstrap
- Every Git commit in this project carries `Signed-off-by: Chao Liu <chao.liu.zevorn@gmail.com>` and does NOT carry any `Co-Authored-By: Claude` line

### Runtime vs Authoring-Time Error Separation

- **Runtime-fatal** (routes to dedicated error section, blocks use, user-visible Chinese message):
  - `fetch` failure for `data/questions.md` or `data/results.md` (CORS, 404, network)
  - Parser syntax errors (malformed expression, unknown dimension, missing prompt, wrong option count, missing required metadata, duplicate key, duplicate display name, duplicate result code, invalid hex color, unknown accessory/background enum)
  - Missing `default_default_default` entry in the parsed results dictionary
  - `answers.length` mismatch in `score()` length guard
- **Authoring-time lint** (warnings only, non-blocking, shown only in `test.html` or console during development):
  - Question count ≠ 16
  - Result dictionary entries never matched by any reachable `classify()` triple
- **Integration test only** (asserted by fixtures, not by production runtime):
  - Reachability of all 5 grades under the per-grade fixture answer sheets



## Output File Convention

This template is used to produce the main output file (e.g., `plan.md`).

### Translated Language Variant

When `alternative_plan_language` resolves to a supported language name through merged config loading, a translated variant of the output file is also written after the main file. Humanize loads config from merged layers in this order: default config, optional user config, then optional project config; `alternative_plan_language` may be set at any of those layers. The variant filename is constructed by inserting `_<code>` (the ISO 639-1 code from the built-in mapping table) immediately before the file extension:

- `plan.md` becomes `plan_<code>.md` (e.g. `plan_zh.md` for Chinese, `plan_ko.md` for Korean)
- `docs/my-plan.md` becomes `docs/my-plan_<code>.md`
- `output` (no extension) becomes `output_<code>`

The translated variant file contains a full translation of the main plan file's current content in the configured language. All identifiers (`AC-*`, task IDs, file paths, API names, command flags) remain unchanged, as they are language-neutral.

When `alternative_plan_language` is empty, absent, set to `"English"`, or set to an unsupported language, no translated variant is written. Humanize does not auto-create `.humanize/config.json` when no project config file is present.

--- Original Design Draft Start ---

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

--- Original Design Draft End ---
