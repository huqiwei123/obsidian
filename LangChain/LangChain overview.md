### Install LangChain
```bash
pip install -U langchain
```
针对不同模型厂商提供了独立的包，`langchain-openai` `langchain-anthropic`
### Build a basic agent
Agent = 模型（Model）+ 外围控制系统（Harness）
`create_agent`是LangChain提供的最小可用AI Agent框架，用来把LLM变成可执行任务的系统，而不仅仅是聊天模型
```python
from langchain.agents import create_agent

# 定义工具，本质是一个可以被调用的函数
def get_weather(city: str) -> str

# 定义模型和工具的agent
agent = create_agent(model, tools, system_prompt)

# agent.invoke接收的是一个对象，对象中的messages是一个消息栈（不断累计的对话状态）
result = agent.invoke({
			"messages": [{
				"role": "user", 
				"content": "What's the weather in San Francisco?"
				}]
		})

# 自定义模型
from langchain_openai import ChatOpenAI

# ChatOpenAI是接入兼容OpenAI协议的模型接口
deepseek_model = ChatOpenAI(model, base_url, api_key)
```
1. Standard model interface: 标准模型接口，统一不同大模型的调用方式
2. Highly configurable harness: 高度可配置的Agent外壳/运行框架
3. Built on top of LangGraph: 基于LangGraph构建，LangChain Agent本身运行在LangGraph上
4. Debug with LangSmith: 用LangSmith来观测和调试Agent执行过程
`InMemorySaver()`是LangGraph/LangChain里用来做短期记忆存储的一个组件，保存agent的运行状态（messages、变量、步骤），用于下一轮继续对话
每次调用agent.invoke(...)本质上是一个有状态系统
```python
agent = create_agent(model=model, tools=[], system_prompt, checkpointer=InMemorySaver())

agent.invoke( {"messages": [{"role": "user", "content": content}]}, config={"configurable": {"thread_id": "great-gatsby-lc"}}, )
```
agent.invoke本身是无状态无记忆的，第一次调用invoke时，将状态保存到InMemorySaver['great-gatsby-lc']中，后续agent.invoke时，从InMemorySaver找到历史会话记录

