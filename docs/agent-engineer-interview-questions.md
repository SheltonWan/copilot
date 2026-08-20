# 智能体工程师面试题库

---

## 一、基础概念（考察理论功底）

### Q1.1 什么是 AI 智能体？它与普通 Chatbot 的本质区别是什么？

**参考答案要点**：
- 智能体 = LLM + 工具 + 记忆 + 规划 + 编排
- 普通 Chatbot 是单轮/多轮对话的被动响应；智能体能主动调用工具、制定计划、在环境中执行动作并观察反馈
- 核心区别在于「自主性」：智能体拥有 Agentic Loop（感知→推理→行动→观察），而 Chatbot 只有推理→输出

### Q1.2 请描述智能体工作循环（Agentic Loop）的四个阶段及各阶段作用。

**参考答案要点**：
1. **思考（Think）**：分析当前状态，拆解任务，决定下一步行动
2. **行动（Act）**：调用工具或生成代码，与外部环境交互
3. **观察（Observe）**：接收工具返回结果，感知环境变化
4. **更新状态（Update）**：将观察结果纳入上下文，更新记忆
5. 循环终止条件：目标达成 / 超过最大步数 / 异常 / 用户中断

### Q1.3 参照 SAE 自动驾驶分级，AI 智能体通常分为哪六个自主性等级？

**参考答案要点**：
- L0 无自主（普通 Chatbot）
- L1 辅助建议（Copilot 代码补全）
- L2 半自动（AI 代码审查 + 一键修复）
- L3 条件自主（CI/CD 修复智能体）
- L4 高度自主（自主研究智能体）
- L5 完全自主（AGI，理论阶段）

### Q1.4 请画出智能体的层次模型，并说明各层职责。

**参考答案要点**：
- **元认知层**：自我反思、目标修正、学习策略调整
- **规划层**：任务分解、路径选择、优先级排序
- **执行层**：工具调用、代码生成、动作执行
- **感知层**：环境感知、输入理解、上下文构建

---

## 二、工具使用（考察核心机制）

### Q2.1 请设计一个搜索文档的工具定义（JSON Schema），并说明设计原则。

**参考答案要点**：
```json
{
  "name": "search_documents",
  "description": "在知识库中搜索相关文档片段",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {"type": "string", "description": "搜索查询词"},
      "top_k": {"type": "integer", "default": 5}
    },
    "required": ["query"]
  }
}
```
- 描述要精确（模型据此决定调用时机）
- 参数尽量简洁，提供合理默认值
- 名称语义化（动词+名词模式）

### Q2.2 并行工具调用（Parallel Tool Calls）的核心价值是什么？举例说明。

**参考答案要点**：
- 将 N 次串行调用的延迟压缩为 1 次
- 示例：同时查两个城市天气 + 当前时间，模型在一次推理中发出 3 个 tool_call，等待全部返回后一次回答
- 关键是判断哪些工具调用之间无依赖关系

### Q2.3 工具权限如何分级？对于高风险操作，你会如何设计安全机制？

**参考答案要点**：
| 级别 | 说明 |
|------|------|
| 只读（Read） | 查询类，无副作用 |
| 写入（Write） | 可撤销 |
| 执行（Execute） | 代码/命令 |
| 外部通信（External） | 第三方 API |
| 不可逆（Irreversible） | 删除/发送 |

高风险操作应引入 **Human-in-the-Loop 确认机制** + 操作审计日志。

### Q2.4 如果模型频繁调用错误的工具，你会从哪些角度排查？

**参考答案要点**：
1. 工具描述是否足够精确、语义无歧义
2. 工具名称是否与其他工具有重叠
3. 系统提示中工具使用规则是否清晰
4. 是否存在"工具过多导致选择困难"（考虑分组或按需注入）
5. 是否需要增加 Few-shot 示例引导正确调用

---

## 三、记忆与上下文管理（考察架构能力）

### Q3.1 请解释智能体记忆系统的四个层次及其对应的存储方案。

**参考答案要点**：
| 记忆层次 | 内容 | 存储方案 |
|----------|------|----------|
| 工作记忆 | 当前对话上下文 | 进程内存 |
| 短期记忆 | 会话级历史 | Redis（TTL 自动过期） |
| 长期记忆 | 用户偏好、领域知识 | PostgreSQL + 向量库 |
| 语义记忆 | 向量化知识 | Pinecone/Milvus + RAG |

### Q3.2 当对话历史超出 Token 限制时，你有哪几种上下文压缩策略？

**参考答案要点**：
1. **摘要压缩**：调用轻量模型对历史对话摘要，保留要点
2. **滑动窗口**：保留最近 N 轮完整对话
3. **重要性过滤**：标记关键信息（用户偏好、核心决策）永久保留
4. **结构化存档**：将历史结构化后存入外部 DB，按需检索

### Q3.3 请对比基础 RAG 和 Agentic RAG 的区别。

**参考答案要点**：
- **基础 RAG**：查询 → 单次检索 → 生成（线性流程）
- **Agentic RAG**：查询 → 判断是否需要检索 → 生成搜索词 → 检索 → 评估质量 → 质量不够则换策略重试 → 生成
- 核心差异：Agentic RAG 引入智能体规划能力，自适应检索深度、多源融合、迭代精化

### Q3.4 请列举至少四种高级 RAG 技术并简述原理。

**参考答案要点**：
| 技术 | 原理 |
|------|------|
| **HyDE** | 先生成假设答案，以其 Embedding 检索 |
| **GraphRAG** | 知识图谱存储，图遍历检索 |
| **Self-RAG** | 动态判断是否需要检索及检索质量 |
| **多路召回** | BM25（稀疏）+ 稠密检索并行 |

---

## 四、规划与推理（考察算法理解）

### Q4.1 请对比 ReAct 和 Plan-and-Execute 两种规划模式的优劣及适用场景。

**参考答案要点**：
| 维度 | ReAct | Plan-and-Execute |
|------|-------|-----------------|
| 流程 | 逐步推理+行动交替 | 先定计划再执行 |
| 灵活性 | 高，动态调整 | 低，按计划执行 |
| 效率 | 可能冗余调用 | 工具调用更高效 |
| 适用 | 探索性任务 | 结构化任务 |

### Q4.2 ReAct 中 `max_iterations` 设置为多少合适？为什么？

**参考答案要点**：
- 建议 10-20 步
- 过少：复杂任务无法完成
- 过多：陷入无限循环风险
- 应配合 `early_stopping` 检测到 Final Answer 时提前终止
- 还应对单步耗时设置超时，防止单次工具调用长时间挂起

### Q4.3 LATS 是如何将 MCTS 应用于语言智能体的？相比 ReAct 有什么优势？

**参考答案要点**：
- 每个状态节点代表一段对话状态
- 动作展开后生成多个候选下一步，用 LLM 评估每个子状态分数
- 回溯更新父节点 → 选择高分路径继续展开
- 优势：能找到全局最优解，避免 ReAct 贪心式的局部最优
- 代价：需要多次 LLM 评估，计算成本高

### Q4.4 如何处理任务分解中的依赖关系？请画一个 DAG 示例。

**参考答案要点**：
- 识别串行依赖（必须等待前置完成）vs 可并行任务
- 使用拓扑排序确定执行顺序
- 示例：T1（需求）→ T2（架构）→ [T3（前端）, T4（后端）] 可并行 → T5（集成测试）→ T6（部署）

---

## 五、多智能体系统（考察系统设计）

### Q5.1 请对比三种多智能体协作模式（编排者、对话式、层级式）的优劣及选型建议。

**参考答案要点**：
| 模式 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| 编排者 | 中心可控、易调试 | 单点瓶颈 | 明确分工的任务 |
| 对话式 | 灵活、去中心化 | 共识困难 | 头脑风暴、创意 |
| 层级式 | 可扩展、职责清晰 | 通信开销大 | 大型复杂项目 |

### Q5.2 MCP 和 A2A 协议分别解决什么问题？它们如何互补？

**参考答案要点**：
- **MCP（Model Context Protocol）**：智能体 ↔ 工具/资源，标准化单智能体调用外部能力的方式
- **A2A（Agent-to-Agent）**：智能体 ↔ 智能体，标准化多智能体间的任务委派与通信
- 互补关系：MCP 管"智能体怎么用工具"，A2A 管"智能体之间怎么协作"

### Q5.3 多智能体系统中如何解决冲突（两个智能体意见不一致）？

**参考答案要点**：
1. **投票机制**：多智能体表决，少数服从多数
2. **置信度加权**：根据各智能体在该领域的置信度加权
3. **仲裁智能体**：引入独立裁判智能体
4. **人工升级**：关键决策交给人裁决
5. **结构化辩论**：各智能体呈交论据，综合分析

---

## 六、提示工程（考察实战经验）

### Q6.1 系统提示（System Prompt）应包含哪些核心要素？

**参考答案要点**：
1. 角色定义 + 领域专注
2. 能力边界（能/不能做什么）
3. 行为规范（操作约束）
4. 工具使用规则（调用时机、约束、错误处理）
5. 输出格式要求

### Q6.2 什么是 Metaprompting（元提示）？请设计一个示例。

**参考答案要点**：
- 让 LLM 参与生成或优化自身的提示
- 示例：给模型一段任务描述，让它自动生成角色、能力边界、输出格式等 System Prompt
- 价值：减少人工调优成本，实现提示的自适应

### Q6.3 提示注入的防御策略有哪些？

**参考答案要点**：
1. **输入净化**：过滤 `ignore previous instructions` 等模式
2. **指令隔离**：使用 XML 标签明确分隔系统指令和用户输入
3. **特权分离**：系统指令始终最高优先级，不可被覆盖
4. **输出审核**：检查输出是否包含敏感信息或违反规则
5. **最小权限**：工具只授予必要权限

### Q6.4 Few-shot 示例数量是否越多越好？为什么？

**参考答案要点**：
- 不是。3-5 个高质量、多样化的示例通常最佳
- 过多：消耗 Token、可能导致过拟合特定模式、模型忽略其他指令
- 关键：示例应覆盖边缘情况（正常路径 + 异常处理）

---

## 七、评估与测试（考察质量意识）

### Q7.1 如何评估一个智能体系统的质量？请列举关键维度和指标。

**参考答案要点**：
| 维度 | 指标 | 方法 |
|------|------|------|
| 任务完成率 | 成功/失败比 | E2E 测试用例 |
| 工具调用准确性 | 正确调用率 | 调用日志分析 |
| 推理质量 | 推理正确性 | LLM-as-Judge |
| 响应效率 | 延迟、Token 消耗 | 性能监控 |
| 鲁棒性 | 异常处理 | 对抗性测试 |
| 安全性 | 有害输出率 | 红队测试 |

### Q7.2 LLM-as-Judge 有哪些常见偏见？如何缓解？

**参考答案要点**：
1. **位置偏见**：Judge 倾向评高第一个回答 → 交换位置多轮评测
2. **长度偏见**：倾向给长回答高分 → 限定输出长度或归一化
3. **风格偏见**：偏好特定语气 → 多 Judge 模型取平均
4. **自我增强**：Judge 偏好自身风格的输出 → 使用不同家族的模型交叉评估
5. 定期用人工标注校准 Judge 准确性

### Q7.3 SWE-bench 和 GAIA 分别测试什么能力？

**参考答案要点**：
- **SWE-bench**：真实 GitHub Issue 修复，测试代码理解、定位、修改的全流程软件工程能力
- **GAIA**：多步工具使用的问答，测试推理+规划+工具调用+信息综合的通用助手能力

---

## 八、框架与平台（考察技术选型）

### Q8.1 LangChain 和 LangGraph 的核心区别是什么？

**参考答案要点**：
- **LangChain**：链式调用框架，适合线性工作流
- **LangGraph**：有向图状态机，支持循环和条件分支，适合复杂智能体工作流
- 选型：简单链式 → LangChain；需要循环/条件/状态管理 → LangGraph

### Q8.2 MCP Server 的三大原语是什么？请简述各用途。

**参考答案要点**：
1. **Resources**：向 LLM 暴露数据（文件、数据库记录、API 响应）
2. **Tools**：LLM 可调用的函数（读写操作、执行代码）
3. **Prompts**：可复用的提示模板

### Q8.3 你会在什么场景下选择 AutoGen vs CrewAI？

**参考答案要点**：
- **AutoGen**：研究场景、需要灵活的多智能体对话、学术/实验性项目
- **CrewAI**：生产级团队协作、需要明确角色定义和任务序列、企业应用
- 核心差异：AutoGen 更灵活但更复杂，CrewAI 更结构化更易上手

---

## 九、安全与合规（考察安全意识）

### Q9.1 OWASP LLM Top 10 中排名前三的威胁是什么？

**参考答案要点**：
1. **LLM01 提示注入**：恶意指令覆盖系统提示
2. **LLM02 不安全输出处理**：未验证的模型输出注入下游系统
3. **LLM03 训练数据投毒**：恶意数据影响模型行为

### Q9.2 什么是"过度代理权限"（Excessive Agency）？如何防范？

**参考答案要点**：
- 智能体被授予超出必要范围的权限
- 防范：最小权限原则（只给必须的工具权限）、操作分级审批（高风险操作需人工确认）、权限动态衰减（敏感操作超时自动降权）

### Q9.3 EU AI Act 对智能体系统有哪些合规要求？

**参考答案要点**：
- 高风险 AI 系统需提供详细技术文档
- 必须有人工监督机制
- 透明度要求（告知用户正在与 AI 交互）
- 准确率、鲁棒性、网络安全的持续监控
- 事件报告义务

---

## 十、生产部署（考察工程能力）

### Q10.1 智能体的可靠性设计包括哪些方面？

**参考答案要点**：
1. **幂等性**：每个工具调用携带 idempotency_key
2. **熔断**：LLM API 连续失败时快速失败而非持续重试
3. **降级**：主模型不可用时切换到备用模型
4. **超时控制**：单步执行设定硬超时（建议 60-120s）
5. **断点续传**：长时间任务支持暂停/恢复
6. **优雅降级**：非核心工具不可用时跳过继续执行

### Q10.2 请描述智能体的灰度发布流程。

**参考答案要点**：
1. **Shadow Mode**：新版本并行运行，不影响用户，只对比结果
2. **Canary Release**：1% → 5% → 20% → 50% → 100%，每阶段观察指标
3. **Feature Flag**：按用户组/地区 AB 测试不同提示或模型
4. 关键监控：成功率、延迟分布、用户反馈、异常率

### Q10.3 如何控制智能体系统的 LLM 调用成本？

**参考答案要点**：
| 策略 | 方法 |
|------|------|
| **模型路由** | 简单任务用小模型，复杂任务才用大模型 |
| **语义缓存** | 相似度 >0.95 的查询复用结果 |
| **批处理** | 非实时任务使用 Batch API |
| **Token 优化** | 精简系统提示，动态裁剪上下文 |
| **本地模型** | 非关键路径用 Ollama 等本地模型 |

---

## 十一、编程实战（考察代码能力）

### Q11.1 请实现一个带超时和重试的 LLM 调用封装函数。

**参考答案**：
```python
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
async def call_llm(prompt: str, timeout: int = 30) -> str:
    try:
        return await asyncio.wait_for(
            primary_llm.invoke(prompt),
            timeout=timeout
        )
    except asyncio.TimeoutError:
        raise Exception("LLM call timeout")
    except RateLimitError:
        return await fallback_llm.invoke(prompt)
```

### Q11.2 请实现一个简单的记忆管理类，支持存储和语义检索。

**参考答案**：
```python
class AgentMemory:
    def __init__(self, vector_db, max_working_size=20):
        self.working = []          # 当前工作记忆
        self.vector_db = vector_db # 长期语义存储
        self.max_working = max_working_size

    def add(self, event: dict):
        self.working.append(event)
        if len(self.working) > self.max_working:
            summary = self._summarize(self.working[:-5])
            self.vector_db.upsert(summary)
            self.working = self.working[-5:]

    def search(self, query: str, top_k=5):
        working_results = self._filter_working(query)
        long_term = self.vector_db.search(query, top_k=top_k)
        return working_results + long_term
```

### Q11.3 请设计一个 Human-in-the-Loop 审批门控工具类。

**参考答案**：
```python
class HumanApprovalTool:
    HIGH_RISK = {"delete", "send_email", "deploy", "payment"}

    async def execute(self, action: dict) -> dict:
        if action.get("type") in self.HIGH_RISK:
            approval = await self.request_approval(action)
            if not approval.approved:
                return {"status": "rejected", "reason": approval.reason}
        return await self._execute(action)

    async def request_approval(self, action: dict, timeout=3600):
        ticket = await self.create_approval_ticket(action)
        return await self.wait_for_decision(ticket, timeout)
```

### Q11.4 请用 LangGraph 实现一个简单的 ReAct 循环。

**参考答案**：
```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class State(TypedDict):
    messages: list
    iteration: int

def should_continue(state: State) -> str:
    if state["iteration"] >= 10:
        return "end"
    last_msg = state["messages"][-1]
    if "Final Answer" in last_msg:
        return "end"
    if "Action:" in last_msg:
        return "tools"
    return "end"

workflow = StateGraph(State)
workflow.add_node("agent", call_model)
workflow.add_node("tools", execute_tools)
workflow.add_conditional_edges("agent", should_continue, {
    "tools": "tools", "end": END
})
workflow.add_edge("tools", "agent")
workflow.set_entry_point("agent")
app = workflow.compile()
```

---

## 十二、开放讨论题（考察综合深度）

### Q12.1 你认为当前智能体系统最大的技术瓶颈是什么？为什么？

**开放讨论，以下为可参考方向**：
- 长上下文下的推理一致性衰减
- 工具调用的可靠性（解析错误、错误调用）
- 安全与自主性的矛盾
- 成本与延迟的权衡
- 评估体系的不成熟（缺乏标准化 Benchmark）

### Q12.2 推理模型（如 o1、DeepSeek-R1）的兴起对智能体架构设计有什么影响？

**参考方向**：
- 减少外部 ReAct 循环，更多推理在模型内部完成
- 工具调用决策更精准，但每步延迟增加
- 适合作为 Planner/思考者，搭配轻量执行模型
- 提示策略需调整（推理模型对指令的敏感度不同）

### Q12.3 如果让你从零设计一个企业级智能体平台，你会如何做技术选型？

**参考方向**：
- 编排层：LangGraph（有状态工作流）
- 记忆：Redis（会话） + PostgreSQL（持久化） + Milvus（向量）
- 监控：LangFuse + OpenTelemetry + Grafana
- 部署：Kubernetes + 异步任务队列
- 安全：API Gateway + WAF + 内容审核层
- 模型层：多模型路由（强大模型做推理，轻量模型做执行）

### Q12.4 智能体领域的 "幻觉" 问题如何缓解？

**参考方向**：
- **RAG 约束**：所有事实性回答必须基于检索到的源文档
- **结构化输出**：减少自由文本生成的不可控性
- **交叉验证**：多模型独立推理后交叉比对
- **不确定性表达**：明确标注置信度，不确定时主动说不确定
- **事实核查层**：输出后增加独立验证步骤
- **来源引用**：每句事实声明附上来源链接

---

## 评分参考

| 级别 | 要求 |
|------|------|
| **初级（L1-L2）** | 能回答一至四章基础题，了解 ReAct 和 RAG 核心概念 |
| **中级（L3-L4）** | 能回答五至八章设计题，有生产部署经验，能写 LangGraph 代码 |
| **高级（L5）** | 能回答全部题目，在开放讨论中有深度见解，有企业级平台设计经验 |
| **专家** | 上述全部 + 能独立设计智能体架构、能带队攻坚、有论文或开源贡献 |

---

> 本文档基于《智能体专家知识体系》编制，涵盖理论、实战、设计、部署全链路面试评估。
