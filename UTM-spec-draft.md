# UTM — Unified Typed Markup · Mini Spec (Draft)

- **Working name / 工作名**: UTM (Unified Typed Markup) — placeholder, replaceable / 占位名，可替换
- **Version / 版本**: 0.1 (draft / 草案)
- **Status / 状态**: Request for review / 征求评审
- **File extension / 文件后缀**: `.utm`

> EN: This is a one-page design sketch, not a finished standard. It fuses the
> strong practices surveyed across Markdown/CommonMark, AsciiDoc, reStructuredText,
> Org-mode, MyST and the Mermaid/Graphviz hosting model, while dropping the
> presentation-coupling and the duplicated block syntaxes those formats carry.
>
> 中文：这是一页设计草图，不是成品标准。它融合了我们调研过的
> Markdown/CommonMark、AsciiDoc、reStructuredText、Org-mode、MyST 以及
> Mermaid/Graphviz 托管模式中的优秀做法，同时砍掉了这些格式里的"展现耦合"
> 和"多套重复块语法"。

---

## 1. Design goals & non-goals / 设计目标与非目标

**EN — Goals**
1. Plain-text first: a `.utm` file is fully readable as text, no rendering required.
2. One block primitive: code, diagrams, tables, math, callouts all share a single
   *typed block* syntax — no per-content ad-hoc grammar.
3. Addressable everything: every block can carry a stable `id`; references are
   resolved and **validated at build time**.
4. Host, don't invent: graphics embed an external DSL (Mermaid/Graphviz/D2/…);
   the format defines only the hosting protocol, never a diagram language.
5. Backend-agnostic: no raw-HTML escape hatch; semantics are not tied to HTML.

**EN — Non-goals**
- Not a presentation/layout language (no styling, no theming in the source).
- Not a config/data format (TOML/JSON do that); UTM is prose + structure.
- No bidirectional/backlink graph in core (an optional tooling concern, §9).

**中文 — 目标**
1. 文本优先：`.utm` 文件无需渲染即可完整阅读。
2. 单一块原语：代码、图形、表格、公式、提示框共用一种*带类型块*语法——
   不为每种内容单设语法。
3. 万物可寻址：每个块都能挂稳定 `id`；引用在**构建时解析并校验**。
4. 只托管不发明：图形内嵌外部 DSL（Mermaid/Graphviz/D2…）；格式只定义托管
   协议，绝不内建图形语言。
5. 后端无关：没有原始 HTML 逃逸口；语义不绑定 HTML。

**中文 — 非目标**
- 不是展现/排版语言（源码中无样式、无主题）。
- 不是配置/数据格式（那是 TOML/JSON 的事）；UTM 是正文 + 结构。
- 核心不含双向/反向链接图谱（属可选工具层，见 §9）。

---

## 2. Document model / 文档模型

**EN** — A document is a sequence of **blocks**. There are exactly two block
shapes:
- **Flow blocks**: paragraphs, headings, lists — body is parsed as inline UTM.
- **Typed blocks**: fenced, body handling decided by the block *type* (raw or flow).

Every block MAY carry an **attribute object** `{#id .class key=val}`. Inline
content lives only inside flow blocks.

**中文** — 一篇文档是**块**的序列。块只有两种形态：
- **流式块**：段落、标题、列表——其正文按内联 UTM 解析。
- **类型块**：带围栏，正文如何处理由块*类型*决定（raw 原样 / flow 解析）。

每个块都*可*携带**属性对象** `{#id .class key=val}`。内联内容只存在于流式块中。

---

## 3. The typed-block primitive / 类型块原语

**EN** — One fence for all structured content. The fence is a run of `=` (≥3).
A type name dispatches handling; the attribute object is optional.

```
=== <type> <attrs>?
<body>
===
```

- Nesting uses longer fences (`====` wraps `===`).
- The **type registry** declares each type's body mode: `raw` (verbatim, e.g.
  `code`, `figure`, `math`) or `flow` (parsed, e.g. `note`, `aside`).
- Unknown types are a build **warning**, body preserved as raw (forward-compat).

**中文** — 所有结构化内容共用一种围栏。围栏是连续的 `=`（≥3 个）。类型名决定
处理方式；属性对象可选。

```
=== <类型> <属性>?
<正文>
===
```

- 嵌套用更长的围栏（`====` 包住 `===`）。
- **类型注册表**声明每种类型的正文模式：`raw`（原样，如 `code`、`figure`、
  `math`）或 `flow`（解析，如 `note`、`aside`）。
- 未知类型产生构建**告警**，正文按 raw 保留（向前兼容）。

**Fence rationale / 围栏选型** — EN: `=` is chosen over `:` and `-`. Within UTM,
headings use ATX `#` (not setext) and there is no `---`/`===` thematic-break or
frontmatter rule, so a run of `=` carries **no competing meaning** and also reads
as a strong divider. `-` is rejected: a run of `-` collides with thematic breaks,
YAML frontmatter, setext H2 underlines and list markers.
中文：选 `=` 而非 `:` 或 `-`。UTM 内部标题用 ATX `#`（非 setext），且没有
`---`/`===` 这类分隔线或 frontmatter 规则，所以一串 `=` **不承载任何其他含义**，
视觉上也像一道强分隔。`-` 被否决：一串 `-` 会与分隔线、YAML frontmatter、
setext 二级标题下划线、列表符号全都冲突。

### EBNF (draft) / EBNF（草稿）

```ebnf
document      = { block } ;
block         = flow-block | typed-block ;

typed-block   = fence , SP , type , [ SP , attrs ] , NL ,
                body ,
                close-fence ;
fence         = "===" , { "=" } ;            (* open: N equals signs, N>=3 *)
close-fence   = fence ;                       (* same length as open *)
type          = NAME ;                        (* e.g. code, table, figure *)
body          = { LINE } ;                    (* raw or flow per registry *)

attrs         = "{" , { attr-item , [ SP ] } , "}" ;
attr-item     = id-attr | class-attr | kv-attr ;
id-attr       = "#" , NAME ;
class-attr    = "." , NAME ;
kv-attr       = NAME , "=" , value ;
value         = bare-word | quoted-string ;

flow-block    = heading | list | paragraph ;
NAME          = ALPHA , { ALPHA | DIGIT | "-" | "_" } ;
```

---

## 4. Attributes & identifiers / 属性与标识符

**EN**
- `{#budget}` sets the block id `budget`. Ids MUST be unique per document.
- `{.warning}` adds a semantic class (no styling implied — semantics only).
- `{caption="Annual cost"}` and other `key=val` pairs are type-defined params.
- Headings auto-derive an id from their text unless one is given.

**中文**
- `{#budget}` 设定块 id 为 `budget`。文档内 id 必须唯一。
- `{.warning}` 添加语义类（不含样式，只是语义）。
- `{caption="年成本"}` 等 `key=val` 是各类型自定义的参数。
- 标题若未显式给 id，则按文本自动生成。

---

## 5. Inline content & links / 内联内容与链接

**EN** — Three link forms, all build-time validated for internal/cross-doc:

| Form | Meaning |
|------|---------|
| `[text](https://…)` | external link |
| `[text](#budget)` | internal ref to block `budget`, explicit text |
| `[[#budget]]` | **auto-ref**: link text pulled from target's caption/heading |
| `[text](other.utm#budget)` | cross-document ref (validated) |

- External link options live in the attribute object: `[text](url){rel=nofollow target=_blank}`.
- A broken `#id` or `other.utm#id` is a build **error** (this is the AsciiDoc
  edge HTML never had).

**中文** — 三种链接形式，内部/跨文档引用均构建时校验：

| 形式 | 含义 |
|------|------|
| `[文字](https://…)` | 外部链接 |
| `[文字](#budget)` | 指向块 `budget` 的内部引用，文字自定义 |
| `[[#budget]]` | **自动引用**：链接文字取自目标的 caption/标题 |
| `[文字](other.utm#budget)` | 跨文档引用（校验） |

- 外链选项放进属性对象：`[文字](url){rel=nofollow target=_blank}`。
- 断掉的 `#id` 或 `other.utm#id` 是构建**错误**（这正是 AsciiDoc 有、HTML 从无
  的能力）。

---

## 6. Tables — dual form + optional compute / 表格——双形态 + 可选计算

**EN** — One block type `table`, two interchangeable bodies, parsed to one model.

**(a) Visual form** — human-written, source looks like a table:
```
=== table {#budget caption="Annual cost"}
| Plan     | Months | Rate |
|----------|-------:|-----:|
| Org      |      1 |   30 |
| AsciiDoc |      2 |   30 |
===
```

**(b) Data form** — machine-generatable, spans & compute declared as attrs:
```
=== table {#budget format=csv header=1 compute="Total = Months * Rate"}
Plan,     Months, Rate, Total
Org,      1,      30,
AsciiDoc, 2,      30,
===
```

- `span` for merged cells is declared, never drawn: `span="r2c1:2x1"` (no ASCII
  art — keeps it parseable; fixes rST's grid-table pain).
- `compute` formulas reference columns by header name or letter; evaluation is a
  spec-defined pure function set (portable, not editor-bound — fixes Org's
  Emacs lock-in).

**中文** — 单一块类型 `table`，两种可互换正文，解析为同一模型。

**(a) 可视化形态** — 人手写，源码就像表：
```
=== table {#budget caption="年成本"}
| 方案     | 人月 | 单价 |
|----------|-----:|-----:|
| Org      |    1 |   30 |
| AsciiDoc |    2 |   30 |
===
```

**(b) 数据形态** — 可机器生成，跨格与计算用属性声明：
```
=== table {#budget format=csv header=1 compute="小计 = 人月 * 单价"}
方案,     人月, 单价, 小计
Org,      1,    30,
AsciiDoc, 2,    30,
===
```

- 合并单元格用 `span` 声明、绝不画线：`span="r2c1:2x1"`（无 ASCII 画框，保证可
  解析；解决 rST 网格表之痛）。
- `compute` 公式按表头名或列字母引用；求值是 spec 定义的纯函数集（可移植、不绑
  编辑器——解决 Org 的 Emacs 锁定）。

---

## 7. Graphics — host protocol / 图形——托管协议

**EN** — Block type `figure` hosts an external diagram DSL. The format defines
the *protocol*, not the language.

```
=== figure {#flow kind=mermaid caption="Review flow"}
graph LR
  A[Draft] --> B{Review}
  B -->|ok|   C[Publish]
  B -->|back| A
===
```

- `kind` selects a pluggable renderer (`mermaid`, `graphviz`, `d2`, `plantuml`, …).
- Body is `raw` and passed verbatim to that renderer.
- A conforming processor MUST expose the renderer registry and MUST NOT
  interpret the body itself. Unknown `kind` ⇒ warning + body preserved.
- `#flow` makes the figure referenceable: `see [[#flow]]`.

**中文** — 块类型 `figure` 托管外部图形 DSL。格式定义*协议*，不定义语言。

```
=== figure {#flow kind=mermaid caption="评审流程"}
graph LR
  A[草稿] --> B{评审}
  B -->|通过| C[发布]
  B -->|打回| A
===
```

- `kind` 选择可插拔渲染器（`mermaid`、`graphviz`、`d2`、`plantuml`…）。
- 正文为 `raw`，原样交给该渲染器。
- 合规处理器必须暴露渲染器注册表，且不得自行解释正文。未知 `kind` ⇒ 告警 +
  保留正文。
- `#flow` 让该图可被引用：`见 [[#flow]]`。

---

## 8. Conformance / 一致性

**EN** — A conforming processor MUST:
1. Parse the typed-block primitive (§3) and the attribute object (§4).
2. Build a document model where every block's id is unique and resolvable.
3. Emit an **error** on any unresolved internal/cross-doc reference (§5).
4. Treat unknown block `type` and unknown figure `kind` as **warnings**, never
   errors, preserving body verbatim (forward compatibility).
5. NOT require any specific editor, and NOT depend on raw HTML.

A test suite (à la CommonMark/`toml-test`) accompanies the spec: input `.utm` ⇒
expected document-model JSON. / 配套测试集（仿 CommonMark / `toml-test`）：
输入 `.utm` ⇒ 期望的文档模型 JSON。

**中文** — 合规处理器必须：
1. 解析类型块原语（§3）与属性对象（§4）。
2. 构建文档模型，其中每个块 id 唯一且可解析。
3. 对任何无法解析的内部/跨文档引用报**错误**（§5）。
4. 把未知块 `type` 和未知图 `kind` 当**告警**而非错误，原样保留正文（向前兼容）。
5. 不依赖任何特定编辑器，不依赖原始 HTML。

---

## 9. Open questions / 待定问题

**EN**
- Backlinks/graph: keep out of core, or add an optional `[[wikilink]]` layer +
  index tool? (Org-roam / Obsidian territory.)
- Compute scope: how rich should the portable function set be (sum/avg only, or
  expressions)? Where to stop before reinventing a spreadsheet.
- Inline math & footnotes: confirm they are flow-block features, not new fences.
- Attribute value typing: bare-word coercion rules (numbers/bools) — borrow
  TOML's typing discipline?

**中文**
- 反向链接/图谱：核心不收，还是加一个可选 `[[wikilink]]` 层 + 索引工具？
  （即 Org-roam / Obsidian 的地盘。）
- 计算范围：可移植函数集该多丰富（只 sum/avg，还是支持表达式）？在重造电子表格
  之前于何处收手。
- 内联数学与脚注：确认它们是流式块特性，而非新围栏。
- 属性值类型：裸词的类型推断规则（数字/布尔）——是否借用 TOML 的类型纪律？
```
