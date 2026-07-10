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
Prompt Caching 并不是 LangChain 自己缓存 Prompt，而是利用模型提供商（如 OpenAI、Anthropic、Gemini、Bedrock）的缓存能力。LangChain 的作用是统一封装各家的缓存接口，并通过 Middleware 自动标记稳定的 Prompt（如 System Prompt、Tool Schema），从而提高缓存命中率，最终达到降低 Token 成本和缩短响应时间的目的。
