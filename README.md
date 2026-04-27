# Skill是什么
Skills扩展了cluade code/openclaw的功能，模型在工作室时，会将一个简短的skills文档中的“description”加载进入上下文。模型将通过这个description的用途和用法，在执行过程中来决定是否要使用该skills。

[使用 skills 扩展 Claude - Claude Code Docs](https://code.claude.com/docs/zh-CN/skills)

[What are skills? - Agent Skills](https://agentskills.io/what-are-skills)

# 1 Skill调用框架
以openclaw最新的配置（v 2026.03.23）为例，所有的skills的名称和decription会被加载到系统提示词中，并以xml的方式编排成list供模型自己匹配，一次只能匹配一个skill使用。需要注意的是claude code的调用方式也许有所不同，所以在构造skills脚手架时，**<font style="background-color:#FBDE28;">并不能只按这一种模板</font>**。

```plain
function buildSkillsSection(params: { skillsPrompt?: string; readToolName: string }) {
    const trimmed = params.skillsPrompt?.trim();
    if (!trimmed) return [];
    return [
      "## Skills (mandatory)",
      "Before replying: scan <available_skills> <description> entries.",
      `- If exactly one skill clearly applies: read its SKILL.md at <location> with \`${readToolName}\`, then follow it.`,
      "- If multiple could apply: choose the most specific one, then read/follow it.",
      "- If none clearly apply: do not read any SKILL.md.",
      "Constraints: never read more than one skill up front; only read after selecting.",
      "- When a skill drives external API writes, assume rate limits...",
      trimmed,   // ← 这里插入 <available_skills> XML 块
      "",
    ];
  }

------------------------------------------------------------------
// <available_skills> XML 块
 <available_skills>                  
    <skill>                           
      <name>deploy</name>             
      <description>Deploy to prod     
      </description>                  
      <location>~/.agents/skills/     
        deploy/SKILL.md</location>    
    </skill>                          
    ...                               
  </available_skills> 
```



Skill 注入 System Prompt 的完整链路 ：                               

```plain
                      ┌─────────────────────────────────┐
                      │    SKILL.md 文件（多个来源）       │
                      │  ---                            │
                      │  name: deploy                   │
                      │  description: Deploy to prod    │
                      │  ---                            │
                      │  部署步骤指令...                  │
                      └──────────────┬──────────────────┘
                                     │
                      ① loadSkillEntries()
                         workspace.ts:293
                      ↓
            ┌─────────────────────────────────────┐
            │ 按优先级从6个来源加载 SKILL.md：        │
            │  extra < bundled < managed <         │
            │  ~/.agents/skills < .agents/skills   │
            │  < skills/（工作区）                   │
            │                                     │
            │ 同名 skill 后者覆盖前者（Map）          │
            │ 解析 frontmatter → SkillEntry[]      │
            └──────────────┬──────────────────────┘
                           │
              ② filterSkillEntries()
                 过滤 config/eligibility
                           │
              ③ 过滤 disableModelInvocation === true 的skill
                 (这些 skill 不进入 prompt)
                           ↓
            ┌─────────────────────────────────────┐
            │ applySkillsPromptLimits()            │
            │  workspace.ts:567                   │
            │                                     │
            │  三层降级策略：                        │
            │  Tier1: formatSkillsForPrompt()     │
            │    → <name> + <description> + <loc> │
            │    预算内? ✓ 使用                     │
            │                                     │
            │  Tier2: formatSkillsCompact()        │
            │    → <name> + <location> (无desc)    │
            │    预算内? ✓ 使用                     │
            │                                     │
            │  Tier3: 二分搜索截断                  │
            │    → 找最多能放下的 skill 子集         │
            └──────────────┬──────────────────────┘
                           │
                生成 skillsPrompt 字符串
                (XML格式的 <available_skills> 块)
                           │
              ④ resolveSkillsPromptForRun()
                 attempt.ts:1716
                 (优先使用 snapshot 缓存，否则现场构建)
                           ↓
            ┌─────────────────────────────────────┐
            │ buildEmbeddedSystemPrompt()          │
            │  → buildAgentSystemPrompt()          │
            │    system-prompt.ts:176              │
            │                                     │
            │  拼接最终 system prompt:              │
            │  ┌───────────────────────────┐      │
            │  │ "You are a personal..."    │      │
            │  │ ## Tooling                 │      │
            │  │ ## Tool Call Style         │      │
            │  │ ## Safety                  │      │
            │  │                            │      │
            │  │ ## Skills (mandatory) ← ⑤  │      │
            │  │ buildSkillsSection()       │      │
            │  │                            │      │
            │  │ ## Workspace               │      │
            │  │ ## Docs                    │      │
            │  │ ...                        │      │
            │  └───────────────────────────┘      │
            └──────────────┬──────────────────────┘
                           │
              ⑥ createSystemPromptOverride()
                 → session.agent.setSystemPrompt(prompt)
                           ↓
            ┌─────────────────────────────────────┐
            │  LLM API 调用                        │
            │  system prompt 中包含：               │
            │                                     │
            │  ## Skills (mandatory)               │
            │  Before replying: scan               │
            │  <available_skills> <description>    │
            │  entries.                            │
            │  - If exactly one skill clearly      │
            │    applies: read its SKILL.md at     │
            │    <location> with `read`, then      │
            │    follow it.                        │
            │  - If multiple could apply: choose   │
            │    the most specific one.            │
            │  - If none clearly apply: do not     │
            │    read any SKILL.md.                │
            │                                     │
            │  <available_skills>                  │
            │    <skill>                           │
            │      <name>deploy</name>             │
            │      <description>Deploy to prod     │
            │      </description>                  │
            │      <location>~/.agents/skills/     │
            │        deploy/SKILL.md</location>    │
            │    </skill>                          │
            │    ...                               │
            │  </available_skills>                 │
            └─────────────────────────────────────┘
```



# 2 Skill能力项
| 能力层 | 具体表现 | 训练信号来源 |
| --- | --- | --- |
| L1 语义匹配 | 用户说"帮我部署"→ 从 10 个 skill 中选中 description 含 "deploy to production" 的那个 | description 与 query 的语义距离 |
| L2 精准拒绝 | 用户问天气 → 10 个 skill 都是代码类 → 不调用任何 skill | 负样本训练 |
| L3 指令遵循 | SKILL.md 说"先跑测试再构建再部署"→ 模型严格按此顺序执行 tool call | 步骤覆盖率 + 顺序正确性 |
| L4 上下文适应 | SKILL.md 中引用了 scripts/validate.sh → 模型知道去执行这个相对路径的脚本 | 文件引用解析正确性 |
| L5 参数传递 | 用户说 /fix-issue 123 → $ARGUMENTS 被替换为 "123" → 模型理解并使用 | 参数出现在 tool call 中 |
| L6 容错恢复 | 执行某步 tool call 失败 → 模型能识别错误并尝试修复或报告 | 失败后行为合理性 |


 

# 3 Skill Pool 标准
不是我们自己造 500 个 skill，而是从真实生态采集最常用的skill。

### 3.1 采集来源
| 来源 | 预期数量 | 优先级 |
| --- | --- | --- |
| [https://github.com/anthropics/skills（官方）](https://github.com/anthropics/skills（官方）) | 17 | P0 |
| [https://github.com/anthropics/claude-code](https://github.com/anthropics/claude-code) 仓库中的插件 skills | ~15 | P0 |
| www.clawhub.ai/skills | 待爬取 | P1 |
| [https://skill.antgroup-inc.cn/](https://skill.antgroup-inc.cn/) 蚂蚁集团内部skill市场 | 待爬取 | P1 |
| GitHub 上 `.claude/skills/` 或 `SKILL.md` 的公开仓库 | 爬取目标 ≥200 | P1 |


### 3.2 质量过滤标准
从野外采集的 `SKILL.md` 须经过过滤：

| 标准 | 要求 | 原因 |
| --- | --- | --- |
| frontmatter 可解析 | YAML 合法，含 `name` 和 `description` | 不合规的模型学不了 |
| description 信息量 | 描述可判断用途 | 太短的 description 无法训练 L1 语义匹配 |
| 指令可操作 | markdown body 含明确步骤（非纯概念性内容） | 纯知识型 skill 无法训练 L3 指令遵循 |
| 可在沙箱执行 | 不依赖特定 SaaS API 密钥、不需特定硬件 | 沙箱中无法验证执行正确性 |


---

### 3.3 领域覆盖标准
采集后按领域标注，要求以下领域均有覆盖：

+ **开发类**：`deploy` / `test` / `lint` / `refactor` / `migrate` / `debug` / `code-review`
+ **文档类**：`docx` / `pdf` / `pptx` / `xlsx` / `markdown-gen` / `api-docs`
+ **设计类**：`frontend-design` / `theme` / `algorithmic-art` / `canvas`
+ **基建类**：`mcp-builder` / `plugin-structure` / `hook-development` / `ci-cd`
+ **分析类**：`data-analysis` / `log-analysis` / `security-scan` / `performance-profiling`
+ **通信类**：`internal-comms` / `brand-guidelines` / `changelog`

> 不要求每个子领域都有，但一级分类必须每类 **≥3 个 skill**。
>

# 4. 轨迹数据标准
### 4.1 轨迹构造方法
不是凭空合成对话，而是基于真实 skill 构造真实使用场景，对于每个 skill ：

1. 读取  `SKILL.md`，理解它能做什么
2. 构造 **3~5 个**真实的 user query（该 skill 应该被调用的场景）
3. 随机组合 **5~20 个** skill（含 `S`）作为 `<available_skills>`
4. 在沙箱中执行完整轨迹：`query → 路由 → 读取 → 执行 → 回复`
5. 有可验证执行正确性的奖励标准，

---

### 4.2 每条轨迹包含的要素
| 要素 | 必须包含 | 说明 |
| --- | --- | --- |
| 完整的 skill 列表 | 5~20 个 skill 的 `name` + `description` | 模型要从中选择 |
| 真实的 SKILL.md 内容 | 原封不动的真实 skill 文件 | 模型要学会读懂任意 skill |
| 真实的 project 上下文 | 与该 skill 领域匹配的项目文件 | 比如 deploy skill 需要有 `package.json` |
| 真实的 tool 执行结果 | 沙箱中 `bash`/`read`/`write` 的真实输出 | 不是模拟的 "Build succeeded" |
| 错误和恢复（部分轨迹） | tool call 失败 → 模型的后续处理 | 训练 L6 容错能力 |


---

### 4.3 质量标准
| 标准 | SFT 数据 | RL 数据 |
| --- | --- | --- |
| 路由正确 | 100%（不正确的不入库） | 允许错误（用于负奖励） |
| 步骤覆盖率 | ≥90%（SKILL.md 关键步骤都被执行） | 不限 |
| 最终回复质量 | 人工/模型评判 ≥4/5 | 不限 |
| 每个 skill 至少 N 条 | ≥3 条不同 query 的轨迹 |  |
| reward奖励准则 | 结果校验正确 | 有可执行的校验准则 |


### 4.4 公开轨迹数据集
与公开的轨迹数据不重合

| 数据集 | 平台 | 描述 |
| --- | --- | --- |
| **agent-trajectory-data** | HuggingFace | LLM Agent轨迹数据 |
| **WebShop** | 研究数据集 | 网页导航任务轨迹 |
| **ScienceWorld** | 研究数据集 | 科学推理任务轨迹 |
| **TrajAgent数据集** | GitHub | 轨迹建模任务数据 |


---

# 5. 沙箱执行环境
### 5.1 设计原则
> 沙箱不是模拟器，是**真实但受限**的执行环境。模型发出的 tool call 真实执行，返回真实结果。
>

---

### 5.2 功能需求
| 模块 | 需求 |
| --- | --- |
| Skill.md 文件等files | 从 Skill Pool 动态挂载 skill 文件到标准路径 |
| Project Fixtures | 按 skill 领域匹配预置的真实项目骨架（可 `npm test`、`python -m pytest` 等） |
| Tool Executor | 执行 `read`/`write`/`edit`/`bash`/`grep`/`glob`，bash 有网络隔离和超时 |
| 状态快照 | 每步记录 FS diff + stdout/stderr，支持重放验证 |
| 并发隔离 | 每条轨迹独立容器/namespace |


---

### 5.3 Project Fixture 标准
> **关键改进**：fixture 必须与 skill 领域匹配，不是通用的。
>

| Skill 领域 | Fixture 要求 |
| --- | --- |
| `deploy` / `test` / `lint` | Node 项目（`package.json`, `src/`, `tests/`），可 `npm test` |
| python 类 | Python 项目（`pyproject.toml`, `src/`, `tests/`），可 `pytest` |
| `docx` / `pdf` / `pptx` / `xlsx` | 空目录 + 模板文件 + Python runtime 有相关库 |
| `frontend-design` | 有 Vite/Next 骨架，可 `npm run dev` |
| infra 类 | 有 `Dockerfile`, `.github/workflows/` 等 |


每个 fixture 标注：**支持的 skill 领域标签列表**。

---

# 6. RL Reward 标准
### 6.1 核心思路
> Reward 不是检查格式，是检查模型**是否真的把事做对了**。
>

---

### 6.2 Reward 分项
$ R_{total} = w_1 \cdot R_{routing} + w_2 \cdot R_{faithful} + w_3 \cdot R_{outcome} $



#### R_routing：选对了 skill 吗
| 场景 | 判定 | 分数 |
| --- | --- | --- |
| 正样本，选中最佳 skill | query-skill 语义相关性最高的被选中 | `1.0` |
| 正样本，选中可接受 skill | 不是最佳但也能完成任务 | `0.5` |
| 正样本，选错或未选 | — | `0.0` |
| 负样本，正确不选 | — | `1.0` |
| 负样本，错误地选了 | — | `0.0` |


> **判定方式**：采集时预标注 `(query → best_skill, acceptable_skills)`。`best_skill` 可以有多个（true ties）。
>

---

#### R_faithful：忠实执行了 skill 指令吗
这是 skill 训练特有的 reward，通用 tool-use 训练没有：

$ R_{faithful} = \alpha \cdot \text{step\_coverage} + \beta \cdot \text{step\_order} + \gamma \cdot \text{no\_hallucinated\_ops} $

| 子项 | 计算 | 说明 |
| --- | --- | --- |
| `step_coverage` | 被执行的关键步骤 / SKILL.md 中标注的总关键步骤 | 例如 skill 说 5 步，执行了 4 步 → `0.8` |
| `step_order` | 被覆盖步骤的相对顺序是否与 skill 一致 | 有序 `1.0` / 部分乱序按 Kendall tau 计算 |
| `no_hallucinated_ops` | 是否执行了 skill 未要求的危险操作 | 无幻觉 `1.0` / 有则 `0.0` |


**关键步骤的标注方式：**

+ 每个 `SKILL.md` 在入池时由 LLM + 人工标注 `key_steps[]`
+ 格式：`[{tool: "bash", pattern: "test|pytest", required: true}, ...]`
+ `required: true` 的步骤必须执行，`required: false` 的步骤执行了加分

---

#### R_outcome：最终结果对吗
| 判定方式 | 适用场景 | 说明 |
| --- | --- | --- |
| 沙箱验证 | 有明确成功标准的 skill（test pass, file generated, build success） | 检查 exit code、文件是否生成、内容是否匹配 |
| LLM Judge | 开放式任务（explain, review, summarize） | 用大模型打分，0~1 |
| 规则匹配 | 回复中应包含特定信息（URL, 数据, 文件名） | 从 tool 输出中提取 ground truth，检查回复是否引用 |


> 三种方式按 skill 类型选择，不硬切一种。
>

---

### 6.3 Reward 工程约束
| 约束 | 说明 |
| --- | --- |
| 三项独立可观测 | 训练 dashboard 分别展示 `R_routing` / `R_faithful` / `R_outcome` |
| 权重可配置 | `w₁` `w₂` `w₃` 不硬编码，支持训练过程中动态调整 |
| 关键步骤标注随 skill 走 | 标注存在 skill pool 中，不在轨迹数据中重复 |
| `R_faithful` 的步骤匹配容错 | `bash("npm run test")` 和 `bash("npm test")` 都应匹配 `pattern: "test"` |
| 延迟预算 | 单条 reward ≤5s（沙箱验证是瓶颈） |


 

