# Episodic Learning Artifact

## Week 4 — Lab 3: LangChain `create_agent` Agent Layer

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**Notebook:** `4_langchain_langgraph/3_lab3.ipynb`
**หัวข้อหลัก:** `create_agent`, Prebuilt Agent Loop, Checkpoint Memory, Structured Response, Middleware และ MCP Browser Tools
**สถานะ:** เรียนและสรุปแนวคิดแล้ว

---

# 1. Context

Week 4 Lab 1 ศึกษา Building Blocks ระดับล่าง:

```text
Model
Messages
Tools
Tool Calls
Tool Results
Structured Output
```

เราต้องเขียน Tool Loop เอง:

```text
Model
→ Tool Request
→ Application Executes
→ ToolMessage
→ Model Again
```

Week 4 Lab 2 เปลี่ยน Manual Loop ให้เป็น LangGraph:

```text
State
Nodes
Edges
Conditional Routing
ToolNode
Checkpointer
```

Graph หลักมีลักษณะ:

```text
START
  ↓
Model
  ├── Tool Calls → Tools → Model
  └── Final Answer → END
```

Lab 3 ขยับขึ้นอีกหนึ่งระดับ โดยให้ Framework ประกอบ Graph มาตรฐานนี้ให้:

```text
Model
+ Tools
+ System Prompt
        ↓
create_agent()
        ↓
Compiled LangGraph Agent
```

Lab นี้จึงไม่ได้ทิ้ง LangGraph แต่ใช้ LangGraph ผ่าน Abstraction ที่สูงขึ้น

---

# 2. Learning Objectives

หลังจบ Lab 3 สามารถอธิบายได้ว่า:

1. `create_agent()` อยู่ระดับใดของ LangChain/LangGraph Abstraction
2. `create_agent()` สร้างส่วนใดของ Agent Loop ให้อัตโนมัติ
3. ทำไม Agent ที่สร้างขึ้นยังเป็น Compiled LangGraph
4. Agent ที่ไม่มี Tools แตกต่างจาก Tool-using Agent อย่างไร
5. `invoke()` และ `ainvoke()` ต่างกันอย่างไร
6. Tools ที่ส่งเข้า `create_agent()` ถูกใช้ใน Model–Tool Loop อย่างไร
7. Checkpointer และ `thread_id` เพิ่ม Conversation Continuity อย่างไร
8. Memory ใน Lab นี้แตกต่างจาก Long-term Semantic Memory อย่างไร
9. `response_format` ทำให้ Agent คืน Structured Response อย่างไร
10. Structured Output ระดับ Agent ต่างจาก Structured Output ระดับ Model อย่างไร
11. Middleware แทรก Logic เข้า Agent Runtime ได้ตรงไหน
12. `wrap_tool_call` ทำงานอย่างไร
13. Middleware สามารถ Log, Block, Retry หรือ Transform Tool Calls อย่างไร
14. Human-in-the-loop พึ่งพา LangGraph Persistence อย่างไร
15. MCP เชื่อม Python Agent กับ External Tool Process อย่างไร
16. Playwright MCP ทำให้ Agent ควบคุม Browser อย่างไร
17. Browser Tool มีความเสี่ยงมากกว่า Read-only Search Tool อย่างไร
18. เมื่อใดควรใช้ `create_agent()`
19. เมื่อใดควรกลับไปสร้าง `StateGraph` เอง

---

# 3. Week 4 Abstraction Progression

```text
Lab 1 — Building Blocks
Model, Messages, Tools, Structured Output

Lab 2 — Orchestration
State, Nodes, Edges, Reducers, Checkpoints

Lab 3 — Agent Abstraction
Prebuilt Model–Tool Graph

Lab ถัดไป — Agent Harness
Planning, Sub-agents, Filesystem และ Context Management
```

Mental Model:

```text
Lab 1
= มีชิ้นส่วนเครื่องยนต์

Lab 2
= ประกอบเครื่องยนต์และระบบควบคุมเอง

Lab 3
= ใช้เครื่องยนต์มาตรฐานที่ Framework ประกอบให้

Harness
= เพิ่มทีม แผน และพื้นที่ทำงานระดับสูง
```

---

# 4. Core Imports

```python
from dotenv import load_dotenv
from pydantic import BaseModel, Field

from langchain.agents import create_agent
from langchain.agents.middleware import wrap_tool_call
from langchain_core.tools import tool

from langgraph.checkpoint.memory import MemorySaver

from langchain_mcp_adapters.client import (
    MultiServerMCPClient,
)

load_dotenv(override=True)
```

หน้าที่:

```text
create_agent
→ สร้าง Agent Graph สำเร็จรูป

@tool
→ สร้าง Model-readable Tools

MemorySaver
→ เก็บ Thread State ใน RAM

response_format
→ กำหนด Final Output Contract

Middleware
→ แทรก Logic เข้า Agent Loop

MultiServerMCPClient
→ เชื่อม External MCP Servers
```

---

# 5. `create_agent()`

ตัวอย่างพื้นฐาน:

```python
agent = create_agent(
    model="openai:gpt-5.4-mini",
    system_prompt=(
        "You are a helpful assistant "
        "who answers concisely."
    ),
)
```

`create_agent()` รับองค์ประกอบหลัก:

```text
Model
System Prompt
Tools
Checkpointer
Response Format
Middleware
```

แล้วประกอบเป็น Agent Runtime

---

# 6. Model String

Model สามารถระบุในรูปแบบ:

```text
provider:model
```

ตัวอย่าง:

```python
model="openai:gpt-5.4-mini"
```

Conceptual Flow:

```text
Model string
→ Resolve provider integration
→ Initialize chat model
→ Attach to Agent Graph
```

ข้อดี:

```text
Configuration สั้น
เปลี่ยน Model ง่าย
ไม่ต้องสร้าง Model Object ทุกครั้ง
```

แต่หากต้องปรับค่าเฉพาะ เช่น Timeout, Temperature หรือ Provider Options อาจสร้าง Model Object เองแล้วส่งเข้า `create_agent()`

---

# 7. Invoking the Agent

```python
result = agent.invoke({
    "messages": [
        {
            "role": "user",
            "content": (
                "Explain the Model Context "
                "Protocol in two sentences."
            ),
        }
    ]
})
```

ผลลัพธ์เป็น Agent State Dictionary

คำตอบสุดท้ายอยู่ที่:

```python
result["messages"][-1].content
```

ดังนั้น:

```text
create_agent result
≠ Plain string

create_agent result
= Final graph state
```

---

# 8. Agent without Tools

Agent แรกไม่มี Tools:

```text
START
  ↓
Model
  ↓
END
```

พฤติกรรมใกล้กับ Chat Model Call แต่ยังอยู่ใน Agent Runtime เดียวกัน

ยังไม่มี:

```text
External action
Tool observation
Multi-round tool loop
```

ดังนั้น Agent ที่ไม่มี Tools ไม่ได้แสดงความเป็น Autonomous Agent มากนัก

---

# 9. Tool-using Agent

เมื่อเพิ่ม Tools:

```python
agent = create_agent(
    model="openai:gpt-5.4-mini",
    tools=[
        get_weather,
        get_population,
    ],
    system_prompt=(
        "You are a travel assistant. "
        "Use tools when needed."
    ),
)
```

Agent Runtime จะมี Flow:

```text
User Request
    ↓
Model
    ├── Final answer → END
    │
    └── Tool calls
            ↓
          Tools
            ↓
        ToolMessages
            ↓
          Model
            ↺
```

นี่คือ Agent Loop รูปแบบเดียวกับ Lab 2

---

# 10. What `create_agent()` Builds

Conceptually:

```text
create_agent()
├── Agent State
├── Message Reducer
├── Model Node
├── Tool Node
├── Tool-call Router
├── Model–Tool Cycle
├── Stop Condition
└── Compiled LangGraph
```

สิ่งที่ Lab 2 เขียนเอง:

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

---

# 11. `create_agent()` Does Not Replace LangGraph

Agent ที่สร้างขึ้นยังรองรับ:

```python
agent.invoke(...)
agent.ainvoke(...)
agent.get_graph()
agent.get_state(...)
agent.get_state_history(...)
```

เมื่อมี Checkpointer

จึงสรุปได้ว่า:

```text
create_agent
ไม่ได้เป็น Runtime ใหม่

create_agent
เป็น Prebuilt LangGraph Pattern
```

---

# 12. Graph Visualization

```python
agent.get_graph().draw_mermaid_png()
```

Graph หลักจะคล้าย:

```text
START
  ↓
Model
  ├── Tools → Model
  └── END
```

Visualization ช่วยยืนยันว่า Agent Abstraction ยังมี Nodes, Edges และ Cycle ภายใน

```text
Agent Abstraction
= Graph ที่ถูกซ่อนรายละเอียดบางส่วน
```

---

# 13. Creating Tools

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
```

และ:

```python
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

Tool Metadata มาจาก:

```text
Function name
Docstring
Type hints
```

เหมือนกับ Lab 1

---

# 14. Tool Loop Hidden by the Framework

เมื่อผู้ใช้ถาม:

```text
What is the weather and population of Rome?
```

Runtime อาจทำ:

```text
1. Model requests get_weather("Rome")
2. Model requests get_population("Rome")
3. Framework executes both tools
4. ToolMessages enter Agent State
5. Model produces final response
```

ผู้พัฒนาไม่ต้องเขียน:

```text
Tool registry
Tool dispatcher
ToolMessage construction
Outer while loop
```

---

# 15. Model Still Chooses the Tools

แม้ Framework Execute Tools ให้ แต่ Model ยังเป็นผู้ตัดสินว่า:

```text
จะใช้ Tool หรือไม่
ใช้ Tool ใด
ส่ง Arguments อะไร
เรียกกี่ครั้ง
```

ดังนั้น:

```text
create_agent
จัดการ Execution Loop

แต่ไม่ได้รับประกัน
Model Decision Quality
```

ยังต้องมี:

```text
Prompt design
Tool descriptions
Argument validation
Call limits
Permissions
Evaluation
```

---

# 16. `invoke()` vs `ainvoke()`

Synchronous:

```python
result = agent.invoke(input_state)
```

Asynchronous:

```python
result = await agent.ainvoke(
    input_state
)
```

## `invoke()`

```text
Block current execution
เหมาะกับ Script และ Notebook ขั้นพื้นฐาน
```

## `ainvoke()`

```text
ไม่ Block Event Loop ระหว่างรอ I/O
เหมาะกับ Web servers
MCP
Browser tools
Concurrent requests
```

Async ช่วยด้าน Throughput และ Responsiveness

แต่:

```text
Async
≠ Higher reasoning quality
```

---

# 17. Thread Memory with a Checkpointer

สร้าง Agent:

```python
memory_agent = create_agent(
    model="openai:gpt-5.4-mini",
    tools=[get_weather],
    checkpointer=MemorySaver(),
)
```

กำหนด Thread:

```python
config = {
    "configurable": {
        "thread_id": "trip-planning"
    }
}
```

Run แรก:

```text
I am planning a trip to London.
```

Run ถัดไป:

```text
What is the weather like where I am going?
```

เมื่อใช้ `thread_id` เดิม Agent สามารถอ้างถึง London จาก State ก่อนหน้าได้

---

# 18. Memory in This Lab

Memory ใน Lab นี้คือ:

```text
Checkpointer
+ thread_id
=
Thread-scoped state persistence
```

เหมาะกับ:

```text
Conversation history
Previous tool results
Current task context
Workflow continuity
```

ไม่ใช่:

```text
Semantic memory
Cross-thread memory
User profile
Knowledge base
Shared organizational memory
```

---

# 19. `MemorySaver` Limitation

`MemorySaver` เก็บข้อมูลใน RAM

```text
Process stops
→ State disappears

Notebook kernel restarts
→ State disappears
```

เหมาะกับ:

```text
Learning
Testing
Development
Short-lived demos
```

Production ควรใช้ Durable Checkpointer ตาม Infrastructure ที่เหมาะสม

---

# 20. Thread Isolation

```text
thread-a
→ Conversation A

thread-b
→ Conversation B
```

หากใช้ Thread ID เดียวกันกับหลาย Users:

```text
Messages may mix
Tool results may leak
Privacy risk
Incorrect context
```

ดังนั้น Thread ID ต้องผูกกับ:

```text
Authenticated user
Conversation ID
Tenant
Session
```

และต้องมี Authorization ก่อนโหลด Thread State

---

# 21. Structured Response at Agent Level

กำหนด Schema:

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

เรียก:

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
```

Structured Result อยู่ที่:

```python
result["structured_response"]
```

---

# 22. Model-level vs Agent-level Structured Output

## Model-level

```text
Prompt
→ Model
→ Structured Object
```

ใช้ใน Lab 1:

```python
llm.with_structured_output(
    Schema
)
```

## Agent-level

```text
Prompt
→ Model
→ Tools
→ Model
→ Structured Object
```

ใช้ใน Lab 3:

```python
create_agent(
    response_format=Schema
)
```

Agent สามารถรวบรวมข้อมูลผ่าน Tools ก่อนสร้าง Final Structured Response

---

# 23. Structured Response Benefits

```text
Predictable final fields
Typed Python object
Simpler API integration
Less text parsing
Clear downstream contract
```

เหมาะกับ:

```text
API responses
Database records
Workflow routing
Reports
Evaluation outputs
```

---

# 24. Structured Response Is Not Truth

Pydantic ตรวจได้ว่า:

```text
city เป็น str
weather เป็น str
population เป็น str
```

แต่ไม่ตรวจว่า:

```text
Weather เป็นข้อมูลจริง
Population เป็นข้อมูลล่าสุด
City ถูกระบุถูกต้อง
Tool data มีคุณภาพ
```

ดังนั้น:

```text
Structured
≠ Verified
```

ต้องเพิ่ม:

```text
Source validation
Recency checks
Business rules
Cross-source verification
Human review
```

---

# 25. Middleware

Middleware คือ Logic ที่แทรกเข้า Agent Execution โดยไม่แก้ Core Loop

สามารถแทรก:

```text
ก่อน Agent Run
ก่อน Model Call
รอบ Model Call
หลัง Model Call
ก่อน Tool Call
รอบ Tool Call
หลัง Tool Call
หลัง Agent Run
```

Mental Model:

```text
Agent Runtime
= สายพานการผลิต

Middleware
= จุดตรวจหรือเครื่องควบคุมบนสายพาน
```

---

# 26. Why Middleware Matters

หากไม่มี Middleware อาจต้องแก้:

```text
Model wrapper
Tool functions
Graph nodes
Agent core logic
```

Middleware ช่วยแยก Cross-cutting Concerns เช่น:

```text
Logging
Tracing
Retries
Rate limits
Permissions
Redaction
Approval
Budgets
Error handling
```

ออกจาก Business Logic

---

# 27. `wrap_tool_call`

ตัวอย่าง:

```python
@wrap_tool_call
def log_tool_calls(
    request,
    handler,
):
    call = request.tool_call

    print(
        f"Calling {call['name']} "
        f"with {call['args']}"
    )

    return handler(request)
```

Flow:

```text
Model proposes Tool Call
        ↓
Middleware receives request
        ↓
Middleware logs or validates
        ↓
handler(request)
        ↓
Tool executes
        ↓
Tool result returns
```

---

# 28. `handler(request)`

```python
return handler(request)
```

หมายถึง:

```text
ส่ง Request ต่อไปยัง Tool Runtime
```

หาก Middleware ไม่เรียก Handler:

```text
Tool will not execute
```

ดังนั้น Middleware สามารถ:

```text
Allow
Reject
Modify
Retry
Short-circuit
```

Tool Call ได้

---

# 29. Logging Middleware

ประโยชน์:

```text
เห็น Tool name
เห็น Arguments
ตรวจลำดับ Tool Calls
วิเคราะห์ Agent behavior
Debug Tool selection
```

แต่ต้องระวัง Logging:

```text
Passwords
Tokens
Personal data
Sensitive query parameters
```

Production Logging ต้องมี Redaction

---

# 30. Permission Middleware

ตัวอย่างเชิงแนวคิด:

```python
@wrap_tool_call
def protect_tools(
    request,
    handler,
):
    tool_name = request.tool_call[
        "name"
    ]

    if tool_name == "delete_database":
        raise PermissionError(
            "Tool is not allowed."
        )

    return handler(request)
```

หลัก:

```text
Tool available to Model
ไม่ได้หมายความว่า
Tool must always be authorized
```

---

# 31. Argument Validation Middleware

```text
Model-generated arguments
        ↓
Middleware validation
        ↓
Tool execution
```

ควรตรวจ:

```text
Type
Allowed values
Path
Domain
Amount
Recipient
Data sensitivity
```

Schema Validation อย่างเดียวอาจตรวจเพียงรูปแบบ ไม่ได้ตรวจ Business Constraints

---

# 32. Error-handling Middleware

Conceptual:

```python
@wrap_tool_call
def handle_tool_errors(
    request,
    handler,
):
    try:
        return handler(request)
    except Exception as exc:
        ...
```

สามารถ:

```text
Convert exception to ToolMessage
Retry transient failures
Fallback to another tool
Stop unsafe execution
Log diagnostic metadata
```

---

# 33. Retry Risk

การ Retry Tool ต้องดูว่า Tool เป็น:

```text
Read-only
หรือ
Side-effecting
```

Read-only Tool:

```text
Search
Read file
Get weather
```

Retry มักปลอดภัยกว่า

Side-effect Tool:

```text
Send email
Submit form
Charge payment
Delete data
```

Retry อาจทำ Action ซ้ำ

ต้องมี Idempotency

---

# 34. Prebuilt Middleware Categories

ตัวอย่างแนวคิด:

```text
Summarization
→ ลด Context เมื่อ Messages ยาว

PII Handling
→ ตรวจหรือ Redact ข้อมูลส่วนบุคคล

Call Limits
→ จำกัด Model/Tool Calls

Fallbacks
→ เปลี่ยน Model เมื่อ Provider ล้มเหลว

Human-in-the-loop
→ Pause ก่อน Action สำคัญ
```

Middleware ทำให้ `create_agent()` ยังปรับแต่งได้ แม้ Core Graph ถูกประกอบอัตโนมัติ

---

# 35. Human-in-the-loop

Flow:

```text
Model proposes action
        ↓
Human approval middleware
        ↓
Graph interrupts
        ↓
Checkpoint saved
        ↓
Human reviews
        ├── Approve
        ├── Edit
        └── Reject
        ↓
Graph resumes
```

เหมาะกับ:

```text
Sending email
Financial transactions
Database mutations
External publication
Deleting data
Browser form submission
```

---

# 36. HITL Requires Persistence

หาก Graph Pause เพื่อรอ Human แต่ไม่มี Checkpointer:

```text
Process restart
→ Pending state may be lost
```

จึงต้องใช้:

```text
Checkpointer
thread_id
interrupt/resume mechanism
```

นี่เป็นเหตุผลที่ Agent Layer ยังพึ่ง LangGraph Runtime

---

# 37. MCP

MCP ย่อมาจาก:

```text
Model Context Protocol
```

ทำหน้าที่เป็นมาตรฐานเชื่อม AI Application กับ:

```text
External tools
Resources
Data sources
Prompts
Services
```

MCP ไม่ใช่ Agent Framework และไม่ใช่ Memory โดยอัตโนมัติ

---

# 38. MCP Architecture in the Lab

```text
Python Agent
    ↓ stdio
MCP Client Adapter
    ↓
Node.js MCP Server
    ↓
Playwright
    ↓
Chrome Browser
```

Tool Implementation ไม่จำเป็นต้องอยู่ใน Python Process เดียวกับ Agent

นี่คือคุณค่าของ Protocol Boundary

---

# 39. `MultiServerMCPClient`

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

จากนั้น:

```python
browser_tools = await client.get_tools()
```

MCP Client:

```text
Starts server process
Discovers tool definitions
Converts MCP tools
Makes them usable by LangChain Agent
```

---

# 40. External Process Boundary

MCP Server รันเป็น Process แยก:

```text
Python process
≠ Node MCP process
≠ Browser process
```

ข้อดี:

```text
Language independence
Tool isolation
Reusable integrations
Standard discovery
Clear process boundaries
```

ข้อเสีย:

```text
More infrastructure
Process startup failures
Transport errors
Version mismatch
Harder debugging
```

---

# 41. Infrastructure Smoke Test

ก่อนให้ Agent ใช้ Playwright ควรตรวจ:

```bash
node --version
npx --version
```

แล้วทดสอบ Browser แบบ Deterministic:

```bash
npx -y playwright@latest screenshot \
  --channel=chrome \
  https://news.ycombinator.com \
  playwright_check.png
```

หลักสำคัญ:

```text
Test infrastructure without AI first
```

ช่วยแยกว่า Error มาจาก:

```text
Node
Playwright
Browser
MCP
Agent
Model
```

---

# 42. Browser Agent

```python
browser_agent = create_agent(
    model="openai:gpt-5.5",
    tools=browser_tools,
    system_prompt=(
        "You are a web research assistant. "
        "Use browser tools and report clearly."
    ),
)
```

เรียก:

```python
result = await browser_agent.ainvoke({
    "messages": [
        {
            "role": "user",
            "content": (
                "Open Hacker News and report "
                "the top three story titles."
            ),
        }
    ]
})
```

Flow:

```text
User task
→ Model plans browser actions
→ MCP browser tool
→ Browser state changes
→ Page observations
→ Model decides next action
→ Final answer
```

---

# 43. Browser Agent vs Search Agent

## Search Agent

```text
Query
→ Search results
→ Read text
```

## Browser Agent

```text
Navigate
Click
Read DOM
Scroll
Fill forms
Possibly submit actions
```

Browser Agent มี Action Surface กว้างกว่า

จึงมี Security Risk มากกว่า

---

# 44. Browser Tool Risks

```text
Indirect prompt injection
Malicious web instructions
Unexpected navigation
Unintended clicks
Form submission
File downloads
Credential exposure
Session leakage
Excessive browsing loop
```

Web content ต้องถือเป็น:

```text
Untrusted data
```

ไม่ใช่ System Instruction

---

# 45. Browser Safety Controls

ควรเพิ่ม:

```text
Domain allowlist
Navigation restrictions
Read-only tool subset
No saved browser profile
No personal credentials
Download restrictions
Click approval
Submit approval
Tool-call limits
Session isolation
Audit logs
```

---

# 46. `--isolated`

Playwright MCP ใช้:

```text
--isolated
```

ช่วยแยก Browser Session ของ Run

ลดการใช้:

```text
Existing cookies
Saved sessions
Personal profile
Previous browsing state
```

แต่ไม่ได้ป้องกัน:

```text
Prompt injection
Malicious web content
Unsafe clicks
Data sent through forms
```

---

# 47. Read-only Browser Policy

สำหรับ Research Agent ควรอนุญาตเฉพาะ:

```text
Navigate
Snapshot
Read page
Inspect text
```

และ Block:

```text
Click dangerous controls
Type credentials
Submit forms
Upload files
Download executable files
```

Tool Least Privilege สำคัญกว่า System Prompt

---

# 48. MCP Is Not Memory

```text
MCP
= External capability protocol

Checkpointer
= Thread state persistence

Store
= Long-term cross-thread memory
```

ไม่ควรสับสนสามแนวคิดนี้

---

# 49. Windows Notebook Workaround

Notebook อาจปรับ Stdio MCP Client บน Windows:

```python
if sys.platform == "win32":
    ...
```

เพื่อหลีกเลี่ยงปัญหา File Descriptor หรือ Error Stream ใน Jupyter

ข้อเสียของการส่ง `stderr` ไป `DEVNULL`:

```text
Error details disappear
Debugging becomes harder
Observability decreases
```

ควรใช้เฉพาะเมื่อจำเป็น

---

# 50. Version Pinning

Notebook ใช้:

```text
@playwright/mcp@latest
```

ข้อดี:

```text
ได้ Version ใหม่
สะดวกสำหรับการเรียน
```

ความเสี่ยง:

```text
Tool names เปลี่ยน
Behavior เปลี่ยน
Arguments เปลี่ยน
Run เดิมทำซ้ำไม่ได้
```

Production ควร Pin:

```text
MCP package version
Node version
Browser version
LangChain version
Prompt version
```

---

# 51. `create_agent()` Gives Us

```text
Compiled LangGraph
Agent message state
Model–tool loop
Tool dispatch
Conditional routing
Checkpointer integration
Middleware hooks
Structured response support
Async APIs
Graph visualization
```

---

# 52. `create_agent()` Does Not Automatically Give Us

```text
Correct tool choice
Safe permissions
Factual verification
Authentication
Thread authorization
Cost control
Retry policy
Idempotency
Human approval
Prompt-injection protection
Reliable browser behavior
```

Framework ให้ Runtime

Application ต้องกำหนด Policy

---

# 53. When to Use `create_agent()`

เหมาะเมื่อ Workflow หลักเป็น:

```text
User request
→ Model
→ Tools when needed
→ Final answer
```

และสามารถปรับผ่าน:

```text
System prompt
Tools
Middleware
Checkpointer
Response schema
```

ตัวอย่าง:

```text
Travel assistant
Customer support
Research assistant
Database Q&A
Browser research
Internal knowledge assistant
```

---

# 54. When to Build a Custom `StateGraph`

ใช้ Custom Graph เมื่อ Workflow มี:

```text
Fixed multi-stage sequence
Several LLM roles
Deterministic validation nodes
Custom branches
Parallel fan-out/fan-in
Multiple approval points
Complex state fields
Custom recovery paths
Subgraphs
```

ตัวอย่าง:

```text
Validate request
→ Research
→ Evidence review
→ Draft
→ Compliance check
→ Human approval
→ Publish
```

Graph ชัดกว่าการหวังให้ Model จัดการทุก Stage จาก Prompt เดียว

---

# 55. `create_agent()` vs Custom Graph

## `create_agent()`

ข้อดี:

```text
Code น้อย
เริ่มเร็ว
Agent loop มาตรฐาน
Middleware พร้อมใช้
```

ข้อจำกัด:

```text
Control flow บางส่วนถูกซ่อน
Custom stages ซับซ้อนขึ้น
อาจต้องพึ่ง Middleware มากเกินไป
```

## Custom `StateGraph`

ข้อดี:

```text
เส้นทางชัด
ควบคุม State ละเอียด
เพิ่ม Deterministic Gates ง่าย
เหมาะกับ Workflow ซับซ้อน
```

ข้อจำกัด:

```text
Code มากกว่า
ต้องจัดการ Graph เอง
ต้องเข้าใจ Runtime ลึกกว่า
```

---

# 56. Mapping Lab 2 to Lab 3

```text
Lab 2 StateGraph
→ create_agent internal graph

Model Node
→ model parameter

ToolNode
→ tools parameter

tools_condition
→ internal agent routing

Tools → Model edge
→ internal agent loop

compile(checkpointer=...)
→ checkpointer parameter

Custom control logic
→ middleware

Final structured field
→ structured_response
```

---

# 57. Middleware vs Graph Node

## Middleware

เหมาะกับ Cross-cutting Logic:

```text
Logging
Retry
Permission
Redaction
Call limits
```

## Graph Node

เหมาะกับ Business Stage:

```text
Research
Validate
Approve
Translate
Publish
```

หลัก:

```text
Middleware
= รอบการทำงาน

Node
= ขั้นตอนของงาน
```

ไม่ควรยัด Workflow หลาย Stage ลง Middleware ทั้งหมด

---

# 58. Common Misconceptions

## ความเข้าใจคลาดเคลื่อนที่ 1

> `create_agent()` ไม่เกี่ยวกับ LangGraph

**ข้อเท็จจริง:**
มันสร้าง Compiled LangGraph Agent

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> ใช้ `create_agent()` แล้วไม่ต้องเข้าใจ State

**ข้อเท็จจริง:**
Input, Output, Checkpoints และ Memory ยังพึ่ง State

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> Agent ที่มี Tools จะใช้ทุก Tool

**ข้อเท็จจริง:**
Model เป็นผู้เลือกว่าจะขอ Tool ใด

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> Checkpointer คือ Long-term Memory

**ข้อเท็จจริง:**
เป็น Thread-scoped State Persistence

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> Structured Response รับประกันข้อมูลถูกต้อง

**ข้อเท็จจริง:**
รับประกัน Schema เป็นหลัก

---

## ความเข้าใจคลาดเคลื่อนที่ 6

> Middleware ใช้แค่ Logging

**ข้อเท็จจริง:**
สามารถ Block, Retry, Transform และ Short-circuit ได้

---

## ความเข้าใจคลาดเคลื่อนที่ 7

> MCP คือ Memory

**ข้อเท็จจริง:**
MCP เป็น Protocol เชื่อม External Capabilities

---

## ความเข้าใจคลาดเคลื่อนที่ 8

> Browser Agent คือ Search Agent ที่เก่งขึ้น

**ข้อเท็จจริง:**
Browser Agent มี Action Surface และความเสี่ยงมากกว่า

---

## ความเข้าใจคลาดเคลื่อนที่ 9

> Async ทำให้ Agent ฉลาดขึ้น

**ข้อเท็จจริง:**
Async ช่วยจัดการ I/O และ Concurrency

---

# 59. Risks Identified

## 59.1 Tool Selection Error

Model เลือก Tool ไม่เหมาะสม

## 59.2 Invalid Arguments

Arguments ผ่าน Schema แต่ผิด Business Rule

## 59.3 Infinite Tool Loop

Agent เรียก Tools ซ้ำหลายรอบ

## 59.4 Thread Leakage

หลาย Users ใช้ Thread ID เดียวกัน

## 59.5 Structured Hallucination

Output ตรง Schema แต่ข้อมูลผิด

## 59.6 Sensitive Logging

Middleware Log Credentials หรือ PII

## 59.7 Retry Side Effects

Action ถูกทำซ้ำ

## 59.8 MCP Process Failure

External Server เริ่มไม่ได้หรือ Transport ขาด

## 59.9 Browser Prompt Injection

เว็บชักนำ Agent ให้ทำ Action อันตราย

## 59.10 Version Drift

MCP หรือ Browser Tool เปลี่ยน Behavior

---

# 60. Production Improvements

```text
Persistent production checkpointer
Authenticated thread ownership
Tool-call limits
Model-call limits
Timeouts
Retries with idempotency
Argument validation
Tool allowlists
Middleware redaction
Human approval
Structured error results
Source verification
Browser domain policy
Read-only browser mode
Pinned MCP versions
Audit tracing
Evaluation tests
```

---

# 61. Suggested Exercise — Tool Logging

ใช้ `wrap_tool_call` บันทึก:

```text
Tool name
Arguments
Start time
End time
Success/failure
```

แล้วตรวจว่า Agent ใช้ Tools ตามที่คาดไว้หรือไม่

---

# 62. Suggested Exercise — Tool Error Recovery

ทำให้ `get_weather()` Raise Exception สำหรับเมืองหนึ่ง

ตรวจว่า Agent:

```text
หยุด
ลองใหม่
เปลี่ยน Tool
หรือแจ้งข้อจำกัด
```

เพิ่ม Middleware เพื่อคืน Error Message ที่ Model เข้าใจได้

---

# 63. Suggested Exercise — Tool-call Budget

จำกัด Tools เช่น:

```text
Maximum 3 tool calls per run
```

ถามคำถามที่ต้องใช้หลาย Tools แล้วดูว่า Agent จัดการ Partial Result อย่างไร

---

# 64. Suggested Exercise — Memory Isolation

สร้าง:

```text
thread-london
thread-rome
```

ให้แต่ละ Thread จำเมืองต่างกัน

จากนั้นถาม:

```text
What is the weather where I am going?
```

ตรวจว่า Context ไม่ปะปน

---

# 65. Suggested Exercise — Structured Agent Result

สร้าง:

```python
class TravelReport(BaseModel):
    city: str
    weather: str
    population: str
    recommendation: str
```

ให้ Agent ใช้ Tools แล้วคืน `structured_response`

ตรวจทั้ง:

```text
Schema
Source quality
Consistency
```

---

# 66. Suggested Exercise — Read-only Browser Middleware

อนุญาตเฉพาะ Tool Names ที่เกี่ยวกับ:

```text
Navigate
Snapshot
Read
Inspect
```

Block Tool ที่เกี่ยวกับ:

```text
Click
Type
Submit
Upload
Download
```

เพื่อสร้าง Research Agent ที่มี Least Privilege

---

# 67. Suggested Exercise — Human Approval

เพิ่ม Approval ก่อน Browser Tool ที่:

```text
Submit form
Create account
Send message
Purchase
```

ทดสอบ:

```text
Approve
Edit
Reject
Resume
```

---

# 68. Patterns Learned

## Prebuilt Agent Pattern

```text
Model
+ Tools
→ create_agent
→ Tool-using graph
```

## Agent State Pattern

```text
Messages
Tool results
Structured response
→ Shared graph state
```

## Thread Memory Pattern

```text
Checkpointer
+ thread_id
→ Conversation continuity
```

## Structured Agent Output

```text
Tool loop
→ Final typed object
```

## Middleware Pattern

```text
Request
→ Policy layer
→ Model/Tool
```

## MCP Adapter Pattern

```text
External process tools
→ Protocol
→ LangChain Agent
```

## Browser Agent Pattern

```text
Model decision
→ Browser action
→ Page observation
→ Next decision
```

---

# 69. Connection to Week 4 Lab 1

Lab 1:

```text
Manual tool binding
Manual dispatch
Manual ToolMessage
Manual loop
```

Lab 3:

```text
Tools parameter
Automatic dispatch
Automatic ToolMessages
Prebuilt agent loop
```

พื้นฐานยังเหมือนเดิม:

```text
Model proposes
Application executes
Tool returns observation
```

---

# 70. Connection to Week 4 Lab 2

Lab 2:

```text
StateGraph
Model Node
ToolNode
Conditional Edges
Checkpointer
```

Lab 3:

```text
create_agent
```

ห่อ Pattern เหล่านั้นเป็น Agent Abstraction

ความเข้าใจ Lab 2 ช่วยให้รู้ว่า:

```text
create_agent
ซ่อนอะไร
และ
ควบคุมอะไรได้
```

---

# 71. Connection to Week 3

CrewAI แบ่งระบบเป็น:

```text
Agents
Tasks
Crews
Processes
```

`create_agent()` เน้น Agent Runtime หนึ่งตัว:

```text
Model
Tools
Messages
Middleware
Memory
```

หากต้องการทีมหลาย Agents อาจประกอบหลาย `create_agent()` Instances ภายใน Graph หรือ Harness ระดับสูงกว่า

---

# 72. Lab 3 Mental Model

```text
User
  ↓
Agent State
  ↓
Model
  ├── Final Response
  │       ↓
  │ Structured Response
  │       ↓
  │      END
  │
  └── Tool Calls
          ↓
      Middleware
          ↓
        Tools
          ↓
     ToolMessages
          ↓
        Model
          ↺
```

Persistence:

```text
Checkpointer
+ thread_id
→ Resume conversation state
```

External Tools:

```text
MCP Client
→ External MCP Server
→ Browser or service
```

---

# 73. Final Lessons

## Lesson 1

`create_agent()` เป็น Prebuilt LangGraph Agent Pattern

## Lesson 2

Agent Abstraction ลด Infrastructure Code แต่ไม่ได้ลบ State หรือ Graph Runtime

## Lesson 3

Model ยังคงเป็นผู้เสนอ Tool Calls ส่วน Runtime เป็นผู้ Execute

## Lesson 4

Checkpointer และ `thread_id` ให้ Thread-scoped Conversation Continuity

## Lesson 5

MemorySaver เหมาะกับ Development ไม่ใช่ Durable Production Memory

## Lesson 6

Agent-level Structured Output สามารถเกิดหลัง Tool Loop หลายรอบ

## Lesson 7

Schema Validation ไม่เท่ากับ Factual Validation

## Lesson 8

Middleware เหมาะกับ Cross-cutting Policies ภายใน Agent Loop

## Lesson 9

`handler(request)` เป็นจุดที่ปล่อยให้ Model หรือ Tool Call เดินต่อ

## Lesson 10

Human Approval พึ่งพา Persistence เพื่อ Pause และ Resume อย่างปลอดภัย

## Lesson 11

MCP ทำให้ Tool Implementation อยู่ต่าง Process หรือภาษาได้

## Lesson 12

Browser Tools เพิ่มทั้ง Capability และ Security Risk

## Lesson 13

Infrastructure ควรถูกทดสอบแบบ Deterministic ก่อนเพิ่ม LLM

## Lesson 14

ใช้ `create_agent()` เมื่อ Workflow เป็น Model–Tool Loop มาตรฐาน

## Lesson 15

ใช้ Custom `StateGraph` เมื่อ Workflow มี Stages, Branches และ Validation Gates เฉพาะทาง

---

# 74. Memory Summary

```text
Week 4 Lab 3 ใช้:
langchain.agents.create_agent

Notebook:
4_langchain_langgraph/3_lab3.ipynb

create_agent:
สร้าง Prebuilt LangGraph Agent

ภายในมี:
Agent State
Model Node
Tool Node
Conditional routing
Model–Tool loop
Message handling

Agent result:
เป็น State Dictionary

Final text:
result["messages"][-1].content

Agent without tools:
ใกล้กับ Chat Model

Agent with tools:
Model
→ Tools
→ Model
→ Final response

invoke:
Synchronous

ainvoke:
Asynchronous
เหมาะกับ MCP และ I/O

Memory:
ใช้ Checkpointer

MemorySaver:
เก็บใน RAM

thread_id:
แยก Conversation State

Memory ใน Lab นี้:
Thread-scoped persistence

ไม่ใช่:
Long-term semantic memory

Structured agent output:
response_format=PydanticModel

Result:
result["structured_response"]

Structured:
รับประกัน Schema

ไม่รับประกัน:
Truth
Recency
Source quality

Middleware:
แทรก Logic รอบ Agent Loop

wrap_tool_call:
รับ request และ handler

handler(request):
ปล่อย Tool Call ให้ Execute

Middleware ใช้กับ:
Logging
Validation
Permissions
Retries
Redaction
Approval
Limits

Human-in-the-loop:
Pause
Checkpoint
Approve/Edit/Reject
Resume

MCP:
Protocol เชื่อม External Tools

Lab ใช้:
MultiServerMCPClient
Playwright MCP
Node.js
Chrome

Browser agent:
Navigate
Observe
Act
Repeat

Browser risks:
Prompt injection
Unsafe clicks
Form submission
Credential exposure
Session leakage

ควรเพิ่ม:
Read-only policy
Domain allowlist
Tool limits
Human approval
Version pinning
Audit logs

create_agent เหมาะกับ:
Standard Model–Tool Loop

Custom StateGraph เหมาะกับ:
Multiple stages
Custom routing
Validation gates
Parallel work
Complex state
```

---

# 75. Next Episode

Lab ถัดไปจะขยับจาก Agent Runtime มาตรฐานไปสู่ Harness ที่มีความสามารถมากขึ้น เช่น:

```text
Planning
Task decomposition
Filesystem
Sub-agents
Context management
Long-running workflows
```

คำถามสำคัญก่อนเข้าสู่ Lab ถัดไปคือ:

> เมื่อ Agent ต้องทำงานที่ยาวและซับซ้อนเกินกว่าจะเก็บทุกอย่างใน Message History เราจะเพิ่ม Planning, External Filesystem และ Sub-agents อย่างไร โดยไม่ปล่อยให้ Context, Cost และสิทธิ์การใช้ Tools ขยายตัวจนควบคุมไม่ได้?
