> 关联：[[index|AI学习]]、[[Day 5：agent经典范式构建]]、[[test]]

# 1 准备LLM客户端
为了方便切换不同的大模型，我们需要一个通用客户端
## 1.1 配置文件.env
1. 准备一个`.env`文件
2. 获取相关大模型配置信息并写入
```bash 
# .env
LLM_API_KEY="ms-5e772247-8d88-47e4-87cb-ddfbb06089e5"
LLM_MODEL_ID="deepseek-ai/DeepSeek-V3.2"
LLM_BASE_URL="https://api-inference.modelscope.cn/v1"

```
## 1.2 客户端文件
```python
import os
from openai import OpenAI
from dotenv import load_dotenv
from typing import List, Dict

load_dotenv()

class LLMClient:
    """
    大模型客户端,用于调用LLM
    """

    def __init__(self, model: str = None ,apikey: str = None, baseUrl :str = None,timeout: int = None):
        
        # 构造函数，如果没有传入参数则使用默认
        self.model = model or os.getenv("LLM_MODEL_ID")
        apikey = apikey or os.getenv("LLM_API_KEY")
        baseUrl = baseUrl or os.getenv("LLM_BASE_URL")
        timeout = timeout or int(os.getenv("LLM_TIMEOUT",60))

        if not all([self.model,apikey,baseUrl]):
            raise ValueError("需模型ID、API Key、baseUrl传入,或在环境变量中定义")
        
        self.client = OpenAI(api_key=apikey,base_url=baseUrl,timeout=timeout)


    def think(self, message: list[Dict[str,str]],temperature: float = 0) -> str:
        """
        调用模型进行思考
        :param message: 输入消息
        :param temperature: 模型温度
        :return: 模型输出
        """
        print(f"[*]正在调用模型：{self.model}")

        try:
            response = self.client.chat.completions.create(
                model = self.model,
                messages=message,
                temperature=temperature,
                stream=True,
            )

            print("[*]模型响应成功...")

            collected_content = []

            for chunk in response:
                content = chunk.choices[0].delta.content or ""
                print(content, end="", flush=True)

                collected_content.append(content)
            print()
            return "".join(collected_content)

        except Exception as e:
            print(f"[!]模型调用失败:{e}")
            return None
    
if __name__ == "__main__":

        llm = LLMClient()
        message = [
            {"role": "system", "content": "你是一个agent开发助手，你需要帮我我做出一个ctf-agent项目，能够辅助或直接解出ctf web题目"},
            {"role": "user", "content": "我的项目，你该怎么做？"},
        ]

        print("[*]----正在调用大模型----")
        print(llm.think(message))

```

到这里我们就手搓了一个LLM客户端了，相当于通过api连接上了大模型，只要改变提示词就会像平常使用ai一样了。
# 2 ReAct 范式

在准备好LLM客户端后，我们将构建第一个，也是最经典的一个智能体范式**ReAct (Reason + Act)**。ReAct由Shunyu Yao于2022年提出，其核心思想是模仿人类解决问题的方式，将**推理 (Reasoning)** 与**行动 (Acting)** 显式地结合起来，形成一个“思考-行动-观察”的循环。

## 2.1 实现

这里我们实现一个agent，回答关于“华为最新手机”的问题，我们需要为智能体提供一个网页搜索工具。在这里我们选用 **SerpApi**，它通过API提供结构化的Google搜索结果，能直接返回“答案摘要框”或精确的知识图谱信息，
```bash
pip install google-search-results
```
安装需要的库
同时，需要前往 [SerpApi官网](https://serpapi.com/) 注册一个免费账户，这是一个google搜索的api，获取你的API密钥，并将其添加到我们项目根目录下的 `.env` 文件中：
```bash
# .env
#... (保留之前的LLM配置)
SERPAPI_API_KEY="YOUR_SERPAPI_API_KEY"
```
一个良好定义的工具应包含以下三个核心要素：

1. **名称 (Name)**： 一个简洁、唯一的标识符，供智能体在 `Action` 中调用，例如 `Search`。
2. **描述 (Description)**： 一段清晰的自然语言描述，说明这个工具的用途。**这是整个机制中最关键的部分**，因为大语言模型会依赖这段描述来判断何时使用哪个工具。
3. **执行逻辑 (Execution Logic)**： 真正执行任务的函数或方法。
我们的第一个工具是 `search` 函数，它的作用是接收一个查询字符串，然后返回搜索结果。
```python
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
            "gl": "cn",  # 国家代码
            "hl": "zh-cn", # 语言代码
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
            # 如果没有直接答案，则返回前三个有机结果的摘要
            snippets = [
                f"[{i+1}] {res.get('title', '')}\n{res.get('snippet', '')}"
                for i, res in enumerate(results["organic_results"][:3])
            ]
            return "\n\n".join(snippets)
        
        return f"对不起，没有找到关于 '{query}' 的信息。"

    except Exception as e:
        return f"搜索时发生错误: {e}"
```
