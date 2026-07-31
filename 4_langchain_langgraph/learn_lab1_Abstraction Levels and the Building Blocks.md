# Week 4 — Lab 1: LangChain Building Blocks

Notebook:

```text
4_langchain_langgraph/1_lab1.ipynb
```

หัวข้อของ Lab คือ **Abstraction Levels and the Building Blocks** โดยยังไม่เริ่มสร้าง LangGraph จริง แต่จะรื้อระบบ Agent ลงมาดูชิ้นส่วนพื้นฐานที่ Framework ชั้นสูงซ่อนไว้ ได้แก่ Model, Messages, Tools, Tool Calls, Tool Results และ Structured Output. 

---

## 1. Learning Objectives

หลังจบ Lab นี้ควรอธิบายได้ว่า:

1. LangChain และ LangGraph อยู่คนละระดับของ Abstraction อย่างไร
2. `ChatOpenAI` ทำหน้าที่อะไร
3. `invoke()` และ `stream()` ต่างกันอย่างไร
4. `SystemMessage`, `HumanMessage`, `AIMessage` และ `ToolMessage` ต่างกันอย่างไร
5. `@tool` เปลี่ยน Python Function ให้เป็น Tool Schema อย่างไร
6. `bind_tools()` ทำอะไรและ **ไม่ได้ทำอะไร**
7. ทำไม LLM ไม่ได้ Execute Tool ด้วยตัวเอง
8. `tool_calls` และ `tool_call_id` ใช้เชื่อม Request กับ Result อย่างไร
9. Manual Tool Loop ทำงานอย่างไร
10. `with_structured_output()` เปลี่ยนคำตอบเป็น Pydantic Object อย่างไร
11. Structured Output รับประกันโครงสร้าง แต่ไม่รับประกันความจริงอย่างไร
12. Layer 1 แตกต่างจาก `create_agent()` และ LangGraph อย่างไร

---

# 2. Prerequisites

จาก Week ก่อนหน้า ควรจำแนวคิดเหล่านี้ได้:

```text
Model call
Messages
Tool schema
Tool request
Application executes tool
Tool result
Agent loop
Structured output
```

Week 4 ไม่ได้เริ่มจากศูนย์ แต่กำลังนำสิ่งที่เคยทำกับ OpenAI Agents SDK และ CrewAI มาดูในระดับที่ต่ำกว่า

Environment หลักของ repository ปัจจุบันกำหนด Python `>=3.12` และมี dependencies เช่น `langchain-openai`, `langgraph`, `langsmith` และ Pydantic-related packages ส่วน Notebook ระบุ Kernel Python 3.12.12. ([GitHub][1])

ไฟล์ `.env` อย่างน้อยควรมี:

```env
OPENAI_API_KEY=...
```

ส่วนตัวอย่าง OpenRouter ต้องมี:

```env
OPENROUTER_API_KEY=...
```

---

# 3. ภาพใหญ่: Four Levels of Abstraction

Notebook แบ่ง Ecosystem ออกเป็นสี่ระดับ:

| Level              | Package                              | สิ่งที่ได้                                 | สิ่งที่เราควบคุม           |
| ------------------ | ------------------------------------ | ------------------------------------------ | -------------------------- |
| 1. Building blocks | `langchain-core`, `langchain-openai` | Models, Messages, Tools, Structured Output | ควบคุม Tool Loop เอง       |
| 2. Orchestration   | `langgraph`                          | Graph, State, Memory, Checkpointing        | ออกแบบ Control Flow        |
| 3. Agent           | `langchain.create_agent`             | Agent Loop สำเร็จรูป                       | กำหนด Model, Tools, Prompt |
| 4. Harness         | `deepagents.create_deep_agent`       | Planning, Sub-agents, Filesystem           | ระบุ Intent ระดับสูง       |

นี่คือกรอบที่ Course ใช้ตลอด Week 4. 

Mental model:

```text
Layer 1
เราประกอบเครื่องยนต์เอง

Layer 2
เราออกแบบถนนและทางแยก

Layer 3
เราได้รถที่วิ่งวนใช้ Tools ได้แล้ว

Layer 4
เราได้ทีมขับรถ พร้อมแผน งานย่อย และพื้นที่ทำงาน
```

LangGraph ไม่ใช่ตัวแทน LangChain แต่เป็น Runtime ระดับต่ำสำหรับ Stateful Orchestration ขณะที่ `create_agent()` ของ LangChain ปัจจุบันสร้าง Agent Runtime บน LangGraph อีกที. ([Docs by LangChain][2])

---

# 4. Imports

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

แยกหน้าที่ได้ดังนี้:

```text
ChatOpenAI
→ Model abstraction

Messages
→ Conversation objects

@tool
→ Tool definition

Pydantic
→ Output schema

load_dotenv
→ Load API credentials
```

สังเกตว่า Notebook ยังไม่ Import `langgraph` เพราะ Lab 1 อยู่ใน Layer 1 เท่านั้น. 

---

# 5. First Model Call

```python
llm = ChatOpenAI(model="gpt-5.4-mini")

message = (
    "In 1 sentence, what does it mean "
    "for an AI Agent to be autonomous"
)

reply = llm.invoke(message)

print(reply.content)
```

Flow:

```text
String prompt
    ↓
ChatOpenAI.invoke()
    ↓
Provider API
    ↓
AIMessage
    ↓
reply.content
```

จุดสำคัญคือ `invoke()` ไม่ได้คืน String โดยตรง แต่คืน Message Object ที่มีทั้ง Content และ Metadata เช่น Token Usage, Model Name และ Finish Reason ตาม Provider ที่รองรับ. 

---

## `ChatOpenAI` คืออะไร

`ChatOpenAI` เป็น Adapter ระหว่าง Code ของเรากับ OpenAI Chat Model API

แทนที่จะเรียก OpenAI Client โดยตรง:

```text
Application
→ OpenAI-specific API
```

เราใช้:

```text
Application
→ LangChain model interface
→ Provider API
```

ประโยชน์คือ Model Interface มีรูปแบบมาตรฐาน เช่น:

```text
invoke
stream
ainvoke
astream
bind_tools
with_structured_output
```

LangChain ออกแบบ Model Interface เพื่อให้การเปลี่ยน Provider หรือ Model ทำได้ด้วยรูปแบบการเรียกใกล้เคียงกัน. ([Docs by LangChain][3])

---

# 6. `invoke()` ไม่ใช่ Memory

```python
llm.invoke("Hello")
```

เป็น Model Call หนึ่งครั้ง

การเรียกครั้งต่อไป:

```python
llm.invoke("What did I just say?")
```

ไม่ได้เห็นข้อความก่อนหน้าโดยอัตโนมัติ

ต้องส่ง History เอง:

```python
messages = [
    HumanMessage("Hello"),
    AIMessage("Hi"),
    HumanMessage("What did I just say?"),
]
```

ดังนั้น:

```text
Model Object
≠ Conversation Memory
```

`ChatOpenAI` เป็น Client Configuration ไม่ใช่ Session ที่สะสมประวัติเอง

---

# 7. Streaming

```python
for chunk in llm.stream(
    "Tell me a two line poem about autonomous agents"
):
    print(chunk.content, end="", flush=True)
```

ความต่าง:

```text
invoke()
→ รอคำตอบครบแล้วคืนครั้งเดียว

stream()
→ คืนข้อความเป็นชิ้น ๆ ระหว่างการสร้าง
```

Streaming ช่วยเรื่อง User Experience เพราะผู้ใช้เริ่มเห็นผลลัพธ์เร็วขึ้น แต่ไม่ได้ทำให้ Model คิดเร็วขึ้นหรือทำให้คำตอบถูกขึ้น

บาง Provider อาจไม่ส่ง Usage Metadata ระหว่าง Streaming โดย Default; `ChatOpenAI` รองรับ `stream_usage=True` สำหรับกรณีที่ Endpoint รองรับ. 

---

# 8. OpenAI-compatible Provider

Notebook ใช้:

```python
openrouter_llm = ChatOpenAI(
    model="anthropic/claude-haiku-4.5",
    base_url="https://openrouter.ai/api/v1",
    api_key=os.getenv("OPENROUTER_API_KEY"),
)
```

Mental model:

```text
ChatOpenAI interface
        ↓
Custom base_url
        ↓
OpenRouter-compatible endpoint
        ↓
Selected model
```

สิ่งนี้แสดงว่า Adapter เดียวสามารถใช้กับ Endpoint ที่รองรับ OpenAI-compatible API ได้สำหรับพื้นฐาน เช่น Invocation และ Tool Calling บางรูปแบบ. 

อย่างไรก็ตาม เอกสารปัจจุบันของ LangChain ระบุว่า `ChatOpenAI` มุ่งรองรับ OpenAI API Specification โดยตรง ฟิลด์เฉพาะของ Third-party Providers เช่น Reasoning Metadata อาจไม่ถูกเก็บครบ และแนะนำ Provider-specific integrations เมื่อต้องใช้ความสามารถเฉพาะทาง. ([Docs by LangChain][4])

ดังนั้น:

```text
API-compatible
≠ Feature-compatible
```

ต้องตรวจแยก:

```text
Tool calling
Structured output
Streaming
Usage metadata
Reasoning fields
Multimodal support
Error format
```

---

# 9. Messages

```python
messages = [
    SystemMessage(
        "You are a terse assistant "
        "who answers in exactly five words."
    ),
    HumanMessage("What is the capital of France?"),
]

reply = llm.invoke(messages)
```

ประเภทหลัก:

```text
SystemMessage
→ นโยบายและบทบาทระดับระบบ

HumanMessage
→ ข้อความจากผู้ใช้

AIMessage
→ คำตอบหรือ Tool Request จากโมเดล

ToolMessage
→ ผลลัพธ์จาก Tool ที่ Application รันแล้ว
```

Notebook แสดงด้วยว่า Message Objects กับ Plain Dictionaries สามารถส่งความหมายเดียวกันได้:

```python
messages_as_dicts = [
    {
        "role": "system",
        "content": "You are a terse assistant..."
    },
    {
        "role": "user",
        "content": "What is the capital of France?"
    },
]
```

Message Classes เพิ่ม Type Safety และ Methods ที่สะดวก ส่วน Dictionary เป็นรูปแบบพื้นฐานที่คุ้นเคยจาก Provider APIs. 

---

# 10. Creating a Tool

```python
@tool
def get_share_price(symbol: str) -> float:
    """Return the current share price for a given ticker symbol."""

    fake_prices = {
        "AAPL": 241.5,
        "GOOG": 168.2,
        "AMZN": 198.0,
    }

    return fake_prices.get(symbol.upper(), 0.0)
```

`@tool` แปลง Python Function ให้กลายเป็น Object ที่มี:

```text
name
description
argument schema
invoke()
```

แหล่งที่มาของ Metadata:

```text
Function name
→ Tool name

Docstring
→ Tool description

Type hints
→ Argument schema
```

นี่คล้าย `@function_tool` ใน OpenAI Agents SDK. 

ตรวจได้ด้วย:

```python
print(get_share_price.name)
print(get_share_price.description)
print(get_share_price.args)
```

---

## Tool Description สำคัญมาก

โมเดลไม่ได้อ่าน Source Code ของ Function

สิ่งที่โมเดลเห็นคือ Schema ประมาณ:

```json
{
  "name": "get_share_price",
  "description": "Return the current share price...",
  "parameters": {
    "symbol": {
      "type": "string"
    }
  }
}
```

ดังนั้น Docstring ที่กำกวมทำให้ Model เลือก Tool หรือสร้าง Arguments ผิดได้

แต่ต้องจำว่า:

```text
Tool description
= Guidance สำหรับ LLM

Tool implementation
= Authority และ Validation จริง
```

Security ไม่ควรฝากไว้ใน Docstring

---

# 11. Calling a Tool Directly

```python
result = get_share_price.invoke({
    "symbol": "AAPL"
})
```

นี่คือ Application เรียก Tool เองโดยตรง

```text
Application
→ Tool
→ Result
```

ยังไม่มี LLM เข้ามาตัดสินใจ

ส่วน Tool-using Agent จะเป็น:

```text
User request
→ LLM proposes tool call
→ Application validates and executes
→ Result returns to LLM
```

---

# 12. `bind_tools()`

```python
llm_with_tools = llm.bind_tools([
    get_share_price
])
```

สิ่งที่เกิดขึ้น:

```text
Original model
+
Tool schemas
=
Model configured to request tools
```

`bind_tools()` แปลง Tool Definitions เป็น Provider Tool Schema แล้วแนบ Schema ไปกับ Model Invocation. ([Docs by LangChain][4])

สิ่งที่ **ไม่เกิดขึ้น**:

```text
Tool ไม่ได้ถูกรันทันที
ไม่มี Agent Loop
ไม่มี Tool Dispatcher
ไม่มี Retry
ไม่มี Permission Check
ไม่มี State หรือ Memory
```

ดังนั้น:

> `bind_tools()` สอนให้โมเดลรู้ว่ามี Tools อะไร แต่ไม่ได้มอบมือให้โมเดลรัน Tools เอง

---

# 13. Model Requests a Tool

```python
response = llm_with_tools.invoke(
    "What is the share price of Amazon?"
)

print(response.content)
print(response.tool_calls)
```

คำตอบอาจมีลักษณะ:

```python
{
    "name": "get_share_price",
    "args": {
        "symbol": "AMZN"
    },
    "id": "call_...",
    "type": "tool_call"
}
```

Model ไม่ได้คืนราคาหุ้น แต่คืน **คำขอให้ Application เรียก Tool**

```text
Model output
= Proposed action

Application
= Authorized executor
```

LangChain แปลง Tool Requests จาก Provider ต่าง ๆ ให้อยู่ในรูปมาตรฐานผ่าน `AIMessage.tool_calls`. 

---

# 14. Manual Tool Loop

Notebook ทำดังนี้:

```python
conversation = [
    HumanMessage(
        "What is the share price of Amazon?"
    )
]

ai_message = llm_with_tools.invoke(conversation)
conversation.append(ai_message)

for call in ai_message.tool_calls:
    if call["name"] == "get_share_price":
        result = get_share_price.invoke(call["args"])

        conversation.append(
            ToolMessage(
                content=str(result),
                tool_call_id=call["id"],
            )
        )

final = llm_with_tools.invoke(conversation)

print(final.content)
```

---

## Execution Flow

```text
1. HumanMessage
        ↓
2. Model sees available tools
        ↓
3. AIMessage requests get_share_price
        ↓
4. Application reads tool_calls
        ↓
5. Application runs Python function
        ↓
6. ToolMessage stores result
        ↓
7. Model sees original request + tool call + tool result
        ↓
8. Model generates final natural-language answer
```

นี่คือ Agent Loop ระดับพื้นฐานที่เคยทำใน Week 1 แต่ครั้งนี้ใช้ LangChain Message และ Tool Abstractions. 

---

# 15. ทำไมต้อง Append `AIMessage`

โค้ดต้องเก็บ:

```python
conversation.append(ai_message)
```

เพราะ History ต้องมีหลักฐานว่า Model เคยขอ Tool Call ใด

ลำดับที่ Provider คาดหวัง:

```text
Human message
AI tool request
Tool result
AI final response
```

ถ้าส่งเฉพาะ:

```text
Human message
Tool result
```

Model จะไม่รู้ว่า Tool Result ตอบ Tool Request ใด

---

# 16. `tool_call_id`

```python
ToolMessage(
    content=str(result),
    tool_call_id=call["id"],
)
```

`tool_call_id` ทำหน้าที่เหมือน Tracking Number

ตัวอย่าง Model ขอหลาย Tools:

```text
call_001 → get_share_price(AAPL)
call_002 → get_weather(Bangkok)
```

ผลลัพธ์ต้องเชื่อมกลับให้ถูก:

```text
Tool result AAPL → call_001
Weather Bangkok → call_002
```

หาก ID ผิด Model อาจเชื่อม Result กับ Request ผิด หรือ Provider ปฏิเสธ Message Sequence

---

# 17. `for` กับ `while`

Notebook ใช้:

```python
for call in ai_message.tool_calls:
```

ส่วนนี้รองรับ **หลาย Tool Calls ในหนึ่ง Model Response**

ตัวอย่าง:

```text
AIMessage
├── Tool call A
└── Tool call B
```

แต่ Notebook ยังไม่ใช่ Agent Loop แบบสมบูรณ์ เพราะหลังส่ง Tool Results แล้วเรียก Model อีกเพียงครั้งเดียว:

```python
final = llm_with_tools.invoke(conversation)
```

ถ้า `final` ขอ Tool เพิ่ม ระบบจะหยุด

Agent Loop ที่สมบูรณ์ต้องมี:

```python
while True:
    response = llm_with_tools.invoke(conversation)
    conversation.append(response)

    if not response.tool_calls:
        break

    for call in response.tool_calls:
        ...
```

Mental model:

```text
for
= หลาย Tool Calls ในรอบเดียว

while
= หลาย Model–Tool Rounds
```

LangChain `create_agent()` ซึ่งจะเรียนใน Day 3 มี Agent Loop สำเร็จรูปที่ทำงานต่อจน Model ให้ Final Output หรือถึง Stop Condition. ([Docs by LangChain][2])

---

# 18. จุดอ่อนของ Dispatcher ปัจจุบัน

Notebook ใช้:

```python
if call["name"] == "get_share_price":
```

เมื่อมี Tools มากขึ้นจะกลายเป็น:

```python
if ...
elif ...
elif ...
```

วิธีที่ดีกว่า:

```python
tool_registry = {
    get_share_price.name: get_share_price,
}

tool_name = call["name"]

if tool_name not in tool_registry:
    raise ValueError(
        f"Unknown tool: {tool_name}"
    )

selected_tool = tool_registry[tool_name]
result = selected_tool.invoke(call["args"])
```

ข้อดี:

```text
เพิ่ม Tool ง่าย
ตรวจ Unknown Tool ได้
รวม Validation ได้
เพิ่ม Permission Policy ได้
ทำ Logging ได้
```

---

# 19. Robust Manual Loop

โครงสร้างที่ใกล้ Production กว่า:

```python
tools = [
    get_share_price,
]

tool_registry = {
    current_tool.name: current_tool
    for current_tool in tools
}

model = llm.bind_tools(tools)

conversation = [
    HumanMessage(
        "What is the share price of Amazon?"
    )
]

max_rounds = 5

for _ in range(max_rounds):
    response = model.invoke(conversation)
    conversation.append(response)

    if not response.tool_calls:
        print(response.content)
        break

    for call in response.tool_calls:
        tool_name = call["name"]

        if tool_name not in tool_registry:
            tool_result = (
                f"Error: unknown tool {tool_name}"
            )
        else:
            try:
                tool_result = tool_registry[
                    tool_name
                ].invoke(call["args"])
            except Exception as exc:
                tool_result = (
                    f"Tool execution failed: {exc}"
                )

        conversation.append(
            ToolMessage(
                content=str(tool_result),
                tool_call_id=call["id"],
            )
        )
else:
    raise RuntimeError(
        "Maximum tool rounds exceeded"
    )
```

เพิ่มสิ่งที่ Notebook ขั้นพื้นฐานยังไม่มี:

```text
Tool registry
Unknown tool handling
Exception handling
Maximum rounds
Repeated tool rounds
```

---

# 20. Structured Output

```python
class Company(BaseModel):
    name: str = Field(
        description="The company name"
    )
    ticker: str = Field(
        description="The stock ticker symbol"
    )
    founded_year: int = Field(
        description="The year the company was founded"
    )
```

จากนั้น:

```python
structured_llm = llm.with_structured_output(
    Company
)

company = structured_llm.invoke(
    "Tell me about Amazon the technology company"
)
```

ผลลัพธ์ไม่ใช่ String แต่เป็น Object:

```python
Company(
    name="Amazon",
    ticker="AMZN",
    founded_year=1994,
)
```

จึงเข้าถึงได้ด้วย:

```python
company.ticker
```

Notebook ใช้ Pydantic เพื่อกำหนด Output Contract ระหว่าง Model กับ Application. 

---

# 21. `with_structured_output()` ทำอะไร

Conceptual flow:

```text
Pydantic model
        ↓
JSON schema
        ↓
Model constrained/requested to fill schema
        ↓
Response parsed
        ↓
Pydantic validation
        ↓
Typed Python object
```

เอกสารปัจจุบันระบุว่า `with_structured_output()` สามารถใช้ Native JSON Schema ของ Provider หรือ Function-calling mechanism ตามการตั้งค่าและ Provider. ([Docs by LangChain][4])

---

# 22. Structured Output ไม่ใช่ Ground Truth

Pydantic ตรวจได้ว่า:

```text
name เป็น str
ticker เป็น str
founded_year เป็น int
```

แต่ไม่ตรวจว่า:

```text
ชื่อบริษัทถูกหรือไม่
Ticker มีอยู่จริงหรือไม่
ปีที่ก่อตั้งถูกหรือไม่
ข้อมูลเป็นปัจจุบันหรือไม่
```

ดังนั้น:

```text
Schema valid
≠ Factually correct
```

หากข้อมูลสำคัญ ต้องเพิ่ม:

```text
Database lookup
Source retrieval
Cross-validation
Deterministic business rules
Human review
```

---

# 23. Layer 1 ให้และไม่ให้อะไร

## สิ่งที่ได้

```text
Unified model interface
Messages
Streaming
Tools
Tool schemas
Tool-call representation
Structured output
```

## สิ่งที่ยังต้องเขียนเอง

```text
Tool loop
State
Memory
Retries
Timeout
Routing
Max iterations
Checkpointing
Human approval
Observability policy
```

นี่จึงเป็น Building Blocks Layer ไม่ใช่ Agent Framework สำเร็จรูป. 

---

# 24. เชื่อมโยงกับ Week ก่อนหน้า

## OpenAI Agents SDK

```text
Runner
จัดการ Tool Loop ให้
```

## CrewAI

```text
Crew + Process
จัดการ Task Orchestration
```

## LangChain Layer 1

```text
เราจัดการ Loop เอง
```

Notebook ตั้งใจให้รู้สึก “หยาบกว่า” Framework ชั้นสูง เพื่อให้เห็นว่า Framework กำลังซ่อนอะไรไว้. 

---

# 25. Misconceptions ที่ต้องแก้

### “LangChain คือ Agent Framework อย่างเดียว”

ไม่ใช่ คำว่า LangChain ครอบคลุมทั้ง Core Building Blocks และ Agent Abstraction หลายระดับ

### “`bind_tools()` รัน Tool”

ไม่รัน มันเพียงส่ง Tool Schemas ให้ Model รู้จัก

### “LLM เรียก Python Function เอง”

ไม่ใช่ Model สร้าง Tool Request แล้ว Application เป็นผู้ Execute

### “`tool_calls` คือผลลัพธ์ของ Tool”

ไม่ใช่ เป็นคำขอให้ Tool ทำงาน

### “`ToolMessage` คือข้อความจากผู้ใช้”

ไม่ใช่ เป็นผลลัพธ์จาก Tool ที่เชื่อมกับ Tool Request ผ่าน `tool_call_id`

### “`for` loop ใน Notebook คือ Agent Loop สมบูรณ์”

ไม่ใช่ มันรองรับหลาย Calls ในรอบเดียว แต่ยังไม่มีหลาย Model–Tool Rounds

### “Structured Output ทำให้ข้อมูลถูกต้อง”

ไม่ใช่ มันควบคุม Schema

### “LangGraph มาแทน LangChain”

ไม่ใช่ LangGraph เป็น Orchestration Runtime ระดับต่ำ และ LangChain `create_agent()` ปัจจุบันทำงานบน LangGraph Runtime. ([Docs by LangChain][2])

---

# 26. Exercise ของ Lab

Notebook ให้สร้าง Tool ที่สอง เช่น:

```python
@tool
def get_weather(city: str) -> str:
    """Return a fictional weather report for a city."""

    reports = {
        "Bangkok": "Sunny, 34°C",
        "London": "Cloudy, 16°C",
    }

    return reports.get(
        city.title(),
        "Weather unavailable",
    )
```

จากนั้น Bind ทั้งสอง Tools:

```python
tools = [
    get_share_price,
    get_weather,
]

llm_with_tools = llm.bind_tools(tools)
```

แล้วถามคำถามที่ต้องใช้ทั้งสอง:

```text
What is Amazon's share price,
and what is the fictional weather
in Bangkok?
```

เป้าหมายไม่ใช่เพียงให้ได้คำตอบ แต่ให้สังเกตว่า:

```text
Model ขอ Tools กี่ตัว
ขอพร้อมกันหรือทีละตัว
Arguments ถูกหรือไม่
Tool call IDs ต่างกันหรือไม่
หลังรับผลแล้ว Model ขอ Tool เพิ่มหรือไม่
```

Notebook กำหนดให้ทำ Tool Loop ด้วยตนเองจนกว่า Model จะให้ Final Answer. 

---

# 27. Checklist ก่อนจบ Lab

ตอบคำถามเหล่านี้ให้ได้:

### `ChatOpenAI` คือ Model หรือ Agent?

เป็น Model Adapter ไม่ใช่ Agent Loop

### `invoke()` คืนอะไร?

โดยทั่วไปคืน `AIMessage`

### `stream()` คืนอะไร?

คืน Message Chunks ระหว่าง Generation

### `@tool` ใช้อะไรสร้าง Schema?

Function name, docstring และ type hints

### `bind_tools()` ทำอะไร?

ผูก Tool Schemas กับ Model

### ใครเป็นคน Execute Tool?

Application Code

### Tool Request อยู่ที่ไหน?

```python
ai_message.tool_calls
```

### Tool Result ส่งกลับอย่างไร?

ผ่าน `ToolMessage`

### ทำไมต้องมี `tool_call_id`?

เพื่อเชื่อม Result กับ Tool Request ที่ถูกต้อง

### Structured Output ใช้อะไร?

Pydantic Model

### Pydantic รับประกันความจริงหรือไม่?

ไม่ รับประกัน Type และ Structure เป็นหลัก

### Lab นี้ใช้ LangGraph หรือยัง?

ยัง เป็น Layer 1 Building Blocks

---

# แก่นของ Week 4 Lab 1

```text
Model
→ เสนอคำตอบหรือ Action

Tool Schema
→ บอก Model ว่าทำอะไรได้

Application
→ Execute Action จริง

ToolMessage
→ ส่ง Observation กลับ

Loop
→ ทำซ้ำจนได้ Final Answer

Pydantic
→ ทำ Output ให้อยู่ในรูปที่ระบบอ่านได้
```

บทเรียนสำคัญที่สุดคือ:

> **Agent ไม่ใช่เพียง Model ที่รู้จัก Tools แต่เป็นระบบ Loop ที่เชื่อม Model Decision, Application Execution และ Environment Feedback เข้าด้วยกัน**

และ:

> **LangChain Layer 1 ให้ภาษากลางสำหรับ Model, Messages, Tools และ Structured Output แต่ยังปล่อยให้เราควบคุม Tool Loop เอง เพื่อให้เข้าใจชัดเจนว่า Agent Framework ชั้นสูงกำลังทำงานอะไรแทนเราอยู่**

[1]: https://raw.githubusercontent.com/ed-donner/agents/main/pyproject.toml "raw.githubusercontent.com"
[2]: https://docs.langchain.com/oss/python/langchain/agents?utm_source=chatgpt.com "Agents - Docs by LangChain"
[3]: https://docs.langchain.com/oss/python/langchain/overview?utm_source=chatgpt.com "LangChain overview - Docs by LangChain"
[4]: https://docs.langchain.com/oss/python/integrations/chat/openai "ChatOpenAI integration - Docs by LangChain"
