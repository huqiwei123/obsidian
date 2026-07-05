An agent is a model calling tools in a loop until a given task is complete.
思考 -> 工具调用 -> 观察结果
如果模型判断需要外部信息或执行动作，会生成：
```json
{  
	"tool_name": "get_weather",  
	"arguments": {  
		"city": "San Francisco"  
	}  
}
```
The job of a harness: get the model the right context at the right time for the given task.
harness是围绕循环的整套系统：model模型、prompt控制模型行为的规则边界系统、tools模型可以调用的外部能力、middleware中间层逻辑（retry、memory summarization、权限系统）

create_agent是一个高可配置的harness
Building on that, you can configure the basics directly with the `model=` `tools=` `system_prompt=` parameters

agent = model + harness
![[Pasted image 20260704220505.png|500]]
* Model：Agent配置的模型
* Tools：提供给Agent的工具，通过`from langchain.tools import tool`中的`@tool`注解标记函数，将其定义为Agent可调用的工具
* System prompt：系统提示词
* Structed output：`response_format=`使LLM返回特定格式的响应
* invoke方法参数config是运行框架的配置，控制Agent怎么运行，context是当前调用的上下文，比如当前调用的一些上下文信息（当前调用用户的user_id），智能体能够动态识别到不同调用的用户（动态上下文），并将动态信息（当前调用的用户的id、company、权限角色）作为参数传给工具
* Streaming：流式输出，`agent.stream_events()`
* `create_agent`具有极强的可扩展性，中间件是实现定制化的基础单元，每个中间件处理一个特定功能，在恰当的时机挂接到代理循环中，并能与其他中间件自由组合

#### 配置harness
除了模型+工具的循环式调用，harness还需要建立一整套能力（middleware）
1. 执行环境：Tools, filesystem, sandboxes, code execution
2. 上下文管理：Summarization, memory, skills, prompt caching
3. 规划与委派：Todo lists, subagents for parallel, isolated work
4. 容错：Retries, fallbacks, call limits
5. 护栏：拦截用户输入、Tool输出、模型输出等隐私信息
6. 人工控制：Human-in-the-loop approval before high-impact actions
#### 执行环境
执行环境给智能体一个工作区，工作区有可调用的工具，跨越多轮读写的文件系统和运行脚本和执行shell指令
```python
create_agent(model, tools, middleware=[FilesystemMiddleware(backend=StateBackend())])
```
#### 上下文管理
模型具有一个固定大小的上下文窗口，填充了用户对话历史、tool调用请求、tool返回结果、中间推理步骤等
```python
create_agent(model, tools, middleware=[
			FilesystemMiddleware(backend=backend),
			SummarizationMiddleware(model=model, backend=backend),
			MemoryMiddleware(backend=backend, sources=["./AGENTS.md"]),
			SkillsMiddleware(backend=backend, sources=["./skills/"]),
			])
```
给agent增加了一组上下文管理中间件，Filesystem Summarization Memory Skills
Filesystem给agent一个可读可写的虚拟文件系统环境
Summarization给agent一个上下文压缩器
Memory给agent长期结构化记忆
Skills给agent特定领域的技能包
#### 规划与委派
主agent能将复杂任务拆解为多个子任务，将每个子任务委派给多个具有独立上下文的子agent进行并行处理，同时防止主agent上下文污染
```python
create_agent(model, tools, middleware=[
			FilesystemMiddleware(backend=backend),
			TodoListMiddleware(),
			SubAgentMiddleware(
				backend=backend,
				subagents=[
					{
						"name": "researcher",
						"description": "Use the search tool to research the question and summarize key points.",
						"tools": [search],
						"model": "anthropic:claude-sonnet-4-6",
						"middleware": [],
					}
				]
			)
			])
```
#### 容错
生产阶段的agent遇到在开发阶段未出现的故障时（限速、模型调用超时、短暂性API错误），需要具备一定的容错能力，容错中间件能够处理架构层级的错误，工具和业务逻辑不需要主动通过try/catch进行捕获
```python
create_agent(model, tools, middleware=[
			ModelRetryMiddleware(max_retries=3),
			ToolRetryMiddleware(max_retries=2),
			])
```
#### 护栏
系统级强约束控制层，有些规则不能靠提示词来约束，必须在系统层强制执行，Guardrails会在agent运行过程中拦截数据流，确保合规规则在进入模型上下文之前就被执行
```python
create_agent(model, tools, middleware=[PIIMiddleware("email")])
```
#### 人工控制
在agent自动执行过程中，把“关键决策点”交给人来批准，而不是完全让模型自主运行
```python
create_agent(model, tools, middleware=[HumanInTheLoopMiddleware(interrupt_on={"write_file": True})])
```
