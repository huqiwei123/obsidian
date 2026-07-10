模型基于上下文决定什么时候调用工具并提供什么输入参数
### 创建工具
使用`@tool`装饰器将一个方法定义为一个工具，函数的docstring表示为工具的描述，用于模型理解在何时使用它
```python
from langchain.tools import tool

@tool
def search_database(query: str, limit: int = 10) -> str:
    """Search the customer database for records matching the query.

    Args:
        query: Search terms to look for
        limit: Maximum number of results to return
    """
    return f"Found {limit} results for '{query}'"
```
### 自定义工具属性
#### 自定义工具名
工具名默认为函数名，通过`@tool(web_search)`可自定义工具名
#### 自定义工具描述
`@tool("calculator", description="Performs arithmetic calculations. Use this for any math problems.")`
#### 高级方案定制
通过Pydantic或JSON schemas定义工具复杂的输入
```python
from pydantic import BaseModel, Field
from typing import Literal

# 通过类定义的方式定义了一个工具输入参数的详细方案，对工具的每个参数可以定义默认值和详细描述，便于模型理解
class WeatherInput(BaseModel):
    """Input for weather queries."""
    location: str = Field(description="City name or coordinates")
    units: Literal["celsius", "fahrenheit"] = Field(
        default="celsius",
        description="Temperature unit preference"
    )
    include_forecast: bool = Field(
        default=False,
        description="Include 5-day forecast"
    )
    
weather_schema = {
    "type": "object",
    "properties": {
        "location": {"type": "string"},
        "units": {"type": "string"},
        "include_forecast": {"type": "boolean"}
    },
    "required": ["location", "units", "include_forecast"]
}

@tool(args_schema=WeatherInput/weather_schema)
def get_weather(location: str, units: str = "celsius", include_forecast: bool = False) -> str:
    """Get current weather and optional forecast."""
    temp = 22 if units == "celsius" else 72
    result = f"Current weather in {location}: {temp} degrees {units[0].upper()}"
    if include_forecast:
        result += "\nNext 5 days: Sunny"
    return result
```
### 访问上下文
工具调用能通过ToolRuntime参数访问运行时的上下文（会话历史、用户数据、持久化记忆）
#### State
State represents short-term memory that exists for the duration of a conversation.
Agent当前这一次对话执行过程中的“内存数据”，保存了当前会话里动态变化的信息，如已经发生的消息，用户临时偏好，当前任务进度，工具调用结果，Agent中间产生的变量
#### Context
在调用过程中不可变配置，用户id和session info
#### Store

