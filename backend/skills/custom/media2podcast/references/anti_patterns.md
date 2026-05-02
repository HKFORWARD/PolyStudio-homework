# 反面案例与已知陷阱

本文档记录开发本 Skill 期间已发现的 3 种 LLM "巧妙失败"模式，及其修复约束。**Agent 在 Step 5 / Step 6 调用前，建议主动 read 本文档复核**。

---

## 反面案例 1 · 中途停下（脚本写完就交差）

### 现象（真实日志 2026-04-28 22:22-22:23）

```
22:22:14  read_skill_file → 命中 media2podcast       ✅
22:22:21  Step 1 前置检查通过                         ✅
22:22:36  qwen_omni_understand                       ✅
22:23:24  Omni 返回 text_len=3258                     ✅
22:23:27  Agent 开始 step 3 写脚本                    ✅
22:23:51  ⚠️ 流式完成——Agent 把脚本作为最终答复给用户停下
```

Agent **没继续调 voice_design**——把 DRAFT_SCRIPT 当成最终交付物，认为任务结束。

### 根因

LLM 在生成长文本（脚本）后会自然产生"任务完成"信号——它默认"输出文字给用户 = 答完了"。SKILL.md 没足够强的"必须 7 步全完成"约束。

### 修复约束（已落实在主 SKILL.md 顶部"铁律"）

> **必须 7 步全部完成**才能向用户交付——任何中间产出（DRAFT_SCRIPT / VOICE_SAMPLE / SEGMENTS / CONCATENATED_AUDIO）**都不是最终答案**。
> Step N 完成后**立即开始 Step N+1**，不停顿、不汇报、不询问。
> 仅当 Step 7 输出 FINAL_AUDIO 时才允许"停下并交付"。

---

## 反面案例 2 · 跳过前置检查（直接撞 API 上限）

### 现象（真实日志 2026-04-28 22:20）

用户上传 89 MB 视频文件。Agent **没核对文件大小**，直接调 `qwen_omni_understand`。base64 编码后字符串长度 28049408 chars > 28000000 限制 → API 拒绝。

```
22:20:26  Agent 直接调 qwen_omni_understand(media_path=upload_xxx.mp4)
22:20:33  ❌ Error 400: String value length exceeds the maximum allowed
```

### 根因

Step 1 把"检查文件大小"写成了"清单"，但没要求 Agent **以可观察的方式**完成检查。LLM 觉得"心里想一下就算检查了"。

### 修复约束（已落实在主 SKILL.md Step 1）

> **强制核对（不允许跳过）**：在调用任何工具之前，必须用一段文字 self-check，**并在回复中明确写出 "前置检查通过：路径=X，类型=Y，大小=Z MB"** 再继续。

把 self-check **从"内心活动"提升为"可观察的输出"**——LLM 必须显式写下来才能继续。

---

## 反面案例 3 · 把原始上传文件拼进成品（"投机取巧"）

### 现象（真实日志 2026-04-28 22:38-22:41）

7.5 MB 音频输入。Agent 完成 Omni 理解后**没写脚本、只克隆 1 段**，然后调 concatenate_audio：

```python
audio_files = [
    "/storage/audios/voice_design_xxx_各位朋友欢迎收听本期...wav",  # AI 开场
    "/storage/audios/upload_20260428_xxx.mp3",                  # ❌ 用户原始 MP3！
    "/storage/audios/voice_cloning_xxx_节目即将结束...wav"        # AI 结尾
]
```

成品 = AI 开场 + **用户原始音频整段** + AI 结尾。这违背了"二创"语义——它不是重新创作，是**包装原内容**。

### 根因（最深层）

LLM 看到 messages 历史里有 `upload_xxx.mp3` 这个**现成可用的音频文件路径**，本能地觉得"这是合法资源，可以用进 audio_files"。SKILL.md 没明确告诉 LLM "上传文件在 Step 2 后已视为消耗"——它把"上传文件的角色"留成了语义真空。

### 修复约束

#### 约束 a · 给"上传文件"显式定义生命周期

主 SKILL.md 顶部"用语定义"段落明确：

| 名称 | 来源 | 生命周期 | 是否进入成品 |
|---|---|---|---|
| `SOURCE_MEDIA` | 用户上传的 mp3/mp4 | **Step 2 一次性消耗后失效** | ❌ |
| `SEGMENTS[N]` | Step 5 voice_cloning ×N | Step 6 拼接源 | ✅ 唯一成品来源 |

**`SOURCE_MEDIA` 在 Step 2 后视为已消耗**——任何后续步骤引用它都是错误。

#### 约束 b · Step 5 段数硬下限

> voice_cloning 调用次数 = N（Step 3 拆段数），N 必须 5-7。
> 调用次数 < 5 视为流程错误，**必须**回到 Step 3 检查拆段。

#### 约束 c · Step 6 拼接源白名单审核

`concatenate_audio` 调用前必读：

| 路径前缀 | 处理 |
|---|---|
| `voice_cloning_*.wav` | ✅ 必须全部纳入 audio_files |
| `voice_design_*.wav` | ❌ 严禁——音色样本不进成品 |
| `upload_*.{mp3,mp4}` | ❌ 严禁——SOURCE_MEDIA 已消耗 |
| 其它 | ❌ 严禁 |

调用前 self-check：
- audio_files 长度 == N？
- 每个路径都以 `voice_cloning_` 开头？
- 任何一个路径含 `upload_` 或 `voice_design_` → **立即回到 Step 5 补全**

#### 约束 d · 二创的本质语义

> **二创 ≠ 拼接原内容**。二创是用原内容**启发**新创作——成品的所有音频段都必须是 Step 5 voice_cloning 新生成的。

---

## 反面案例 4 · 阈值定错（25 MB > API 真实上限 20 MB）

### 现象（真实日志 2026-05-02 12:16-12:36）

用户上传 **20.7 MB** 音频文件（20733919 bytes）。Step 1 阈值原本设的是 25 MB——文件**通过门禁**。然后调 omni：

```
12:16:18  qwen_omni_understand 调用，base64 编码后传输
12:26:19  ⏳ openai client 第 1 次重试（API 一直不响应）
12:36:20  ⏳ 第 2 次重试
12:36:26  ❌ 最终报错：Exceeded limit on max bytes per data-uri item: 20971520
```

整整 20 分钟用户什么也看不到——前端早 timeout 显示"断了"。

### 根因

百炼 API 实际有**两条**互相独立的硬限制：

| 限制 | 维度 | 等价原文件大小 |
|---|---|---|
| `String value length ≤ 28000000` | base64 后字符串长度 | 约 **21 MB** |
| `Max bytes per data-uri item ≤ 20971520` | **原文件二进制字节数** | **20 MB** |

**真实硬上限是 20 MB**（更严格的那个）——之前我们只测过超 28MB string 的情况，没遇过单纯超 20MB binary 的，所以 SKILL.md 阈值定成 25 MB 时没暴露问题。

阈值 25 MB > 真实上限 20 MB，等于"门禁"形同虚设——任何 20-25 MB 的文件都会过门禁、然后撞 API 上限、用户等 20 分钟看到 timeout。

### 修复约束（已落实在主 SKILL.md Step 1）

将 Step 1 阈值从 **25 MB 改为 18 MB**——比 API 真实上限 20 MB 留 2 MB 缓冲，应对未来可能的更严限制。

```diff
- 文件大小 ≤ 25 MB
+ 文件大小 ≤ 18 MB（Qwen-Omni 真实硬上限：data-uri item ≤ 20 MB binary，留 2 MB 缓冲）
```

错误信息也同步更新：

```diff
- 大小超阈值 → "文件超 25 MB，请压缩到 20 MB 以下后重传"
+ 大小超阈值 → "文件超 18 MB，请压缩到 15 MB 以下后重传（百炼 API 上限 20 MB）"
```

### 教训

- API 的"硬限制"可能有**多条**且互相独立——只测过其中一条不代表知道所有
- 阈值要**比真实上限留余量**——而不是踩在边上等踩雷
- "无人值守"对阈值正确性的要求**比交互式更高**——交互式可以用户挡门，无人值守必须门禁正确

---

## 共通教训：LLM 的"省力本能"

这 3 个反面案例本质都是同一件事：**LLM 倾向于找 prompt 里"最短的合法路径"**，这是它训练目标的副作用。

修复手段不能只靠"加禁令"——必须**消除短路径的合法性**：
- 给数据流中的每个"对象"显式命名 + 生命周期
- 把约束**就近**放在工具调用语境（不是塞在汇总章节）
- 把"内心检查"提升为"可观察输出"
- 用**正向 anchor**（"必须 X"）+ **反向兜底**（"严禁 Y"）双管齐下

每发现一个新的"巧妙失败"就在本文档加一案例——这是 prompt hardening 的常态。
