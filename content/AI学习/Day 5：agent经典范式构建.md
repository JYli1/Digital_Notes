## 0x01 简介：智能体核心能力与挑战

现代智能体的核心能力在于将大语言模型（LLM）的推理能力与外部世界联通。它能够自主理解用户意图、拆解复杂任务，并通过调用代码解释器、搜索引擎、API等“工具”来获取信息、执行操作，最终达成目标。

然而，智能体并非万能，它同样面临挑战：
*   来自大模型本身的“幻觉”问题。
*   在复杂任务中可能陷入推理循环。
*   对工具的错误使用。

为了更好地组织智能体的“思考”与“行动”过程，业界涌现出多种经典架构范式，本章主要学习以下三种：
1.  **ReAct (Reasoning and Acting)：** “边想边做”，动态调整。
2.  **Plan-and-Solve：** “三思而后行”，先规划后执行。
3.  **Reflection：** “反思修正”，通过自我批判优化结果。

**为何要“手搓轮子”？**
尽管LangChain、LlamaIndex等成熟框架优势显著，但亲手实现这些范式能带来以下好处：
*   深入了解背后的设计机制和原理。
*   暴露出项目中的工程挑战（如模型输出解析、工具调用重试、防止死循环）。
*   培养系统设计能力，从框架“使用者”转变为应用“创造者”。

## 0x02 环境准备与基础工具定义

在开始构建智能体之前，需要搭建开发环境并定义基础组件，以避免重复劳动，专注于核心逻辑。

### 2.1 安装依赖库

本书实战主要使用 Python 语言（建议 Python 3.10+）。需要安装 `openai` 库用于与大语言模型交互，以及 `python-dotenv` 库用于安全地管理 API 密钥。

```bash
pip install openai python-dotenv
```

### 2.2 配置 API 密钥

为了通用性，将模型服务的相关信息（模型ID、API密钥、服务地址）配置在环境变量中。在项目根目录下创建 `.env` 文件，并添加以下内容：

```dotenv
# .env file
LLM_API_KEY="YOUR-API-KEY"
LLM_MODEL_ID="YOUR-MODEL"
LLM_BASE_URL="YOUR-URL"
```

### 2.3 封装基础 LLM 调用函数

为了代码结构清晰、易于复用，定义一个专属的LLM客户端类，封装所有与模型服务交互的细节。

```python
import os
from openai import OpenAI
from dotenv import load_dotenv
from typing import List, Dict

load_dotenv()

class HelloAgentsLLM:
    """
    定制LLM客户端，用于调用任何兼容OpenAI接口的服务，并默认使用流式响应。
    """
    def __init__(self, model: str = None, apiKey: str = None, baseUrl: str = None, timeout: int = None):
        self.model = model or os.getenv("LLM_MODEL_ID")
        apiKey = apiKey or os.getenv("LLM_API_KEY")
        baseUrl = baseUrl or os.getenv("LLM_BASE_URL")
        timeout = timeout or int(os.getenv("LLM_TIMEOUT", 60))
        
        if not all([self.model, apiKey, baseUrl]):
            raise ValueError("模型ID、API密钥和服务地址必须被提供或在.env文件中定义。")

        self.client = OpenAI(api_key=apiKey, base_url=baseUrl, timeout=timeout)

    def think(self, messages: List[Dict[str, str]], temperature: float = 0) -> str:
        """
        调用大语言模型进行思考，并返回其响应。
        """
        print(f"🧠 正在调用 {self.model} 模型...")
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=temperature,
                stream=True,
            )
            
            print("✅ 大语言模型响应成功:")
            collected_content = []
            for chunk in response:
                content = chunk.choices[0].delta.content or ""
                print(content, end="", flush=True)
                collected_content.append(content)
            print()
            return "".join(collected_content)

        except Exception as e:
            print(f"❌ 调用LLM API时发生错误: {e}")
            return None

if __name__ == '__main__':
    try:
        llmClient = HelloAgentsLLM()
        exampleMessages = [
            {"role": "system", "content": "You are a helpful assistant that writes Python code."},
            {"role": "user", "content": "写一个快速排序算法"}
        ]
        print("---"调用LLM"---")
        responseText = llmClient.think(exampleMessages)
        if responseText:
            print("\n\n---"完整模型响应"---")
            print(responseText)
    except ValueError as e:
        print(e)
```

## 0x03 ReAct (Reasoning and Acting) 范式

ReAct 范式（由 Shunyu Yao 于 2022 年提出）模仿人类解决问题的方式，将**推理 (Reasoning)** 与**行动 (Acting)** 显式地结合起来，形成一个“思考-行动-观察”的循环。

### 3.1 ReAct 的工作流程

ReAct 巧妙地认识到思考与行动是相辅相成的。通过特殊的提示工程，LLM 每一步的输出都遵循固定轨迹：
*   **Thought (思考)：** 智能体的“内心独白”，分析当前情况、拆解任务、规划下一步或反思结果。
*   **Action (行动)：** 智能体决定采取的具体动作，通常是调用一个外部工具，例如 `Search['华为最新款手机']`。
*   **Observation (观察)：** 执行 Action 后从外部工具返回的结果，例如搜索结果摘要或 API 返回值。

智能体将不断重复 **Thought -> Action -> Observation** 循环，将新的观察结果追加到历史记录中，形成不断增长的上下文，直到其认为已找到最终答案并输出结果。这种机制特别适用于：
*   需要外部知识的任务（如查询实时信息）。
*   需要精确计算的任务（将数学问题交给计算器工具）。
*   需要与 API 交互的任务。

### 3.2 工具的定义与实现

如果说大语言模型是智能体的大脑，那么工具 (Tools) 就是其与外部世界交互的“手和脚”。一个良好定义的工具应包含三个核心要素：
1.  **名称 (Name)：** 简洁、唯一的标识符，供智能体在 `Action` 中调用（如 `Search`）。
2.  **描述 (Description)：** 清晰的自然语言描述，说明工具用途。**这是最关键部分**，LLM 依赖它判断何时使用工具。
3.  **执行逻辑 (Execution Logic)：** 真正执行任务的函数或方法。

**示例：实现搜索工具的核心逻辑 (基于 SerpApi)**

```python
import os
from serpapi import SerpApiClient

def search(query: str) -> str:
    """
    一个基于SerpApi的实战网页搜索引擎工具。
    它会智能地解析搜索结果，优先返回直接答案或知识图谱信息。
    """
    print(f"🔍 正在执行 [SerpApi] 网页搜索: {query}")
    try:
        api_key = os.getenv("SERPAPI_API_KEY")
        if not api_key:
            return "错误:SERPAPI_API_KEY 未在 .env 文件中配置。"

        params = {
            "engine": "google",
            "q": query,
            "api_key": api_key,
            "gl": "cn",
            "hl": "zh-cn",
        }
        
        client = SerpApiClient(params)
        results = client.get_dict()
        
        # 智能解析:优先寻找最直接的答案
        if "answer_box_list" in results:
            return "\n".join(results["answer_box_list"])
        if "answer_box" in results and "answer" in results["answer_box"]:
            return results["answer_box"]["answer"]
        if "knowledge_graph" in results and "description" in results["knowledge_graph"]:
            return results["knowledge_graph"]["description"]
        if "organic_results" in results and results["organic_results"]:
            snippets = [
                f"[{i+1}] {res.get('title', '')}\n{res.get('snippet', '')}"
                for i, res in enumerate(results["organic_results"][:3])
            ]
            return "\n\n".join(snippets)
        
        return f"对不起，没有找到关于 '{query}' 的信息。"

    except Exception as e:
        return f"搜索时发生错误: {e}"
```

**构建通用的工具执行器**

当智能体需要使用多种工具时，需要一个统一的管理器来注册和调度。`ToolExecutor` 类负责此功能。

```python
from typing import Dict, Any

class ToolExecutor:
    """
    一个工具执行器，负责管理和执行工具。
    """
    def __init__(self):
        self.tools: Dict[str, Dict[str, Any]] = {}

    def registerTool(self, name: str, description: str, func: callable):
        """
        向工具箱中注册一个新工具。
        """
        if name in self.tools:
            print(f"警告:工具 '{name}' 已存在，将被覆盖。")
        self.tools[name] = {"description": description, "func": func}
        print(f"工具 '{name}' 已注册。")

    def getTool(self, name: str) -> callable:
        """
        根据名称获取一个工具的执行函数。
        """
        return self.tools.get(name, {}).get("func")

    def getAvailableTools(self) -> str:
        """
        获取所有可用工具的格式化描述字符串。
        """
        return "\n".join([
            f"- {name}: {info['description']}" 
            for name, info in self.tools.items()
        ])
```

### 3.3 ReAct 智能体的编码实现

将LLM客户端和工具执行器组装起来，构建一个完整的 ReAct 智能体。

**系统提示词设计**

提示词是 ReAct 机制的基石，它为 LLM 提供了行动的操作指令。

```python
# ReAct 提示词模板
REACT_PROMPT_TEMPLATE = """
请注意，你是一个有能力调用外部工具的智能助手。

可用工具如下:
{tools}

请严格按照以下格式进行回应:

Thought: 你的思考过程，用于分析问题、拆解任务和规划下一步行动。
Action: 你决定采取的行动，必须是以下格式之一:
- `{{tool_name}}[{{tool_input}}]`:调用一个可用工具。
- `Finish[最终答案]`:当你认为已经获得最终答案时。
- 当你收集到足够的信息，能够回答用户的最终问题时，你必须在Action:字段后使用 Finish[最终答案] 来输出最终答案。

现在，请开始解决以下问题:
Question: {question}
History: {history}
"""
```
此模板定义了智能体与 LLM 交互规范：角色定义、工具清单、格式规约 (Thought/Action) 和动态上下文。

**核心循环的实现**

`ReActAgent` 的核心是一个循环，不断地“格式化提示词 -> 调用LLM -> 执行动作 -> 整合结果”。

```python
# 假设 llm_client.py 和 tool_executor.py 已定义
# from llm_client import HelloAgentsLLM
# from tool_executor import ToolExecutor
import re # 需要导入re模块

class ReActAgent:
    def __init__(self, llm_client: HelloAgentsLLM, tool_executor: ToolExecutor, max_steps: int = 5):
        self.llm_client = llm_client
        self.tool_executor = tool_executor
        self.max_steps = max_steps
        self.history = []

    def run(self, question: str):
        self.history = []
        current_step = 0

        while current_step < self.max_steps:
            current_step += 1
            print(f"---" 第 {current_step} 步 ---")

            tools_desc = self.tool_executor.getAvailableTools()
            history_str = "\n".join(self.history)
            prompt = REACT_PROMPT_TEMPLATE.format(
                tools=tools_desc,
                question=question,
                history=history_str
            )

            messages = [{"role": "user", "content": prompt}]
            response_text = self.llm_client.think(messages=messages)
            
            if not response_text:
                print("错误:LLM未能返回有效响应。")
                break

            # ... (后续的解析、执行、整合步骤，需在实际代码中补充完整)
            thought, action = self._parse_output(response_text)
            
            if thought:
                print(f"思考: {thought}")

            if not action:
                print("警告:未能解析出有效的Action，流程终止。")
                break

            if action.startswith("Finish"):
                final_answer = re.match(r"Finish[(.*)]", action).group(1)
                print(f"🎉 最终答案: {final_answer}")
                return final_answer
            
            tool_name, tool_input = self._parse_action(action)
            if not tool_name or not tool_input:
                print(f"无效的Action格式: {action}")
                continue

            print(f"🎬 行动: {tool_name}[{tool_input}]")
            
            tool_function = self.tool_executor.getTool(tool_name)
            if not tool_function:
                observation = f"错误:未找到名为 '{tool_name}' 的工具。"
            else:
                observation = tool_function(tool_input)

            print(f"👀 观察: {observation}")
            self.history.append(f"Action: {action}")
            self.history.append(f"Observation: {observation}")

        print("已达到最大步数，流程终止。")
        return None

    def _parse_output(self, text: str):
        """解析LLM的输出，提取Thought和Action。"""
        thought_match = re.search(r"Thought: (.*)", text, re.DOTALL) # Added re.DOTALL for multiline thoughts
        action_match = re.search(r"Action: (.*)", text, re.DOTALL) # Added re.DOTALL for multiline actions
        thought = thought_match.group(1).strip() if thought_match else None
        action = action_match.group(1).strip() if action_match else None
        return thought, action

    def _parse_action(self, action_text: str):
        """解析Action字符串，提取工具名称和输入。"""
        match = re.match(r"(\w+)[(.*)]", action_text)
        if match:
            return match.group(1), match.group(2)
        return None, None
```

### 3.4 ReAct 的特点、局限性与调试技巧

**特点：**
*   **高可解释性：** 通过 `Thought` 链清晰展示智能体决策过程。
*   **动态规划与纠错能力：** “走一步，看一步”，根据 `Observation` 动态调整。
*   **工具协同能力：** LLM 负责推理规划，工具负责执行具体任务。

**局限性：**
*   **对LLM自身能力的强依赖：** LLM 推理、指令遵循、格式化输出能力不足易导致流程中断。
*   **执行效率问题：** 任务完成需多次调用 LLM，串行“思考-行动”循环可能导致高耗时和费用。
*   **提示词的脆弱性：** 对提示词模板的微小变动敏感，并非所有模型能稳定遵循格式。
*   **可能陷入局部最优：** 步进式决策可能缺乏全局规划，陷入“原地打转”。

**调试技巧：**
*   **检查完整的提示词：** 打印每次 LLM 调用前的完整提示词。
*   **分析原始输出：** 当解析失败时，打印 LLM 原始文本以判断问题来源。
*   **验证工具的输入与输出：** 确保 tool_input 格式正确且 observation 可理解。
*   **调整提示词中的示例 (Few-shot Prompting)：** 通过成功案例引导模型。
*   **尝试不同的模型或参数：** 更换能力更强的模型或调整 `temperature` 参数 (通常设为0)。

## 0x04 Plan-and-Solve 范式

Plan-and-Solve 范式将任务处理明确分为两个阶段：**先规划 (Plan)，后执行 (Solve)**。这与 ReAct 的反应式风格不同，更像是在动工前先绘制完整的蓝图。

### 4.1 Plan-and-Solve 的工作原理

核心动机是解决思维链在处理多步骤、复杂问题时容易“偏离轨道”的问题。
*   **规划阶段 (Planning Phase)：** 智能体接收用户问题后，首先将其分解并制定出一个清晰、分步骤的行动计划（LLM 调用产物）。
*   **执行阶段 (Solving Phase)：** 获得完整计划后，智能体严格按照计划逐一执行，每一步可能是独立的 LLM 调用或结果加工处理，直到完成所有步骤。

这种“先谋后动”策略使得智能体在处理需要长远规划的复杂任务时，能保持更高的目标一致性。尤其适用于结构性强、可清晰分解的复杂任务。

### 4.2 规划阶段

规划阶段的目标是让大语言模型接收原始问题，并输出一个清晰、分步骤的行动计划。

**规划阶段的提示词设计**

```python
PLANNER_PROMPT_TEMPLATE = """
你是一个顶级的AI规划专家。你的任务是将用户提出的复杂问题分解成一个由多个简单步骤组成的行动计划。
请确保计划中的每个步骤都是一个独立的、可执行的子任务，并且严格按照逻辑顺序排列。
你的输出必须是一个Python列表，其中每个元素都是一个描述子任务的字符串。

问题: {question}

请严格按照以下格式输出你的计划,```python与```作为前后缀是必要的:
```python
["步骤1", "步骤2", "步骤3", ...] 
```
"""
```
此提示词通过角色设定、任务描述和严格的格式约束（Python 列表字符串）确保输出质量和稳定性。

**规划器 (`Planner` 类)**

将规划逻辑封装成 `Planner` 类。

```python
import ast # 需要导入ast模块

# 假定 llm_client.py 中的 HelloAgentsLLM 类已经定义好
# from llm_client import HelloAgentsLLM

class Planner:
    def __init__(self, llm_client: HelloAgentsLLM):
        self.llm_client = llm_client

    def plan(self, question: str) -> list[str]:
        prompt = PLANNER_PROMPT_TEMPLATE.format(question=question)
        messages = [{"role": "user", "content": prompt}]
        
        print("---" 正在生成计划 ---")
        response_text = self.llm_client.think(messages=messages) or ""
        
        print(f"✅ 计划已生成:\n{response_text}")
        
        try:
            plan_str = response_text.split("```python")[1].split("```")[0].strip()
            plan = ast.literal_eval(plan_str)
            return plan if isinstance(plan, list) else []
        except (ValueError, SyntaxError, IndexError) as e:
            print(f"❌ 解析计划时出错: {e}")
            print(f"原始响应: {response_text}")
            return []
        except Exception as e:
            print(f"❌ 解析计划时发生未知错误: {e}")
            return []
```

### 4.3 执行器与状态管理

执行器不仅负责调用大语言模型解决子问题，还承担着状态管理，记录每一步执行结果并作为上下文提供给后续步骤。

**执行器的提示词**

```python
EXECUTOR_PROMPT_TEMPLATE = """
你是一位顶级的AI执行专家。你的任务是严格按照给定的计划，一步步地解决问题。
你将收到原始问题、完整的计划、以及到目前为止已经完成的步骤和结果。
请你专注于解决“当前步骤”，并仅输出该步骤的最终答案，不要输出任何额外的解释或对话。

# 原始问题:
{question}

# 完整计划:
{plan}

# 历史步骤与结果:
{history}

# 当前步骤:
{current_step}

请仅输出针对“当前步骤”的回答:
"""
```

**执行器 (`Executor` 类) 与 `PlanAndSolveAgent`**

```python
# 假定 llm_client.py 已定义
# from llm_client import HelloAgentsLLM

class Executor:
    def __init__(self, llm_client: HelloAgentsLLM):
        self.llm_client = llm_client

    def execute(self, question: str, plan: list[str]) -> str:
        history = ""
        
        print("\n---" 正在执行计划 ---")
        
        for i, step in enumerate(plan):
            print(f"\n-> 正在执行步骤 {i+1}/{len(plan)}: {step}")
            
            prompt = EXECUTOR_PROMPT_TEMPLATE.format(
                question=question,
                plan=plan,
                history=history if history else "无",
                current_step=step
            )
            
            messages = [{"role": "user", "content": prompt}]
            
            response_text = self.llm_client.think(messages=messages) or ""
            
            history += f"步骤 {i+1}: {step}\n结果: {response_text}\n\n"
            
            print(f"✅ 步骤 {i+1} 已完成，结果: {response_text}")

        final_answer = response_text
        return final_answer

class PlanAndSolveAgent:
    def __init__(self, llm_client: HelloAgentsLLM):
        self.llm_client = llm_client
        self.planner = Planner(self.llm_client)
        self.executor = Executor(self.llm_client)

    def run(self, question: str):
        print(f"\n---" 开始处理问题 ---
问题: {question}")
        
        plan = self.planner.plan(question)
        
        if not plan:
            print("\n---" 任务终止 --- \n无法生成有效的行动计划。")
            return

        final_answer = self.executor.execute(question, plan)
        
        print(f"\n---" 任务完成 ---
最终答案: {final_answer}")
```

## 0x05 Reflection (自我反思与迭代) 范式

Reflection 机制为智能体引入一种事后 (post-hoc) 的自我校正循环，使其能够像人类一样，审视自己的工作，发现不足，并进行迭代优化。

### 5.1 Reflection 机制的核心思想

灵感来源于人类学习过程：**执行 -> 反思 -> 优化** 三步循环。
*   **执行 (Execution)：** 智能体尝试完成任务，生成初步解决方案（“初稿”）。
*   **反思 (Reflection)：** 独立 LLM 扮演“评审员”，审视“初稿”，评估事实性错误、逻辑漏洞、效率问题、遗漏信息，生成结构化反馈。
*   **优化 (Refinement)：** 智能体将“初稿”和“反馈”作为新上下文，再次调用 LLM 修正，生成“修订稿”。

这种机制的价值在于：提供内部纠错回路、将一次性任务执行转变为持续优化、构建临时“短期记忆”。

### 5.2 记忆模块设计

Reflection 的核心在于迭代，而迭代前提是能够记住之前的尝试和获得的反馈。因此，一个“短期记忆”模块是必需品。

**`Memory` 类**

```python
from typing import List, Dict, Any, Optional

class Memory:
    """
    一个简单的短期记忆模块，用于存储智能体的行动与反思轨迹。
    """
    def __init__(self):
        self.records: List[Dict[str, Any]] = []

    def add_record(self, record_type: str, content: str):
        """
        向记忆中添加一条新记录。
        """
        record = {"type": record_type, "content": content}
        self.records.append(record)
        print(f"📝 记忆已更新，新增一条 '{record_type}' 记录。")

    def get_trajectory(self) -> str:
        """
        将所有记忆记录格式化为一个连贯的字符串文本，用于构建提示词。
        """
        trajectory_parts = []
        for record in self.records:
            if record['type'] == 'execution':
                trajectory_parts.append(f"---" 上一轮尝试 (代码)---
{record['content']}")
            elif record['type'] == 'reflection':
                trajectory_parts.append(f"---" 评审员反馈 ---
{record['content']}")
        
        return "\n\n".join(trajectory_parts)

    def get_last_execution(self) -> Optional[str]:
        """
        获取最近一次的执行结果 (例如，最新生成的代码)。
        """
        for record in reversed(self.records):
            if record['type'] == 'execution':
                return record['content']
        return None
```

### 5.3 Reflection 智能体的编码实现

将提示词逻辑和 `Memory` 模块整合到 `ReflectionAgent` 类中。

**提示词设计**

Reflection 机制需要多个不同角色的提示词协同工作。

**初始执行提示词 (Execution Prompt)**

```python
INITIAL_PROMPT_TEMPLATE = """
你是一位资深的Python程序员。请根据以下要求，编写一个Python函数。
你的代码必须包含完整的函数签名、文档字符串，并遵循PEP 8编码规范。

要求: {task}

请直接输出代码，不要包含任何额外的解释。
"""
```

**反思提示词 (Reflection Prompt)**

```python
REFLECT_PROMPT_TEMPLATE = """
你是一位极其严格的代码评审专家和资深算法工程师，对代码的性能有极致的要求。
你的任务是审查以下Python代码，并专注于找出其在**算法效率**上的主要瓶颈。

# 原始任务:
{task}

# 待审查的代码:
```python
{code}
```

请分析该代码的时间复杂度，并思考是否存在一种**算法上更优**的解决方案来显著提升性能。
如果存在，请清晰地指出当前算法的不足，并提出具体的、可行的改进算法建议（例如，使用筛法替代试除法）。
如果代码在算法层面已经达到最优，才能回答“无需改进”。

请直接输出你的反馈，不要包含任何额外的解释。
"""
```

**优化提示词 (Refinement Prompt)**

```python
REFINE_PROMPT_TEMPLATE = """
你是一位资深的Python程序员。你正在根据一位代码评审专家的反馈来优化你的代码。

# 原始任务:
{task}

# 你上一轮尝试的代码:
{last_code_attempt}
评审员的反馈：
{feedback}

请根据评审员的反馈，生成一个优化后的新版本代码。
你的代码必须包含完整的函数签名、文档字符串，并遵循PEP 8编码规范。
请直接输出优化后的代码，不要包含任何额外的解释。
"""
```

**智能体封装 (`ReflectionAgent` 类)**

```python
# 假设 llm_client.py 和 memory.py 已定义
# from llm_client import HelloAgentsLLM
# from memory import Memory

class ReflectionAgent:
    def __init__(self, llm_client: HelloAgentsLLM, max_iterations=3):
        self.llm_client = llm_client
        self.memory = Memory()
        self.max_iterations = max_iterations

    def run(self, task: str):
        print(f"\n---" 开始处理任务 ---
任务: {task}")

        # --- 1. 初始执行 ---
        print("\n---" 正在进行初始尝试 ---")
        initial_prompt = INITIAL_PROMPT_TEMPLATE.format(task=task)
        initial_code = self._get_llm_response(initial_prompt)
        self.memory.add_record("execution", initial_code)

        # --- 2. 迭代循环:反思与优化 ---
        for i in range(self.max_iterations):
            print(f"\n---" 第 {i+1}/{self.max_iterations} 轮迭代 ---")

            # a. 反思
            print("\n-> 正在进行反思...")
            last_code = self.memory.get_last_execution()
            reflect_prompt = REFLECT_PROMPT_TEMPLATE.format(task=task, code=last_code)
            feedback = self._get_llm_response(reflect_prompt)
            self.memory.add_record("reflection", feedback)

            # b. 检查是否需要停止
            if "无需改进" in feedback:
                print("\n✅ 反思认为代码已无需改进，任务完成。")
                break

            # c. 优化
            print("\n-> 正在进行优化...")
            refine_prompt = REFINE_PROMPT_TEMPLATE.format(
                task=task,
                last_code_attempt=last_code,
                feedback=feedback
            )
            refined_code = self._get_llm_response(refine_prompt)
            self.memory.add_record("execution", refined_code)
        
        final_code = self.memory.get_last_execution()
        print(f"\n---" 任务完成 ---
最终生成的代码:\n```python\n{final_code}\n```")
        return final_code

    def _get_llm_response(self, prompt: str) -> str:
        """辅助方法，用于调用LLM并获取完整的流式响应。"""
        messages = [{"role": "user", "content": prompt}]
        response_text = self.llm_client.think(messages=messages) or ""
        return response_text
```

## 0x06 本章小结

本章从零开始编码实现了三种业界经典的智能体构建范式：ReAct、Plan-and-Solve 与 Reflection。

**核心知识点回顾：**
*   **ReAct：** 构建了一个能与外部世界交互的智能体，通过“思考-行动-观察”动态循环，利用搜索引擎回答自身知识库无法覆盖的实时性问题。其核心优势在于**环境适应性**和**动态纠错能力**，适合探索性、需要外部工具输入的任务。
*   **Plan-and-Solve：** 实现了一个先规划后执行的智能体，解决了需要多步推理的数学应用题。其核心优势在于**结构性**和**稳定性**，特别适合处理逻辑路径确定、内部推理密集的任务。
*   **Reflection (自我反思与迭代)：** 构建了一个具备自我优化能力的智能体，通过“执行-反思-优化”的迭代循环，将初始方案优化为高性能版本。其核心价值在于能显著**提升解决方案的质量**，适用于对结果的准确性和可靠性有极高要求的场景。

**不同 Agent Loop 的选择策略：**
*   **ReAct：** 适用于需要外部工具、环境不确定、需要动态调整的任务。
*   **Plan-and-Solve：** 适用于任务逻辑清晰、可分解、需要精确规划的任务。
*   **Reflection：** 适用于对结果质量要求极高、可以接受多轮迭代的任务。

