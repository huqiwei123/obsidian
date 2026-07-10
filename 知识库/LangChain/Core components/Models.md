除了文本生成，模型也支持工具调用，结构化输出，多模态，推理
模型是智能体的推理引擎，驱动智能体做决策、工具调用、语义理解、给出最终答案
* With agents - 在创建agent时动态指定不同模型
* Standalone - 不含循环的纯模型调用，这种模式下延迟低、token消耗低、可控性高、智能性低、适用于简单任务 `model.invoke(prompt)`

### 初始化模型
`init_chat_model` 从模型提供商中初始化一个chat model

```python
import os
from langchain.chat_models import init_chat_model

os.environ["OPENAI_API_KEY"] = "sk-..."
model = init_chat_model("gpt-5.5")
```
LangChain支持主流的模型提供商integration packages，对不同的模型提供统一的接口来根据模型名切换模型，各模型提供商根据统一接口进行适配，实现不同模型之间的无感切换
LangChain Model Interface -> Provider Adapter -> Native API call
>LangChain通过统一模型接口+provider插件化架构，让所有大模型API被抽象成同一个调用方式，从而实现模型无关的应用开发

### 模型参数
* model(String): 模型名称 / 模型提供商:模型名
* api_key: 模型访问密钥，不写死在代码里，通常放到环境变量中
* temperature: 控制模型输出的随机性
* max_tokens: 限制响应输出总token数
* timeout: 模型响应最大等待时延
* max_retries: 模型请求最大重试次数
### 模型方法
* Invoke方法：使用invoke方法，并传入单条消息或消息列表，对于消息列表，Each message has a role that models use to indicate who sent the message in the conversation
* stream方法：流式输出，stream()返回一个包含输出块的迭代器
  astream_events()：Runnable的执行事件流
* batch：批量并行模型请求

### 工具调用
model.bind_tools([...])
```python
from langchain.tools import tool

@tool
def get_weather(location: str) -> str:
    """Get the weather at a location."""
    return f"It's sunny in {location}."


model_with_tools = model.bind_tools([get_weather])

response = model_with_tools.invoke("What's the weather like in Boston?")
for tool_call in response.tool_calls:
    # View tool calls made by the model
    print(f"Tool: {tool_call['name']}")
    print(f"Args: {tool_call['args']}")
```
1. Tool execution loop:
	1. message + tool description -> model
	2. model -> which tool(ai message)
	3. message = message + ai message
	4. invoke tool
	5. message = message + tool result
	6. message + tool description -> model
	7. model -> result(finished)
2. Forcing tool calls: 强制模型必须使用工具
   `model.bind_tools([tool_1], tool_choice="any")`
3. Parallel tool calls: 大模型可能根据语义多次调用同一工具（不同入参）
4. Streaming tool calls
### 结构化输出
LangChain支持提供模型格式化输出方案，以确保模型输出在后续处理过程中能够以结构化的方式被解析，LangChain支持多种方案和方法迫使模型以结构化的方式输出内容
1. Pydantic
   ```python
   from pydantic import BaseModel, Field
	
	class Movie(BaseModel):
	    """A movie with details."""
	    title: str = Field(description="The title of the movie")
	    year: int = Field(description="The year the movie was released")
	    director: str = Field(description="The director of the movie")
	    rating: float = Field(description="The movie's rating out of 10")
	
	model_with_structure = model.with_structured_output(Movie)
	response = model_with_structure.invoke("Provide details about the movie Inception")
	print(response)
   ```
   类本身也可以看做是可调用的方法，结构化的过程本质上是把Movie作为一个工具调用，title year director rating是工具参数
2. TypedDict
   ```python
	from typing_extensions import TypedDict, Annotated

	class MovieDict(TypedDict):
	    """A movie with details."""
	    title: Annotated[str, ..., "The title of the movie"]
	    year: Annotated[int, ..., "The year the movie was released"]
	    director: Annotated[str, ..., "The director of the movie"]
	    rating: Annotated[float, ..., "The movie's rating out of 10"]
	
	model_with_structure = model.with_structured_output(MovieDict)
	response = model_with_structure.invoke("Provide details about the movie Inception")
	print(response)
   ```
3. JSON Scheme
   ```python
	import json

	json_schema = {
	    "title": "Movie",
	    "description": "A movie with details",
	    "type": "object",
	    "properties": {
	        "title": {
	            "type": "string",
	            "description": "The title of the movie"
	        },
	        "year": {
	            "type": "integer",
	            "description": "The year the movie was released"
	        },
	        "director": {
	            "type": "string",
	            "description": "The director of the movie"
	        },
	        "rating": {
	            "type": "number",
	            "description": "The movie's rating out of 10"
	        }
	    },
	    "required": ["title", "year", "director", "rating"]
	}
	
	model_with_structure = model.with_structured_output(
	    json_schema,
	    method="json_schema",
	)
	response = model_with_structure.invoke("Provide details about the movie Inception")
	print(response)
   ```
### Model profiles
LangChain给每个chat model挂上的一份“能力字典”，用来告诉程序，这个模型支持什么，不支持什么，在程序流中可以根据模型的自身属性进行决策`model.profile["image_inputs"]`
### Prompt caching
Prompt Caching 并不是 LangChain 自己缓存 Prompt，而是利用模型提供商（如 OpenAI、Anthropic、Gemini、Bedrock）的缓存能力。LangChain 的作用是统一封装各家的缓存接口，并通过 Middleware 自动标记稳定的 Prompt（如 System Prompt、Tool Schema），从而提高缓存命中率，最终达到降低 Token 成本和缩短响应时间的目的
LangChain在agents场景中，引入middleware，让系统可以自动识别并缓存稳定不变的部分prompt
识别稳定prompt -> 标记缓存点 -> 复用token计算结果
### Server-side tool use
tool在模型提供商侧执行
```
# 模型绑定的不再是本地的工具Function，而是声明一个tool，在模型提供商处调用
tool = {"type": "web_search"}
model_with_tools = model.bind_tools([tool])
```
模型返回的不再是tool_calls，而是content_blocks
### Rate limiting
LangChain提供了一个统一接口`from langchain_core.rate_limiters import InMemoryRateLimiter`，一个本地进程内的token bucket限流器
系统里有一个“桶”，桶里有许可令牌，每次请求要拿许可令牌才能执行，许可令牌会按时间慢慢补充
```python
rate_limiter = InMemoryRateLimiter(request_per_second=0.1)
```
一个基于token bucket的本地限流器，用来在请求发出前控制LLM调用频率，从而避免出发provider的RPM/TPM限制，并稳定agent系统的并发行为
* Custom base URL:
  ```python
	  model = init_chat_model(
	    model="MODEL_NAME",
	    model_provider="openai",
	    base_url="BASE_URL",
	    api_key="YOUR_API_KEY",
	)
  ```
* HTTP代理访问
  ```python
	from langchain_openai import ChatOpenAI

	model = ChatOpenAI(
	    model="gpt-5.5",
	    openai_proxy="http://proxy.example.com:8080"
	)
  ```
### Token usage
跨模型记录token使用情况（按模型分组）
1. callback handler
	```python
		from langchain.chat_models import init_chat_model
		from langchain_core.callbacks import UsageMetadataCallbackHandler
		
		model_1 = init_chat_model(model="gpt-5.4-mini")
		model_2 = init_chat_model(model="claude-haiku-4-5-20251001")
		
		callback = UsageMetadataCallbackHandler()
		result_1 = model_1.invoke("Hello", config={"callbacks": [callback]})
		result_2 = model_2.invoke("Hello", config={"callbacks": [callback]})
		print(callback.usage_metadata)
	```
2. context manager
 ```python
	from langchain.chat_models import init_chat_model
	from langchain_core.callbacks import get_usage_metadata_callback
	
	model_1 = init_chat_model(model="gpt-5.4-mini")
	model_2 = init_chat_model(model="claude-haiku-4-5-20251001")
	
	with get_usage_metadata_callback() as cb:
	    model_1.invoke("Hello")
	    model_2.invoke("Hello")
	    print(cb.usage_metadata)
 ```  
### Invocation config
运行时控制层机制，在不改变模型定义的情况下，对单次调用进行“行为注入+观测+控制”的配置入口
```python
response = model.invoke(
    "Tell me a joke",
    config={
        "run_name": "joke_generation",      # Custom name for this run
        "tags": ["humor", "demo"],          # Tags for categorization
        "metadata": {"user_id": "123"},     # Custom metadata
        "callbacks": [my_callback_handler], # Callback handlers
    }
)
```
这个config是一个RunnableConfig运行配置对象
* run_name：给单次调用命名，在LangSmith/trace里这个调用会显示调用名称
* tags：给这次调用打标签
* metadata：自定义业务信息
* callbacks：执行过程hook，可以监听token生成、tool call、usage metadata、error
在模型调用过程中动态配置可变模型（创建模型实例时不指定具体模型）：
```python
from langchain.chat_models import init_chat_model

configurable_model = init_chat_model(temperature=0)

configurable_model.invoke(
    "what's your name",
    config={"configurable": {"model": "gpt-5-nano"}},  # Run with GPT-5-Nano
)
configurable_model.invoke(
    "what's your name",
    config={"configurable": {"model": "claude-sonnet-4-6"}},  # Run with Claude
)
```
带默认值的可配置模型（运行时覆盖）
```python
first_model = init_chat_model(
        model="gpt-5.4-mini",
        temperature=0,
        configurable_fields=("model", "model_provider", "temperature", "max_tokens"),
        config_prefix="first",  # Useful when you have a chain with multiple models
)
first_model.invoke("what's your name")

first_model.invoke(
    "what's your name",
    config={
        "configurable": {
            "first_model": "claude-sonnet-4-6",
            "first_temperature": 0.5,
            "first_max_tokens": 100,
        }
    },
)
```
### Dynamic model selection
动态模型选择在运行时基于当前状态和上下文进行模型动态选择，他不是在业务中写死，而是通过拦截用户请求，根据用户请求状态或上下文信息动态判断选用模型
```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse


basic_model = ChatOpenAI(model="gpt-5.4-mini")
advanced_model = ChatOpenAI(model="gpt-5.5")

@wrap_model_call
def dynamic_model_selection(request: ModelRequest, handler) -> ModelResponse:
    """Choose model based on conversation complexity."""
    message_count = len(request.state["messages"])

    if message_count > 10:
        # Use an advanced model for longer conversations
        model = advanced_model
    else:
        model = basic_model

    return handler(request.override(model=model))

agent = create_agent(
    model=basic_model,  # Default model
    tools=tools,
    middleware=[dynamic_model_selection]
)
```
使用`@wrap_model_call`创建一个middleware，在用户请求时修改模型
