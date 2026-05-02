# 第 10 周反思笔记

> 完成时间：2026-04-28
> 作业主题：基于 Skill 的自动化多模态内容二创
> 仓库：https://github.com/HKFORWARD/PolyStudio-homework/tree/homework/week10

---

## 一、环境踩坑录（耗时最多但最值得记）

### 1. shell `OPENAI_API_KEY` 抢走 `.env` 的优先级

我 `.zshrc` 里长期 export 着一个 OpenAI 官方 key（`sk-proj-...`），结果 PolyStudio 项目的 `.env` 里写的 SiliconFlow key 一直被它"吃掉"——后端实际拿着 OpenAI 的 key 去打 SiliconFlow 的接口，必然 401。

诊断走了好几轮弯路：先以为 URL 错（`.com` vs `.cn`），然后怀疑 key 失效，最后才意识到是**python-dotenv 默认行为：不覆盖已存在的环境变量**。

修复就一行：[main.py:16](backend/app/main.py#L16) `load_dotenv()` → `load_dotenv(override=True)`。

**记忆点**：以后任何 `.env` 不生效的问题，先 `env | grep ` 看 shell 里有没有同名变量在抢戏。

### 2. conda 环境激活后 `which python` 还指向别处

`conda activate polystudio` 之后 `which python` 仍返回 `deepseek` env 的路径——导致我第一次安装依赖装到了错地方。后来习惯用绝对路径 `/Users/.../envs/polystudio/bin/python` 直接调用，绕开 zshrc 的 PATH 怪事。

### 3. 跨工具复制丢失文件权限

`start.sh` 的执行位 `+x` 不知道在哪一步丢了，运行时直接 "Permission denied"。**新拿到一个项目第一件事**：`find . -name "*.sh" -exec chmod +x {} +`。

### 4. macOS qlmanage 强制方形输出

把高图（1000×1380）转成 PNG 时被剪成 2000×2000 方形，下半部分内容丢了。换成 Chrome headless 才保留宽高比。

---

## 二、认知被翻转过几次（Task 1 的"啊哈"瞬间）

### 翻转 1：以为 Skill 加载有缓存——实际没有

直觉上扫描 `skills/` 目录这么贵的操作肯定要缓存住——结果 [scan_available_skills](backend/app/services/skill_service.py#L68) 没有 `@lru_cache`、没有全局变量，**每次 /chat 请求都现读磁盘**。这"看似低效"的设计反而是"Skill 修改后无需重启"的实现机制。

**先入为主的"性能优化直觉"在 LLM Agent 项目里有时是错的**——开发体验有时比微秒级性能更值钱。

### 翻转 2：以为 Progressive Loading 是分块加载——实际是 metadata + on-demand

我以为这个词意味着"把大文件切成几段流式读"。实际是 system prompt **只注入 name + description + 路径**，正文一个字节都不进 prompt；Agent 判断匹配后**主动调 `read_skill_file_tool`** 按需读。token 占用 = 常数级。

### 翻转 3：以为 Agent 听 SKILL.md 步骤是因为硬编码执行器——实际全靠 prompt

我赌肯定有某个状态机/调度器在解析 SKILL.md 的"步骤 1→2→3"然后强制执行。直到读到 [agent_service.py:82-87](backend/app/services/agent_service.py#L82) 这一行才翻转：

```python
agent = create_react_agent(name=..., model=..., tools=tools, prompt=full_prompt)
```

只 4 个参数。**没有 `steps` / `workflow` / `pipeline`**。SKILL.md 的步骤通过 `prompt=full_prompt` 作为普通字符串注入——本质和 system prompt 第一行写"你是友好助手"是同一个机制。

**这次翻转是整个作业最值钱的认知**——它直接预言了 Task 2 实测的所有"漏洞"。

### 翻转 4：以为工具间结果传递有专门 wiring——实际是 messages 共享黑板

读 [stream_processor.py](backend/app/services/stream_processor.py) 看到 `agent.astream({"messages": ...})` 才反应过来：所有工具结果都被框架包成 `ToolMessage` 追加到 messages 列表，下一轮 LLM 看完整列表后**自主**决定下一步——没有任何"工具间专线"。

---

## 三、Task 2 实战迭代（工程闭环的真实样子）

写完 320 行 SKILL.md 真机测了一次，**漏了两件事**：

| 漏洞 | 现象 | 根因 |
|---|---|---|
| 中途停下 | Agent 写完脚本就把脚本作为最终答复，没继续走 step 4-7 | SKILL.md 没明确"7 步全完成才算结束" |
| 跳过前置检查 | Agent 没核对文件大小，直接调 omni 撞 28MB 限制 | Step 1 没强制要求"文字 self-check 后再继续" |

**修复办法不是改 Python 代码——是继续往 prompt 里加更"凶"的话**：

1. 顶部加 **「铁律 5 条」**，第 1 条就是"必须 7 步全完成才算结束"
2. Step 1 强制写出 "前置检查通过：路径=X..." 文字才能继续
3. Step 3 末尾直接对 LLM 喊话：**"如果你刚写完脚本想告诉用户——停止这个念头，立即调 voice_design"**

第 3 条这种"anthropomorphic instruction"措辞反直觉但真有效。**这次迭代亲身验证了翻转 3 的结论**：SKILL.md 是 prompt 文字、LLM 完全可以"理解错"边界。**可靠性 = 指令遵循能力 × prompt 措辞质量**。

---

## 四、设计哲学沉淀（可迁移到下一个 Skill）

写 SKILL.md 不是写文档——是**给 LLM 当 prompt**。这意味着：

- **默认值就是无人值守**——任何"没说"的参数都要在 SKILL.md 里写死，不能让用户中途填
- **"禁止行为"列表比"应该做"更有效**——明确禁止"询问/确认/中途停下"比鼓励"自动跑完"管用
- **失败 fail fast 比 retry forever 更工程化**——1 次重试拿走 70% 瞬时抖动收益，3 次性价比骤降还可能耗光配额
- **优雅降级**——核心功能能交付就交付（BGM 库空 → 跳过混音直接给无 BGM 版本）

---

## 五、对自己学习方向的思考

我学这个课**不是为了搞 AI 研究，是为了用 AI 工具做内容生成**——播客、博客、视频。这次作业最值钱的不是理解 ReAct 范式，而是看懂了**怎么用 SKILL.md 把多个工具串成一个无人值守工作流**。

具体能套用到我自己的场景：
- video → podcast 这条链路，未来可以平移成 video → 小红书图文 / video → 短视频脚本
- "默认值汇总 + 禁止行为汇总 + 失败兜底汇总" 这套 SKILL.md 三段式写法，可以复用到任何无人值守任务

未来 1-2 周想做的小实验：

- [ ] 给 `media2podcast` 扩展双人对谈版本（约 30% 工作量）
- [ ] 写一个 video → 小红书图文 Skill（同样的 7 步模板，换工具）
- [ ] 把 SKILL.md 三段式骨架抽成自己的 template，下次创建新 Skill 直接套
