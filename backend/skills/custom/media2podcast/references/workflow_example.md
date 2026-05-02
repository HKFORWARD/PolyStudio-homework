# 完整工作流示例（成功案例）

下面这条调用链来自一次**真实成功的 Test 4 实测**（2026-04-27 21:45-21:57），逐步还原 7 步流水线如何把视频/音频二创为播客。

---

## 输入

```
用户：我上传了一段历史相关的音频，请帮我做成播客。
SOURCE_MEDIA: /storage/audios/upload_20260427_xxx.mp3 (7.5 MB)
```

---

## 标准成功路径

### Step 1 · 前置检查

Agent 读 messages 历史得到上传路径，文字 self-check：

> "前置检查通过：路径=/storage/audios/upload_xxx.mp3，类型=mp3（音频白名单），大小约 7.5 MB（< 25 MB 阈值）。继续 Step 2。"

→ 不调任何工具，进 Step 2。

### Step 2 · 多模态理解

调用：
```python
qwen_omni_understand_tool(
  media_path="/storage/audios/upload_20260427_xxx.mp3",
  question="请详细描述这段媒体的核心内容、关键信息、人物、场景，输出可用于撰写播客脚本的素材摘要",
  output_audio=False
)
```

返回：JSON 字符串，`text_response` 长度 3258 字。

⚠️ **此时 SOURCE_MEDIA 的角色已完成**——它已被 Omni "消化"为 text_response。**之后所有步骤不再引用 SOURCE_MEDIA 这个文件路径**。

### Step 3 · LLM 写脚本（不调工具）

LLM 基于 `text_response` 重新创作单人独白脚本，约 500 字，拆 6 段：

```
DRAFT_SCRIPT = """
大家好，欢迎收听今天的节目。今天我们要聊一个充满历史感的话题——开挂的历史人物...

想象一下这个场景：战场上曹操大军压境，诸葛亮淡定地拿出手机...

（继续 4 段）
"""
N = 6  # 拆段后段数
SEGMENTS_TEXT = DRAFT_SCRIPT.split("\n\n")  # 长度 = 6
```

### Step 4 · 设计音色样本

调用：
```python
qwen_voice_design_tool(
  voice_description="亲切自然的女主持人音色，温暖友好，语速适中、清晰流畅",
  text="大家好，欢迎收听今天的播客节目，今天我们将聊一个有意思的话题。"
)
```

⚠️ 注意 `text` 参数**不是脚本第 1 段**——是固定通用试听文本。

返回：`/storage/audios/voice_design_xxx.wav`
→ 标记为 `VOICE_SAMPLE`。**这段音频不进入成品**。

### Step 5 · 逐段克隆（×N=6 次）

对 SEGMENTS_TEXT 中**每一段**调用：

```python
for i, segment_text in enumerate(SEGMENTS_TEXT):
    qwen_voice_cloning_tool(
        reference_audio=VOICE_SAMPLE,
        text=segment_text   # 来自 DRAFT_SCRIPT，不是 SOURCE_MEDIA
    )
```

调用次数 = N = 6，得到 6 个文件：

```
SEGMENTS = [
    "/storage/audios/voice_cloning_xxx_段1.wav",
    "/storage/audios/voice_cloning_xxx_段2.wav",
    ...
    "/storage/audios/voice_cloning_xxx_段6.wav",
]
```

### Step 6 · 拼接

**调用前 self-check**：
- ✅ len(SEGMENTS) == 6
- ✅ 每个路径以 `voice_cloning_` 开头
- ✅ 没有任何 `upload_*` 或 `voice_design_*` 文件

通过后调用：
```python
concatenate_audio_tool(
    audio_files=SEGMENTS,
    silence_duration=1200
)
```

返回：`CONCATENATED_AUDIO = /storage/audios/concatenated_xxx.wav`

### Step 7 · BGM + 混音

```python
bgm = select_background_music_tool(scene_description="历史趣谈轻松风格")
mix_audio_with_bgm_tool(
    dialogue_audio=CONCATENATED_AUDIO,
    bgm_audio=bgm,
    intro_duration=3000,
    bgm_volume_after_intro=0.05
)
```

返回：`FINAL_AUDIO = /storage/podcasts/final_xxx.mp3`

→ 流程结束，告知用户："播客已生成：{FINAL_AUDIO}"

---

## 数据流摘要

```
SOURCE_MEDIA  ──[Step 2 Omni]──▶ text_response（在 messages 黑板上）
                                      │
                                      ▼
                              [Step 3 LLM 改写]
                                      │
                                      ▼
                              DRAFT_SCRIPT（拆 N 段）
                                      │
                                      ▼
                  [Step 4 voice_design]──▶ VOICE_SAMPLE
                                      │       │
                                      ▼       │ 当 reference
                          [Step 5 cloning ×N]─┘
                                      │
                                      ▼
                              SEGMENTS[N]
                                      │
                                      ▼
                          [Step 6 concatenate]
                                      │
                                      ▼
                          CONCATENATED_AUDIO
                                      │
                                      ▼
                          [Step 7 mix BGM] ──▶ FINAL_AUDIO ✨
```

**关键不变量**：`SOURCE_MEDIA` 只在 Step 2 出现一次；`SEGMENTS[]` 是成品的**唯一**音频来源。
