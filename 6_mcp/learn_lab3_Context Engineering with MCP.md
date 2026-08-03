# Week 6 — Lab 3: Context Engineering with MCP

Notebook:

```text
6_mcp/3_lab3.ipynb
```

Lab 1 สอนวิธี **ใช้ MCP Server** ส่วน Lab 2 สอนวิธี **สร้าง MCP Server** ของตัวเอง

Lab 3 ยกระดับขึ้นอีกขั้น โดยไม่ได้มอง MCP เป็นเพียงช่องทางเรียก Tools แต่ใช้ MCP เพื่อประกอบ **Context Architecture** ให้ Agent:

```text
Long-term memory
+ Fresh web information
+ Agentic RAG
+ Live external integrations
```

Notebook แบ่งเนื้อหาเป็นสี่ส่วนตาม Context Sources เหล่านี้อย่างชัดเจน.

---

## Learning Objectives

หลังจบ Lab นี้ควรเข้าใจว่า:

1. Context Engineering ต่างจาก Prompt Engineering อย่างไร
2. Knowledge Graph Memory ทำให้ข้อมูลอยู่ข้าม Agent Runs ได้อย่างไร
3. Web Search ช่วยแก้ปัญหาความรู้ล้าสมัยของ Model อย่างไร
4. Tool Filtering เป็นส่วนหนึ่งของ Context Engineering อย่างไร
5. Agentic RAG ต่างจาก Traditional RAG อย่างไร
6. Agent ใช้ Web Search และ Vector Store ร่วมกันอย่างไร
7. Qdrant MCP เก็บและค้น Semantic Knowledge อย่างไร
8. Knowledge Graph Memory ต่างจาก Vector Memory อย่างไร
9. Live Integration ต่างจาก Web Search อย่างไร
10. Massive MCP และ Local Market Server สลับใช้งานกันอย่างไร
11. Context ที่มากขึ้นอาจทำให้ระบบแย่ลงได้อย่างไร
12. Memory Poisoning, Staleness และ Prompt Injection เกิดขึ้นที่จุดใด
13. Trace ช่วยตรวจ Context Selection และ Tool Use อย่างไร
14. Production Context Layer ต้องมี Metadata, Provenance และ Expiry อย่างไร

---

# 1. Context Engineering คืออะไร

Prompt Engineering มักเน้นว่า:

```text
ควรเขียนคำสั่งอย่างไร
เพื่อให้ Model ตอบดีขึ้น
```

Context Engineering มองกว้างกว่า:

```text
ข้อมูลอะไรควรถูกส่งเข้า Model
เมื่อใด
จากที่ใด
ในรูปแบบใด
และควรให้ Model เห็น Tools อะไรบ้าง
```

Mental model:

```text
Prompt Engineering
= เขียนคำสั่งให้ดี

Context Engineering
= จัดสภาพแวดล้อมข้อมูลให้ Model ตัดสินใจได้ดี
```

Lab นี้ประกอบ Context จากสี่แหล่ง:

```text
1. Long-term memory
2. Web search
3. Agentic RAG
4. Live service integration
```

สิ่งสำคัญคือ Context Engineering ไม่ได้หมายถึงส่งข้อมูลให้มากที่สุด

```text
More context
≠ Better context
```

Context ที่ดีควร:

```text
Relevant
Current
Trustworthy
Small enough
Properly scoped
Traceable
```

---

# 2. Imports สำคัญ

```python
from agents import Agent, Runner, trace

from agents.mcp import (
    MCPServerStdio,
    create_static_tool_filter,
)

from pathlib import Path
from datetime import datetime
```

องค์ประกอบใหม่ที่สำคัญคือ:

```python
create_static_tool_filter
```

ใช้จำกัดว่า Agent จะเห็น Tools ใดจาก MCP Server

นี่เป็น Context Engineering เพราะ Tool Definitions เองก็เป็นส่วนหนึ่งของ Context ที่ Model ได้รับ.

---

# 3. Windows MCP Workaround

เหมือน Labs ก่อนหน้า Notebook Patch `stdio_client` เพื่อส่ง `stderr` ของ MCP Server ไปที่ `DEVNULL` บน Windows/Jupyter:

```python
agents.mcp.server.stdio_client = functools.partial(
    agents.mcp.server.stdio_client,
    errlog=subprocess.DEVNULL,
)
```

จุดประสงค์คือหลีกเลี่ยง:

```text
io.UnsupportedOperation: fileno
```

ข้อดี:

```text
MCP Server เริ่มได้ใน Windows Notebook
```

ข้อเสีย:

```text
Startup errors ถูกซ่อน
Debugging ยากขึ้น
เป็นการแก้ Internal Library Function
```

Production ควรส่ง Errors ไป Log File หรือ Structured Logging แทนการทิ้งทั้งหมด

---

# Part 1 — Long-term Memory

# 4. ปัญหาของ Conversation Memory

Agent ปกติจำข้อมูลได้เฉพาะสิ่งที่อยู่ใน Conversation Context

```text
Conversation starts
→ Agent knows messages

Conversation ends
→ Context disappears
```

การสร้าง Agent Object ใหม่หรือเริ่ม Session ใหม่อาจทำให้ข้อมูลเดิมหายไป

Lab จึงเพิ่ม Persistent Memory:

```text
Agent
→ Memory MCP Server
→ Disk-backed knowledge graph
```

---

# 5. Knowledge Graph Memory Server

Notebook ใช้ Official MCP Memory Server:

```text
@modelcontextprotocol/server-memory
```

กำหนดตำแหน่งจัดเก็บ:

```python
memory_path = os.path.abspath(
    "memory/memory.json"
)
```

และส่งผ่าน Environment Variable:

```python
memory_params = {
    "command": "npx",
    "args": [
        "-y",
        "@modelcontextprotocol/server-memory",
    ],
    "env": {
        "MEMORY_FILE_PATH": memory_path
    },
}
```

Memory Server เก็บ Entities, Observations และ Relations ลง `memory/memory.json` เพื่อให้ข้อมูลอยู่ข้าม MCP Process และ Agent Runs.

---

# 6. Knowledge Graph Mental Model

ตัวอย่างข้อมูล:

```text
Entity:
Ed

Observations:
- Is an LLM engineer
- Teaches a course about AI Agents
- Teaches MCP

Relations:
Ed
→ teaches
→ AI Agent course
```

โครงสร้าง:

```text
Entity
├── Observation
├── Observation
└── Relation → another entity
```

ต่างจากการเก็บ Conversation ทั้งหมด เพราะระบบพยายามเก็บ **ข้อเท็จจริงที่สกัดแล้ว**

---

# 7. Discover Memory Tools

```python
async with MCPServerStdio(
    params=memory_params,
    client_session_timeout_seconds=60,
) as server:
    memory_tools = await server.list_tools()
```

Memory Server อาจเปิด Tools สำหรับ:

```text
Create entities
Add observations
Create relations
Search nodes
Open nodes
Read graph
Delete data
```

Tool Names จริงควรตรวจจาก `memory_tools` เพราะอาจเปลี่ยนตาม Package Version

หลักเดิมยังเหมือน Lab 1:

```text
Connect
→ List tools
→ Inspect capabilities
→ Give server to Agent
```

---

# 8. Writing Memory

Instructions:

```python
instructions = """
You use your entity tools as a persistent memory
to store and recall information about your conversations.
"""
```

ข้อมูลที่ให้ Agent:

```text
My name is Ed.
I'm an LLM engineer.
I'm teaching a course about AI Agents.
The course includes MCP.
```

Agent Run:

```python
async with MCPServerStdio(...) as mcp_server:
    agent = Agent(
        name="agent",
        instructions=instructions,
        model="gpt-5.4-mini",
        mcp_servers=[mcp_server],
    )

    result = await Runner.run(
        agent,
        request,
    )
```

Model ต้องตัดสินใจเองว่า:

```text
ข้อมูลใดควรเก็บ
ควรสร้าง Entity อะไร
ควรเพิ่ม Observation ใด
ควรสร้าง Relation หรือไม่
```

---

# 9. Reading Memory in a New Run

Notebook ปิด MCP Context แล้วเปิดใหม่ จากนั้นสร้าง Agent ใหม่และถาม:

```text
My name's Ed. What do you know about me?
```

Agent สามารถค้นข้อมูลจาก `memory/memory.json` ที่ Agent ก่อนหน้าเขียนไว้.

สิ่งนี้พิสูจน์ว่า:

```text
Agent object
≠ Memory owner

Conversation
≠ Memory storage

Persistent file
= Long-term memory state
```

---

# 10. Memory Lifecycle

```text
First agent run
→ Learn fact
→ Store entity/observation
→ Write memory.json
→ MCP process stops

Second agent run
→ New MCP process
→ Read same memory.json
→ Search graph
→ Recall fact
```

Memory จึงอยู่ข้าม:

```text
Agent instances
MCP process instances
Conversation runs
```

ตราบใดที่ File เดิมยังอยู่

---

# 11. Memory Is Not Automatically Correct

Model เป็นผู้ตัดสินใจว่าจะเก็บอะไร

จึงอาจเกิด:

```text
เก็บข้อมูลผิด
สรุปเกินสิ่งที่ผู้ใช้พูด
สร้าง Entity ซ้ำ
เชื่อม Relation ผิด
เก็บข้อมูลชั่วคราวเป็นข้อเท็จจริงถาวร
```

ตัวอย่าง:

```text
User:
I'm temporarily working in Phuket.

Bad memory:
Ed lives permanently in Phuket.
```

นี่คือ **Memory Distortion**

---

# 12. Memory Poisoning

หากข้อมูลที่ Agent อ่านมาจากแหล่งที่ไม่น่าเชื่อถือ แล้ว Agent เก็บข้อมูลนั้นลง Long-term Memory:

```text
Untrusted content
→ Agent interprets as fact
→ Persistent memory
→ Future decisions use false fact
```

เรียกว่า:

```text
Memory poisoning
```

อันตรายกว่า Hallucination ชั่วคราว เพราะข้อมูลผิดถูกนำกลับมาใช้ในอนาคต

Production ควรเก็บ Metadata เช่น:

```text
Source
Created time
Last verified time
Confidence
Owner
Expiry
Evidence
```

---

# 13. Knowledge Graph เหมาะกับข้อมูลแบบใด

เหมาะกับ:

```text
บุคคล
องค์กร
ความสัมพันธ์
Preference
Role
เหตุการณ์
ข้อเท็จจริงที่มี Entity ชัดเจน
```

ตัวอย่าง:

```text
Ed
→ works as
→ LLM engineer
```

ไม่เหมาะนักกับการเก็บบทความยาวทั้งฉบับ เพราะ Knowledge Graph ต้องสกัดเป็น Entities และ Relationships ก่อน

---

# Part 2 — Web Search

# 14. ทำไมต้องมี Web Search

Model มี Knowledge Cutoff และไม่รู้เหตุการณ์ที่เกิดหลัง Training

```text
Model knowledge
= Static snapshot

Web search
= Fresh external context
```

Lab ใช้ Tavily ซึ่งมี MCP Server สำหรับงาน Search และส่งผลลัพธ์ที่จัดรูปแบบให้ Agent ใช้งานง่าย.

Environment Variable:

```env
TAVILY_API_KEY=tvly-...
```

---

# 15. Tavily MCP Server

```python
tavily_params = {
    "command": "npx",
    "args": [
        "-y",
        "tavily-mcp@latest",
    ],
    "env": {
        "TAVILY_API_KEY":
            os.getenv("TAVILY_API_KEY")
    },
}
```

เปิดและตรวจ Tools:

```python
async with MCPServerStdio(
    params=tavily_params,
    client_session_timeout_seconds=60,
) as server:
    tavily_tools = await server.list_tools()
```

Tavily Server เปิดหลาย Capabilities เช่น Search, Extract, Crawl, Map และ Research แต่ Lab ต้องการเพียง Web Search.

---

# 16. Tool Filtering

สร้าง Static Filter:

```python
search_only = create_static_tool_filter(
    allowed_tool_names=[
        "tavily_search"
    ]
)
```

จากนั้น:

```python
MCPServerStdio(
    params=tavily_params,
    tool_filter=search_only,
)
```

ผลคือ Agent เห็นเพียง:

```text
tavily_search
```

ไม่เห็น Tools ที่หนักกว่า เช่น Crawl หรือ Deep Research.

นี่เป็น Context Engineering เพราะ Tool Catalog คือส่วนหนึ่งของ Context

```text
All Tavily tools
→ More capability
→ More choices
→ Larger context
→ More confusion and cost

Search only
→ Smaller decision surface
→ More predictable behavior
```

---

# 17. Tool Filtering ไม่ใช่ Security Boundary ทั้งหมด

Tool Filter ช่วยจำกัด Tools ที่ SDK เปิดให้ Agent

แต่ไม่ได้แก้:

```text
API key permissions
Remote server trust
Prompt injection ในผล Search
Data privacy
Source quality
```

จึงควรมองว่าเป็น:

```text
Capability curation
```

ไม่ใช่ Complete Authorization System

---

# 18. Web-search Task

Instructions:

```text
Search the web and briefly summarize the takeaways.
```

Task:

```text
Research the latest news on Amazon stock price
and briefly summarize its outlook.
```

Notebook ใส่วันที่ปัจจุบันจาก:

```python
datetime.now().strftime("%Y-%m-%d")
```

ลงใน Request ด้วย.

เหตุผลคือคำว่า:

```text
latest
recent
today
```

มีความหมายขึ้นกับเวลา

การใส่วันที่ช่วยให้ Model และ Search Query มี Temporal Anchor ที่ชัดเจน

---

# 19. Expected Web-search Loop

```text
Receive research request
        ↓
Call tavily_search
        ↓
Receive ranked results
        ↓
Read titles, snippets and sources
        ↓
Possibly search again
        ↓
Synthesize outlook
        ↓
Return concise summary
```

Agent ไม่ควรตอบจาก Model Knowledge อย่างเดียว เพราะ Request ระบุข้อมูลล่าสุด

---

# 20. Search Result Is Not Ground Truth

Search อาจคืน:

```text
ข่าวเก่า
บทความซ้ำ
บทความ SEO
ความคิดเห็น
ข่าวลือ
ข้อมูลคนละวันที่
```

Agent ยังต้อง:

```text
เปรียบเทียบวันที่
แยกข่าวกับความคิดเห็น
ตรวจหลายแหล่ง
ระบุความไม่แน่นอน
```

โดยเฉพาะข้อมูลหุ้น:

```text
News
≠ Price data
≠ Financial statement
≠ Investment recommendation
```

---

# 21. Tool Curation as Context Engineering

แนวคิดสำคัญของ Part 2 คือ:

```text
Context Engineering
ไม่ได้เลือกเพียง Documents

แต่เลือกด้วยว่า
Model ควรมี Tools อะไร
```

ตัวอย่าง:

```text
Task:
Summarize recent news

Required:
Search

Not required:
Crawl entire site
Map site structure
Run deep research
```

การไม่ให้ Tool ที่ไม่จำเป็นช่วยลด:

```text
Token usage
Latency
Tool-selection errors
Unexpected side effects
```

---

# Part 3 — Agentic RAG

# 22. Traditional RAG

Traditional RAG มักมี Pipeline:

```text
Documents
→ Chunk
→ Embed
→ Vector database
→ User query
→ Retrieve chunks
→ Model answer
```

Knowledge Base ถูกสร้างไว้ล่วงหน้าโดย Application หรือ Data Pipeline

---

# 23. Agentic RAG

Lab นิยาม Agentic RAG ว่า Agent เป็นผู้ตัดสินใจเองว่า:

```text
ควรค้นอะไร
ข้อมูลใดควรเก็บ
ควรเก็บเมื่อใด
ควรค้น Knowledge Base เมื่อใด
```

Flow:

```text
Agent researches
→ Selects important facts
→ Stores them
→ Later searches them
→ Answers from stored knowledge
```

Notebook ใช้ Tavily สำหรับ Research และ Qdrant MCP สำหรับ Vector Storage.

---

# 24. Qdrant Local Vector Store

```python
vectordb_path = Path(
    "memory/qdrant"
)
```

Server Parameters:

```python
vectorstore_params = {
    "command": "uvx",
    "args": [
        "mcp-server-qdrant"
    ],
    "env": {
        "QDRANT_LOCAL_PATH":
            str(vectordb_path),

        "COLLECTION_NAME":
            "knowledge",
    },
}
```

Qdrant ทำงาน Local โดยเก็บข้อมูลไว้ที่:

```text
memory/qdrant/
```

และใช้ Collection:

```text
knowledge
```

---

# 25. Vector Tools

Notebook ระบุว่า Qdrant MCP เปิดความสามารถหลักสองแบบ:

```text
Store text
Find relevant text
```

Mental model:

```text
Store
→ Text
→ Embedding
→ Vector database

Find
→ Query embedding
→ Similarity search
→ Relevant stored text
```

ครั้งแรกอาจช้า เพราะ Local Embedding Model ต้องถูกดาวน์โหลดและเริ่มใช้งาน

---

# 26. Research-and-store Agent

Agent ได้รับ MCP Servers สองตัว:

```text
Tavily Search
+
Qdrant Vector Store
```

Instructions:

```text
Research topics on the web.
Store information worth keeping.
When asked what you know,
search the knowledge base.
```

Task:

```text
Research the latest news on Nvidia
and store the key facts
in your knowledge base.
```

Run ถูกจำกัดด้วย:

```python
max_turns=20
```

---

# 27. Expected Agentic RAG Loop

```text
Receive Nvidia research task
        ↓
Call Tavily search
        ↓
Read current results
        ↓
Select important facts
        ↓
Call Qdrant store tool
        ↓
Store facts as semantic text
        ↓
Possibly repeat search/store
        ↓
Return completion summary
```

สิ่งสำคัญคือ Application ไม่ได้ Preload Documents ให้

Agent เป็นผู้สร้าง Corpus เอง

---

# 28. Retrieval-only Agent

หลัง Research Run Notebook เปิด Agent ใหม่ที่มีเพียง:

```text
Qdrant MCP Server
```

ไม่มี Tavily Search

จากนั้นถาม:

```text
Based on your knowledge base,
what's the latest on Nvidia?
```

Agent จึงต้องตอบจากสิ่งที่ Agent ก่อนหน้าเก็บไว้เท่านั้น.

นี่แยกสองขั้นออกจากกัน:

```text
Acquisition
→ Research and store

Retrieval
→ Search stored knowledge and answer
```

---

# 29. Agentic RAG vs Web Search

## Web Search

```text
สดกว่า
ค้นจาก Internet ทุกครั้ง
ขึ้นกับ Network และ Search API
```

## Vector Knowledge Base

```text
ค้นเร็วจากข้อมูลที่เก็บแล้ว
อยู่ข้าม Agent Runs
ไม่ต้องค้นเว็บทุกครั้ง
แต่อาจล้าสมัย
```

Pattern ที่สมเหตุสมผล:

```text
Search when freshness is required
Store durable insights
Retrieve before searching again
Refresh when data expires
```

---

# 30. Knowledge Graph vs Vector Store

| ประเด็น        | Knowledge Graph Memory        | Vector Store               |
| -------------- | ----------------------------- | -------------------------- |
| หน่วยข้อมูล    | Entity, observation, relation | Text chunk                 |
| วิธีค้น        | Entity/relation lookup        | Semantic similarity        |
| โครงสร้าง      | Explicit                      | Implicit                   |
| เหมาะกับ       | บุคคล ความสัมพันธ์ Preference | ข่าว บทความ Notes          |
| การอธิบาย      | เข้าใจโครงสร้างง่าย           | ดูเหตุผล Retrieval ยากกว่า |
| Duplicate risk | Duplicate entities            | Duplicate chunks           |
| Persistence    | `memory.json`                 | Qdrant local storage       |

Mental model:

```text
Knowledge graph
= จำว่าใครเกี่ยวข้องกับอะไร

Vector store
= จำข้อความใดมีความหมายใกล้กับคำถาม
```

---

# 31. Agentic RAG Risk

Agent เป็นผู้เลือกข้อมูลที่จะเก็บ

จึงอาจ:

```text
เก็บข้อมูลผิด
ไม่เก็บข้อเท็จจริงสำคัญ
เก็บความคิดเห็นเป็นข้อเท็จจริง
ไม่เก็บแหล่งที่มา
เก็บข่าวเก่า
เก็บข้อมูลซ้ำ
```

จากนั้น Retrieval จะทำให้ข้อมูลผิดนั้นกลับมาใช้อีก

```text
Bad research
→ Bad storage
→ Confident retrieval
→ Repeated wrong answer
```

---

# 32. Missing Provenance

Task บอกให้เก็บ “key facts” แต่ไม่ได้บังคับให้เก็บ:

```text
URL
Publisher
Published date
Retrieved date
Quote
Confidence
Expiry
```

Production ควรเก็บ Metadata พร้อม Text:

```json
{
  "text": "Nvidia announced...",
  "source_url": "...",
  "published_at": "...",
  "retrieved_at": "...",
  "topic": "NVDA",
  "confidence": 0.8,
  "expires_at": "..."
}
```

เพื่อให้ตรวจสอบและ Refresh ได้

---

# 33. “Latest” ไม่ควรถูกเก็บแบบถาวร

ข้อความเช่น:

```text
The latest Nvidia news is...
```

จะหมดอายุอย่างรวดเร็ว

ควรเก็บพร้อม:

```text
As-of date
Published date
Expiry
```

หรือเปลี่ยนข้อความเป็น:

```text
On 2026-08-03, source X reported...
```

เพื่อไม่ทำให้ข้อมูลเก่าถูกตีความว่าเป็นข้อมูลปัจจุบันในอนาคต

---

# Part 4 — Live Integrations

# 34. Search กับ Integration ต่างกันอย่างไร

Web Search ให้:

```text
บทความ
ข่าว
ความคิดเห็น
Search snippets
```

Live Integration ให้:

```text
ข้อมูลจากระบบเฉพาะทาง
ผ่าน API หรือ Business Service
```

Lab ใช้ Market Data จาก Massive เป็นตัวอย่าง Integration.

---

# 35. Massive MCP

หากมี:

```env
MASSIVE_API_KEY=...
```

Notebook กำหนด:

```python
market_params = {
    "command": "uvx",
    "args": [
        "--from",
        "git+https://github.com/"
        "massive-com/mcp_massive@v0.10.0",
        "mcp_massive",
    ],
    "env": {
        "MASSIVE_API_KEY":
            massive_api_key
    },
}
```

Course Pin Server ที่ Version:

```text
v0.10.0
```

แทนการใช้ `latest` ในส่วนนี้.

---

# 36. Local Fallback

หากไม่มี Massive API Key:

```python
market_params = {
    "command": "uv",
    "args": [
        "run",
        "-m",
        "backend.market_server",
    ],
}
```

Local Server ใช้ FastMCP และเปิด Tool:

```python
lookup_share_price(
    symbol: str
) -> float
```

ซึ่งเรียก `get_share_price(symbol)` จาก Market Domain Layer.

---

# 37. Market Fallback Architecture

```text
MASSIVE_API_KEY exists?
    ├── Yes
    │    ↓
    │ Massive MCP Server
    │    ↓
    │ Real market provider
    │
    └── No
         ↓
      Local market_server
         ↓
      Market simulator
```

ข้อดี:

```text
Lab ทำงานได้แม้ไม่มี External API Key
```

นี่คือ **Interface-level Fallback**

Agent ยังคงได้รับ Market Tool ผ่าน MCP แม้ Implementation ด้านหลังเปลี่ยนไป

---

# 38. Integration Task

Instructions:

```text
You answer questions about the stock market.
```

Request:

```text
What was the most recent price
that Apple (AAPL) traded at?
```

Model:

```text
gpt-5.4-mini
```

Agent เชื่อมกับ Market MCP Server แล้วใช้ Tool เพื่อหาคำตอบ.

---

# 39. Expected Integration Loop

```text
User asks AAPL price
        ↓
Model recognizes live data required
        ↓
Call market-data tool
        ↓
Market MCP calls provider
        ↓
Return price
        ↓
Model presents answer
```

นี่ต่างจากตอบด้วย Model Knowledge:

```text
Model memory
→ potentially stale

Market integration
→ provider result
```

---

# 40. Real กับ Simulated Data ต้องแยกให้ชัด

Local Fallback คืน `float` เช่นเดียวกับ Real Provider

Agent อาจไม่รู้ว่า:

```text
Price is real
หรือ
Price is simulated
```

ดังนั้น Public Result ที่ดีกว่าควรเป็น:

```json
{
  "symbol": "AAPL",
  "price": 210.4,
  "source": "massive",
  "mode": "live",
  "as_of": "2026-08-03T..."
}
```

หรือ:

```json
{
  "symbol": "AAPL",
  "price": 210.4,
  "source": "market_simulator",
  "mode": "simulated"
}
```

ไม่ควรคืนเพียงตัวเลขโดยไม่มี Provenance

---

# 41. “Most Recent Price” มีความกำกวม

อาจหมายถึง:

```text
Last trade
Real-time quote
Delayed quote
End-of-day close
Previous close
Simulated price
```

Free Data Plan หรือ Local Fallback อาจไม่ได้ให้ Real-time Last Trade จริง

ดังนั้นคำตอบควรระบุ:

```text
Data source
Timestamp
Market status
Whether delayed
Whether simulated
```

ไม่ควรสรุปเกิน Capability ของ Provider

---

# 42. Four Context Sources Together

```text
Knowledge graph
→ Personal and relational facts

Web search
→ Fresh public information

Vector store
→ Reusable semantic knowledge

Live integration
→ Structured operational data
```

Context แต่ละประเภทแก้ปัญหาคนละแบบ:

| Context source | คำถามที่ตอบ                         |
| -------------- | ----------------------------------- |
| Memory         | “เราเคยรู้อะไรเกี่ยวกับบุคคลนี้?”   |
| Web search     | “โลกภายนอกมีอะไรเกิดขึ้นล่าสุด?”    |
| Vector RAG     | “เราเคยค้นคว้าและเก็บอะไรไว้?”      |
| Integration    | “ระบบจริงมีสถานะหรือค่าอะไรตอนนี้?” |

---

# 43. Context Selection

Agent ไม่ควรใช้ Context ทุกแหล่งทุกครั้ง

ตัวอย่าง:

```text
ถามชื่อผู้ใช้
→ Memory

ถามข่าวล่าสุด
→ Web search

ถามจากงานวิจัยเดิม
→ Vector store

ถามยอดเงินหรือราคาจากระบบ
→ Integration
```

เลือกผิดแหล่งอาจทำให้:

```text
ข้อมูลเก่า
ข้อมูลแพงเกินไป
Latency สูง
คำตอบผิดประเภท
```

Context Engineering จึงเกี่ยวกับ **Routing** ด้วย

---

# 44. Context Budget

ทุก MCP Tool เพิ่ม:

```text
Tool name
Description
Input schema
Result
```

ลงใน Agent Context

เมื่อมี Servers จำนวนมาก:

```text
Tool catalog ใหญ่ขึ้น
→ Tokens มากขึ้น
→ Model เลือกยากขึ้น
→ Latency เพิ่ม
→ Tool collision เพิ่ม
```

แนวทาง:

```text
ใช้ Tool filters
แยก Specialized Agents
โหลด Tools ตาม Task
จำกัด Result size
สรุปข้อมูลก่อนส่ง Model
```

---

# 45. State Surfaces

Lab นี้มี Persistent State หลายชนิด:

```text
memory/memory.json
→ Knowledge graph state

memory/qdrant/
→ Vector knowledge state

Web results
→ Ephemeral external context

Market provider
→ Live external state

OpenAI trace
→ Execution evidence
```

แต่ละ State มี Lifecycle ต่างกัน

---

# 46. State Divergence

ตัวอย่าง:

```text
Knowledge graph บอกว่าผู้ใช้ทำงานตำแหน่งเดิม
แต่ข้อมูลจริงเปลี่ยนแล้ว

Qdrant เก็บข่าวเมื่อเดือนก่อน
แต่ Agent เรียกว่า latest

Search result บอกข่าวใหม่
แต่ Memory ยังเก็บข่าวเก่า

Market simulator คืนราคา
แต่ Agent รายงานเหมือนเป็นราคา Live
```

Production ต้องมี Policy สำหรับ:

```text
Conflict resolution
Freshness
Expiry
Source priority
Revalidation
```

---

# 47. Context Quality Dimensions

ควรประเมิน Context จาก:

```text
Relevance
Freshness
Accuracy
Authority
Completeness
Cost
Latency
Privacy
```

ไม่มี Context Source ที่ดีที่สุดทุกด้าน

ตัวอย่าง:

```text
Web search
→ Fresh
→ แต่ Source quality ไม่แน่นอน

Knowledge graph
→ Structured
→ แต่อาจเก่า

Vector store
→ Semantic retrieval ดี
→ แต่ Explainability ต่ำกว่า

Live API
→ Structured
→ แต่อาจล่มหรือมีค่าใช้จ่าย
```

---

# 48. Observability

ทุก Agent Run ใช้:

```python
with trace(...):
```

Trace ช่วยดู:

```text
Agent เลือก Context source ใด
Tool ใดถูกเรียก
Query อะไรถูกส่ง
ข้อมูลอะไรถูกเก็บ
ข้อมูลใดถูก Retrieve
ผลลัพธ์จาก Provider คืออะไร
```

สิ่งที่ควรตรวจใน Trace:

```text
Memory run
→ Agent stored facts or not

Search run
→ Only tavily_search was exposed

RAG run
→ Search happened before store

Retrieval run
→ No web search was available

Market run
→ Tool was called rather than guessed
```

---

# 49. Trace ไม่ได้พิสูจน์ Context Quality

Trace แสดงว่า:

```text
Tool ถูกเรียก
Result ถูกส่งกลับ
```

แต่ไม่พิสูจน์ว่า:

```text
ข้อมูลถูกต้อง
Source น่าเชื่อถือ
ข้อมูลควรถูกเก็บ
Retrieval ตรงกับคำถาม
Market data เป็น Live
```

ต้องเสริมด้วย:

```text
Source metadata
Deterministic validation
Freshness checks
Domain rules
Human review
```

---

# 50. Security Risks

## Memory Poisoning

ข้อมูลผิดถูกเก็บระยะยาว

## Search Prompt Injection

หน้าเว็บพยายามสั่ง Agent

## Vector-store Poisoning

Agent เก็บข้อความอันตรายหรือข้อมูลผิดใน RAG Corpus

## Privacy Leakage

ข้อมูลส่วนตัวถูกเก็บใน Memory โดยไม่มี Consent

## Remote MCP Exposure

API Keys และ Queries ถูกส่งไป External Service

## Excessive Tools

Agent มี Capability มากเกิน Task

---

# 51. Package-version Risks

Notebook ใช้ Packages บางตัวแบบ:

```text
tavily-mcp@latest
server-memory without explicit version
mcp-server-qdrant without explicit version
```

Run ในอนาคตอาจได้รับ:

```text
Tool names ใหม่
Schemas ใหม่
Behavior ใหม่
Breaking changes
```

ส่วน Massive Server ถูก Pin ที่ `v0.10.0`

Production ควร Pin ทุก Server Version ที่ผ่านการทดสอบแล้ว

---

# 52. Common Misconceptions

### “Context Engineering คือใส่ Context ให้เยอะที่สุด”

ไม่จริง ต้องเลือกข้อมูลที่เกี่ยวข้องและเหมาะกับ Task

### “Long-term Memory คือ Conversation History”

ไม่เหมือนกัน Memory เป็นข้อมูลที่สกัดและ Persist แยกจาก Messages

### “Knowledge Graph กับ Vector Store เหมือนกัน”

ไม่เหมือน Knowledge Graph เก็บ Structure ส่วน Vector Store เก็บ Semantic Similarity

### “Agentic RAG ดีกว่า Traditional RAG เสมอ”

ไม่จริง Agent อาจเลือกและเก็บข้อมูลผิด

### “Search Result เป็นข้อเท็จจริง”

ไม่จริง Search เป็นเพียง Candidate Sources

### “Tool Filter ทำให้ Server ปลอดภัยทั้งหมด”

ไม่จริง Filter ลด Exposed Tools แต่ไม่แทน Authentication และ Trust Review

### “MCP Integration คืนข้อมูลจริงเสมอ”

ไม่จริง Lab มี Local Simulator Fallback

### “Latest ใน Vector Store หมายถึงล่าสุดจริง”

ไม่จริง ต้องดูวันที่และ Refresh Policy

---

# 53. Production Improvements

```text
Pin every MCP server version
Store source and timestamp metadata
Use per-user memory isolation
Require consent for personal memory
Add memory update and deletion workflows
Filter MCP tools per task
Validate search sources
Store citations with vector records
Add document expiry
Deduplicate stored knowledge
Expose live/simulated mode
Return timestamps with market data
Capture MCP logs
Use correlation IDs
Add tool-call and token budgets
```

---

# 54. Exercises

## Exercise 1 — Inspect Memory File

เปิด:

```text
memory/memory.json
```

ตรวจ:

```text
Entities ถูกสร้างอย่างไร
Observations ตรงกับคำพูดหรือไม่
Relations ถูกต้องหรือไม่
มีข้อมูลเกินสิ่งที่ผู้ใช้พูดหรือไม่
```

---

## Exercise 2 — Memory Correction

บอก Agent:

```text
I am no longer an LLM engineer.
I now work as a product researcher.
```

ตรวจว่า Agent:

```text
เพิ่มข้อมูลใหม่
แก้ข้อมูลเดิม
หรือเก็บข้อเท็จจริงที่ขัดกันทั้งคู่
```

---

## Exercise 3 — Tool Filter

ทดลองให้ Tavily Server เปิดทุก Tools แล้วเปรียบเทียบกับ `search_only`

วัด:

```text
จำนวน Tool Definitions
Token usage
Tool selection
Latency
Calls ที่ไม่จำเป็น
```

---

## Exercise 4 — RAG Metadata

เปลี่ยนข้อมูลที่เก็บเป็นรูปแบบ:

```text
Fact
Source URL
Published date
Retrieved date
```

จากนั้นถาม Agent ให้แสดง Source ของทุก Claim

---

## Exercise 5 — Staleness Test

เก็บข้อมูลหนึ่งชุดลง Qdrant แล้วค้น Web ใหม่

ให้ Agent เปรียบเทียบ:

```text
Stored knowledge
vs
Current search results
```

และ Update เฉพาะสิ่งที่เปลี่ยน

---

## Exercise 6 — Market Provenance

แก้ `market_server.py` ให้คืน:

```json
{
  "symbol": "AAPL",
  "price": 100,
  "mode": "simulated",
  "source": "local_market_server",
  "as_of": "..."
}
```

เพื่อไม่ให้ Agent รายงาน Simulated Price ว่าเป็น Live Price

---

# Checklist

```text
□ Memory MCP Server เปิดได้
□ memory/memory.json ถูกสร้าง
□ Agent เขียน Entity และ Observation ได้
□ Agent ใหม่อ่าน Memory เดิมได้
□ TAVILY_API_KEY ถูกโหลด
□ Tavily MCP list_tools() ได้
□ Tool filter เปิดเฉพาะ tavily_search
□ Search Request มีวันที่กำกับ
□ Qdrant MCP เปิดได้
□ memory/qdrant ถูกสร้าง
□ Agent ค้น Web และ Store Knowledge ได้
□ Retrieval Agent ใช้ Qdrant โดยไม่มี Web Search
□ เข้าใจ Graph Memory กับ Vector Memory
□ Massive Server หรือ Local Fallback เปิดได้
□ Market Agent เรียก Tool แทนการเดาราคา
□ ตรวจ Source ว่า Live หรือ Simulated
□ ตรวจ Trace ของทั้งสี่ Parts
```

---

# แก่นของ Week 6 — Lab 3

```text
Knowledge Graph MCP
= Persistent structured memory

Tavily MCP
= Fresh public information

Qdrant MCP
= Persistent semantic knowledge

Market MCP
= Live operational integration

Tool filter
= Context and capability curation

Trace
= Evidence of context selection
```

บทเรียนสำคัญที่สุดคือ:

> **Context Engineering ไม่ใช่การเขียน Prompt ให้ยาวขึ้น แต่คือการออกแบบว่าข้อมูลและความสามารถชนิดใดควรเข้าสู่ Agent ในแต่ละช่วงของงาน**

อีกบทเรียนคือ:

> **MCP ทำให้ Context Sources ที่ต่างกันมาก—Memory, Search, Vector Database และ Live API—ถูกนำเสนอให้ Agent ผ่าน Interface รูปแบบเดียวกันได้**

และแก่นเชิง Production คือ:

> **Context ที่ Agent ใช้ต้องมี Source, Timestamp, Scope, Ownership และ Expiry เพราะ Persistent Context ที่ผิดหรือล้าสมัยอาจอันตรายกว่าการไม่มี Context เลย**
