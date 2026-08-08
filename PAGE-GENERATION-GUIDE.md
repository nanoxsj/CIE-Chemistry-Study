# CIE Chemistry 章节页面生成指南

本指南用于把新章节（例如 24 Electrochemistry、25 Equilibria 等）生成成与现有章节一致的学习页面。按顺序执行即可。

## 1. 项目与素材位置

```text
项目目录   /Users/xiesj/Documents/ChatGPT/CIE_Chemistry
GitHub     nanoxsj/CIE-Chemistry-Study（公开，main + gh-pages）
线上地址   https://nanoxsj.github.io/CIE-Chemistry-Study/

笔记 PDF   /Users/xiesj/Documents/ALevel_Course/CIE_Chemistry/2025-2027/2025-2027/
           （新考纲编号，例如 23.1 Lattice Energy - Born-Haber Cycles.pdf.pdf）
专题题包   /Users/xiesj/Documents/ALevel_Course/CIE_Chemistry/Topical Past Papers/Topical Past Papers/
           A2/ 与 AS/ 子目录；Save My Exams 导出，旧编号（如 5.1、5.2），需映射到新考纲章节
考纲       /Users/xiesj/Documents/ALevel_Course/CIE_Chemistry/664563-2025-2027-syllabus.pdf
```

题包文件名中 `C` 结尾 = Multiple Choice，`SQ` 结尾 = Structured Questions；本项目的章节页**只做大题（SQ），不做选择题**。

## 2. 目标结构（以第 23 章为例）

```text
a-level/23-chemical-energetics.qmd              # 主页面：知识点 + worked example + include 题目
a-level/23-chemical-energetics-structured.qmd   # 大题页：全部 SQ + 逐点答案
assets/questions/chemical-energetics/*.webp     # 题目图（可独立提取的图）
assets/notes/chemical-energetics/*.webp         # 笔记图（知识点区插图）
resources/original-papers/chemical-energetics/*.pdf  # 原始专题 PDF（只进 main，不进 gh-pages）
_site/                                          # quarto render 产物（.gitignore 忽略）
```

## 3. 读取素材

笔记与题包都是 Save My Exams 的图片型 PDF，`pdftotext` 只能提出少量残字，必须渲染后 OCR：

```bash
# 渲染（200 dpi 对表格/公式更稳）
pdftoppm -r 200 -png "<note.pdf>" /private/tmp/chem/pages/<key>
tesseract /private/tmp/chem/pages/<key>-01.png stdout -l eng --psm 4
```

- 题包结构：前面是题目页（页眉 `Page N of X`），后面是 Model Answers；用 `pdftotext -raw` 找「Model Answers」出现的页号，划分题目区与答案区。
- 题目文字尽量用 200 dpi OCR 重读一遍，表格数值以 Model Answers 中的计算过程为准互相校验（例如 K₂O 晶格能 −2280、CaCl₂ −2260）。
- OCR 输入文件**不要放 `/tmp`**：tesseract 打不开 `/tmp`（符号链接）路径且静默失败；放 `/private/tmp/...` 或项目内真实路径。

## 4. 题目选择规则

1. 本项目的章节页**只收 Structured Questions**，不收 MC。
2. 优先收「带图且图可以独立提取」的题；原题就是纯文字（含可重排的数据表）的题也收。
3. 原题带图但图无法独立提取的题：**跳过，不加入页面**，不改写成纯文字题。
4. 不使用答案页的图：`pdfimages -list` 的对象页号必须落在题目页范围内。
5. 第 23 章共 6 个题包（Easy/Medium/Hard × 5.1/5.2），22 道完整大题全部可收（5 道带图，图全部可独立提取）。

## 5. 图片提取

### 题目图（Save My Exams 题包，纯栅格）

```bash
# 只提取题目页范围内的对象
pdfimages -png -p -f <起始页> -l <结束页> <pdf> /private/tmp/chem/figs/<key>
```

有 smask 的对象成对合成白底 WEBP（Python + Pillow）：

```python
im = Image.open(img).convert("RGB")
m  = Image.open(mask).convert("L")
assert im.size == m.size
white = Image.new("RGB", im.size, "white")
white.paste(im, mask=m)
white.save(out, "WEBP", lossless=True, method=6)
```

### 笔记图（栅格底图 + 矢量文字混合）

笔记里的图是「栅格底图 + 矢量覆盖文字」，`pdfimages` 直接提取会丢文字。正确做法是用高分辨率页面渲染 + 图框裁剪：

```bash
pdftoppm -r 200 -f <页> -l <页> -png <note.pdf> /private/tmp/chem/we/<key>
```

```python
# 用 pdfplumber 读该页大图（width>100）的放置框 (x0, top, x1, bottom)
# scale = 200/72，裁剪 (x0*scale-10, top*scale-10, x1*scale+10, bottom*scale+10)，存 lossless WEBP
```

### 验证（当前环境 view_image 不可用）

- 四角必须为白色；
- 把提取图按 pdfplumber 框贴回 150 dpi 页面渲染做像素比对：MAE 通常 < 15、差异像素（>25）占比 < 15%；
- OCR 页面裁剪区确认图内标签与题目/知识点一致（例如 CaO 循环应有 Step A–G）。

## 6. 页面内容规范

### 主页面（a-level/XX-xxx.qmd）

- YAML：`title: "23 Chemical energetics"`、`subtitle: "..."`。
- `## Learning objectives`：本节能力目标。
- 每个小节用 `::: {.study-card}`（概念）与 `::: {.formula-panel}`（公式）组织，公式用 LaTeX（KaTeX 渲染）。
- **中英混排**：解释用中文，关键专有名词与定义句保留英文原文（定义加粗），例如：
  > **The lattice energy, $\Delta H^\ominus_{\mathrm{latt}}$, is the enthalpy change when 1 mole of an ionic compound is formed from its gaseous ions, under standard conditions.**
- 笔记中的 **worked example 放在相关知识点之后**，用 `::: {.worked-example}`，内含 `<span class="we-tag">Worked example · 笔记例题</span>` 与 `<div class="we-answer">` 答案框；KCl/MgO 循环图、MgCl₂ 能量水平图等源自 worked example 的图归位到例题框内。
- 章末：`## Chapter checklist`（勾选框）+ `### Original question PDFs`（raw 链接，见第 7 节）。

### 大题页（a-level/XX-xxx-structured.qmd）

- 按题包分组标题：`### 5.1 Chemical Energetics - Easy`（ASCII 连字符）。
- 每题一个 `::: {.question-card}`，标题统一：
  ```text
  ### Example N · 简短主题 · 5.1 SQ Easy Q1
  ```
  N 为该章节内从 1 开始的连续编号。
- 题目保留原始英文与原始图像（`{.question-figure}` + `fig-alt`）；表格用 Markdown 重排。
- 答案统一放在：
  ```markdown
  <details class="answer-panel"><summary>Show mark scheme / model answer</summary>
  - **(a)** ... `[1]`
  </details>
  ```
  按 PDF Model Answers 逐点整理并标注分值；计算题写完整过程。
- 原答案明显有误或数据缺失时，在题卡后加 `::: {.small-note}` 的 Source check 说明（例如第 23 章 5.1 Median Q5c 表格缺行、5.2 Median Q3a 答案误写 KBr）。

## 7. 原始 PDF 链接

1. 把题包 PDF 复制到 `resources/original-papers/<topic>/`，文件名用新考纲编号（如 `23.1-lattice-energy-born-haber-easy.pdf`）。
2. 页面里统一用 raw 地址（不要用相对路径，否则 PDF 会进 gh-pages 导致部署超时）：

```markdown
- [23.1 Lattice Energy and Born–Haber Cycles – Easy](https://raw.githubusercontent.com/nanoxsj/CIE-Chemistry-Study/main/resources/original-papers/chemical-energetics/23.1-lattice-energy-born-haber-easy.pdf)
```

3. PDF 只保留在 `main` 分支，**不要**进入 `gh-pages`。

## 8. 接线与渲染

1. `_quarto.yml`：`project.render` 增加新章节 qmd；`website.sidebar` 对应位置加 href 与 text。
2. `index.qmd`：加一章 `::: {.topic-card}`，列出知识点与题量。
3. 渲染（Quarto 需要写 `~/Library/Caches/quarto`，沙箱下用 require_escalated）：

```bash
quarto render --no-cache
```

4. 检查生成的 HTML：

```bash
rg -o 'class="[^"]*question-card"' _site/a-level/23-chemical-energetics.html | wc -l   # = 题目数
rg -o '<details class="answer-panel">' _site/a-level/23-chemical-energetics.html | wc -l
rg -o 'src="\.\./assets/(questions|notes)/chemical-energetics/[^"]+"' _site/a-level/23-chemical-energetics.html | sort -u
```

确认：题卡数 = 题目数、每个答案区都在、所有图引用存在且 `_site` 中有对应文件、没有占位文本。

5. **div 配平检查**（每次改完必查）：`::: {.xxx}` 必须成对闭合；脚本逐行配对即可发现未闭合位置。`</details>` 后可以紧跟 `:::`（参考物理项目写法）。

## 9. 发布

1. 提交并推送 main：

```bash
git add -A
git commit -m 'Add chapter N ...'
git push --progress origin main
```

（`.git` 在沙箱内只读，git 写操作需要 require_escalated。）

2. 发布 gh-pages。新仓库没有 gh-pages 分支时 `quarto publish gh-pages` 会报
   `remote origin does not have a branch named "gh-pages"`，直接用手动方案：

```bash
pages_dir=$(mktemp -d /tmp/cie-pages.XXXXXX)
git branch gh-pages 2>/dev/null
git worktree add "$pages_dir" gh-pages
rm -rf "$pages_dir"/*
cp -R _site/. "$pages_dir"/
git -C "$pages_dir" add -A
git -C "$pages_dir" commit -m 'Built site for gh-pages'
git -C "$pages_dir" push --progress origin HEAD:gh-pages
git worktree remove "$pages_dir"
```

3. 验证：

```bash
# 轮询直到 built
gh api repos/nanoxsj/CIE-Chemistry-Study/pages --jq '.status'
# 线上访问
curl -s -o /dev/null -w '%{http_code}\n' https://nanoxsj.github.io/CIE-Chemistry-Study/
curl -s -o /dev/null -w '%{http_code}\n' https://nanoxsj.github.io/CIE-Chemistry-Study/a-level/23-chemical-energetics.html
```

## 10. 常见坑

- **答案页图混入**：pdfimages 对象页号超出题目页范围 = 答案页图，禁止使用。
- **笔记图丢文字**：笔记图是栅格+矢量混合，别用 pdfimages，用 200 dpi 页面渲染裁剪。
- **OCR 静默失败**：输入文件放 `/tmp` 时 tesseract 打开失败且无输出；放真实路径并检查 stderr。
- **表格数值 OCR 错**：以 Model Answers 的计算过程反推校验每个数；数值与答案对不上时，先怀疑 OCR 而不是答案。
- **div 未闭合**：`::: {.question-card}` 少一个 `:::` 会导致后面的所有卡片结构塌陷；每次编辑后跑配平脚本。
- **PDF 相对链接失效**：从 `a-level/` 子目录链接资源要用 `../resources/...`；发布前务必换成 raw GitHub 地址。
- **gh-pages 混入 PDF / .DS_Store**：Pages 只应包含 HTML/CSS/WEBP；发布前检查 `git -C "$pages_dir" ls-tree` 确认没有 `.pdf` 和 `.DS_Store`。
- **标题格式不统一**：始终用 `Example N · 主题 · 5.x SQ 难度 Q#`。
