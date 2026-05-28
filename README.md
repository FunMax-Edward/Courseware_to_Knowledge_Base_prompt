# 课件 → 应试备考知识库（Claude Code Prompt 模板）

> 这份模板用于把任意一门大学课程的 PDF 课件（含可选的学长手写笔记）系统性地转写成结构化、应试导向、图文并茂的 Markdown 知识库。
> 已在《数字集成电路分析设计》一门课上验证可行（6 章课件 + 28 页手写笔记 = 379 页，总产出 460 KB Markdown）。

---

## 怎么用这份模板

1. 打开本文件，**只修改下面 §1「我的情况」部分的占位符**（用 `{{...}}` 标记），其他部分不要动
2. 把整份文件的内容**全部复制**，粘贴给 VS Code 里的 Claude Code 作为第一条消息
3. Claude 会按 §3 的工作流程一步步执行，期间会**主动找你确认 demo 格式**，确认后才大规模铺开
4. 全程 1.5–2.5 小时，期间可以做别的事

---

## §1 我的情况（**请你修改这一节，其他保持不变**）

```
课程名：{{比如：信号与系统 / 模拟电子技术 / 高等数学}}
工程目录（绝对路径）：{{比如：C:/Users/你的名字/Desktop/XX课-备考工程}}
源资料目录（绝对路径）：{{比如：C:/Users/你的名字/Desktop/XX课-备考工程/源资料}}

源资料清单（请列出文件名 + 类型）：
- 文件1.pdf — 类型：{{ 可识别文字PDF / 扫描件PDF / 手写笔记PDF }} — 大致内容：{{比如：第1章 引论}}
- 文件2.pdf — 类型：{{ ... }} — 大致内容：{{...}}
- 文件3.pdf — 类型：{{ ... }} — 大致内容：{{...}}
- ...

是否有手写笔记（学长/学姐/自己）：{{ 有 / 没有 }}
  └─ 如果有：笔记文件名：{{比如：XX学长备考笔记.pdf}}

考试目标：应试，老师从课件出题，知识点来自课件
学习偏好：只覆盖课件内容、不外延课外知识；生涩术语可解释；公式必须 LaTeX 化；电路图/曲线图直接引用课件原图
```

---

## §2 期望最终产出（目录结构）

```
{{工程目录}}/
├── 源资料/                      # 用户提供的原始 PDF
│   └── *.pdf
├── scripts/                     # Python 解析脚本
│   ├── extract_pdf.py           # 每页 → PNG + .txt OCR
│   └── extract_images.py        # 抠出每页嵌入的图片
├── parsed/                      # 中间产物（解析结果）
│   ├── ch01/p001.png, p001.txt, ...
│   ├── ch02/...
│   └── ch_yingjie/...           # 手写笔记（如有）
└── kb/                          # ★ 最终知识库
    ├── INDEX.md                 # 主索引（速览表 + 各章详细目录 + 考察方向）
    ├── ch01.md                  # 第 1 章
    ├── ch02.md                  # ...
    ├── chXX.md
    ├── yingjie.md               # 手写笔记（如有）
    ├── _parts/                  # 中间产物（subagent 分段输出，合并后可不用看）
    │   └── chXX/pNNN-pMMM.md
    └── images/                  # 从 PDF 抠出的内容图（已过滤掉 PPT 母版背景）
        ├── ch01/p001_img02.png, _index.md, ...
        └── chXX/...
```

---

## §3 工作流程（请严格按这 4 个阶段执行）

### 阶段 0：开工前确认（必做）

1. **读 §1 占位符**，逐项确认：课程名、工程路径、源资料清单、是否有手写笔记
2. 如果用户**没替换占位符**或信息不全 → **必须先问清楚**，不要瞎猜
3. 在工程目录下 `mkdir -p scripts parsed kb/_parts kb/images`
4. 用 TodoWrite 建立任务清单：
   ```
   - 解析 ch01 PDF
   - 解析 ch02 PDF
   - ...
   - 解析手写笔记（如有）
   - 抠图过滤
   - 做 ch01 p2-p5 demo 让用户审格式
   - 批量处理剩余各章
   - 合并 + 生成 INDEX
   ```

### 阶段 1：本地解析（不耗 API）

把 §5 的两个 Python 脚本**完整复制**写入 `scripts/extract_pdf.py` 和 `scripts/extract_images.py`，然后逐个 PDF 跑：

```bash
# 课件章节按数字编号 01, 02, ...
python scripts/extract_pdf.py "源资料/第1章.pdf" 01
python scripts/extract_pdf.py "源资料/第2章.pdf" 02
# ...
# 手写笔记用 _yingjie / _notes 等字符串编号
python scripts/extract_pdf.py "源资料/学长笔记.pdf" "_yingjie"
```

然后抠图：

```bash
python scripts/extract_images.py "源资料/第1章.pdf" 01
python scripts/extract_images.py "源资料/第2章.pdf" 02
# ...
# 手写笔记是整页扫描件，不需要抠图，跳过
```

跑完后报告统计：每章页数、抠图数、过滤掉多少背景/重复。

### 阶段 2：Demo 阶段（**强制：先小步走再放量**）

⚠️ **关键纪律：不要一上来就铺所有章节，先做 5 页让用户审格式**

1. 选第 1 章的 P2–P5（P1 通常是封面，跳过）
2. 按 §6 的 Markdown 格式规范写到 `kb/_parts/ch01/p002-p005.md`
3. 严格遵守 §8 的「32MB 上限避坑」—— 每次 Read 最多 1–2 张图
4. 完成后**主动停下来**告诉用户：「Demo 已完成，请审格式，确认风格无误后我再批量推进」
5. 用户审稿后会反馈调整，调整完再进阶段 3

### 阶段 3：批量处理（subagent 并发）

只有用户对 demo 格式签字确认后才进入此阶段。

策略：

- 把每章拆成 ~25–30 页一段，每段交给一个 subagent（`general-purpose` 类型）
- 每个 subagent 写到 **独立的** `kb/_parts/chXX/pNNN-pMMM.md`，避免多个 subagent 抢写同一文件
- 主线程**并发**发起所有 subagent（一次 Agent 工具调用块里写多个）
- 每个 subagent 的 prompt 用 §9 的模板

页数拆分建议（实测安全的颗粒度）：

| 单章页数 | 拆几个 subagent |
|---:|---:|
| ≤ 30 | 1 个 |
| 31–60 | 2 个 |
| 61–90 | 3 个 |

如果某个 subagent 撞了 32MB 上限 → 让它**纯 .txt 模式重跑**（不读任何 PNG，只用 OCR 文字 + 图片索引清单填图片引用）。

### 阶段 4：合并 + 主索引

所有 _parts 都完成后：

1. **合并**：对每个章节，按页号顺序把 `kb/_parts/chXX/*.md` 拼到 `kb/chXX.md`，开头加章节头：
   ```markdown
   # 第 X 章 标题（English）
   
   > **来源**：源资料/xxx.pdf，共 NN 页
   > **课件抠图**：[`images/chXX/`](images/chXX/)
   > **原始 OCR + 整页图**：`parsed/chXX/`
   
   ---
   ```

2. **生成 [`kb/INDEX.md`](kb/INDEX.md)**，包含：
   - 章节速览表（章节 / 主题 / 页数 / 文件 / 大小）
   - 每章详细目录（`## P{N}` 标题列表，每条带可点链接）
   - 推荐学习路径（如有手写笔记，建议「先看笔记 → 再过课件主线」）
   - 各章重点考察方向（基于手写笔记反查，没笔记则基于课件章节命名推断）

3. **验收报告**：用一张表给用户看
   ```
   | 章节 | 主题 | 页数 | 已转写 | 文件 |
   |---|---|---:|---:|---|
   | ch01 | ... | 28 | 27 | kb/ch01.md  22 KB |
   | ...
   ```

---

## §4 三大核心约束（**贯穿全程的纪律**）

### 4.1 应试导向 —— 只讲课件内容

- 课件出现过的：必须写
- 课件没有的：**严禁外延**，包括：
  - 不要补充课本里没提的公式推导
  - 不要做「前向埋点」（如「这点在 ch5 还会出现」）
  - 不要补课外名词解释
- 生涩术语**可以**解释（例如 P&R 展开 = Place & Route，DRC = Design Rule Check）—— 这属于「解释课件出现的缩写」，不算外延
- 课件原话表述晦涩时，可以**白话改写**，但内容范围不能扩大

### 4.2 图文并茂 —— 抠图引用，不手画

- 课件里的电路图、版图、曲线、截面图 → 用 `extract_images.py` 抠出来 → Markdown 里用相对路径 `![](images/chXX/pNNN_imgKK.png)` 引用
- 配一句**照课件说**的图说（这张图展示的是什么），不要自己脑补
- **禁止**用 ASCII 字符画电路、禁止用 mermaid 画课件已有的图（除非课件本来就是流程图/状态机风格）

### 4.3 公式必须 LaTeX 化

- 行内：`$V_{OH} - V_{OL}$`
- 块级：
  ```
  $$ NM_H = V_{OH} - V_{IH} $$
  ```
- 用 KaTeX 兼容语法，方便 VS Code Markdown Preview Enhanced 实时渲染

---

## §5 关键脚本（**完整代码，请直接保存使用**）

### scripts/extract_pdf.py

每页渲染成 PNG（200 DPI）+ 用 PyMuPDF 提取该页文字到 .txt。

```python
"""
用法：
  python extract_pdf.py "<pdf_path>" <chapter_id>

例：
  python extract_pdf.py "源资料/第1章.pdf" 01
  python extract_pdf.py "源资料/学长笔记.pdf" "_yingjie"

输出：
  parsed/ch{id}/p{N:03d}.png  — 200 DPI 渲染图
  parsed/ch{id}/p{N:03d}.txt  — 该页可识别文字（扫描件会是空文件）
  parsed/ch{id}/_meta.md      — 章节元信息
"""
import sys
import fitz  # pip install pymupdf
from pathlib import Path

DPI = 200


def extract(pdf_path: str, chapter_id: str, project_root: Path):
    doc = fitz.open(pdf_path)
    out_dir = project_root / "parsed" / f"ch{chapter_id}"
    out_dir.mkdir(parents=True, exist_ok=True)

    zoom = DPI / 72
    mat = fitz.Matrix(zoom, zoom)
    for pno in range(doc.page_count):
        page = doc[pno]
        # 整页渲染图
        pix = page.get_pixmap(matrix=mat, alpha=False)
        pix.save(str(out_dir / f"p{pno+1:03d}.png"))
        # 可识别文字
        text = page.get_text("text")
        (out_dir / f"p{pno+1:03d}.txt").write_text(text, encoding="utf-8")

    meta = [
        f"# ch{chapter_id} 解析元信息",
        f"- 来源：{Path(pdf_path).name}",
        f"- 总页数：{doc.page_count}",
        f"- DPI：{DPI}",
        f"- 文字可识别页数：{sum(1 for f in out_dir.glob('p*.txt') if f.stat().st_size > 20)}",
    ]
    (out_dir / "_meta.md").write_text("\n".join(meta), encoding="utf-8")
    print(f"OK ch{chapter_id}: {doc.page_count} pages -> {out_dir}")
    doc.close()


if __name__ == "__main__":
    pdf_path = sys.argv[1]
    chapter_id = sys.argv[2]
    project_root = Path(__file__).parent.parent
    extract(pdf_path, chapter_id, project_root)
```

### scripts/extract_images.py

把每页里嵌入的图片资源抠出来，两层过滤：(1) 占页面 ≥ 90% 的判为 PPT 母版背景丢弃；(2) 同位置叠放（IoU > 0.7）的只保留像素最大的那张。

```python
"""
用法：
  python extract_images.py "<pdf_path>" <chapter_id>

输出：
  kb/images/ch{id}/p{N}_img{K}.png
  kb/images/ch{id}/_index.md   — 每页图片清单（含尺寸和 bbox）
"""
import sys
import fitz
from pathlib import Path

MIN_W, MIN_H = 60, 60
BG_AREA_RATIO = 0.90   # >= 90% 页面面积 → 视为 PPT 母版背景
IOU_DUP = 0.70         # IoU 超过此阈值视为同位置叠放


def iou(a, b):
    ax0, ay0, ax1, ay1 = a
    bx0, by0, bx1, by1 = b
    ix0, iy0 = max(ax0, bx0), max(ay0, by0)
    ix1, iy1 = min(ax1, bx1), min(ay1, by1)
    if ix1 <= ix0 or iy1 <= iy0:
        return 0.0
    inter = (ix1 - ix0) * (iy1 - iy0)
    aa = (ax1 - ax0) * (ay1 - ay0)
    bb = (bx1 - bx0) * (by1 - by0)
    return inter / (aa + bb - inter + 1e-6)


def extract(pdf_path: str, chapter_id: str, project_root: Path):
    doc = fitz.open(pdf_path)
    out_dir = project_root / "kb" / "images" / f"ch{chapter_id}"
    out_dir.mkdir(parents=True, exist_ok=True)

    lines = [f"# ch{chapter_id} 图片索引", f"来源：{Path(pdf_path).name}\n"]
    total, bg_skipped, dup_skipped = 0, 0, 0

    for pno in range(doc.page_count):
        page = doc[pno]
        page_area = page.rect.width * page.rect.height
        infos = page.get_image_info(xrefs=True)
        if not infos:
            continue

        cands = []
        for k, info in enumerate(infos, 1):
            xref = info.get("xref", 0)
            if not xref:
                continue
            bbox = info.get("bbox", (0, 0, 0, 0))
            bw, bh = bbox[2] - bbox[0], bbox[3] - bbox[1]
            if page_area > 0 and (bw * bh) / page_area >= BG_AREA_RATIO:
                bg_skipped += 1
                continue
            try:
                pix = fitz.Pixmap(doc, xref)
            except Exception as e:
                print(f"  ! p{pno+1} img{k} open failed: {e}")
                continue
            if pix.width < MIN_W or pix.height < MIN_H:
                continue
            if pix.colorspace and pix.colorspace.name not in ("DeviceRGB", "DeviceGray"):
                pix = fitz.Pixmap(fitz.csRGB, pix)
            cands.append({
                "k": k, "pix": pix, "bbox": tuple(round(x, 1) for x in bbox),
                "pixels": pix.width * pix.height,
            })

        cands.sort(key=lambda x: x["pixels"], reverse=True)
        kept = []
        for c in cands:
            if any(iou(c["bbox"], k["bbox"]) > IOU_DUP for k in kept):
                dup_skipped += 1
                continue
            kept.append(c)

        kept.sort(key=lambda x: x["k"])
        if kept:
            lines.append(f"## p{pno+1}")
        for c in kept:
            fname = f"p{pno+1:03d}_img{c['k']:02d}.png"
            c["pix"].save(str(out_dir / fname))
            lines.append(f"- `{fname}` {c['pix'].width}x{c['pix'].height} bbox={c['bbox']}")
            total += 1

    (out_dir / "_index.md").write_text("\n".join(lines), encoding="utf-8")
    print(f"OK ch{chapter_id}: {total} kept, {bg_skipped} bg, {dup_skipped} dup -> {out_dir}")
    doc.close()


if __name__ == "__main__":
    pdf_path = sys.argv[1]
    chapter_id = sys.argv[2]
    project_root = Path(__file__).parent.parent
    extract(pdf_path, chapter_id, project_root)
```

依赖：`pip install pymupdf`（一次性安装即可，已装则跳过）。

---

## §6 Markdown 格式规范（**每页必须遵守**）

每页一个 `## P{N}` 二级标题块，结构如下：

```markdown
## P{N} 中文小标题（English Heading）

### 核心要点
- 3–5 条 bullet，每条 ≤ 20 字
- 提炼这一页最关键的结论/概念/公式名字
- 把英文术语用 **加粗** 标出来

### 课件描述 / 详细内容
（按课件原话叙述，可适当白话改写，可用表格/小标题）

| 概念 | 含义 | 备注 |
|---|---|---|
| ... | ... | ... |

### 关键公式
$$ V_{OH} - V_{OL} = V_{sw} $$

行内公式用 $V_{OH}$ 这样的写法。

### 图示

![Figure X.Y — 简短图名](images/ch{XX}/p{NNN}_img{KK}.png)

> 一句话照课件说这张图画了什么（不要脑补课件没说的内容）。

---
```

规则要点：

1. **每页之间用 `---` 分隔**
2. **页号 N** 用真实页号（不补零），图片文件名用补零三位（`p007_img02.png`）
3. **英文术语**首次出现时给中英对照并加粗：**电压传输特性（Voltage Transfer Characteristic, VTC）**
4. **没有图的页**就不要写「图示」一节，不要硬塞
5. **没有公式的页**也不要硬塞公式一节
6. **图片引用路径**永远是相对路径 `images/chXX/pNNN_imgKK.png`，不要写绝对路径
7. **没找到对应图**时不要瞎引用，宁可缺图也不要错图

特殊：**手写笔记**用 `## 笔记 P{N}（手写）`，下面结构略调整：

```markdown
## 笔记 P{N}（手写）

### 涵盖知识点
- ...
- ...

### 内容转录
（按学长笔记的逻辑顺序，把手写内容整理成可读文字。学长画的箭头、批注要保留语义。）

### 关键公式（如有）
$$ ... $$

### 原扫描页

![笔记 P{N}](../parsed/ch_yingjie/p{NNN}.png)

> 学长重点圈出/标星的位置：...
```

---

## §7 应试导向的具体写法对照

| 不要这样写 ✗ | 要这样写 ✓ |
|---|---|
| 「这个概念的更深层物理含义是……（课件未提）」 | 「课件给出的定义：……」 |
| 「该公式还可推导出更一般形式……」 | 只列课件那个公式 |
| 「这与第 5 章的 XX 现象密切相关，建议先看 5.3」 | 不做前向引用 |
| 「实际工业界还会考虑……」 | 不补充课外 |
| 「P&R」 | **P&R（Place & Route，布局布线）** ← 缩写展开属于解释，允许 |
| 把图自己用 ASCII 画 | 直接 `![](images/chXX/pNNN_imgKK.png)` |
| 把电路图用 mermaid 重画 | 直接引用课件原图 |

---

## §8 32MB API 上限避坑（**最容易翻车的地方**）

### 8.1 原因

Claude API 单次请求**编码后**不能超过 32MB。一张课件 PNG 渲染图典型 0.5–1.2 MB，**经过 base64 编码后膨胀 ~33%**，所以：

- 一次请求 Read 3 张整页 PNG ≈ 安全
- 一次请求 Read 5 张整页 PNG + 累计上下文已经较大 → **必爆**
- 实测翻车案例：subagent 在同一请求里 Read 了 5 张整页 PNG + 多张抠图 → `Request too large (max 32MB)`

### 8.2 安全策略

| 场景 | 怎么做 |
|---|---|
| 文字可识别 PDF | **优先只读 `.txt`**，PyMuPDF 已经把文字提取出来了，不需要再让 LLM 看图 |
| 需要看图（公式排版、流程图、表格） | **每次 Read 最多 1–2 张图**，看完立刻 Edit 写入对应 md，不要囤积 |
| 抠图（已经过滤过的） | 体积小（几十 KB 居多），可以一次读 2–3 张 |
| 手写笔记 / 扫描件 PDF | **必须**逐页读 PNG，但一次最多 1 页 |

### 8.3 应急方案：撞了上限怎么办

subagent 中途撞 `Request too large` → 整个 subagent 任务失败 → 主线程**重新发起**一个**纯 .txt 模式**的 subagent：

> Prompt 关键句：「请**完全不要 Read 任何 PNG 文件**，只用 `parsed/chXX/pNNN.txt` 的 OCR 文字 + `kb/images/chXX/_index.md` 的图片清单。所有图片引用按 _index.md 里列出的文件名填入，不需要验证图片内容。」

这种纯文字模式生成的 md 质量略低（不能针对图说写得很贴切），但能 100% 保证不撞上限。

---

## §9 Subagent 任务 Prompt 模板（批量阶段用）

每个 subagent 用这个模板（替换 `{{...}}`）：

```
你的任务：把 {{课程名}} 第 {{X}} 章的 P{{起始页}}–P{{结束页}} 共 {{N}} 页课件，转写成应试导向的中文 Markdown。

# 输入
- OCR 文字：c:/.../parsed/ch{{XX}}/p{{NNN}}.txt（每页一个，必读）
- 整页渲染图：c:/.../parsed/ch{{XX}}/p{{NNN}}.png（仅在 OCR 残缺或公式排版关键时才读）
- 抠图清单：c:/.../kb/images/ch{{XX}}/_index.md（按这里列的文件名引用图，路径写 images/ch{{XX}}/pNNN_imgKK.png）
- 已有 demo 样板：c:/.../kb/_parts/ch01/p002-p005.md（**严格仿照它的格式风格**）

# 输出
单个文件：c:/.../kb/_parts/ch{{XX}}/p{{起始页:03d}}-p{{结束页:03d}}.md

# 格式（每页一块）
## P{N} 中文标题（English）

### 核心要点
- 3–5 条

### 课件描述
（叙述/表格）

### 关键公式（如有）
$$ ... $$

### 图示（如有）
![](images/ch{{XX}}/pNNN_imgKK.png)
> 一句话说明

---

# 三条铁律
1. **应试导向**：只覆盖课件内容，不外延、不前向引用、不补课外
2. **32MB 防御**：单次请求最多 Read 1 张 PNG；优先用 .txt + _index.md，能不读图就不读
3. **图片引用按 _index.md 实际清单填**，不要瞎写文件名

# 撞 32MB 的应急
如果某次 Read 失败 → 立刻切换到「纯 .txt 模式」：完全不 Read PNG，只靠 .txt 和 _index.md 完成剩余页面

# 完成后报告
- 处理页数 / 跳过页数
- 是否触发 32MB
- 输出文件大小
- 是否有需要人工核对的页（如 OCR 极少、纯图页）
```

---

## §10 最终验收清单

主线程做完阶段 4 后，给用户的总结报告必须包含：

- [ ] 每章页数 vs 实际转写页数（容许封面/纯图过渡页跳过，需说明）
- [ ] 每章 md 文件大小
- [ ] `kb/INDEX.md` 是否包含速览表 + 详细目录 + 学习路径建议
- [ ] 抽样验证：随机抽 3 页，检查图片引用是否能正常打开、公式 LaTeX 是否合规
- [ ] 提示用户安装 VS Code 插件：**Markdown Preview Enhanced** + **Markdown All in One**
- [ ] 列出推荐学习路径（有手写笔记则建议先看笔记，无则按章节顺序）

---

## 附：常见踩坑速查

| 现象 | 原因 | 解决 |
|---|---|---|
| `Request too large (max 32MB)` | 一次 Read 太多图 | 改用纯 .txt 模式 / 拆小批 |
| 抠图都是同一张 PPT 母版背景 | 没启用 BG_AREA_RATIO 过滤 | 用本模板的 `extract_images.py`（已含过滤） |
| 同位置叠了好几张同款图 | PPT 用了图层遮罩 | IoU 去重已含 |
| 中文 PDF 文字提取乱码 | PDF 用了字体子集化 | 改用扫描件思路，逐页让 LLM 读 PNG |
| LaTeX 公式 VS Code 不渲染 | 没装 Markdown Preview Enhanced | 装它 |
| 图片在预览里显示不出来 | 引用路径写错（绝对路径 / 反斜杠） | 一律用相对路径 + 正斜杠 `images/chXX/...` |
| subagent 同时写一个文件冲突 | 没拆 parts 文件 | 每个 subagent 写独立 `pNNN-pMMM.md` |
| 手写笔记部分页不清楚 | 字迹潦草 | 标 `[手写不清]` 占位，不要瞎猜 |

---

**模板结束。**

把整份内容粘给 Claude Code 后，它会先确认 §1 的占位符，然后按 §3 走完整个流程。期间会主动停下来让你审 demo 格式，确认后再批量推。
