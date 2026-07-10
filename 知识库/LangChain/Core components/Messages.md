Messages是模型上下文的基础单元，代表了模型的输入和输出，携带有和模型交互过程中的内容和表示会话状态的元数据：
* Role：表示消息类型（system、user）
* Content
* Metadata
### 基本使用
创建消息对象 -> 传递给模型调用 model.invoke(message)
```python
from langchain.chat_models import init_chat_model
from langchain.messages import HumanMessage, AIMessage, SystemMessage

model = init_chat_model("gpt-5-nano")

system_msg = SystemMessage("You are a helpful assistant.")
human_msg = HumanMessage("Hello, how are you?")

# Use with chat models
messages = [system_msg, human_msg]
response = model.invoke(messages)  # Returns AIMessage
```
### Text prompts
Text prompt are strings - ideal for straightforward generation tasks where you don't need to retain conversation history.
### Message prompts
将消息封装成不同的对象SystemMessage HumanMessage AIMessage，以list的形式传递给模型调用
```python
from langchain.messages import SystemMessage, HumanMessage, AIMessage

messages = [
    SystemMessage("You are a poetry expert"),
    HumanMessage("Write a haiku about spring"),
    AIMessage("Cherry blossoms bloom...")
]
response = model.invoke(messages)
```
### Dictionary format
将每条消息表示为字典的形式
```python
messages = [
    {"role": "system", "content": "You are a poetry expert"},
    {"role": "user", "content": "Write a haiku about spring"},
    {"role": "assistant", "content": "Cherry blossoms bloom..."}
]
response = model.invoke(messages)
```
### 消息类型
* System message
* Human message
* AI message
* Tool message
### System message
系统级消息代表了指导模型的初始指令，可以使用系统级指令为模型设置语调、角色、建立响应指南（预先设置模型的行为倾向）
### Human message
用户输入和交互的消息，text images audio files 和 任何多模态内容
### AI message
代表模型调用的输出，包括多模态数据，tool calls, 模型提供商的元数据，属性如下：
* text：消息text文本内容
* content：消息原始内容（模型返回的原始内容）
* content_blocks：LangChain标准化后的结构化内容（对不同模型厂商返回不同格式内容进行统一和标准化）
* tool_calls：模型调用工具决策
* id：消息唯一识别号
* usage_metadata：消息使用元数据，包括token计数（输入token、输出token数、总token数、缓存token、reasoning token等）
* response_metadata：消息响应元数据
### tool calls
从响应消息的tool_calls字段可以获取一个工具列表，遍历工具列表，读取每个工具的工具名、入参、工具id信息
```python
for tool_call in response.tool_calls:
    print(f"Tool: {tool_call['name']}")
    print(f"Args: {tool_call['args']}")
    print(f"ID: {tool_call['id']}")
```
### token usage
```python
response.usage_metadata

{'input_tokens': 8,
 'output_tokens': 304,
 'total_tokens': 312,
 'input_token_details': {'audio': 0, 'cache_read': 0},
 'output_token_details': {'audio': 0, 'reasoning': 256}}
```
Streaming and chuncks
```python
chunks = []
for chunk in model.stream():
	chunks.append(chunk)
	print(chunk.text)
```
### Tool message
工具调用的结果消息封装
* content：工具调用输出结果（stirng）
* tool_call_id：和模型给定工具id匹配的被调用工具id
* name：被调用的工具名
* artifact：不给模型看，但程序可以访问（工具返回的结果部分给模型看，部分保留给后端业务访问）
### Message content
用户可以提供多模态类型的数据
原生模型消息（消息格式需要匹配模型定义的格式）
```python
# Provider-native format (e.g., OpenAI)
human_message = HumanMessage(content=[
    {"type": "text", "text": "Hello, how are you?"},
    {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}}
])
```
LangChain统一消息格式
```python
# List of standard content blocks
human_message = HumanMessage(content_blocks=[
    {"type": "text", "text": "Hello, how are you?"},
    {"type": "image", "url": "https://example.com/image.jpg"},
])
```
### 标准内容块
Message objects implement a `content_blocks` property that will lazily parse the `content` attribute into a standard, type-safe representation.
#### TextContentBlock 标准文本内容输出
* type：Aways "text"
* text：文本内容
* annotations：文本标注信息，这段文字的额外说明、来源、引用、定位信息
* extras：保存模型特殊字段，在content_block对不同模型标准化时未预留的字段、
#### ReasoningContentBlock 推理内容块
* type：Always "reasoning"
* reasoning：推理内容
* extras：保存模型特殊字段
#### ImageContentBlock
* type: "image"
* url: 指向图片位置的url
* base64：Base64编码的图片数据
* id：内容块的唯一识别号（模型生成的图片需要给一个标识用于识别是哪一个图片，这样后续提问时，模型可以进行定位）
* mime_type：图片MIME类型（image/jpeg、image/png）
#### AudioContentBlock
* type："audio”
* url：音频位置url
* base64：Base64编码的音频数据
* id
* mime_type：音频MIME类型（audio/mpeg、audio/wav）
#### VideoContentBlock
* type："video"
* url
* base64
* id
* mime_type：（video/mp4、video/webm）
#### FileContentBlock
* type："file"
* url
* base64
* id
* mime_type：（application/pdf）
#### PlainTextContentBlock
* type："text-plain"
* text：文本内容
* mime_type：text/plain、text/markdown
#### ToolCall
模型决定调用应用侧工具/函数时产生的标准内容块
* type："tool_call"
* name：要调用的工具名
* args：传递给工具的参数对象
* id：本次工具调用的唯一识别号，后续 ToolMessage 的 tool_call_id 必须和它匹配
```python
{
    "type": "tool_call",
    "name": "search",
    "args": {"query": "weather"},
    "id": "call_123"
}
```
#### ToolCallChunk
流式输出中的工具调用片段，用于 streaming tool calls
* type："tool_call_chunk"
* name：正在调用的工具名
* args：部分工具参数，可能是不完整 JSON 字符串
* id：工具调用 id
* index：该 chunk 在流中的位置
```python
{
    "type": "tool_call_chunk",
    "name": "search",
    "args": "{\"query\":",
    "id": "call_123",
    "index": 0
}
```
#### InvalidToolCall
格式错误或无法解析的工具调用，常见于模型输出的工具参数不是合法 JSON
* type："invalid_tool_call"
* name：尝试调用但失败的工具名
* args：模型生成的错误参数
* error：错误原因
```python
{
    "type": "invalid_tool_call",
    "name": "search",
    "args": "{bad json",
    "error": "Could not parse tool arguments"
}
```
#### ServerToolCall
由模型提供商侧执行的工具调用，不是本地应用执行的工具调用
* type："server_tool_call"
* id：server-side tool call 的唯一识别号
* name：provider 侧工具名
* args：工具参数，通常是 JSON 字符串
```python
{
    "type": "server_tool_call",
    "id": "srv_123",
    "name": "web_search",
    "args": "{\"query\":\"LangChain\"}"
}
```
#### ServerToolCallChunk
流式输出中的 provider 侧工具调用片段
* type："server_tool_call_chunk"
* id：server-side tool call id
* name：provider 侧工具名
* args：部分工具参数，可能是不完整 JSON 字符串
* index：该 chunk 在流中的位置
```python
{
    "type": "server_tool_call_chunk",
    "id": "srv_123",
    "name": "web_search",
    "args": "{\"query\":",
    "index": 0
}
```
#### ServerToolResult
provider 侧工具执行后的结果，例如 web search 的搜索结果
* type："server_tool_result"
* tool_call_id：对应的 ServerToolCall id
* id：server tool result 的唯一识别号
* status：执行状态，"success" 或 "error"
* output：工具输出结果
```python
{
    "type": "server_tool_result",
    "tool_call_id": "srv_123",
    "id": "result_123",
    "status": "success",
    "output": [{"title": "LangChain docs", "url": "https://docs.langchain.com"}]
}
```
#### NonStandardContentBlock
模型提供商特有内容块的 escape hatch，用于 LangChain 标准内容块暂时无法覆盖的 provider-specific 数据结构
* type："non_standard"
* value：provider 原生数据结构
```python
{
    "type": "non_standard",
    "value": {
        "provider_specific_type": "custom_event",
        "payload": {}
    }
}
```