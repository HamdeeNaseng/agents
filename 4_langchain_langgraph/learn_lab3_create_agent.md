# Week 4 — Lab 3: `create_agent` Agent Layer

Notebook:

```text
4_langchain_langgraph/3_lab3.ipynb
```

Lab 1 ให้ Building Blocks เช่น Model, Messages, Tools และ Structured Output ส่วน Lab 2 ให้เราประกอบ Agent Loop เองด้วย StateGraph, Model Node, ToolNode และ Conditional Edge

Lab 3 ขยับขึ้นมาอีกหนึ่งระดับ:

```text
Model + Tools + Prompt
        ↓
create_agent()
        ↓
Compiled LangGraph Agent
```

`create_agent()` สร้าง Graph ที่เรียก Model และ Tools วนซ้ำจนกว่าจะได้คำตอบสุดท้ายหรือถึงเงื่อนไขหยุด จึงเป็น Agent Layer ที่ลด Code ของ Runtime ลงอย่างมาก แต่ยังเปิดให้ปรับพฤติกรรมผ่าน Middleware, Checkpointer และ Structured Response ได้. 

---

## Learning Objectives

เมื่อจบ Lab นี้ควรอธิบายได้ว่า:

1. `create_agent()` ซ่อน Graph ส่วนใดจาก Lab 2
2. Model string รูปแบบ `provider:model` ทำงานอย่างไร
3. `invoke()` และ `ainvoke()` ต่างกันอย่างไร
4. การส่ง Tools เข้า `create_agent()` สร้าง Tool Loop อย่างไร
5. ทำไม Agent ที่สร้างขึ้นยังเป็น Compiled LangGraph
6. Checkpointer และ `thread_id` เพิ่ม Conversation Memory อย่างไร
7. `response_format` สร้าง Structured Response อย่างไร
8. Middleware แทรก Logic เข้า Agent Loop ตรงไหนได้บ้าง
9. `wrap_tool_call` ต่างจากการแก้ Tool Function โดยตรงอย่างไร
10. MCP ทำให้ Agent ใช้ Tools จาก Process ภายนอกอย่างไร
11. Browser Agent มีความเสี่ยงมากกว่า Read-only Tool อย่างไร
12. เมื่อใดควรใช้ `create_agent()` และเมื่อใดควรสร้าง `StateGraph` เอง

---

# 1. ตำแหน่งของ `create_agent`

ภาพรวม Week 4:

```text
Layer 1 — Building Blocks
Models, Messages, Tools, Structured Output

Layer 2 — LangGraph
State, Nodes, Edges, Routing, Persistence

Layer 3 — create_agent
Prebuilt Model–Tool Agent Loop

Layer 4 — Deep Agents
Planning, Sub-agents, Filesystem และ Harness ระดับสูง
```

`create_agent()` ไม่ใช่ Runtime คนละโลกกับ LangGraph แต่เป็น Harness ที่สร้าง Agent บน LangGraph Runtime อีกชั้นหนึ่ง จึงได้รับความสามารถด้าน Graph, Persistence และ Human-in-the-loop จากฐานเดียวกัน. 

Mental model:

```text
Lab 2
เราสร้างเครื่องยนต์และวงจรควบคุมเอง

Lab 3
Framework ประกอบเครื่องยนต์มาตรฐานให้
แล้วเราเพิ่มอุปกรณ์ผ่าน Middleware
```

---

# 2. Imports สำคัญ

```python
from dotenv import load_dotenv
from pydantic import BaseModel, Field

from langchain.agents import create_agent
from langchain.agents.middleware import wrap_tool_call
from langchain_core.tools import tool

from langgraph.checkpoint.memory import MemorySaver
from langchain_mcp_adapters.client import MultiServerMCPClient

load_dotenv(override=True)
```

แยกหน้าที่:

```text
create_agent
→ สร้าง Agent Graph สำเร็จรูป

@tool
→ สร้าง Tools

MemorySaver
→ Checkpoint State ตาม thread_id

response_format
→ กำหนด Structured Response

Middleware
→ แทรก Logic รอบ Model หรือ Tool Calls

MultiServerMCPClient
→ โหลด Tools จาก MCP Servers
```

Notebook ปัจจุบันแบ่งเนื้อหาเป็น Agent พื้นฐาน, Tools, Memory, Structured Output, Middleware และ MCP Browser Tools. 

---

# 3. Agent แบบเรียบง่ายที่สุด

```python
agent = create_agent(
    model="openai:gpt-5.4-mini",
    system_prompt=(
        "You are a helpful assistant "
        "who answers concisely."
    ),
)

result = agent.invoke({
    "messages": [
        {
            "role": "user",
            "content": (
                "What is the Model Context Protocol, "
                "in two sentences?"
            ),
        }
    ]
})

print(result["messages"][-1].content)
```

Model สามารถส่งเป็น String รูปแบบ:

```text
provider:model
```

เช่น:

```text
openai:gpt-5.4-mini
```

`create_agent()` จะเตรียม Model และ Agent State ให้ จากนั้น `invoke()` รับ Input เป็น State Dictionary และคืน State หลัง Graph ทำงานเสร็จ ไม่ได้คืน String ตรง ๆ. 

ผลลัพธ์สุดท้ายจึงอ่านจาก:

```python
result["messages"][-1].content
```

---

## Agent แบบไม่มี Tools เป็น Agent หรือไม่

ในตัวอย่างแรกยังไม่มี Tool Loop ที่เกิดขึ้นจริง เพราะ Model มีเพียง Prompt และ Messages:

```text
START
  ↓
Model
  ↓
END
```

มันใช้ Agent State และ Agent Runtime เดียวกัน แต่พฤติกรรมใกล้กับ Chat Model มากกว่า Autonomous Tool-using Agent

เมื่อเพิ่ม Tools จึงเห็น Agent Loop ชัดเจน:

```text
Model
  ├── Final answer → END
  └── Tool calls → Tools → Model
```

---

# 4. `invoke()` และ `ainvoke()`

Synchronous:

```python
result = agent.invoke(input_state)
```

Asynchronous:

```python
result = await agent.ainvoke(input_state)
```

Input และ Output มีรูปแบบเดียวกัน ต่างกันที่ `ainvoke()` ไม่ Block Event Loop ระหว่างรอ Model หรือ External Tools จึงเหมาะกับ Applications ที่ต้องรับหลาย Requests หรือรอ I/O หลายรายการพร้อมกัน

Notebook เตรียม `ainvoke()` ไว้ตั้งแต่ต้น เพราะ MCP Browser Tools ในส่วนท้ายถูกโหลดและเรียกผ่าน Async APIs. 

Mental model:

```text
invoke
= โทรแล้วถือสายรอ

ainvoke
= โทรไว้ แล้วไปจัดการงานอื่นระหว่างรอ
```

Async ไม่ได้ทำให้ Model ตอบถูกขึ้น แต่ช่วยให้ Application ใช้เวลารอ Network ได้มีประสิทธิภาพขึ้น

---

# 5. เพิ่ม Tools

Notebook สร้าง Mock Tools:

```python
@tool
def get_weather(city: str) -> str:
    """Return today's weather for a city."""

    pretend = {
        "London": "rainy, 14 degrees",
        "Rome": "sunny, 27 degrees",
    }

    return pretend.get(
        city,
        "clear, 20 degrees",
    )


@tool
def get_population(city: str) -> str:
    """Return the population of a city."""

    pretend = {
        "London": "8.9 million",
        "Rome": "2.8 million",
    }

    return pretend.get(
        city,
        "unknown",
    )
```

จากนั้น:

```python
agent = create_agent(
    model="openai:gpt-5.4-mini",
    tools=[
        get_weather,
        get_population,
    ],
    system_prompt=(
        "You are a travel assistant. "
        "Use your tools to answer "
        "questions about cities."
    ),
)
```

เมื่อถามข้อมูลสภาพอากาศและประชากร Rome ตัว Agent จะจัดการวงจรทั้งหมด:

```text
User asks
    ↓
Model requests tools
    ↓
Framework executes tools
    ↓
ToolMessages added to State
    ↓
Model sees results
    ↓
Final response
```

นี่คือ Tool Loop ที่ Lab 1 เขียนเอง และ Lab 2 ประกอบด้วย Model Node, ToolNode และ Conditional Edge. 

---

# 6. สิ่งที่ `create_agent()` สร้างให้

ใน Lab 2 เราต้องเขียนประมาณนี้:

```python
builder.add_node("model", model_node)
builder.add_node("tools", ToolNode(tools))

builder.add_edge(START, "model")
builder.add_conditional_edges(
    "model",
    tools_condition,
)
builder.add_edge("tools", "model")

graph = builder.compile()
```

Lab 3 ลดเหลือ:

```python
agent = create_agent(
    model=model,
    tools=tools,
)
```

Conceptually:

```text
create_agent()
├── Agent State
├── Model Node
├── Tool Node
├── Conditional Router
├── Model–Tool Cycle
├── Message Reducer
└── Compiled Graph
```

Agent ที่คืนมาจึงใช้ Graph APIs ได้:

```python
agent.get_graph()
agent.invoke(...)
agent.ainvoke(...)
```

---

# 7. พิสูจน์ว่าเป็น LangGraph

```python
display(
    Image(
        agent.get_graph().draw_mermaid_png()
    )
)
```

Graph ที่มี Tools จะเห็นโครงสร้างหลัก:

```text
START
  ↓
Model
  ├── Tools → Model ↺
  └── END
```

นี่คือ Graph รูปแบบเดียวกับที่สร้างด้วยมือใน Lab 2 เพียงแต่ Framework ประกอบให้ผ่าน Function เดียว. 

ประโยชน์คือเราไม่ต้องเขียน Infrastructure Code ซ้ำ

Trade-off คือ Control Flow มาตรฐานบางส่วนถูกซ่อนไว้หลัง Abstraction

---

# 8. Memory ด้วย Checkpointer

โดย Default:

```python
agent.invoke(first_message)
agent.invoke(second_message)
```

เป็นคนละ State Thread หากไม่มี Checkpointer และ Config ที่เชื่อมกัน

Notebook เพิ่ม:

```python
memory_agent = create_agent(
    model="openai:gpt-5.4-mini",
    tools=[get_weather],
    checkpointer=MemorySaver(),
)
```

แล้วกำหนด:

```python
config = {
    "configurable": {
        "thread_id": "trip-planning"
    }
}
```

เรียกสองครั้งด้วย Config เดิม:

```python
memory_agent.invoke(
    {
        "messages": [
            {
                "role": "user",
                "content": (
                    "I am planning a trip "
                    "to London."
                ),
            }
        ]
    },
    config=config,
)

result = memory_agent.invoke(
    {
        "messages": [
            {
                "role": "user",
                "content": (
                    "What is the weather like "
                    "where I am going?"
                ),
            }
        ]
    },
    config=config,
)
```

Agent สามารถเชื่อม “where I am going” กับ London เพราะ Checkpointer โหลด Messages จาก Thread เดิม. 

---

## สิ่งที่เรียกว่า Memory ใน Lab นี้

```text
MemorySaver
+ thread_id
=
Thread-scoped conversation persistence
```

ไม่ใช่:

```text
Semantic long-term memory
Cross-user memory
Knowledge base
User profile shared across threads
```

และ `MemorySaver` เก็บข้อมูลใน RAM:

```text
Kernel restart
→ State หาย
```

จึงเหมาะกับ Notebook และ Development มากกว่า Durable Production Storage

---

# 9. Structured Output ระดับ Agent

Lab 1 ใช้:

```python
llm.with_structured_output(...)
```

Lab 3 ใช้ Structured Output กับ Agent ทั้งตัว:

```python
class CityReport(BaseModel):
    city: str = Field(
        description="The city name"
    )

    weather: str = Field(
        description=(
            "A short weather description"
        )
    )

    population: str = Field(
        description="The population"
    )
```

สร้าง Agent:

```python
report_agent = create_agent(
    model="openai:gpt-5.4-mini",
    tools=[
        get_weather,
        get_population,
    ],
    response_format=CityReport,
)
```

ใช้งาน:

```python
result = report_agent.invoke({
    "messages": [
        {
            "role": "user",
            "content": (
                "Give me a report on London."
            ),
        }
    ]
})

report = result["structured_response"]
```

Agent สามารถใช้ Tools หลายรอบก่อนสร้าง Object สุดท้าย และ Structured Object จะอยู่ใน Key:

```python
result["structured_response"]
```

LangChain เลือก Strategy สำหรับ Structured Output ตามความสามารถของ Model/Provider เช่น Provider-native Schema หรือ Tool-based Strategy. 

---

## Structured Output ระดับ Model กับ Agent

### Model level

```text
Prompt
→ Model
→ Structured Object
```

### Agent level

```text
Prompt
→ Model
→ Tools
→ Model
→ Structured Object
```

ดังนั้น Agent Structured Output เหมาะเมื่อข้อมูลสุดท้ายต้องผ่าน Retrieval หรือ Actions ก่อน

แต่ยังต้องจำ:

```text
Pydantic valid
≠ Factually correct
```

Schema ตรวจว่า `population` เป็น String แต่ไม่ตรวจว่าจำนวนประชากรถูกต้องหรือเป็นข้อมูลล่าสุด

---

# 10. Middleware คืออะไร

Middleware คือ Code ที่แทรกเข้าไปในจุดต่าง ๆ ของ Agent Execution โดยไม่ต้องแก้ Core Agent Loop

จุดที่แทรกได้ เช่น:

```text
ก่อน Agent เริ่ม
ก่อน Model Call
รอบ Model Call
หลัง Model ตอบ
รอบ Tool Call
หลัง Agent จบ
```

Official Middleware แบ่งเป็น Node-style Hooks ซึ่งรัน ณ จุดที่กำหนด และ Wrap-style Hooks ซึ่งครอบ Model หรือ Tool Call ทำให้สามารถรัน Logic ก่อนและหลัง Call รวมถึง Retry หรือ Short-circuit ได้. ([Docs by LangChain][1])

Mental model:

```text
Agent Loop
= สายพานหลัก

Middleware
= จุดตรวจที่ติดตั้งตามสายพาน
```

---

# 11. `wrap_tool_call`

Notebook สร้าง Middleware:

```python
@wrap_tool_call
def log_tool_calls(
    request,
    handler,
):
    call = request.tool_call

    print(
        f"[middleware] calling "
        f"{call['name']} "
        f"with {call['args']}"
    )

    return handler(request)
```

ส่วนสำคัญที่สุด:

```python
return handler(request)
```

`handler` คือขั้นตอนที่ปล่อยให้ Tool Call เดินต่อ

Flow:

```text
Tool call proposed
    ↓
Middleware receives request
    ↓
Log name and arguments
    ↓
handler(request)
    ↓
Tool executes
    ↓
Result returns through middleware
```

จากนั้นส่ง Middleware เข้า Agent:

```python
watched_agent = create_agent(
    model="openai:gpt-5.4-mini",
    tools=[
        get_weather,
        get_population,
    ],
    middleware=[
        log_tool_calls,
    ],
)
```

`wrap_tool_call` เหมาะกับ Logging, Error Handling, Retry, Argument Validation และ Tool Policies. 

---

## Middleware สามารถไม่เรียก `handler` ได้

ตัวอย่าง Conceptual Permission Gate:

```python
@wrap_tool_call
def protect_tools(
    request,
    handler,
):
    tool_name = request.tool_call["name"]

    if tool_name == "delete_database":
        raise PermissionError(
            "Tool is not allowed"
        )

    return handler(request)
```

หรือแก้ Argument ก่อน Execute:

```text
Original args
→ Middleware validates/transforms
→ Tool receives safe args
```

ดังนั้น Middleware ไม่ได้มีไว้เพียง Log แต่สามารถเปลี่ยนพฤติกรรม Agent Loop ได้จริง

---

# 12. Prebuilt Middleware

Notebook กล่าวถึง Middleware สำเร็จรูป เช่น:

```text
SummarizationMiddleware
→ ย่อ Conversation เมื่อ Context ยาว

PIIMiddleware
→ ตรวจหรือ Redact ข้อมูลส่วนบุคคล

Model/Tool Call Limits
→ จำกัด Cost และ Loop

Retries/Fallbacks
→ จัดการ Provider หรือ Tool Failures

HumanInTheLoopMiddleware
→ Pause ก่อน Tool สำคัญ
```

LangChain มี Middleware สำหรับ Summarization, Human Approval, Call Limits, Model Fallbacks และ Guardrails โดย Middleware ทั้งหมดทำงานภายใน Compiled LangGraph ที่ `create_agent()` สร้าง. ([Docs by LangChain][2])

---

# 13. Human-in-the-loop

สำหรับ Tool ที่มี Side Effect เช่น:

```text
ส่ง Email
เขียนไฟล์
แก้ Database
Execute SQL
ลบข้อมูล
ซื้อขายสินทรัพย์
```

Flow ที่ปลอดภัยกว่า:

```text
Model proposes tool
    ↓
HumanInTheLoopMiddleware
    ↓
Graph pauses and checkpoints
    ↓
Human:
Approve / Edit / Reject
    ↓
Graph resumes
```

Human-in-the-loop Middleware ใช้ LangGraph Interrupts และ Persistence จึงต้องมี Checkpointer เพื่อ Pause และ Resume อย่างทนทาน. ([Docs by LangChain][3])

นี่แสดงให้เห็นความสัมพันธ์:

```text
create_agent
→ LangGraph runtime
→ Checkpoints
→ Durable human approval
```

---

# 14. เตรียม Node และ Playwright

ส่วนท้ายของ Notebook ใช้ MCP Server ที่รันผ่าน Node.js จึงตรวจ:

```bash
node --version
npx --version
```

Notebook ระบุให้ใช้ Node 22 หรือใหม่กว่า และให้ Restart Application ทั้งหมดหลังติดตั้ง เพราะ Notebook Process เดิมอาจยังไม่เห็น PATH ใหม่. 

จากนั้น Smoke Test Playwright โดยไม่ใช้ AI:

```bash
npx -y playwright@latest screenshot \
  --channel=chrome \
  https://news.ycombinator.com \
  playwright_check.png
```

จุดประสงค์คือแยก Infrastructure Test ออกจาก Agent Test:

```text
Node works?
Playwright works?
Chrome works?
Screenshot works?
```

ต้องผ่านก่อนจึงเพิ่ม LLM เข้าไป

นี่เป็นแนวทาง Debug ที่ดี:

> ตรวจ Tool Chain แบบ Deterministic ก่อน แล้วจึงให้ Agent ใช้ Tool

---

# 15. Windows Notebook Adjustment

Notebook มี Workaround สำหรับปัญหา Stdio MCP Process ภายใน Jupyter บน Windows:

```python
if sys.platform == "win32":
    import subprocess
    from functools import partial
    import langchain_mcp_adapters.sessions as mcp_sessions

    mcp_sessions.stdio_client = partial(
        mcp_sessions.stdio_client,
        errlog=subprocess.DEVNULL,
    )
```

มันเปลี่ยนจุดหมาย Error Log ของ MCP Child Process เพื่อหลีกเลี่ยงปัญหา File Descriptor ของ Notebook. 

ข้อควรระวัง:

```text
stderr → DEVNULL
```

ทำให้ Error บางอย่างหายจากหน้าจอ จึงลด Observability

ควรใช้เฉพาะ Environment ที่มีปัญหานี้ และไม่ควร Copy Workaround ไปใช้ใน Python Scripts หรือ Production โดยไม่จำเป็น

---

# 16. MCP Client

สร้าง Client:

```python
client = MultiServerMCPClient({
    "playwright": {
        "transport": "stdio",
        "command": "npx",
        "args": [
            "-y",
            "@playwright/mcp@latest",
            "--isolated",
        ],
    }
})
```

จากนั้นโหลด Tools:

```python
browser_tools = await client.get_tools()
```

Architecture:

```text
Python Agent Process
        ↓ stdio
Node MCP Server Process
        ↓
Playwright
        ↓
Chrome Browser
```

MCP เป็นมาตรฐานสำหรับเชื่อม AI Applications กับ External Tools และ Data Sources โดย Tool Implementation สามารถอยู่ใน Process หรือ Technology Stack อื่นได้. 

จากมุมมอง Agent:

```text
Python @tool
MCP browser tool
```

กลายเป็น Tool Interface ที่ใช้งานใน List เดียวกันได้

---

# 17. Browser Agent

```python
browser_agent = create_agent(
    model="openai:gpt-5.5",
    tools=browser_tools,
    system_prompt=(
        "You are a web research assistant. "
        "Use the browser tools to complete "
        "the task, then report clearly."
    ),
)
```

เรียกแบบ Async:

```python
result = await browser_agent.ainvoke({
    "messages": [
        {
            "role": "user",
            "content": (
                "Go to Hacker News and tell me "
                "the titles of the top three "
                "stories on the front page."
            ),
        }
    ]
})
```

Agent สามารถเลือก Browser Tools เช่น Navigate, Inspect Page หรือ Read Elements แล้ววน Tool Loop จนได้ข้อมูลที่ต้องการ. 

---

# 18. Browser Tools เปลี่ยนระดับความเสี่ยง

Mock Weather Tool:

```text
Input
→ Dictionary lookup
→ Text output
```

Browser Tool:

```text
Input
→ เปิดเว็บจริง
→ โหลด Untrusted Content
→ Interact กับ Browser
→ อาจเปลี่ยน External State
```

ความเสี่ยงเพิ่มขึ้น:

```text
Indirect prompt injection
Malicious web content
Unintended clicks
Form submissions
Downloads
Credential exposure
Session leakage
```

`--isolated` ช่วยแยก Browser Session ของ MCP Run แต่ไม่ได้ทำให้เนื้อหาบนเว็บน่าเชื่อถือหรือ Tool Actions ปลอดภัยทั้งหมด

ระบบจริงควรเพิ่ม:

```text
Domain allowlist
No personal browser profile
No saved credentials
Download restrictions
Tool-call budgets
Read-only policy
Approval before click/submit
Audit logs
```

---

# 19. MCP Tools ไม่ใช่ MCP Memory

MCP หมายถึง Protocol สำหรับเชื่อม External Capabilities

```text
MCP Server
อาจให้:
Tools
Resources
Prompts
```

ไม่ได้หมายความว่า:

```text
MCP
= Agent Memory System
```

ใน Lab นี้ MCP Server ให้ Browser Tools ส่วน Conversation Memory ยังมาจาก LangGraph Checkpointer

---

# 20. `@latest` และ Reproducibility

Notebook ใช้:

```text
@playwright/mcp@latest
```

เหมาะกับการเรียนเพราะได้ Package ใหม่ แต่ผลการรันในอนาคตอาจเปลี่ยนเมื่อ Package อัปเดต

Production ควร Pin Version:

```text
@playwright/mcp@<tested-version>
```

และเก็บ:

```text
Node version
MCP package version
Browser version
LangChain version
Prompt version
```

เพื่อทำให้ Runs ทำซ้ำได้ใกล้เคียงเดิม

---

# 21. `create_agent()` ให้และไม่ให้อะไร

## สิ่งที่ให้

```text
Compiled LangGraph
Model–Tool Loop
Messages State
Tool Dispatch
Structured Response
Checkpointer Integration
Middleware Hooks
Async Invocation
Graph Visualization
```

## สิ่งที่ยังต้องออกแบบเอง

```text
Tool permissions
Prompt quality
Validation
Retry policies
Budgets
Authentication
Thread isolation
Side-effect approval
Source verification
Browser policies
Evaluation
```

Framework ให้ Runtime ที่ดี แต่ Application ยังถือ Authority และ Policy

---

# 22. เมื่อใดควรใช้ `create_agent()`

เหมาะเมื่อ Workflow หลักคือ:

```text
User request
→ Model
→ Tools as needed
→ Final response
```

และสิ่งที่ต้องปรับสามารถทำได้ผ่าน:

```text
System prompt
Tools
Middleware
Checkpointer
Structured response
```

ตัวอย่าง:

```text
Research assistant
Support agent
Database assistant
Travel assistant
Browser research agent
```

---

# 23. เมื่อใดควรสร้าง `StateGraph` เอง

ใช้ Custom Graph เมื่อมี:

```text
หลาย Stages ที่ต้องรันตามลำดับแน่นอน
หลาย LLM Roles
Deterministic Validation Nodes
Custom Branching
Parallel Workflows
Approval เฉพาะบางเส้นทาง
Multiple State Fields
Custom Retry/Recovery Paths
Subgraphs
```

ตัวอย่าง:

```text
Validate request
→ Research
→ Fact check
→ Draft
→ Legal approval
→ Publish
```

การบังคับ Workflow แบบนี้ผ่าน Agent Prompt อย่างเดียวจะควบคุมยากกว่า Graph ที่กำหนด Edges ชัดเจน

---

# 24. Mapping Lab 2 → Lab 3

| Lab 2 แบบ Manual Graph | Lab 3 `create_agent()` |
| ---------------------- | ---------------------- |
| StateGraph             | สร้างให้อัตโนมัติ      |
| Model Node             | Model parameter        |
| ToolNode               | Tools parameter        |
| tools_condition        | Agent routing ภายใน    |
| Edge Tools → Model     | Agent loop ภายใน       |
| compile(checkpointer)  | `checkpointer=`        |
| Custom wrappers        | Middleware             |
| Final State Field      | `structured_response`  |

แก่นสำคัญ:

```text
create_agent()
ไม่ได้ลบ LangGraph

มันห่อ LangGraph
ให้ใช้ง่ายขึ้น
```

---

# 25. Misconceptions ที่ต้องแก้

### “`create_agent()` เป็น Framework คนละตัวจาก LangGraph”

ไม่ใช่ มันคืน Compiled LangGraph Agent

### “เมื่อใช้ `create_agent()` เราไม่ต้องเข้าใจ State”

ไม่จริง ผลลัพธ์ยังเป็น Agent State และ Memory ยังพึ่ง Checkpointer กับ `thread_id`

### “ใส่ Tools แล้ว Agent จะใช้ทุก Tool”

ไม่เสมอ Model เป็นผู้ตัดสินว่าจะขอ Tool ใดและเมื่อใด

### “Structured Response ทำให้ข้อมูลถูกต้อง”

ไม่จริง มันทำให้ Output มี Schema ที่คาดเดาได้

### “Middleware เป็นเพียง Logging”

ไม่ใช่ สามารถ Block, Retry, Transform หรือเปลี่ยน Execution ได้

### “MCP เป็น Memory”

ไม่ใช่ เป็น Protocol เชื่อม External Systems

### “Browser Agent เป็นเพียง Search Agent”

ไม่เสมอ Browser Tools อาจมี Action Capabilities มากกว่า Retrieval

### “Async ทำให้ Model ฉลาดขึ้น”

ไม่ ทำให้ Application จัดการ I/O Concurrently ได้ดีขึ้น

---

# 26. แบบฝึกหัดที่แนะนำ

## Exercise 1 — Tool Error Middleware

สร้าง Middleware ที่เปลี่ยน Tool Exception เป็นข้อความที่ Model เข้าใจ:

```python
@wrap_tool_call
def handle_tool_errors(
    request,
    handler,
):
    try:
        return handler(request)
    except Exception as exc:
        # Return an appropriate ToolMessage
        ...
```

ตรวจว่า Agent สามารถแก้ Arguments แล้วลองใหม่หรือไม่

---

## Exercise 2 — Tool Call Limit

จำกัด Tool Calls ต่อ Run แล้วถามคำถามที่ต้องใช้หลาย Tools

ตรวจว่า Agent:

```text
หยุดอย่างไร
รายงานข้อจำกัดหรือไม่
เกิด Partial Answer หรือไม่
```

---

## Exercise 3 — Memory Isolation

สร้าง:

```text
thread-a
thread-b
```

ใส่เมืองปลายทางต่างกัน แล้วตรวจว่า Agent ไม่สลับข้อมูลกัน

---

## Exercise 4 — Structured Browser Result

สร้าง Pydantic Model:

```python
class Story(BaseModel):
    rank: int
    title: str


class HackerNewsReport(BaseModel):
    stories: list[Story]
```

แล้วให้ Browser Agent คืน `structured_response`

---

## Exercise 5 — Browser Read-only Policy

สร้าง Middleware ที่อนุญาตเฉพาะ Tools กลุ่ม:

```text
navigate
snapshot
read
```

และ Block Tools ที่:

```text
click
type
submit
upload
```

เพื่อเปรียบเทียบ Browser Research กับ Browser Automation

---

# Checklist ก่อนจบ Lab

### `create_agent()` คืนอะไร

Compiled LangGraph Agent

### Tool Loop ต้องเขียนเองหรือไม่

ไม่ สำหรับ Agent Pattern มาตรฐาน

### Agent Input และ Output เป็นอะไร

LangGraph State Dictionary ที่มี `messages` และ Fields อื่นตาม Configuration

### Memory เพิ่มอย่างไร

ใช้ Checkpointer และ `thread_id`

### Structured Object อยู่ที่ไหน

```python
result["structured_response"]
```

### Middleware ทำงานตรงไหน

ตาม Hooks ก่อน/หลังหรือรอบ Model และ Tool Calls

### `handler(request)` ทำอะไร

ปล่อยให้ Wrapped Model/Tool Call ดำเนินการต่อ

### MCP ทำอะไร

เชื่อม Agent กับ Tools หรือ Data จาก Server ภายนอกผ่าน Protocol มาตรฐาน

### ทำไม MCP Browser ใช้ `ainvoke()`

Tools ถูกโหลดและสื่อสารผ่าน Async External Process APIs ใน Lab นี้

### Lab 3 ต่างจาก Lab 2 อย่างไร

Lab 2 สร้าง Graph ด้วยมือ ส่วน Lab 3 ใช้ Prebuilt Agent Graph และปรับด้วย Middleware

---

# แก่นของ Week 4 Lab 3

```text
create_agent
= Prebuilt LangGraph Agent Loop

Model
= ผู้ตัดสินใจ

Tools
= ความสามารถภายนอก

Checkpointer
= Thread State Persistence

Structured Response
= Typed Final Contract

Middleware
= จุดควบคุมภายใน Loop

MCP
= มาตรฐานเชื่อม External Tools
```

บทเรียนสำคัญที่สุดคือ:

> **`create_agent()` ไม่ได้แทนที่ LangGraph แต่นำ Graph รูปแบบมาตรฐานที่เราสร้างใน Lab 2 มาประกอบให้พร้อมใช้ เราจึงเขียน Code น้อยลงโดยยังได้รับ State, Persistence และ Tool Loop จาก LangGraph Runtime**

และ:

> **ความสะดวกของ Agent Abstraction ไม่ได้ลดหน้าที่ด้าน Security หรือ Correctness เมื่อ Tools มีพลังมากขึ้น โดยเฉพาะ Browser และ Side-effect Tools เราต้องควบคุมผ่าน Middleware, Permissions, Checkpoints และ Human Approval ไม่ใช่ฝากทุกอย่างไว้กับ System Prompt**

[1]: https://docs.langchain.com/oss/python/langchain/middleware/custom?utm_source=chatgpt.com "Custom middleware - Docs by LangChain"
[2]: https://docs.langchain.com/oss/python/langchain/middleware/overview?utm_source=chatgpt.com "Overview - Docs by LangChain"
[3]: https://docs.langchain.com/oss/python/langchain/human-in-the-loop?utm_source=chatgpt.com "Human-in-the-loop - Docs by LangChain"
