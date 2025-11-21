---
title: LLM-app-frame
date: 2025-11-17 11:05:59
tags: LLM
category: Notes
except: 大模型应用开发框架
---

## Python 常用框架

- Langchain
- LlamaIndex

## 基于RAG架构的开发

- 大模型知识冻结
- 大模型幻觉

## 基于Agent架构的开发  

## 模型调用的分类

1. 按照模型功能不同：

   - 非对话模型
   - 对话模型
   - 嵌入模型

2. 按照模型调用时参数书写位置

   - 硬编码方式，将参数写在代码中
   - 使用环境变量的方式
   - 使用配置文件的方式

3. 具体API的调用

   - 使用LangChain提供的API
   - 使用OpenAI 官方的API
   - 使用其他平台提供的API

```python
# 非对话模型
from langchain_openai import OpenAI
# 对话模型
from langchain_openai import ChatOpenAI
```

## 参数设置

- model_name
  - 模型名称，如"gpt-3.5-turbo"、"gpt-4"等
- base_url
  - 模型API的基础URL，如"https://api.openai.com/v1"
- api_key
  - 模型API的密钥，用于身份验证

- temperature
  - 温度，控制生成文本的随机性，取值范围为0～1
    - 值越低，输出越确定、保守，适合事实回答
    - 值越高，输出越多样、有创意，适合创意写作
  - 精确模式
    - 0.5或更低，生成的文本更加安全可靠
  - 平衡模式
    - 通常是0.8，生成的文本通常有一定的多样性，又能保持较好的连贯性和准确性
  - 创意模式
    - 通常是1，生成的文本更有创意，但也更容易出现语法错误或不合逻辑的内容
- max_tokens
  - 限制生成文本的最大长度，防止输出过长
  - 设置建议
    - 短回复：128-256
    - 常规对话、多轮对话：512-1024
    - 长内容生成：1024-4096
  - token
    - 大模型处理文本的最小单位，相当于自然语言中的词或字，输出时逐个token依次生成
    - 一般收费标准1个token约等于1-1.8个汉字或3-4个英文字母
    - [token字符转化工具](https://platform.openai.com/tokenizer)

## 对话模型的message

- SystemMessage
  - 设定AI行为规则或背景信息。比如设定AI的初始状态、行为模式或对话的总体目标。比如：“作为一个代码专家”，或者“返回JSON格式”。通常是输入消息序列中的第一个传递
- HumanMessage
  - 表示来自用户输入
- AIMessage
  - 存储AI回复的内容，可以是文本，也可以是调用工具的请求
- ChatMessage
  - 可以自定义角色的通用消息类型
- FunctionMessage/ToolMessage
  - 函数调用/工具消息，用于函数调用结果的消息类型

## 模型调用的方法

- invoke
  - 处理单条输入，等待LLM完全推理完成后再返回调用结果
- stream
  - 流式响应，逐字输出LLM的响应结果
- batch
  - 处理批量输入
有相应的异步方法，与asyncio和await语法一起使用实现并发
- astream
  - 异步流式响应
- ainvoke
  - 异步处理单条输入
- abatch
  - 异步处理批量输入
- astream_log
  - 异步流式返回中间步骤，以及最终结果

## 提示词模板

- PromptTemplate
  - 用于生成字符串提示
- ChatPromptTemplate
  - 聊天提示模板，用于组合各种角色的消息模板，传入聊天模型
- XxxMessagePromptTemplate
  - 消息模板词模板，包括：SystemMessagePromptTemplate、HumanMessagePromptTemplate、AIMessagePromptTemplate、ChatMessagePromptTemplate等
- FewShotPromptTemplate
  - 样本提示词模板，通过示例来教模型如何回答
- PipelinePrompt
  - 管道提示词模板，将多个提示词模板组合起来一起使用
- 自定义模板
  - 允许基于其他模板类来定制自己的提示词模板

```python
from Langchain_core.prompts import PromptTemplate

prompt_template = PromptTemplate.from_template(
    template="请回答以下问题：{question}",
)
prompt_template.format(question="你是谁") # 返回值类型为字符串
prompt_template.invoke(input={"question": "你是谁"}) # 返回值类型为PromptValue

prompt_template = PromptTemplate.from_template(
    template="请回答以下问题：{question}, 并保持回答的专业和准确性, 返回格式{format}",
    partial_variables={"question": "AI能够做什么",},
)
prompt_template.format(format="JSON")
```

### ChatPromptTemplate

```python
from Langchain_core.prompts import ChatPromptTemplate

chat_prompt_template = ChatPromptTemplate.from_messages(
    messages=[
        ("system", "你是一个{role}，名字叫{name}"),
        ("human", "{question}"),
    ],
    #input_variables=["role", "name", "question"],
) 
## 返回类型为ChatPromptValue
chat_prompt_template.invoke(input={"role": "专业的代码助手", "name": "AI助手", "question": "你是谁"})  
## 返回类型为List
chat_prompt_template.format_messages(role="专业的代码助手", name="AI助手", question="你是谁")

## 实现ChatPromptValue、List、字符串之间的转换
chat_prompt_template.to_string()
chat_prompt_template.to_messages()
```

## 插入消息列表

```python
# 当ChatPromptTemplate模板中的消息类型和个数不确定的时候，可以使用MessagePlaceholder
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.prompts.chat import MessagePlaceholder
from langchain_core.messages import HumanMessage, AIMessage

chat_prompt_template = ChatPromptTemplate.from_messages(
    messages=[
        ("system", "你是一个{role}，名字叫{name}"),
        MessagePlaceholder(variable_name="history"),
        ("human", "{question}"),
    ],
)

chat_prompt_template.invoke({
  "role": "专业的代码助手", 
  "name": "AI助手", 
  # "messages": [("human", "你是谁")],
  "history": [HumanMessage(content="你是谁"), AIMessage(content="我是一个专业的代码助手")],
  "question": "我刚才的问题是什么？"
})
```

## 少量示例提示词模板

FewShotPromptTemplate与PromptTemplate一起使用
FewShotChatMessagePromptTemplate 与ChatPromptTemplate一起使用

```python
example_prompt_template = FewShotPromptTemplate(
    example_prompt=PromptTemplate.from_template(
        template="问题：{question}\n回答：{answer}",
    ),
    examples=[
        {"question": "你是谁", "answer": "我是一个专业的代码助手"},
        {"question": "你能做什么", "answer": "我可以回答代码相关的问题"},
    ],
    suffix="问题：{question}\n回答：",
    input_variables=["question"],
)
```

```python
from langchain_core.prompts import FewShotChatMessagePromptTemplate
example_prompt_template = FewShotChatMessagePromptTemplate(
    example_prompt=ChatPromptTemplate.from_messages(
        messages=[
            ("human", "{question} 是多少？"),
            ("ai", "{answer}"),
        ],
    ),
    examples=[
        {"question": "2 🌈 2", "answer": "4"},
        {"question": "2 🌈 3", "answer": "8"},
    ],
)
final_prompt_template = ChatPromptTemplate.from_messages(
    messages=[
        ("system", "你是一个数学专业的助手"),
        example_prompt_template,
        ("human", "{question}"),
    ],
)
```
