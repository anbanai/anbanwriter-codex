---
name: article-visual-design
description: Manages images for WeChat article (公众号图文) content including cover generation (封面), content illustration (配图), compression, and CDN upload. Use when generating or processing images for WeChat articles. Also use when user mentions '封面', '配图', '插图', '视觉设计', '图片上传', 'generate cover', or when the article pipeline calls for image planning, generation, or uploading.
---

# 公众号图文图片管理

## MCP 工具

| MCP 工具 | 说明 |
|----------|------|
| `generate_image` (channel_id, prompt, image_type, output_path, task_id, ref_image_path, upload_to_cdn?) | 生成单张图片。`upload_to_cdn=true`（**生成与上传原子化**）时在同一调用内完成"生成→保存→压缩→上传微信 CDN"，直接返回 `wechat_url` + `media_id`；上传失败返回 `upload_error`（生成不浪费，仅重试上传）。返回 download_url（始终为可 HTTP fetch 的存储 URL，不再返回 base64 data URL）、file_path、wechat_url、media_id |
| `upload_image` (channel_id, file_path) | 上传**外部/下载来的**图片到微信 CDN，返回 CDN URL。**已生成的图不再用此工具**——生成时直接用 `generate_image(upload_to_cdn=true)` 原子上传；仅当 `generate_image` 返回 `upload_error` 时作为重传兜底 |
| `download_image` (channel_id, url) | 下载在线图片 |
| `compress_image` (file_path) | 压缩图片 |

---

## 核心原则：配置优先，账号/内容细化，Writer 无关

公众号"模板"由三个**正交**维度组成：图片视觉（`style`）、写作风格（`writing_style`）、排版样式（`theme`）。三者各自独立解析，互不推导——**写作风格绝不决定图片视觉**。

**Writer YAML 仅定义文字风格，不携带任何视觉/封面字段**（曾经的 `cover_style`/`cover_prompt` 已移除）。

视觉风格的**权威来源**是任务已解析的 `style` 字段（由 `get_channel_profile` 按 `task > template > plan > channel` 解析，并通过 profile 的 `style` / `style_source` / `template_style` 字段返回）：

- **有配置值**（`style_source` 为 task/template/plan/channel 之一）→ 以它为**权威视觉锚点**。下面的三维分析只做**细化充实**（配色、情绪、构图），**不得偏离或冲突**。例如配置了"温暖自然的生活摄影"，分析就只能往暖色调、自然光、真实场景细化，**不得**生成维多利亚木刻/黑白版画等冲突风格。
- **无配置值**（所有层级都未配置视觉）→ 执行完整三维分析兜底：图片视觉风格由账号定位、内容主题、目标受众三个维度独立确定，与 writer 选择无关。即使使用 dan-koe（犀利深刻写作风格）为养生账号写文章，图片也应使用温暖自然的视觉风格，而非维多利亚版画。

三维风格分析详见 [references/cover.md](references/cover.md)。

---

## 封面图生成

封面设计规范详见 [references/cover.md](references/cover.md)。

### 流程

1. 从 `get_channel_profile` 结果中读取任务已解析的视觉风格（**权威来源**）：`style`（解析后的视觉风格描述/关键词）、`style_source`（task / template / plan / channel）、`template_style`（任务带模板时）；同时读取账号定位、关键词、受众信息
2. 读取文章内容（`$DIR/04-article-final.md`）提取主题和关键意象
3. **视觉风格确定（配置优先，分析兜底）**：
   - 若 `style` 非空 → 以它为 `$VISUAL_STYLE` 的核心锚点，三维分析只做**补充细化**（配色、情绪、构图），**不得覆盖或偏离**配置的视觉方向
   - 若 `style` 为空（所有层级都未配置视觉）→ 执行完整三维分析兜底（账号定位 + 内容主题 + 目标受众）→ 确定视觉风格、色彩基调、情绪氛围
4. 从零构建封面 prompt（视觉方向取自任务解析的 `style` 字段——**不从 writer YAML 推视觉**，writer 仅决定文字风格，已不再携带任何视觉/封面字段）
5. 构建 prompt 时（如 MCP 工具可用）可调用 `list_resources(category="image_presets")` 获取封面预设模板列表，再用 `get_resource(category="image_presets", name="cover-default")` 等获取具体预设模板，将 `{{ARTICLE_TITLE}}`、`{{ARTICLE_SUMMARY}}`、`{{VISUAL_STYLE}}`、`{{ASPECT_RATIO}}` 等变量替换为实际内容。如果工具不可用，直接从零构建 prompt
6. 调用 `generate_image(channel_id="$CHANNEL_ID", prompt=封面提示词, image_type="cover", output_path="$DIR/cover.png", task_id="$TASK_ID", size="2.35:1", upload_to_cdn=true)` 生成——**生成与上传原子化**：同一调用内完成生成→保存→压缩→上传微信 CDN，直接返回 `media_id` + `wechat_url`。**不再单独调用 `upload_image`**。
7. 从 `generate_image` 返回值取 `media_id` + `wechat_url`。若返回 `upload_error`（生成成功但上传失败），用 `upload_image(channel_id="$CHANNEL_ID", file_path="$DIR/cover.png")` 单独重传即可，**无需重新生成**。

### 产出

- `$DIR/cover.png` — 封面图文件
- `media_id` — 微信素材 ID
- `$VISUAL_STYLE` — 三维分析确定的视觉风格描述
- `$COLOR_PALETTE` — 色彩基调
- `$COVER_PATH` — 封面图路径（供配图参考链使用）

---

## 图片内容规划

在生成任何内容配图之前，必须先创建 `image-plan.md`，为每张配图规划具体内容和视觉方向。这是确保配图与文章内容关联性的核心环节。

设计规范详见 [references/content.md](references/content.md)。

### 规划流程

#### 步骤 1：提取章节内容

读取 `$DIR/04-article-final.md`，对每个 `##` 章节提取：

- **核心论点**：该章节要传达的关键信息（1 句话）
- **情感基调**：理性分析、温暖鼓励、犀利批判、诗意沉思、轻松幽默等
- **具体素材**：章节中使用的案例、比喻、场景描述、引用等
- **原文摘录**：可直接支撑配图 prompt 的章节原句或短段（写入 `source_excerpt`）

#### 步骤 2：分配构图类型

从 8 种构图类型中为每章配图选择不同类型（参见 [references/content.md](references/content.md)）：
- 中心聚焦、对角线流动、三分法、前景/背景、俯拍、特写、留白主导、重复图案
- 3 张以上配图时，必须使用 3 种以上不同构图

#### 步骤 3：写入 image-plan.md

按模板写入 `$DIR/image-plan.md`，包含总体策略和每章配图的详细规划。每张图必须包含 `chapter_title`、`core_point`、`source_excerpt`、`visual_subject`、`composition_type`、`prompt_strategy`。

### 产出

- `$DIR/image-plan.md` — 配图内容规划文档

---

## 内容配图设计与生成

设计规范详见 [references/content.md](references/content.md)。

### 参考链

**所有内容配图使用封面图作为参考锚点**：

```
所有内容配图：ref_image_path="$DIR/cover.png"（始终用封面，不用上一张）
```

为什么始终用封面：使用上一张会导致风格漂移（每张图的微小差异累积放大），封面是风格锚点。

### 生成流程

对 `image-plan.md` 中每个章节配图执行：

1. 根据 image-plan 中的分析构建 prompt（必须包含章节具体概念、比喻或案例）
2. 调用 `generate_image`（`channel_id="$CHANNEL_ID"`, `prompt=章节提示词`, `image_type="content"`, `output_path="$DIR/img_N.png"`, `task_id="$TASK_ID"`, `ref_image_path="$DIR/cover.png"`, **`upload_to_cdn=true`**）——**生成与上传原子化**：同一调用内完成生成→保存→压缩→上传微信 CDN，返回值直接带 `wechat_url` + `media_id`。**不再有独立的 `upload_image` 阶段**。
3. 从 `generate_image` 返回值取 `wechat_url` → 作为该图的 CDN URL。若返回 `upload_error`（生成成功但上传失败），记到记录的 `upload_error` 字段并用 `upload_image(file_path="$DIR/img_N.png")` 单独重传（**不重新生成**）。
4. **每张图返回后立即原子写 `$DIR/images.json`**：先写临时文件 `$DIR/.images.json.tmp` → `fsync` → `rename` 覆盖 `$DIR/images.json`。**绝不要"攒齐所有图再一次性写"**——那是丢失窗口；逐张落盘使中断最多丢失"正在生成的那一张"，已上 CDN 的全部安全。每条记录在原有审计字段基础上增加 `wechat_url` 和 `media_id`。
5. 将 `![描述](CDN_URL)` 插入到章节关键段落之后（不紧跟 `##` 标题，不在章节末尾）

### Prompt 要求

- 必须包含章节中的具体概念、比喻或案例作为视觉主体
- 必须引用 `image-plan.md` 的 `source_excerpt` 或等价章节原文信息
- 避免抽象通用描述（如"商务场景"、"科技背景"）
- 不同章节的 prompt 必须有明显区别
- 视觉风格和色彩与封面一致

---

## 质量验证

生成完成后执行 5 项检查：

- [ ] **文件完整性**：所有图片文件存在且可访问
- [ ] **风格一致性**：读取 `$DIR/images.json`，确认所有内容配图记录 `ref_image_path="$DIR/cover.png"`
- [ ] **视觉多样性**：读取 `$DIR/image-plan.md` 与 `$DIR/images.json`，确认 3 张以上配图使用 3 种以上不同 `composition_type`
- [ ] **内容关联性**：逐项对照 `image-plan.md`，确认每个 prompt 引用了 `source_excerpt` 或章节具体内容（非通用描述）
- [ ] **审计完整性**：每条图片记录必须包含 `chapter_title`、`composition_type`、`visual_subject`、`prompt_source_excerpt`、`ref_image_path`、`image_type`、`quality_status`、`wechat_url`、`media_id`

未通过检查时：重试对应配图（更换 prompt 措辞），仍失败则记录问题继续后续章节。

---

## 保存结果

- 将含 CDN 图片链接的文章覆盖写回 `$DIR/04-article-final.md`
- 将所有配图信息保存为 `$DIR/images.json`（含 index, image_type, chapter_title, composition_type, visual_subject, prompt, prompt_source_excerpt, ref_image_path, file_path, url, wechat_url, media_id, quality_status）

---

## 技术规范

**微信图片限制**：
- 最大尺寸：10MB（超出会被自动压缩）
- 最大宽度：1920px（保持比例压缩）
- 支持格式：JPG、PNG、GIF、WebP

**公众号常用比例**：
- 正文配图：16:9 或 4:3 横版
- 封面图（公众号封面）：2.35:1（900x383px 标准）
- 正方形配图：1:1

---

## 构图类型选择指南

根据章节内容主题推荐构图类型：

| 章节主题 | 推荐构图 | 原因 |
|----------|----------|------|
| 开篇引入 / 总述 | 中心聚焦 | 建立视觉焦点，统领全文 |
| 过程 / 变化 / 方法 | 对角线流动 | 表达动态感和方向性 |
| 平衡分析 / 多角度 | 三分法 | 自然和谐，适合并列观点 |
| 层次 / 上下文 / 环境 | 前景/背景 | 表达深度和空间关系 |
| 细节 / 数据 / 质感 | 特写 | 强调微观细节和真实感 |
| 沉思 / 哲理 / 极简 | 留白主导 | 营造意境和呼吸感 |
| 节奏 / 规律 / 重复 | 重复图案 | 表达秩序感和韵律感 |
| 总览 / 全貌 / 结构 | 俯拍 | 展现整体结构和关系 |

---

## 常见失败与修复

| 问题 | 原因 | 修复 |
|------|------|------|
| 配图与内容无关 | prompt 使用抽象通用描述 | 从章节中提取具体概念/比喻/案例作为视觉主体 |
| 风格不一致 | 未使用封面作为参考图 | 确保所有内容图传入 `ref_image_path="$DIR/cover.png"` |
| 所有配图构图雷同 | 未在 image-plan 中分配不同构图 | 3+ 张图时强制使用 3+ 种构图类型 |
| 封面与文章主题脱节 | 封面 prompt 缺少内容主题关联 | 在封面 prompt 中包含文章的视觉隐喻 |
