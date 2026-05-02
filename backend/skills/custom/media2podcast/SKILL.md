---
name: media2podcast
description: 根据上传的视频或音频自动二创播客，全程一次性完成、无需中途干预。
---

# 媒体到播客的自动二创

## ⚠️ 铁律（贯穿全流程，绝不破例）

1. **必须 7 步全部完成**才能向用户交付——任何中间产出（DRAFT_SCRIPT / VOICE_SAMPLE / SEGMENTS / CONCATENATED_AUDIO）**都不是最终答案**。
2. **二创 ≠ 拼接原内容**——二创是用原内容**启发**新创作。成品的所有音频段必须是 Step 5 voice_cloning 新生成的。
3. Step N 完成后**立即开始 Step N+1**——不停顿、不汇报中间产出、不询问用户。
4. 仅当 Step 7 输出 `FINAL_AUDIO` 时，才能向用户报告"播客已生成"——这是**唯一**允许"停下并交付"的时刻。
5. 如果你刚写完脚本，正在考虑"是不是该把脚本告诉用户"——**停止这个念头，立即调 `qwen_voice_design_tool`**。

## 用语与生命周期

| 名称 | 来源 | 生命周期 | 是否进入成品 |
|---|---|---|---|
| `SOURCE_MEDIA` | 用户上传的 mp3/mp4 等媒体 | **Step 2 一次性消耗后失效** | ❌ |
| `DRAFT_SCRIPT` | Step 3 LLM 输出的播客脚本 | Step 3-5 内部使用 | ❌ |
| `VOICE_SAMPLE` | Step 4 voice_design 输出的样本 | 仅 Step 5 当 reference | ❌ |
| `SEGMENTS[N]` | Step 5 voice_cloning ×N 输出 | Step 6 拼接源 | ✅ **唯一成品来源** |
| `CONCATENATED_AUDIO` | Step 6 拼接结果 | Step 7 混音输入 | ❌（中间态） |
| `FINAL_AUDIO` | Step 7 混音输出 | 交付用户 | ✅ |

⚠️ **`SOURCE_MEDIA` 在 Step 2 后视为已消耗**——任何后续步骤引用它都是错误。

## 适用场景

**触发条件（必须同时满足）**：
- ✅ 用户已上传视频（mp4/mov/avi/mkv/webm）或音频（mp3/wav/m4a）文件
- ✅ 用户表达了"做成播客"、"二创"、"提炼播客"或类似意图

**不适用**：
- ❌ 用户只给主题文字、没上传任何媒体 → 应使用 `podcast-creator`
- ❌ 用户上传图片 → 不在本 Skill 处理范围
- ❌ 视频拼接 / 字幕 / 长视频处理 → 不在本 Skill 处理范围

## 核心工作流程

完整成功案例见 `references/workflow_example.md`；3 个已知失败模式与修复见 `references/anti_patterns.md`——**Step 5 / Step 6 调用前建议主动 `read_skill_file` 加载 anti_patterns.md 复核**。

### 第 1 步 · 前置检查（门禁拦截，不调任何工具）

**强制核对（必须以可观察的方式完成）**：在调用任何工具之前，**必须在回复中明确写出一行**：

> "前置检查通过：路径=X，类型=Y，大小=Z MB"

否则视为未做检查。

**3 项条件全过才放行**：
- `media_path` 非空字符串
- 后缀在白名单：`.mp4 .m4v .mov .avi .mkv .webm .mp3 .m4a .wav`
- 文件大小 ≤ **18 MB**（Qwen-Omni 真实硬上限：data-uri item ≤ 20 MB binary，留 2 MB 缓冲）

**未通过 → 终止流程**：
- 后缀不在白名单 → "仅支持 mp4/mov/avi/mkv/webm 视频或 mp3/wav/m4a 音频格式"
- 大小超阈值 → "文件超 18 MB，请压缩到 15 MB 以下后重传（百炼 API 上限 20 MB）"
- `media_path` 为空 → "未检测到上传文件，请先上传视频或音频"

通过 → **立即**进 Step 2。

### 第 2 步 · 多模态理解（消耗 SOURCE_MEDIA）

**调用工具**：`qwen_omni_understand_tool`

**固定参数**：
- `media_path` = SOURCE_MEDIA 路径
- `question` = `"请详细描述这段媒体的核心内容、关键信息、人物、场景，输出可用于撰写播客脚本的素材摘要"` （**严禁修改这句**）
- `media_type` = 留空（按扩展名自动识别）
- `output_audio` = `false`（这一步只要文字）

**完成条件**：
- 工具返回的 JSON 字符串可解析，`text_response` 字段长度 ≥ 100 字
- 把 `text_response` 作为下一步素材摘要

**⚠️ 重要语义转折**：本步执行后，`SOURCE_MEDIA` **视为已消耗**——后续步骤**严禁**再引用其路径。

**失败兜底（fail fast，严禁重试）**：
- 含 `exceeds the maximum allowed` 或 `Exceeded limit on max bytes per data-uri item` → "文件过大，请压缩到 15 MB 以下后重传"（API 实际上限 20 MB binary，base64 后 28 MB string——更严的那个生效）
- 含 `未配置 DASHSCOPE_API_KEY` → "系统未配置百炼 API"
- 其它错误 → 原始错误透传给用户，终止流程

### 第 3 步 · LLM 写脚本（不调工具）

**目标**：基于 Step 2 的 `text_response`，**重新创作**一份单人独白脚本。**不是转写，是改写**。

**输出格式硬规定**：
- 形式：单人独白
- 长度：500 字（450-550 区间）
- 拆段：5-7 段，段间用空行（双换行 `\n\n`）分隔
- 每段：80-100 字，可独立朗读

**⚠️ 关键约束（与 Step 5/6 直接相关，请记住）**：
- 拆段后段数 = N，**N 必须满足 5 ≤ N ≤ 7**
- N 决定 Step 5 voice_cloning 的调用次数
- N 决定 Step 6 audio_files 的长度

**禁止行为**：
- ❌ 元评论（"好的，这是为您准备的脚本..."）
- ❌ 舞台提示（`(笑)`、`(停顿)`、`[音效]`）
- ❌ Markdown 格式（`**加粗**`、`#` 标题）、emoji
- ❌ 询问 / 确认句
- ❌ 视频时间戳引用（如 `(00:15)`）

**⚠️ 完成后立即进 Step 4——严禁停下、严禁向用户展示脚本完整内容、严禁等待评价**。

如果你刚写完脚本，正在考虑"是不是该把脚本告诉用户"——**停止这个念头，立即调 `qwen_voice_design_tool`**。

### 第 4 步 · 设计音色样本（生成 VOICE_SAMPLE）

**目标**：用一句**固定的、与播客内容无关的**试听文本生成音色样本。这段音频**不进入成品**，仅供 Step 5 当 reference。

**调用工具**：`qwen_voice_design_tool`

**固定参数**：
- `voice_description` = `"亲切自然的女主持人音色，温暖友好，语速适中、清晰流畅"` （**严禁修改**）
- `text` = `"大家好，欢迎收听今天的播客节目，今天我们将聊一个有意思的话题。"` （**严禁修改**）

**⚠️ 严禁**：把 `DRAFT_SCRIPT` 第一段、`SOURCE_MEDIA` 内容、播客主题等"内容相关"文字作为 `text` 参数。原因：`VOICE_SAMPLE` 不进成品，内容是什么不重要，**音色一致性**才重要。

**完成条件**：拿到 `audio_url` 路径，标记为 `VOICE_SAMPLE`。

**失败兜底**：返回 `Error:` → **终止流程，严禁重试**，原始错误透传给用户。

### 第 5 步 · 逐段克隆（生成 SEGMENTS[N]）

**目标**：对 `DRAFT_SCRIPT` 的 N 段（`DRAFT_SCRIPT.split("\n\n")`），**逐段调用** voice_cloning，统一使用 `VOICE_SAMPLE` 作为 reference。

**调用工具**：`qwen_voice_cloning_tool`（调用 N 次）

**正向 anchor**：voice_cloning 调用次数 **严格等于 N**，N ∈ [5, 7]。
**反向兜底**：调用次数 < 5 视为流程错误——回到 Step 3 检查脚本拆段是否漏了。

**每次调用的参数**：
- `reference_audio` = `VOICE_SAMPLE`（**全 N 段保持同一个**，严禁中途换）
- `text` = 当前段的完整文本（来自 `DRAFT_SCRIPT.split("\n\n")[i]`，**不是** `SOURCE_MEDIA` 内容）

**⚠️ 路径来源约束**：
- `text` 参数**只能**来自 `DRAFT_SCRIPT`
- **严禁**把 Step 2 的 `text_response` 摘要直接当 text（那是 LLM 改写前的素材，不是脚本）

**单段失败处理**（业内 retry-once 模式）：
- 单段返回 `Error:` → **同参数重试 1 次**
- 二次失败 → 终止整个流程，告知用户"第 X 段克隆失败，请稍后重试"
- **严禁第三次重试**

**完成条件**：
- 收集 N 个 `audio_url`，按段序排列成 `SEGMENTS[]`
- 每个路径必须以 `voice_cloning_` 开头

### 第 6 步 · 拼接（生成 CONCATENATED_AUDIO）

**调用前 self-check（必读，可主动 read_skill_file 加载 anti_patterns.md 复核）**：

| 检查项 | 期望 | 不通过怎么办 |
|---|---|---|
| `len(audio_files) == N` | 5-7 之间，与 Step 5 调用次数一致 | 回到 Step 5 补全 |
| 每个路径以 `voice_cloning_` 开头 | ✅ | 回到 Step 5 补全 |
| 任何一个路径含 `upload_` | ❌ 严禁 | 立即停止——SOURCE_MEDIA 已消耗 |
| 任何一个路径含 `voice_design_` | ❌ 严禁 | 立即停止——VOICE_SAMPLE 不进成品 |

**audio_files 白名单**：

| 路径前缀 | 处理 |
|---|---|
| `voice_cloning_*.wav` | ✅ **必须**全部纳入 |
| `voice_design_*.wav` | ❌ 严禁——音色样本不进成品 |
| `upload_*.{mp3,mp4,...}` | ❌ 严禁——`SOURCE_MEDIA` 已消耗 |
| 其它 | ❌ 严禁——只接受 voice_cloning 产物 |

**调用工具**：`concatenate_audio_tool`

**参数**：
- `audio_files` = `SEGMENTS`（N 个 voice_cloning .wav）
- `silence_duration` = `1200` （毫秒，业内主流播客节奏）

**完成条件**：拿到 `.wav` 路径，标记为 `CONCATENATED_AUDIO`。

**失败兜底**：返回 `Error:` → 终止流程，原始错误透传，**不重试**。

### 第 7 步 · 加 BGM 输出成品（生成 FINAL_AUDIO）

#### 子步 7a · 自动选 BGM
**调用工具**：`select_background_music_tool`
- 输入：脚本主题或 Step 2 摘要
- 工具内部用 LLM 自动从 `storage/bgm/` 按主题匹配

**BGM 库为空时的兜底（优雅降级）**：
- 工具返回空或错误 → **跳过混音**，直接把 `CONCATENATED_AUDIO` 复制为 .mp3 作为 `FINAL_AUDIO` 交付
- **不要因为 BGM 失败就走"包装原 mp3"的捷径**——成品仍必须是 voice_cloning 产物的拼接

#### 子步 7b · 混音（仅当 7a 成功）
**调用工具**：`mix_audio_with_bgm_tool`

| 参数 | 取值 |
|---|---|
| `dialogue_audio` | `CONCATENATED_AUDIO` |
| `bgm_audio` | 7a 选中的 BGM 路径 |
| `intro_duration` | `3000` 毫秒 |
| `bgm_volume_after_intro` | `0.05` (5%) |

**完成条件**：
- 输出 `.mp3` 路径，标记为 `FINAL_AUDIO`
- **此时**（且仅此时）才向用户报告："播客已生成：{FINAL_AUDIO}"
- 流程结束 ✨

**失败兜底**：
- 7b 失败 → 走 7a 的优雅降级（无 BGM 版本）
- 7a + 7b 都失败 → 终止流程，原始错误透传

## 默认值汇总

下列默认值**全部不询问用户**，遇到歧义直接采用：

| 参数 | 固定值 |
|---|---|
| 文件大小上限 | 18 MB（API 硬上限 20 MB binary，留 2 MB 缓冲） |
| 文件类型白名单 | `.mp4 .m4v .mov .avi .mkv .webm .mp3 .m4a .wav` |
| Omni `question` | （见 Step 2，固定不改） |
| Omni `output_audio` | `false` |
| 播客形式 | 单人独白 |
| 脚本字数 | 500 字（450-550） |
| 拆段数 N | 5-7 |
| 段落分隔符 | `\n\n` |
| voice_design `voice_description` | （见 Step 4，固定不改） |
| voice_design `text` | （见 Step 4，固定通用试听文本） |
| 段间静音 | 1200 ms |
| BGM 选择 | LLM 按主题自动匹配 |
| BGM 开场原声 | 3000 ms |
| BGM 背景音量 | 5% (0.05) |
| 单段失败重试 | 1 次 |

## TODO（未来扩展，本期不做）

- 📧 失败时邮件 / IM 通知 → 需先增加 `send_notification_tool`
- 🎙️ 双人对谈支持 → voice_design ×2，voice_cloning 按角色标签选音色
- ⏱️ 长播客（10-15 分钟）支持 → 字数 500 → 1500
- 📦 视频自动压缩 → 需先增加 video compression tool（绕开 28MB 限制）

## 参考资料

- 完整成功调用链：`references/workflow_example.md`
- 已知失败模式与修复：`references/anti_patterns.md`（Step 5/6 调用前建议主动加载）
