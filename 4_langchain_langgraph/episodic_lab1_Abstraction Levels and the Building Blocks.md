# Episodic Learning Artifact

## Week 4 — Lab 1: LangChain Building Blocks

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**Notebook:** `4_langchain_langgraph/1_lab1.ipynb`
**หัวข้อหลัก:** Abstraction Levels, Chat Models, Messages, Tools, Manual Tool Loop และ Structured Output
**สถานะ:** เรียนและสรุปแนวคิดแล้ว

---

# 1. Context

Week 3 ใช้ CrewAI เพื่ออธิบาย Agentic System ผ่านแนวคิด:

```text
Agent
Task
Crew
Process
Tool
Context
Memory
Artifact
```

CrewAI ซ่อนรายละเอียดบางส่วนของ Model และ Tool Loop เอาไว้หลัง Framework

Week 4 เริ่มต้นด้วยการลดระดับ Abstraction ลงมา เพื่อดูส่วนประกอบพื้นฐานของ Agent โดยตรง:

```text
Model
→ Message
→ Tool request
→ Tool execution
→ Tool result
→ Model response
```

Lab 1 ยังไม่ได้สร้าง LangGraph

แต่กำลังศึกษา **Building Blocks Layer** ที่ LangGraph และ Agent Framework ชั้นสูงใช้เป็นพื้นฐาน

---

# 2. Learning Objectives

หลังจบ Lab 1 สามารถอธิบายได้ว่า:

1. LangChain และ LangGraph อยู่คนละระดับของ Abstraction อย่างไร
2. `ChatOpenAI` เป็น Model Adapter ไม่ใช่ Agent อย่างไร
3. `invoke()` และ `stream()` แตกต่างกันอย่างไร
4. `SystemMessage`, `HumanMessage`, `AIMessage` และ `ToolMessage` มีหน้าที่อะไร
5. `@tool` แปลง Python Function เป็น Tool Schema อย่างไร
6. Function name, docstring และ type hints ถูกนำไปสร้าง Tool Metadata อย่างไร
7. `bind_tools()` ทำอะไรและไม่ได้ทำอะไร
8. Model สร้าง Tool Request แต่ Application เป็นผู้ Execute อย่างไร
9. `AIMessage.tool_calls` มีโครงสร้างอย่างไร
10. `ToolMessage` และ `tool_call_id` เชื่อม Tool Result กับ Tool Request อย่างไร
11. Manual Tool Loop ทำงานอย่างไร
12. `for` และ `while` มีบทบาทต่างกันใน Tool Loop อย่างไร
13. `with_structured_output()` เปลี่ยน Model Response เป็น Pydantic Object อย่างไร
14. Structured Output รับประกัน Schema แต่ไม่รับประกันความจริงอย่างไร
15. Layer 1 แตกต่างจาก `create_agent()` และ LangGraph อย่างไร

---

# 3. Prerequisites

ควรเข้าใจแนวคิดต่อไปนี้จาก Week ก่อนหน้า:

```text
LLM call
Message history
Tool schema
Tool request
Tool execution
Tool result
Agent loop
Structured output
Runtime validation
```

ควรใช้ Python ระดับพื้นฐานได้ โดยเฉพาะ:

```text
Functions
Decorators
Lists
Dictionaries
Loops
Exception handling
Type hints
Pydantic models
```

Environment อย่างน้อยต้องมี:

```env
OPENAI_API_KEY=...
```

ถ้าใช้ตัวอย่าง OpenRouter:

```env
OPENROUTER_API_KEY=...
```

---

# 4. Week 4 Abstraction Levels

Notebook แบ่ง Agent Ecosystem เป็นสี่ระดับ

```text
Level 1 — Building Blocks
LangChain Core / Model Integrations

Level 2 — Orchestration
LangGraph

Level 3 — Agent Abstraction
LangChain create_agent()

Level 4 — Agent Harness
Deep Agents
```

---

## Level 1 — Building Blocks

ประกอบด้วย:

```text
Models
Messages
Tools
Tool calls
Structured output
Streaming
```

ผู้พัฒนาต้องเขียนเอง:

```text
Tool loop
State handling
Retries
Routing
Termination
Memory
Approval
```

---

## Level 2 — LangGraph

เพิ่มแนวคิด:

```text
Graph
Nodes
Edges
State
Conditional routing
Checkpointing
Persistence
Human-in-the-loop
```

LangGraph ช่วยควบคุม Workflow ที่มี State และเส้นทางหลายแบบ

---

## Level 3 — Agent

`create_agent()` ให้ Agent Loop สำเร็จรูป

```text
Model
+ Tools
+ Prompt
→ Agent Runtime
```

Framework จัดการ Model–Tool Loop ให้โดยอัตโนมัติ

---

## Level 4 — Harness

เพิ่มความสามารถระดับสูง เช่น:

```text
Planning
Sub-agents
Filesystem
Long-running tasks
Context management
```

Mental Model:

```text
Level 1
= ประกอบเครื่องยนต์เอง

Level 2
= ออกแบบถนนและทางแยก

Level 3
= ได้รถที่วิ่งและใช้ Tools ได้แล้ว

Level 4
= ได้ทีมพร้อมแผนและพื้นที่ทำงาน
```

---

# 5. Core Imports

```python
import os

from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.messages import (
    HumanMessage,
    SystemMessage,
    ToolMessage,
)
from langchain_core.tools import tool
from pydantic import BaseModel, Field

load_dotenv(override=True)
```

หน้าที่:

```text
ChatOpenAI
→ Model abstraction

Messages
→ Conversation representation

@tool
→ Tool definition

Pydantic
→ Structured output contract

load_dotenv
→ Load environment variables
```

Lab นี้ยังไม่ Import `langgraph`

เพราะยังอยู่ใน Building Blocks Layer

---

# 6. `ChatOpenAI`

ตัวอย่าง:

```python
llm = ChatOpenAI(
    model="gpt-5.4-mini"
)
```

`ChatOpenAI` เป็น Adapter ระหว่าง Application กับ Model Provider

```text
Application
→ LangChain model interface
→ Provider API
```

มันให้ Interface เช่น:

```text
invoke()
stream()
ainvoke()
astream()
bind_tools()
with_structured_output()
```

---

# 7. Model Object ไม่ใช่ Agent

```python
llm = ChatOpenAI(...)
```

สิ่งนี้สร้าง Model Client Configuration

ยังไม่มี:

```text
Tool loop
Memory
State
Planning
Retries
Routing
Termination logic
```

ดังนั้น:

```text
ChatOpenAI
≠ Agent
```

Model จะกลายเป็นส่วนหนึ่งของ Agent เมื่อถูกประกอบกับ:

```text
Messages
Tools
Execution loop
State
Stopping conditions
```

---

# 8. First Model Invocation

```python
message = (
    "In 1 sentence, what does it mean "
    "for an AI Agent to be autonomous"
)

reply = llm.invoke(message)
```

Flow:

```text
String prompt
    ↓
llm.invoke()
    ↓
Provider request
    ↓
AIMessage
```

ผลลัพธ์ไม่ใช่ String โดยตรง

แต่เป็น:

```text
AIMessage
```

ข้อความจริงอยู่ใน:

```python
reply.content
```

---

# 9. AIMessage

`AIMessage` สามารถเก็บ:

```text
content
tool calls
response metadata
token usage
model information
finish reason
```

ตัวอย่าง:

```python
print(reply.content)
print(reply.response_metadata)
```

Mental Model:

```text
AIMessage
= Model response envelope
```

`content` เป็นเพียงส่วนหนึ่งของผลลัพธ์ทั้งหมด

---

# 10. `invoke()` ไม่ใช่ Memory

การเรียก:

```python
llm.invoke("My name is Alex")
```

แล้วเรียกใหม่:

```python
llm.invoke("What is my name?")
```

Model ไม่เห็นข้อความก่อนหน้าโดยอัตโนมัติ

ต้องส่ง History เอง:

```python
messages = [
    HumanMessage("My name is Alex"),
    AIMessage("Nice to meet you, Alex."),
    HumanMessage("What is my name?"),
]

reply = llm.invoke(messages)
```

ดังนั้น:

```text
Model instance
≠ Conversation session
≠ Memory
```

---

# 11. Streaming

```python
for chunk in llm.stream(
    "Tell me a two line poem about autonomous agents"
):
    print(
        chunk.content,
        end="",
        flush=True,
    )
```

ความแตกต่าง:

```text
invoke()
→ รอคำตอบเสร็จแล้วคืนทั้งหมด

stream()
→ คืนผลทีละส่วนระหว่างสร้าง
```

Streaming ช่วย:

```text
ลดเวลาที่ผู้ใช้รู้สึกว่ารอ
แสดงคำตอบต่อเนื่อง
รองรับ UI แบบ real-time
```

แต่ไม่ได้ทำให้:

```text
Reasoning ถูกต้องขึ้น
Model คิดเร็วขึ้น
คำตอบมีคุณภาพสูงขึ้น
```

---

# 12. OpenAI-Compatible Providers

ตัวอย่าง:

```python
openrouter_llm = ChatOpenAI(
    model="anthropic/claude-haiku-4.5",
    base_url="https://openrouter.ai/api/v1",
    api_key=os.getenv(
        "OPENROUTER_API_KEY"
    ),
)
```

Flow:

```text
ChatOpenAI interface
        ↓
Custom base URL
        ↓
OpenAI-compatible endpoint
        ↓
Selected provider/model
```

ข้อดี:

```text
เปลี่ยน Endpoint ได้ง่าย
Interface ใกล้เคียงกัน
Reuse application code ได้
```

แต่ต้องระวัง:

```text
API compatibility
≠ Capability parity
```

ความสามารถที่อาจต่างกัน:

```text
Tool calling
Structured output
Streaming
Usage metadata
Reasoning metadata
Multimodal input
Error format
```

---

# 13. Messages

Message Types หลัก:

```text
SystemMessage
HumanMessage
AIMessage
ToolMessage
```

---

## SystemMessage

กำหนด:

```text
บทบาท
นโยบาย
พฤติกรรมระดับสูง
ข้อจำกัดของคำตอบ
```

ตัวอย่าง:

```python
SystemMessage(
    content=(
        "You are a terse assistant "
        "who answers in five words."
    )
)
```

---

## HumanMessage

แทนข้อความจากผู้ใช้

```python
HumanMessage(
    content=(
        "What is the capital of France?"
    )
)
```

---

## AIMessage

แทนคำตอบจาก Model

อาจมี:

```text
Natural-language content
Tool requests
Metadata
```

---

## ToolMessage

แทนผลลัพธ์จาก Tool ที่ Application Execute แล้ว

```python
ToolMessage(
    content="198.0",
    tool_call_id="call_123",
)
```

---

# 14. Message Sequence

Tool-using conversation ที่ถูกต้องมีลำดับ:

```text
HumanMessage
    ↓
AIMessage with tool call
    ↓
ToolMessage
    ↓
AIMessage final answer
```

นี่ไม่ใช่เพียง Conversation Log

แต่เป็น Protocol ระหว่าง:

```text
Model
Application
Tool runtime
```

---

# 15. Messages กับ Dictionaries

Message Objects:

```python
messages = [
    SystemMessage(
        content="You are concise."
    ),
    HumanMessage(
        content="Explain agents."
    ),
]
```

Dictionary form:

```python
messages = [
    {
        "role": "system",
        "content": "You are concise.",
    },
    {
        "role": "user",
        "content": "Explain agents.",
    },
]
```

ทั้งสองแบบสามารถสื่อความหมายใกล้เคียงกัน

Message Classes ให้:

```text
Type safety
Convenient attributes
Tool-call support
Consistent abstractions
```

---

# 16. Creating a Tool

```python
@tool
def get_share_price(
    symbol: str
) -> float:
    """Return the current share price for a ticker."""

    fake_prices = {
        "AAPL": 241.5,
        "GOOG": 168.2,
        "AMZN": 198.0,
    }

    return fake_prices.get(
        symbol.upper(),
        0.0,
    )
```

`@tool` แปลง Python Function ให้เป็น Tool Object

---

# 17. Tool Metadata

Metadata ถูกสร้างจาก:

```text
Function name
→ Tool name

Docstring
→ Tool description

Type hints
→ Argument schema
```

ตรวจได้ด้วย:

```python
print(get_share_price.name)
print(get_share_price.description)
print(get_share_price.args)
```

Tool Schema ที่ Model เห็นมีลักษณะประมาณ:

```json
{
  "name": "get_share_price",
  "description": "Return the current share price for a ticker.",
  "parameters": {
    "symbol": {
      "type": "string"
    }
  }
}
```

---

# 18. Model ไม่เห็น Function Source Code

Model ไม่ได้อ่าน:

```python
fake_prices = {...}
```

มันเห็นเพียง:

```text
Tool name
Description
Argument schema
```

ดังนั้น Tool Description ต้อง:

```text
ชัดเจน
ไม่กำกวม
อธิบาย Input
อธิบาย Output
อธิบายข้อจำกัด
```

แต่:

```text
Tool description
≠ Security boundary
```

Validation และ Permission ต้องอยู่ใน Tool Implementation

---

# 19. Direct Tool Invocation

```python
result = get_share_price.invoke({
    "symbol": "AAPL"
})
```

Flow:

```text
Application
→ Tool
→ Result
```

ยังไม่มี Model เป็นผู้ตัดสินใจ

Direct Tool Invocation เหมาะกับ:

```text
Testing
Debugging
Deterministic workflow
Tool validation
```

---

# 20. `bind_tools()`

```python
llm_with_tools = llm.bind_tools([
    get_share_price
])
```

สิ่งที่เกิด:

```text
Model configuration
+
Tool schemas
=
Tool-aware model
```

Model รู้ว่า:

```text
มี Tool อะไร
Tool ชื่ออะไร
รับ Arguments อะไร
Tool ทำหน้าที่อะไร
```

---

# 21. สิ่งที่ `bind_tools()` ไม่ทำ

`bind_tools()` ไม่ได้:

```text
Execute Tool
Create Agent Loop
Dispatch Tool
Validate Permission
Handle Retry
Store Memory
Maintain State
Stop Infinite Loop
```

ดังนั้น:

```text
bind_tools()
= ให้ Model รู้จัก Tools

ไม่ใช่
= ให้ Model ควบคุม Tools โดยตรง
```

---

# 22. Model Tool Request

```python
response = llm_with_tools.invoke(
    "What is Amazon's share price?"
)
```

Model อาจคืน:

```python
response.tool_calls
```

ตัวอย่าง:

```python
[
    {
        "name": "get_share_price",
        "args": {
            "symbol": "AMZN",
        },
        "id": "call_123",
        "type": "tool_call",
    }
]
```

นี่คือ:

```text
Proposed action
```

ไม่ใช่ Tool Result

---

# 23. Tool Request Mental Model

```text
User asks a question
        ↓
Model decides information is missing
        ↓
Model proposes tool call
        ↓
Application decides whether to execute
```

หลักสำคัญ:

```text
Model proposes
Application authorizes
Tool executes
```

---

# 24. `AIMessage.tool_calls`

`tool_calls` เป็นรายการ เพราะ Model อาจขอหลาย Tools ใน Response เดียว

ตัวอย่าง:

```text
AIMessage
├── get_share_price(AAPL)
└── get_weather(Bangkok)
```

แต่ละ Tool Call มี:

```text
name
args
id
type
```

---

# 25. Manual Tool Loop

ตัวอย่างพื้นฐาน:

```python
conversation = [
    HumanMessage(
        content=(
            "What is Amazon's share price?"
        )
    )
]

ai_message = llm_with_tools.invoke(
    conversation
)

conversation.append(ai_message)

for call in ai_message.tool_calls:
    if call["name"] == "get_share_price":
        result = get_share_price.invoke(
            call["args"]
        )

        conversation.append(
            ToolMessage(
                content=str(result),
                tool_call_id=call["id"],
            )
        )

final = llm_with_tools.invoke(
    conversation
)

print(final.content)
```

---

# 26. Manual Tool Loop Flow

```text
1. User sends request
2. Model sees Tool schemas
3. Model generates Tool Request
4. Application reads tool_calls
5. Application executes Python Function
6. Application creates ToolMessage
7. Tool result is appended to history
8. Model generates final answer
```

นี่คือ Agent Loop ขั้นพื้นฐาน

---

# 27. ทำไมต้อง Append AIMessage

ต้องเก็บ:

```python
conversation.append(ai_message)
```

เพราะ Model history ต้องมี:

```text
คำขอจากผู้ใช้
คำขอใช้ Tool จาก Model
ผลจาก Tool
```

ถ้าไม่มี AIMessage ที่ขอ Tool:

```text
Model อาจไม่รู้ว่า Tool Result
ตอบคำขอใด
```

---

# 28. `tool_call_id`

```python
ToolMessage(
    content=str(result),
    tool_call_id=call["id"],
)
```

`tool_call_id` ทำหน้าที่เชื่อม:

```text
Tool Request
↔
Tool Result
```

Mental Model:

```text
tool_call_id
= Tracking number
```

---

# 29. Multiple Tool Calls

ตัวอย่าง:

```text
call_001
→ get_share_price(AAPL)

call_002
→ get_weather(Bangkok)
```

ผลลัพธ์ต้องเชื่อมให้ถูก:

```text
Price result
→ call_001

Weather result
→ call_002
```

หาก ID ไม่ตรง:

```text
Model อาจจับคู่ Result ผิด
Provider อาจปฏิเสธ Message
Conversation protocol อาจเสีย
```

---

# 30. `for` กับ `while`

Notebook ใช้:

```python
for call in ai_message.tool_calls:
```

`for` จัดการ:

```text
หลาย Tool Calls
ใน Model Response รอบเดียว
```

แต่ Agent Loop ที่สมบูรณ์ต้องรองรับ:

```text
หลาย Model–Tool Rounds
```

จึงต้องใช้ Loop เช่น:

```python
while True:
    response = model.invoke(
        conversation
    )

    conversation.append(response)

    if not response.tool_calls:
        break

    for call in response.tool_calls:
        ...
```

สรุป:

```text
for
= หลาย Tool Calls ต่อรอบ

while
= หลาย Model–Tool Rounds
```

---

# 31. Lab Loop ยังไม่สมบูรณ์

Notebook เรียก Model หลัง Tool Result อีกเพียงครั้งเดียว:

```python
final = llm_with_tools.invoke(
    conversation
)
```

ถ้า `final` ขอ Tool เพิ่ม:

```text
Workflow จะหยุด
```

Agent Loop ที่แข็งแรงต้องทำซ้ำจนกว่า:

```text
Model ไม่มี Tool Calls
หรือ
ถึง Maximum Rounds
```

---

# 32. Tool Registry

แทนการใช้:

```python
if call["name"] == "get_share_price":
```

ควรใช้:

```python
tools = [
    get_share_price,
]

tool_registry = {
    current_tool.name: current_tool
    for current_tool in tools
}
```

Dispatch:

```python
tool_name = call["name"]

if tool_name not in tool_registry:
    raise ValueError(
        f"Unknown tool: {tool_name}"
    )

selected_tool = tool_registry[
    tool_name
]

result = selected_tool.invoke(
    call["args"]
)
```

---

# 33. ประโยชน์ของ Tool Registry

```text
เพิ่ม Tool ได้ง่าย
ไม่ต้องเพิ่ม if/elif จำนวนมาก
ตรวจ Unknown Tool ได้
เพิ่ม Logging ได้
เพิ่ม Permission Policy ได้
เพิ่ม Timeout ได้
เพิ่ม Retry ได้
```

---

# 34. Robust Manual Agent Loop

```python
tools = [
    get_share_price,
]

tool_registry = {
    item.name: item
    for item in tools
}

model = llm.bind_tools(tools)

conversation = [
    HumanMessage(
        content=(
            "What is Amazon's share price?"
        )
    )
]

max_rounds = 5

for _ in range(max_rounds):
    response = model.invoke(
        conversation
    )

    conversation.append(response)

    if not response.tool_calls:
        print(response.content)
        break

    for call in response.tool_calls:
        tool_name = call["name"]

        if tool_name not in tool_registry:
            result = (
                f"Unknown tool: {tool_name}"
            )
        else:
            try:
                result = tool_registry[
                    tool_name
                ].invoke(
                    call["args"]
                )
            except Exception as exc:
                result = (
                    f"Tool failed: {exc}"
                )

        conversation.append(
            ToolMessage(
                content=str(result),
                tool_call_id=call["id"],
            )
        )
else:
    raise RuntimeError(
        "Maximum tool rounds exceeded"
    )
```

---

# 35. Controls ที่ควรเพิ่มใน Tool Loop

```text
Maximum rounds
Unknown tool rejection
Argument validation
Exception handling
Timeout
Retry
Permission checks
Rate limit
Logging
Tracing
Side-effect approval
```

---

# 36. Tool Validation

Model อาจส่ง:

```python
{
    "symbol": ""
}
```

หรือ:

```python
{
    "symbol": "invalid"
}
```

Tool Implementation ต้องตรวจ:

```text
Input format
Allowed values
Data type
Length
Permissions
Business constraints
```

Prompt หรือ Schema อย่างเดียวไม่เพียงพอ

---

# 37. Structured Output

Pydantic Model:

```python
class Company(BaseModel):
    name: str = Field(
        description="The company name"
    )

    ticker: str = Field(
        description=(
            "The stock ticker symbol"
        )
    )

    founded_year: int = Field(
        description=(
            "The year the company was founded"
        )
    )
```

Bind:

```python
structured_llm = (
    llm.with_structured_output(
        Company
    )
)
```

Invoke:

```python
company = structured_llm.invoke(
    "Tell me about Amazon"
)
```

---

# 38. Structured Output Flow

```text
Pydantic model
        ↓
JSON schema
        ↓
Model generates matching fields
        ↓
Response parsing
        ↓
Pydantic validation
        ↓
Typed Python object
```

ผลลัพธ์:

```python
Company(
    name="Amazon",
    ticker="AMZN",
    founded_year=1994,
)
```

ใช้งาน:

```python
print(company.ticker)
```

---

# 39. Structured Output Benefits

```text
Machine-readable result
Predictable fields
Type validation
Simpler downstream code
Clear output contract
Reduced parsing logic
```

เหมาะกับ:

```text
API responses
Database records
Routing decisions
Task boundaries
Evaluation results
```

---

# 40. Structured Output ไม่ใช่ Ground Truth

Pydantic ตรวจได้ว่า:

```text
name เป็น String
ticker เป็น String
founded_year เป็น Integer
```

แต่ไม่ตรวจว่า:

```text
Ticker ถูกต้องหรือไม่
ปีที่ก่อตั้งถูกต้องหรือไม่
บริษัทมีอยู่จริงหรือไม่
ข้อมูลเป็นปัจจุบันหรือไม่
```

ดังนั้น:

```text
Schema valid
≠ Factually correct
```

---

# 41. Structured Validation Layers

```text
Layer 1
Pydantic type validation

Layer 2
Deterministic business rules

Layer 3
External source verification

Layer 4
Cross-source comparison

Layer 5
Human review
```

ตัวอย่าง:

```python
if company.founded_year < 1800:
    raise ValueError(
        "Unrealistic founded year"
    )
```

แต่ Validation แบบนี้ยังไม่ได้ยืนยันว่าปีที่ระบุถูกจริง

---

# 42. Building Blocks ที่ Lab ให้มา

```text
Model invocation
Streaming
Messages
Tool definition
Tool binding
Tool-call representation
Tool-result messages
Structured output
```

---

# 43. สิ่งที่ Lab ยังไม่ให้

```text
Graph
Persistent state
Checkpointing
Conditional routing
Automatic tool loop
Memory
Human approval
Retry policy
Error recovery
Workflow visualization
```

สิ่งเหล่านี้จะถูกเพิ่มในระดับ LangGraph หรือ Agent Abstraction

---

# 44. LangChain Layer 1 กับ OpenAI Agents SDK

## OpenAI Agents SDK

```text
Runner
→ จัดการ Agent Loop
```

## LangChain Building Blocks

```text
Application
→ เขียน Tool Loop เอง
```

LangChain ระดับนี้ให้ Control มากกว่า

แต่ต้องเขียน Runtime Logic มากขึ้น

---

# 45. LangChain Layer 1 กับ CrewAI

## CrewAI

```text
Agent
Task
Crew
Process
```

Framework เน้นทีมและการมอบหมายงาน

## LangChain Building Blocks

```text
Model
Messages
Tools
Tool calls
```

เน้น Protocol ระดับต่ำของ Model และ Tools

---

# 46. LangChain กับ LangGraph

LangChain Building Blocks ให้:

```text
องค์ประกอบ
```

LangGraph ให้:

```text
Control flow
State transitions
Persistence
```

Mental Model:

```text
LangChain components
= ตัวต่อ

LangGraph
= แผนผังวิธีต่อตัวต่อ
```

---

# 47. Misconceptions Corrected

## ความเข้าใจคลาดเคลื่อนที่ 1

> `ChatOpenAI` คือ Agent

**ข้อเท็จจริง:**
เป็น Model Adapter

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> Model จำ Conversation ได้อัตโนมัติ

**ข้อเท็จจริง:**
ต้องส่ง Message History หรือใช้ State/Memory System

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> `bind_tools()` Execute Tool

**ข้อเท็จจริง:**
มันเพียงส่ง Tool Schema ให้ Model

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> Model เรียก Python Function เอง

**ข้อเท็จจริง:**
Model สร้าง Tool Request ส่วน Application Execute

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> `tool_calls` คือ Tool Results

**ข้อเท็จจริง:**
เป็น Proposed Actions

---

## ความเข้าใจคลาดเคลื่อนที่ 6

> `ToolMessage` คือข้อความจาก Human

**ข้อเท็จจริง:**
เป็น Observation จาก Tool Runtime

---

## ความเข้าใจคลาดเคลื่อนที่ 7

> `for` loop รองรับ Agent Loop ทั้งหมด

**ข้อเท็จจริง:**
รองรับหลาย Tool Calls ในรอบเดียว ต้องมี Outer Loop สำหรับหลายรอบ

---

## ความเข้าใจคลาดเคลื่อนที่ 8

> Structured Output ทำให้ข้อมูลจริง

**ข้อเท็จจริง:**
ควบคุม Type และ Structure

---

## ความเข้าใจคลาดเคลื่อนที่ 9

> LangGraph มาแทน LangChain

**ข้อเท็จจริง:**
LangGraph เป็น Stateful Orchestration Runtime ที่ใช้ร่วมกับ LangChain Components

---

# 48. Risks Identified

## 48.1 Infinite Tool Loop

Model อาจเรียก Tool ซ้ำโดยไม่จบ

## 48.2 Unknown Tool

Model อาจขอ Tool ที่ไม่มีใน Registry

## 48.3 Invalid Arguments

Arguments อาจไม่ผ่าน Validation

## 48.4 Tool Failure

Tool อาจ Throw Exception หรือ Timeout

## 48.5 Side Effects

Tool อาจส่ง Email แก้ไฟล์ หรือเรียก API จริง

## 48.6 Message Protocol Error

Tool Result อาจเชื่อมกับ `tool_call_id` ผิด

## 48.7 State Loss

หากไม่เก็บ Messages Model จะไม่เห็นข้อมูลรอบก่อน

## 48.8 Schema Hallucination

Output ผ่าน Schema แต่ข้อมูลผิด

## 48.9 Provider Compatibility

OpenAI-compatible Endpoint อาจรองรับ Feature ไม่ครบ

---

# 49. Production Improvements

```text
Tool registry
Maximum rounds
Argument validation
Exception handling
Timeouts
Retries
Permission checks
Side-effect approval
Structured tool results
Persistent message state
Tracing
Usage budgets
Provider capability checks
Fact verification
```

---

# 50. Exercise — Add a Second Tool

สร้าง:

```python
@tool
def get_weather(
    city: str
) -> str:
    """Return fictional weather for a city."""

    reports = {
        "Bangkok": "Sunny, 34°C",
        "London": "Cloudy, 16°C",
    }

    return reports.get(
        city.title(),
        "Weather unavailable",
    )
```

Bind:

```python
tools = [
    get_share_price,
    get_weather,
]

model = llm.bind_tools(tools)
```

ถาม:

```text
What is Amazon's share price,
and what is the fictional weather
in Bangkok?
```

---

# 51. สิ่งที่ต้องสังเกตจาก Exercise

```text
Model ขอ Tools กี่ตัว
ขอพร้อมกันหรือทีละรอบ
Arguments ถูกต้องหรือไม่
Tool-call IDs ต่างกันหรือไม่
Model ขอ Tool เพิ่มหลังได้ Result หรือไม่
Final response รวมข้อมูลถูกหรือไม่
```

---

# 52. Suggested Exercise — Tool Registry

สร้าง Registry:

```python
tool_registry = {
    tool_item.name: tool_item
    for tool_item in tools
}
```

แล้วใช้ Generic Dispatcher แทน `if/elif`

---

# 53. Suggested Exercise — Maximum Rounds

กำหนด:

```python
max_rounds = 5
```

หากเกิน:

```python
raise RuntimeError(
    "Maximum rounds exceeded"
)
```

เพื่อป้องกัน Infinite Loop และ Cost ที่ควบคุมไม่ได้

---

# 54. Suggested Exercise — Structured Tool Result

แทนการคืน:

```python
198.0
```

ให้ Tool คืน:

```python
{
    "symbol": "AMZN",
    "price": 198.0,
    "currency": "USD",
    "source": "mock",
}
```

ช่วยให้ Model และ Application เข้าใจ Result ชัดขึ้น

---

# 55. Suggested Exercise — Tool Error Message

หาก Ticker ไม่มี:

```python
raise ValueError(
    "Unknown ticker symbol"
)
```

จากนั้นตรวจว่า Agent ใช้ Error Feedback เพื่อ:

```text
ถามผู้ใช้เพิ่ม
แก้ Arguments
หรือแจ้งว่าไม่พบข้อมูล
```

---

# 56. Patterns Learned

## Model Adapter Pattern

```text
Application
→ Standard model interface
→ Provider
```

## Message Protocol Pattern

```text
Human
→ AI
→ Tool
→ AI
```

## Tool Schema Pattern

```text
Python function
→ Decorator
→ Model-readable schema
```

## Model-Proposes/Application-Executes Pattern

```text
LLM proposes action
→ Application authorizes
→ Tool executes
```

## Observation Loop Pattern

```text
Decision
→ Action
→ Observation
→ Next decision
```

## Structured Output Pattern

```text
Natural-language generation
→ Schema
→ Typed object
```

---

# 57. Connection to Week 1

Week 1 Manual Agent Loop:

```text
Model requests action
Application executes
Tool result returned
Loop continues
```

Week 4 Lab 1 ทำสิ่งเดียวกันผ่าน LangChain abstractions:

```text
AIMessage.tool_calls
Tool.invoke()
ToolMessage
```

---

# 58. Connection to Week 2

OpenAI Agents SDK:

```text
Runner
จัดการ Tool Loop
```

LangChain Layer 1:

```text
Developer
จัดการ Tool Loop เอง
```

ข้อดีของ Layer 1:

```text
เห็น Protocol ชัด
ควบคุมได้ละเอียด
เข้าใจ Agent Runtime
```

ข้อเสีย:

```text
Code มากขึ้น
ต้องจัดการ Failure เอง
```

---

# 59. Connection to Week 3

CrewAI ใช้:

```text
Agents
Tasks
Crews
Processes
```

LangChain Layer 1 แสดงฐานล่างของ Agent เหล่านั้น:

```text
Model
Messages
Tools
Tool results
Loop
```

Framework ชั้นสูงเปลี่ยนรูปแบบการใช้งาน

แต่หลักพื้นฐานยังคงเดิม:

```text
Model decides
Application acts
Environment responds
```

---

# 60. Lab 1 Mental Model

```text
User request
    ↓
HumanMessage
    ↓
ChatOpenAI
    ↓
AIMessage
    ↓
Does it contain tool calls?
    ├── No → Final answer
    └── Yes
          ↓
      Application dispatches tool
          ↓
      ToolMessage
          ↓
      Model invoked again
```

Structured output เป็นอีกเส้นทางหนึ่ง:

```text
Prompt
    ↓
Model
    ↓
Pydantic schema
    ↓
Typed object
```

---

# 61. Checklist ก่อนจบ Lab

### `ChatOpenAI` คืออะไร

Model Adapter

### `invoke()` คืนอะไร

โดยทั่วไปคืน `AIMessage`

### `stream()` แตกต่างอย่างไร

คืนผลทีละ Chunk ระหว่าง Generation

### Message ที่ส่งผลจาก Tool คืออะไร

`ToolMessage`

### Tool Schema มาจากอะไร

Function name, docstring และ type hints

### `bind_tools()` ทำอะไร

แนบ Tool Schemas ให้ Model

### ใคร Execute Tool

Application Code

### Tool Request อยู่ที่ไหน

```python
ai_message.tool_calls
```

### `tool_call_id` ใช้ทำอะไร

เชื่อม Tool Result กับ Tool Request

### `for` รองรับอะไร

หลาย Tool Calls ในรอบเดียว

### `while` หรือ Outer Loop รองรับอะไร

หลาย Model–Tool Rounds

### Structured Output ใช้อะไร

Pydantic Model

### Pydantic รับประกันความจริงหรือไม่

ไม่

### Lab นี้ใช้ LangGraph หรือยัง

ยังไม่ใช้

---

# 62. Final Lessons

## Lesson 1

LangChain Layer 1 ให้ Building Blocks แต่ยังไม่ให้ Agent Runtime ที่สมบูรณ์

## Lesson 2

Model Object ไม่ใช่ Agent และไม่ใช่ Memory

## Lesson 3

Messages เป็น Protocol ที่เก็บทั้ง Conversation และ Tool Interaction

## Lesson 4

Tool Decorator เปลี่ยน Python Function เป็น Schema ที่ Model เข้าใจได้

## Lesson 5

`bind_tools()` ทำให้ Model รู้จัก Tool แต่ไม่ได้ Execute Tool

## Lesson 6

Model สร้าง Proposed Action ส่วน Application ถือ Authority ในการ Execute

## Lesson 7

`tool_call_id` เป็นองค์ประกอบสำคัญในการเชื่อม Request กับ Result

## Lesson 8

Agent Loop ต้องรองรับทั้งหลาย Tool Calls และหลาย Model–Tool Rounds

## Lesson 9

Structured Output ช่วยสร้าง Machine-readable Contract

## Lesson 10

Schema Validation ไม่เท่ากับ Factual Validation

## Lesson 11

Framework ชั้นสูงซ่อน Model–Tool Loop เอาไว้ แต่หลักการภายในยังเหมือนเดิม

## Lesson 12

การเข้าใจ Layer 1 ทำให้สามารถ Debug และออกแบบ LangGraph ได้อย่างมีเหตุผลมากขึ้น

---

# 63. Memory Summary

```text
Week 4 Lab 1 เริ่ม LangChain
ที่ Building Blocks Layer

Notebook:
4_langchain_langgraph/1_lab1.ipynb

ยังไม่ใช้ LangGraph

Core components:
ChatOpenAI
Messages
Tools
Tool calls
Tool results
Structured output

ChatOpenAI:
เป็น Model Adapter
ไม่ใช่ Agent
ไม่ใช่ Memory

invoke():
คืน AIMessage

stream():
คืน Message chunks

Message types:
SystemMessage
HumanMessage
AIMessage
ToolMessage

@tool:
แปลง Python function
เป็น Model-readable Tool

Tool schema มาจาก:
Function name
Docstring
Type hints

bind_tools():
แนบ Tool schema ให้ Model

แต่ไม่:
Execute Tool
สร้าง Loop
จัดการ State
จัดการ Memory

Model:
เสนอ Tool Call

Application:
ตรวจและ Execute Tool

Tool request:
AIMessage.tool_calls

Tool result:
ToolMessage

tool_call_id:
เชื่อม Request กับ Result

for loop:
หลาย Tool Calls ในรอบเดียว

outer loop:
หลาย Model–Tool Rounds

Tool Registry:
ใช้ Dispatch Tools
และตรวจ Unknown Tools

Manual loop ควรเพิ่ม:
Maximum rounds
Validation
Exception handling
Timeout
Retry
Permissions
Approval
Tracing

with_structured_output():
ใช้ Pydantic Schema
สร้าง Typed Object

Structured output:
ตรวจ Structure และ Types

แต่ไม่ตรวจ:
Truth
Recency
Source quality

Core Agent Loop:
User
→ Model
→ Tool Request
→ Application Execution
→ Tool Result
→ Model
→ Final Answer
```

---

# 64. Next Episode

Lab ถัดไปจะเริ่มเปลี่ยน Manual Loop ให้เป็น Stateful Graph

แนวคิดที่จะเพิ่ม:

```text
Graph state
Nodes
Edges
Conditional routing
Message accumulation
Tool nodes
Loop termination
```

คำถามสำคัญสำหรับ Lab ถัดไปคือ:

> เมื่อ Tool Loop เริ่มมีหลายเส้นทาง หลายรอบ และต้องเก็บ State เราจะเปลี่ยน Code ที่ใช้ `while` และ `if` ให้กลายเป็น Graph ที่มองเห็น Control Flow ได้ชัดเจน ตรวจสอบได้ และ Resume จากจุดเดิมได้อย่างไร?
