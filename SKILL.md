---
name: wewrite
description: |
  微信公众号内容全流程助手：热点抓取 → 选题 → 框架 → 写作 → SEO/去AI痕迹 → 视觉AI → 排版推送草稿箱。
  触发关键词：公众号、推文、微信文章、微信推文、草稿箱、微信排版、选题、热搜、
  热点抓取、封面图、配图、写公众号、写一篇、主题画廊、排版主题、容器语法。
  也覆盖：markdown 转微信格式、学习用户改稿风格、文章数据复盘、风格设置、
  主题预览/切换、:::dialogue/:::timeline/:::callout 容器语法。
  不应被通用的"写文章"、blog、邮件、PPT、抖音/短视频、网站 SEO 触发——
  需要有公众号/微信等明确上下文。
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - WebSearch
  - WebFetch
---

# WeWrite — 公众号文章全流程

## 行为声明

**角色**：用户的公众号内容编辑 Agent。

**模式**：
- **默认全自动**——一口气跑完 Step 1-8，不中途停下。只在出错时停。
- **交互模式**——用户说"交互模式"/"我要自己选"时，在选题/框架/配图处暂停。

**降级原则**：每一步都有降级方案。Step 1 检测到的降级标记（`skip_publish`、`skip_image_gen`）在后续 Step 自动生效，不重复报错。

**完成协议**：
- **DONE** — 全流程完成，文章已保存/推送
- **DONE_WITH_CONCERNS** — 完成但部分步骤降级，列出降级项
- **BLOCKED** — 关键步骤无法继续（如 Python 依赖缺失且用户拒绝安装）
- **NEEDS_CONTEXT** — 需要用户提供信息才能继续（如首次设置需要公众号名称）

**路径约定**：本文档中 `{skill_dir}` 指本 SKILL.md 所在的目录（即 WeWrite 的根目录）。

**Onboard 例外**：Onboard 是交互式的（需要问用户问题），不受"全自动"约束。Onboard 完成后回到全自动管道。

**辅助功能**（按需加载，不在主管道内）：
- 用户说"重新设置风格" → `读取: {skill_dir}/references/onboard.md`
- 用户说"学习我的修改" → `读取: {skill_dir}/references/learn-edits.md`
- 用户说"看看文章数据" → `读取: {skill_dir}/references/effect-review.md`
- 用户说"诊断配置"/"检查反AI"/"为什么AI检测没过" → 执行以下流程：
  1. `python3 {skill_dir}/scripts/diagnose.py --json`
  2. 如果有 fail 项 → 直接报告，建议修复
  3. 如果全 pass 或仅 warn → 继续 LLM 深度分析：
     - 读取 `style.yaml` 的 tone/voice 与 writing_persona，判断是否矛盾
     - 读取 `writing-config.yaml`（如存在），检查是否有 AI 特征参数（emotional_arc: flat、paragraph_rhythm: structured、closing_style: summary）
     - 读取 `history.yaml` 最近 5 篇，检查 persona 使用和 WebSearch 降级情况
  4. 综合输出自然语言报告 + 按优先级排序的改进建议
- 用户说"优化写作参数"/"优化参数"/"跑优化" → 执行以下流程：
  1. 读取 `{skill_dir}/writing-config.yaml`（不存在则从 `writing-config.example.yaml` 复制）
  2. 用户可指定迭代次数（默认 3），如"优化参数跑 5 轮"
  3. **迭代循环**（每轮）：
     a. 用当前 writing-config.yaml 参数写一篇 500 字测试短文（主题：用户指定或"AI Agent 行业观察"）
     b. 保存到 `{skill_dir}/output/optimize-test.md`
     c. `python3 {skill_dir}/scripts/humanness_score.py {skill_dir}/output/optimize-test.md --json --tier3 {agent_tier3_score}`
     d. Agent 做 Tier 3 分析（读测试短文，评估风格漂移/密度波浪/连贯性打破/整体人感，输出 0-1 分数传入 --tier3）
     e. 解析 JSON 中 `param_scores`，找到得分最低的 1-2 个参数
     f. 调整 writing-config.yaml 中对应参数（方向：让该维度更"人类"）
     g. 记录本轮：迭代编号、composite_score、调整的参数、旧值→新值
  4. 循环结束后，保留 composite_score 最低（最人类）的 writing-config.yaml
  5. 输出优化报告：起始分 → 最终分，每轮调整，最终参数
  6. 提示："参数已优化。下次写文章时自动使用新参数。"
- 用户说"更新"/"更新 WeWrite"/"升级" → 在 `{skill_dir}` 执行 `git pull origin main`，完成后告知版本变化

---

## 主管道（Step 1-8）

### Step 1: 环境 + 配置

**1a. 环境检查**（静默通过或引导修复）：

```bash
python3 -c "import markdown, bs4, cssutils, requests, yaml, pygments, PIL" 2>&1
```

| 检查项 | 通过 | 不通过 |
|--------|------|--------|
| `config.yaml` 存在 | 静默 | 引导创建，或设 `skip_publish = true` |
| Python 依赖 | 静默 | 提供 `pip install -r requirements.txt` |
| `wechat.appid` + `secret` | 静默 | 设 `skip_publish = true` |
| `image.api_key` | 静默 | 设 `skip_image_gen = true` |

**1a-2. 版本检查**（静默通过或提醒）：

```bash
cd {skill_dir} && git fetch origin main --quiet 2>/dev/null
```

比对本地 `{skill_dir}/VERSION` 与远程 `git show origin/main:VERSION`：
- 相同 → 静默通过
- 不同 → 提示用户："WeWrite 有新版本可用（当前 X → 最新 Y），说「更新」即可升级。"**不阻断流程**，继续 Step 1b
- git 不可用（无 .git 目录或 fetch 失败）→ 静默跳过

**1b. 加载风格**：

```
检查: {skill_dir}/style.yaml
```

- 存在 → 提取 `name`、`topics`、`tone`、`voice`、`blacklist`、`theme`、`cover_style`、`author`、`content_style`
- 不存在 → `读取: {skill_dir}/references/onboard.md`，完成后回到 Step 1

如果用户直接给了选题 → 跳到 Step 3（仍需框架选择和素材采集，不可跳过）。

---

### Step 2: 选题

**2a. 热点抓取**：

```bash
python3 {skill_dir}/scripts/fetch_hotspots.py --limit 30
```

**降级**：脚本报错 → WebSearch "今日热点 {topics第一个垂类}"

**2b. 历史去重 + SEO**：

```
读取: {skill_dir}/history.yaml（不存在则跳过）
```

```bash
python3 {skill_dir}/scripts/seo_keywords.py --json {关键词}
```

**降级**：SEO 脚本报错 → LLM 判断

**2c. 生成 10 个选题**：

```
读取: {skill_dir}/references/topic-selection.md
```

每个选题含标题、评分、点击率潜力、SEO 友好度、推荐框架。近 7 天已写的关键词降分。

- 自动模式 → 选最高分
- 交互模式 → 展示 10 个，等用户选

---

### Step 3: 框架 + 素材

**3a. 框架选择**：

```
读取: {skill_dir}/references/frameworks.md
```

5 套框架（痛点/故事/清单/对比/热点解读），自动选推荐指数最高的。

**3b. 素材采集（关键——决定能否通过 AI 检测）**：

纯 LLM 生成的内容无论技巧多好，底层 token 分布仍是 AI 的。通过检测的文章都建立在真实外部信息源之上。

```
WebSearch: "{选题关键词} site:36kr.com OR site:mp.weixin.qq.com OR site:zhihu.com"
WebSearch: "{选题关键词} 数据 报告 2025 2026"
```

采集 5-8 条真实素材（具名来源 + 具体数据/引述/案例）。**禁止编造**。

**降级**：WebSearch 无结果或不可用 → 用 LLM 训练数据中可验证的公开信息。但需告知用户："素材采集未能使用 WebSearch，文章的 AI 检测通过率会降低。建议在编辑锚点处多加入你自己的内容。"

---

### Step 4: 写作

```
读取: {skill_dir}/references/writing-guide.md
读取: {skill_dir}/playbook.md（如果存在，逐条执行，优先于 writing-guide）
读取: {skill_dir}/writing-config.yaml（如果存在，作为写作参数）
读取: {skill_dir}/history.yaml（最近 3 篇的 dimensions 字段）
```

**4a-0. 历史最佳参数参考**（有 history.yaml 且包含 composite_score 时执行）：

读取 history.yaml 中有 `composite_score` 和 `writing_config_snapshot` 的文章，找到得分最低（最人类）的一篇。如果该篇得分比当前 writing-config.yaml 的默认参数对应的历史平均分更好，在写作时**参考**其参数组合（不是覆盖 writing-config.yaml，而是作为"上次这组参数效果好"的提示）。

具体：如果历史最佳文章的某个参数值与当前 writing-config 不同，在写作时倾向使用历史最佳值。如果没有历史数据，跳过此步。

**4a. 维度随机化**：从 writing-guide.md 规则 3.4 维度池随机激活 2-3 个维度，对比历史去重。

**4b. 加载写作人格**：

```
读取: {skill_dir}/personas/{style.yaml 的 writing_persona 字段}.yaml
如果 style.yaml 没有 writing_persona 字段 → 默认 midnight-friend
```

人格文件定义了：语气浓度、数据呈现方式、情绪弧线、段落节奏、不确定性表达模板等。作为 Step 4c 的硬性约束执行。

**优先级**：playbook.md（confidence ≥ 5 的规则）> persona > writing-guide.md。writing-guide 是底线（禁用词等），persona 在此基础上特化风格参数，playbook 中高置信度规则是用户个性化的最终覆盖。playbook 中 confidence < 5 的规则作为软性参考。

**4c. 写文章**：
- H1 标题（20-28 字） + H2 结构，1500-2500 字
- 真实素材锚定：Step 3b 的素材分散嵌入各 H2 段落
- **写作人格**：按 4b 加载的人格参数写作（数据呈现方式、个人声音浓度、不确定性表达等）
- 3 层反检测规则（统计/语言/内容）在初稿阶段全部生效
- 2-3 个编辑锚点：`<!-- ✏️ 编辑建议：在这里加一句你自己的经历/看法 -->`
- 可选容器语法：`:::dialogue`、`:::timeline`、`:::callout`、`:::quote`

保存到 `{skill_dir}/output/{date}-{slug}.md`

---

### Step 5: SEO + 验证

```
读取: {skill_dir}/references/seo-rules.md
```

**5a. SEO**：3 个备选标题 + 摘要（≤54 字）+ 5 标签 + 关键词密度优化

**5b. 去 AI 逐层验证**（writing-guide.md 自检清单，每项必须通过）：

| 层级 | 检查项 | 标准 | 规则 |
|------|--------|------|------|
| 统计 | 句长方差 | 最短与最长句相差 ≥ 30 字 | 1.1 |
| 统计 | 词汇温度 | 任意 500 字 ≥ 3 种温度 | 1.2 |
| 统计 | 段落节奏 | 无连续 2 个相近长度段落 | 1.3 |
| 统计 | 情绪极性 | 负面情绪 ≥ 2 处，无平铺直叙 | 1.4 |
| 统计 | 副词密度 | 无连续两句以副词开头 | 1.5 |
| 统计 | 风格漂移 | 不同 H2 语气/正式度有差异 | 1.6 |
| 语言 | 禁用词 | 命中数 = 0 | 2.1 |
| 语言 | 破句 | ≥ 3 处 | 2.2 |
| 语言 | 意外用词 | ≥ 1 处非常规但说得通的表达 | 2.3 |
| 语言 | 连贯性 | ≥ 1 处跑题再拉回 | 2.4 |
| 内容 | 真实锚定 | 每个 H2 ≥ 1 条真实素材，零编造 | 3.1 |
| 内容 | 具体性 | 每 500 字 ≥ 2 处具体细节 | 3.2 |
| 内容 | 密度波浪 | 高密度段后跟低密度段 | 3.3 |
| 内容 | 维度贯穿 | 激活维度全文可见 | 3.4 |

不通过 → 定向重写该段落。3 次仍不过 → 标注跳过。

**5b-2. 脚本验证**（补充逐项检查）：

Agent 在 5b 逐项检查时同步完成 Tier 3 评估（风格漂移、密度波浪、连贯性打破、整体人感），产出 0-1 分数。

```bash
python3 {skill_dir}/scripts/humanness_score.py {article_path} --json --tier3 {agent_tier3_score}
```

解读 JSON 中 `composite_score`：
- < 30 → 通过，继续 Step 6
- 30-50 → 查看 `param_scores` 中最低分项，定向重写对应段落
- \> 50 → 重大问题，逐个低分项修复，最多 3 轮

---

### Step 6: 视觉 AI

**如果 `skip_image_gen = true`** → 只执行 6a。

```
读取: {skill_dir}/references/visual-prompts.md
```

**6a.** 分析文章结构，生成封面 3 组创意 + 内文 3-6 张配图提示词。

**6b.** 调用 image_gen.py 生成图片，替换 Markdown 占位符。

**降级**：生图失败 → 输出提示词，继续。

---

### Step 7: 预检 + 排版 + 发布

**7a. Metadata 预检**（发布前必须通过）：

| 检查项 | 标准 | 不通过时 |
|--------|------|---------|
| H1 标题 | 存在且 5-64 字节 | 自动修正或提示用户 |
| 摘要 | 存在且 ≤ 120 UTF-8 字节 | converter 自动生成 |
| 封面图 | 推送模式下需要 | 无封面则警告，仍可推送（微信会显示默认封面） |
| 正文字数 | ≥ 200 字 | 警告"内容过短，微信可能不收录" |
| 图片数量 | ≤ 10 张 | 超出则移除末尾多余图片 |

预检全部通过后才进入排版。

**7b. 排版 + 发布**：

**如果 `skip_publish = true`** → 直接走 preview。

```
读取: {skill_dir}/references/wechat-constraints.md
```

Converter 自动处理：CJK 加空格、加粗标点外移、列表转 section、外链转脚注、暗黑模式、容器语法。

```bash
# 发布
python3 {skill_dir}/toolkit/cli.py publish {markdown} --cover {cover} --theme {theme} --title "{title}"

# 降级：本地预览
python3 {skill_dir}/toolkit/cli.py preview {markdown} --theme {theme} --no-open -o {output}.html
```

---

### Step 8: 收尾

**8a. 写入历史**（推送成功或降级都要写，文件不存在则创建）：

```yaml
# → {skill_dir}/history.yaml
- date: "{日期}"
  title: "{标题}"
  topic_source: "热点抓取"  # 或 "用户指定"
  topic_keywords: ["{词1}", "{词2}"]
  framework: "{框架}"
  word_count: {字数}
  media_id: "{id}"  # 降级时 null
  writing_persona: "{人格名}"
  dimensions:
    - "{维度}: {选项}"
  composite_score: {Step 5b-2 的 composite_score}  # 0=人类, 100=AI
  writing_config_snapshot:  # 本次使用的关键参数（从 writing-config.yaml 提取）
    sentence_variance: {值}
    paragraph_rhythm: "{值}"
    emotional_arc: "{值}"
    word_temperature_bias: "{值}"
    broken_sentence_rate: {值}
    tangent_frequency: "{值}"
    style_drift: {值}
    negative_emotion_floor: {值}
  stats: null
```

**8b. 回复用户**：

- 最终标题 + 2 备选 + 摘要 + 5 标签 + media_id
- 编辑建议："文章有 2-3 个编辑锚点，建议花 3-5 分钟加入你自己的话，效果更好。"
- 飞轮提示："编辑完成后说**'学习我的修改'**，下次初稿会更接近你的风格。"

**8c. 后续操作**：

| 用户说 | 动作 |
|--------|------|
| 润色/缩写/扩写/换语气 | 编辑文章 |
| 封面换暖色调 | 重新生图 |
| 用框架 B 重写 | 回到 Step 4 |
| 换一个选题 | 回到 Step 2c |
| 看看有什么主题 | `python3 {skill_dir}/toolkit/cli.py gallery` |
| 换成 XX 主题 | 重新渲染 |
| 看看文章数据 | `读取: {skill_dir}/references/effect-review.md` |
| 学习我的修改 | `读取: {skill_dir}/references/learn-edits.md` |
| 做一个小绿书/图片帖 | `python3 {skill_dir}/toolkit/cli.py image-post img1.jpg img2.jpg -t "标题"` |
| 诊断配置 / 检查反AI / 为什么AI检测没过 | `python3 {skill_dir}/scripts/diagnose.py --json` + LLM 交叉分析 |
| 优化写作参数 / 优化参数 | 迭代循环：写测试短文 → 打分 → 调参（见辅助功能） |

---

## 错误处理

| 步骤 | 降级 |
|------|------|
| 环境检查 | 逐项引导，设降级标记 |
| 热点抓取 | WebSearch 替代 |
| 选题为空 | 请用户手动给选题 |
| SEO 脚本 | LLM 判断 |
| 素材采集（WebSearch） | LLM 训练数据中可验证的公开信息 |
| 维度随机化 | history 空时跳过去重 |
| Persona 文件不存在 | 回退到 midnight-friend（默认） |
| 去 AI 验证 | 3 次重写不过则跳过该项 |
| 生图失败 | 输出提示词 |
| 推送失败 | 本地 HTML |
| 历史写入 | 警告不阻断 |
| 效果数据 | 告知等 24h |
| Playbook 不存在 | 用 writing-guide.md |
