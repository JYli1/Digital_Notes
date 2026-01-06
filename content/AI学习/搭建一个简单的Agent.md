在本案例中，我们的目标是构建一个能处理分步任务的智能旅行助手。需要解决的用户任务定义为："你好，请帮我查询一下今天北京的天气，然后根据天气推荐一个合适的旅游景点。"要完成这个任务，智能体必须展现出清晰的逻辑规划能力。它需要先调用天气查询工具，并将获得的观察结果作为下一步的依据。在下一轮循环中，它再调用景点推荐工具，从而得出最终建议。
## 0x01 准备工作
### 1.1 python库下载
1. `request`库，用于发送http请求调用api，获取天气数据之类的
2. `tavily-python`是一个强大的 AI 搜索 API 客户端，用于获取实时的网络搜索结果，可以在[官网](https://www.tavily.com/)注册后获取 API
3. `openai`是 OpenAI 官方提供的 Python SDK，用于调用 GPT 等大语言模型服务
所以我们安装库：
```bash
pip install requests tavily-python openai
```
(注意python版本最好在3.9/3.10，开始使用的3.11时报错，换到3.9时正常，应该是生态没跟上)
下面开始写代码，因为代码亮不大，所以写在一个文件也行
`以下代码按顺序写在同一文件下即可`
### 1.2 大模型API获取
使用 API 调用大模型需要 API 密钥，这里我们以Qwen为例，可以从[ModelScope](https://modelscope.cn/docs/model-service/API-Inference/intro)获取，它提供Qwen系列的免费（OpenAI）兼容格式的API，每天免费2000次调用。
1. 进入官网登录
2. 在 首页 -> 访问令牌 中
![[file-20260107072401248.png]]
3. 在 模型库 -> 支持体验 -> 推理API-inference -> DeepSeek-V3.2 里面有代码示范，后面需要的一些参数的写法里面有


## 0x02 代码编写
### 2.1 提示词部分
驱动真实 LLM 的关键在于**提示工程（Prompt Engineering）**。我们需要设计一个“指令模板”，告诉 LLM 它应该扮演什么角色、拥有哪些工具、以及如何格式化它的思考和行动。这是我们智能体的“说明书”，它将作为`system_prompt`传递给 LLM。 提示词是让大模型正确工作的重要部分：
```python
AGENT_SYSTEM_PROMPT = """
你是一个智能旅行助手。你的任务是分析用户的请求，并使用可用工具一步步地解决问题。

# 可用工具:
- `get_weather(city: str)`: 查询指定城市的实时天气。
- `get_attraction(city: str, weather: str)`: 根据城市和天气搜索推荐的旅游景点。

# 行动格式:
你的回答必须严格遵循以下格式。首先是你的思考过程，然后是你要执行的具体行动，每次回复只输出一对Thought-Action：
Thought: [这里是你的思考过程和下一步计划]
Action: [这里是你要调用的工具，格式为 function_name(arg_name="arg_value")]

# 任务完成:
当你收集到足够的信息，能够回答用户的最终问题时，你必须在`Action:`字段后使用 `finish(answer="...")` 来输出最终答案。

请开始吧！
"""

```
这是一个示例。

### 2.2 工具 1：查询真实天气
我们将使用免费的天气查询服务 `wttr.in`，它能以 JSON 格式返回指定城市的天气数据。下面是实现该工具的代码：
```python
import requests
import json

def get_weather(city: str) -> str:
    """
    通过调用 wttr.in API 查询真实的天气信息。
    """
    # API端点，我们请求JSON格式的数据
    url = f"https://wttr.in/{city}?format=j1"
    
    try:
        # 发起网络请求
        response = requests.get(url)
        # 检查响应状态码是否为200 (成功)
        response.raise_for_status() 
        # 解析返回的JSON数据
        data = response.json()
        
        # 提取当前天气状况
        current_condition = data['current_condition'][0]
        weather_desc = current_condition['weatherDesc'][0]['value']
        temp_c = current_condition['temp_C']
        
        # 格式化成自然语言返回
        return f"{city}当前天气:{weather_desc}，气温{temp_c}摄氏度"
        
    except requests.exceptions.RequestException as e:
        # 处理网络错误
        return f"错误:查询天气时遇到网络问题 - {e}"
    except (KeyError, IndexError) as e:
        # 处理数据解析错误
        return f"错误:解析天气数据失败，可能是城市名称无效 - {e}"

```
这就像一个python程序，通过调用API实现：
1. 传入城市
2. 到API请求指定城市的天气，返回json格式的天气数据
3. 把json格式化成自然语言并返回

### 2.3 工具 2：搜索并推荐旅游景点
将定义一个新工具 `search_attraction`，它会根据城市和天气状况，互联网上搜索合适的景点：
```python
import os
from tavily import TavilyClient

def get_attraction(city: str, weather: str) -> str:
    """
    根据城市和天气，使用Tavily Search API搜索并返回优化后的景点推荐。
    """
    # 1. 从环境变量中读取API密钥
    api_key = os.environ.get("TAVILY_API_KEY")
    if not api_key:
        return "错误:未配置TAVILY_API_KEY环境变量。"

    # 2. 初始化Tavily客户端
    tavily = TavilyClient(api_key=api_key)
    
    # 3. 构造一个精确的查询
    query = f"'{city}' 在'{weather}'天气下最值得去的旅游景点推荐及理由"
    
    try:
        # 4. 调用API，include_answer=True会返回一个综合性的回答
        response = tavily.search(query=query, search_depth="basic", include_answer=True)
        
        # 5. Tavily返回的结果已经非常干净，可以直接使用
        # response['answer'] 是一个基于所有搜索结果的总结性回答
        if response.get("answer"):
            return response["answer"]
        
        # 如果没有综合性回答，则格式化原始结果
        formatted_results = []
        for result in response.get("results", []):
            formatted_results.append(f"- {result['title']}: {result['content']}")
        
        if not formatted_results:
             return "抱歉，没有找到相关的旅游景点推荐。"

        return "根据搜索，为您找到以下信息:\n" + "\n".join(formatted_results)

    except Exception as e:
        return f"错误:执行Tavily搜索时出现问题 - {e}"

```
就是调用`Tavily Search API`去搜索，其实就是调用一个ai搜索的API接口，得到的答案作为我们的答案返回。

### 2.4 工具归总
最后，我们将所有工具函数放入一个字典，供主循环调用：
```python
# 将所有工具函数放入一个字典，方便后续调用
available_tools = {
    "get_weather": get_weather,
    "get_attraction": get_attraction,
}
```
### 2.5 接入大模型
