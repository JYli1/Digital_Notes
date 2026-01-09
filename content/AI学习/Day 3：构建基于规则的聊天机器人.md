这次实验将尝试复现人工智能历史上一个极具影响力的早期聊天机器人——ELIZA。

## 0x01 ELIZA 的设计思想

ELIZA是由麻省理工学院的计算机科学家`约瑟夫·魏泽鲍姆（Joseph Weizenbaum）`于1966年发布的一个计算机程序，是早期自然语言处理领域的著名尝试之一。ELIZA并非一个单一的程序，而是一个可以执行不同 “脚本” 的框架。其中，最广为人知也最成功的脚本是“DOCTOR”，它模仿了一位罗杰斯学派的非指导性心理治疗师。

ELIZA的工作方式极其巧妙：它从不正面回答问题或提供信息，而是通过识别用户输入中的关键词，然后应用一套预设的转换规则，将用户的陈述转化为一个开放式的提问。例如，当用户说“我为我的男朋友感到难过”时，ELIZA可能会识别出关键词“我为……感到难过”，并应用规则生成回应：“你为什么会为你的男朋友感到难过？”

魏泽鲍姆的设计思想并非要创造一个真正能够“理解”人类情感的智能体，恰恰相反，他想证明的是，通过一些简单的句式转换技巧，机器可以在`完全不理解对话内容的情况下`，营造出一种“智能”和“共情”的假象。然而，出乎他意料的是，许多与ELIZA交互过的人（包括他的秘书）都对其产生了情感上的依赖，深信它能够理解自己。

本节的实践目标即为复现ELIZA的核心机制，以深入理解这种规则驱动方法的优势与根本局限。

该项目本质上就是像我们玩游戏时的剧情对话，只不过是数据增加，并且加上 随机输出 等机制。

---
## 0x02 核心算法

ELIZA的算法流程基于`模式匹配（Pattern Matching）`与`文本替换（Text Substitution）`，可被清晰地分解为以下四个步骤：
1. **关键词识别与排序：**
规则库为每个关键词（如 `mother`, `dreamed`, `depressed`）设定一个优先级。当输入包含多个关键词时，程序会选择优先级最高的关键词所对应的规则进行处理。
2. **分解规则：**
空格分解句子为单词后循环，通过正则匹配关键词
3. **重组规则：**
然后从预定义的回复语句库中提取回答（可以增加多个句子随机回复）
4. **代词转换：**
在重组前，程序会进行简单的代词转换（如 `I` → `you`, `my` → `your`），以维持对话的连贯性。

通过这套机制，ELIZA成功地将复杂的自然语言理解问题，简化为了一个可操作的、基于规则的模式匹配游戏。

---
## 0x03 代码实现
现在，我们将上一节描述的技术原理转化为一个简单的、可运行的Python函数。下面的代码实现了一个迷你版的ELIZA，它包含了一小部分规则，但足以展示其核心工作机制。

```python
import re
import random

# 定义规则库:模式(正则表达式) -> 响应模板列表
rules = {
    r'I need (.*)': [
        "Why do you need {0}?",
        "Would it really help you to get {0}?",
        "Are you sure you need {0}?"
    ],
    r'Why don\'t you (.*)\?': [
        "Do you really think I don't {0}?",
        "Perhaps eventually I will {0}.",
        "Do you really want me to {0}?"
    ],
    r'Why can\'t I (.*)\?': [
        "Do you think you should be able to {0}?",
        "If you could {0}, what would you do?",
        "I don't know -- why can't you {0}?"
    ],
    r'I am (.*)': [
        "Did you come to me because you are {0}?",
        "How long have you been {0}?",
        "How do you feel about being {0}?"
    ],
    r'.* mother .*': [
        "Tell me more about your mother.",
        "What was your relationship with your mother like?",
        "How do you feel about your mother?"
    ],
    r'.* father .*': [
        "Tell me more about your father.",
        "How did your father make you feel?",
        "What has your father taught you?"
    ],
    r'.*': [
        "Please tell me more.",
        "Let's change focus a bit... Tell me about your family.",
        "Can you elaborate on that?"
    ]
}

# 定义代词转换规则
pronoun_swap = {
    "i": "you", "you": "i", "me": "you", "my": "your",
    "am": "are", "are": "am", "was": "were", "i'd": "you would",
    "i've": "you have", "i'll": "you will", "yours": "mine",
    "mine": "yours"
}

def swap_pronouns(phrase):
    """
    对输入短语中的代词进行第一/第二人称转换
    """
    words = phrase.lower().split()
    swapped_words = [pronoun_swap.get(word, word) for word in words]
    return " ".join(swapped_words)

def respond(user_input):
    """
    根据规则库生成响应
    """
    for pattern, responses in rules.items():
        match = re.search(pattern, user_input, re.IGNORECASE)
        if match:
            # 捕获匹配到的部分
            captured_group = match.group(1) if match.groups() else ''
            # 进行代词转换
            swapped_group = swap_pronouns(captured_group)
            # 从模板中随机选择一个并格式化
            response = random.choice(responses).format(swapped_group)
            return response
    # 如果没有匹配任何特定规则，使用最后的通配符规则
    return random.choice(rules[r'.*'])

# 主聊天循环
if __name__ == '__main__':
    print("Therapist: Hello! How can I help you today?")
    while True:
        user_input = input("You: ")
        if user_input.lower() in ["quit", "exit", "bye"]:
            print("Therapist: Goodbye. It was nice talking to you.")
            break
        response = respond(user_input)
        print(f"Therapist: {response}")

```
