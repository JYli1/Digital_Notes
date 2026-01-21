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

