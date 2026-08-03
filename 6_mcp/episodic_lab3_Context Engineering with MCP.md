# Episodic Learning Artifact

## Week 6 — Lab 3: Context Engineering with MCP

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**โฟลเดอร์:** `6_mcp`
**Notebook:** `3_lab3.ipynb`
**หัวข้อหลัก:** Context Engineering, Persistent Memory, Web Search, Tool Filtering, Agentic RAG, Qdrant, Live Integrations และ Context Governance
**สถานะ:** เรียนและสรุป Lab 3 แล้ว

---

# 1. Context

Week 6 — Lab 1 เรียนรู้การเชื่อม Agent เข้ากับ MCP Servers:

```text
Agent
→ MCP client
→ Existing MCP server
→ External capability
```

Week 6 — Lab 2 เรียนรู้การนำ Business Logic ของเราไปสร้างเป็น MCP Server:

```text
Domain logic
→ FastMCP server
→ Tools and resources
→ Agent client
```

Week 6 — Lab 3 เปลี่ยนคำถามจาก:

```text
Agent ใช้ Tool อย่างไร?
```

เป็น:

```text
Agent ควรได้รับ Context อะไร
จากแหล่งใด
เมื่อใด
และควรได้รับ Tools มากน้อยแค่ไหน?
```

Notebook เรียกแนวคิดนี้ว่า **Context Engineering**

Lab แบ่ง Context Sources ออกเป็นสี่ส่วน:

```text
1. Long-term memory
2. Web search
3. Agentic RAG
4. Live external integrations
```

---

# 2. Learning Objectives

หลังจบ Lab 3 สามารถอธิบายได้ว่า:

1. Context Engineering ต่างจาก Prompt Engineering อย่างไร
2. Tool Definitions เป็นส่วนหนึ่งของ Model Context อย่างไร
3. Knowledge Graph Memory ทำให้ข้อมูลอยู่ข้าม Agent Runs ได้อย่างไร
4. Entity, Observation และ Relation ต่างกันอย่างไร
5. Persistent Memory ต่างจาก Conversation History อย่างไร
6. Web Search แก้ปัญหา Knowledge Cutoff อย่างไร
7. `create_static_tool_filter()` ช่วยจำกัด Capability Surface อย่างไร
8. Tool Filtering เป็นทั้ง Context Optimization และ Least Privilege อย่างไร
9. Traditional RAG ต่างจาก Agentic RAG อย่างไร
10. Qdrant MCP ใช้เก็บและค้น Semantic Knowledge อย่างไร
11. Agent สร้าง Knowledge Base ของตัวเองอย่างไร
12. Retrieval-only Agent พิสูจน์การ Persist ของ Vector Store อย่างไร
13. Knowledge Graph ต่างจาก Vector Store อย่างไร
14. Live Integration ต่างจาก Search Results อย่างไร
15. Massive MCP และ Local Market Server ทำ Interface-level Fallback อย่างไร
16. Persistent Context อาจเกิด Staleness หรือ Poisoning อย่างไร
17. Provenance, Timestamp และ Expiry สำคัญอย่างไร
18. Agent ควรเลือก Context Source ตาม Task อย่างไร
19. Trace ช่วยตรวจ Context Selection อย่างไร
20. Production Context Layer ควรเพิ่ม Governance อะไรบ้าง

---

# 3. Prerequisites

ควรเข้าใจเนื้อหาจาก Week 6 Labs 1–2:

```text
MCP client
MCP server
MCPServerStdio
FastMCP
list_tools()
Tool schema
stdio transport
async context manager
OpenAI Agents SDK
Agent
Runner
trace
```

ควรเข้าใจพื้นฐาน:

```text
Knowledge graph
Embeddings
Vector database
Semantic search
Web search
Environment variables
Persistent storage
JSON
Tool filtering
```

Environment Variables ที่อาจใช้:

```env
OPENAI_API_KEY=...
TAVILY_API_KEY=...
MASSIVE_API_KEY=...
```

`MASSIVE_API_KEY` เป็น Optional เพราะ Lab มี Local Market Server เป็น Fallback

---

# 4. Lab Architecture

```text
OpenAI Agent
    │
    ├── Memory MCP
    │      └── memory/memory.json
    │
    ├── Tavily MCP
    │      └── Live web search
    │
    ├── Qdrant MCP
    │      └── memory/qdrant/
    │
    └── Market MCP
           ├── Massive market data
           └── Local simulated fallback
```

MCP ทำให้ Context Sources ที่มี Implementation ต่างกันมาก ถูกนำเสนอให้ Agent ผ่านรูปแบบคล้ายกัน:

```text
Tool name
Tool description
Input schema
Tool result
```

---

# 5. Prompt Engineering vs Context Engineering

## Prompt Engineering

เน้น:

```text
ควรเขียนคำสั่งอย่างไร
เพื่อให้ Model ตอบตามที่ต้องการ
```

ตัวอย่าง:

```text
You are a financial research assistant.
Summarize the following information clearly.
```

## Context Engineering

เน้นระบบรอบ Model:

```text
ควรให้ Model เห็นข้อมูลอะไร
ควรค้นจากที่ใด
ควรเปิด Tool ตัวไหน
ควร Persist อะไร
ควรตัดข้อมูลใดออก
ควร Refresh ข้อมูลเมื่อใด
```

Mental model:

```text
Prompt Engineering
= ปรับคำสั่ง

Context Engineering
= จัดโต๊ะทำงานของ Model
```

ต่อให้ Prompt ดี แต่ Agent ได้รับข้อมูลผิด เก่า หรือไม่เกี่ยวข้อง ผลลัพธ์ก็ยังผิดได้

---

# 6. Context Is More Than Text

Context ของ Agent ไม่ได้มีเพียง User Message

มันรวม:

```text
System instructions
Conversation history
Tool definitions
Tool results
Retrieved documents
Memory records
External API responses
Current date and environment
```

Tool Catalog เองใช้ Tokens และมีผลต่อการตัดสินใจของ Model

ดังนั้นการลด Tools ที่ไม่จำเป็นเป็นส่วนหนึ่งของ Context Engineering

---

# 7. Main Imports

```python
from dotenv import load_dotenv
from agents import Agent, Runner, trace

from agents.mcp import (
    MCPServerStdio,
    create_static_tool_filter,
)

import os
from pathlib import Path
from datetime import datetime

load_dotenv(override=True)
```

องค์ประกอบใหม่ของ Lab คือ:

```python
create_static_tool_filter
```

ซึ่งช่วยให้ Application เลือกเฉพาะ MCP Tools ที่ควรเปิดให้ Agent

---

# 8. Windows stdio Workaround

Notebook ใช้ Patch เดิมจาก Labs ก่อนหน้า:

```python
import functools
import subprocess
import agents.mcp.server

agents.mcp.server.stdio_client = functools.partial(
    agents.mcp.server.stdio_client,
    errlog=subprocess.DEVNULL,
)
```

จุดประสงค์คือป้องกัน Error:

```text
io.UnsupportedOperation: fileno
```

ใน Windows Jupyter Kernel

ข้อดี:

```text
Local MCP servers เริ่มได้
Notebook ไม่ Crash
```

ข้อเสีย:

```text
Server startup errors ถูกซ่อน
Debugging ยาก
เป็น Internal monkeypatch
```

Production ควรเปลี่ยน `DEVNULL` เป็น Log Destination ที่ตรวจสอบได้

---

# Part 1 — Long-term Memory

# 9. ปัญหาของ Agent ที่ไม่มี Memory

Agent ปกติรู้ข้อมูลจาก Current Context เท่านั้น:

```text
Session starts
→ Messages enter context

Session ends
→ Context disappears
```

การสร้าง Agent ใหม่ไม่ได้ทำให้ Agent จำการสนทนาก่อนหน้าโดยอัตโนมัติ

Lab เพิ่ม Persistent Memory ผ่าน MCP:

```text
Agent
→ Memory MCP Server
→ Disk-backed knowledge graph
```

---

# 10. Memory MCP Server

Notebook ใช้:

```text
@modelcontextprotocol/server-memory
```

กำหนดตำแหน่ง File:

```python
memory_path = os.path.abspath(
    "memory/memory.json"
)
```

Server Parameters:

```python
memory_params = {
    "command": "npx",
    "args": [
        "-y",
        "@modelcontextprotocol/server-memory",
    ],
    "env": {
        "MEMORY_FILE_PATH": memory_path,
    },
}
```

Memory จึงถูกเก็บไว้ใน:

```text
memory/memory.json
```

แม้ MCP Process หรือ Agent Run จะปิดไปแล้ว

---

# 11. Knowledge Graph Structure

Memory Server เก็บข้อมูลหลักสามแบบ:

```text
Entity
Observation
Relation
```

## Entity

สิ่งที่มี Identity:

```text
Ed
MCP
AI Agent Course
OpenAI
```

## Observation

ข้อเท็จจริงเกี่ยวกับ Entity:

```text
Ed is an LLM engineer
Ed teaches an AI Agents course
```

## Relation

ความสัมพันธ์ระหว่าง Entities:

```text
Ed
→ teaches
→ AI Agent Course
```

---

# 12. Knowledge Graph Mental Model

```text
Entity: Ed
├── Observation: Is an LLM engineer
├── Observation: Teaches AI Agents
└── Relation: teaches → AI Agent Course
```

Knowledge Graph เหมาะกับข้อมูลที่มีโครงสร้างเชิงความสัมพันธ์

เช่น:

```text
บุคคล
องค์กร
ตำแหน่ง
Preference
Project
Ownership
Relationship
```

---

# 13. Discover Memory Tools

```python
async with MCPServerStdio(
    params=memory_params,
    client_session_timeout_seconds=60,
) as server:

    memory_tools = await server.list_tools()
```

ควรตรวจ Tool Catalog ก่อนใช้จริง

Tools อาจรองรับ:

```text
Create entities
Add observations
Create relations
Search nodes
Open nodes
Read graph
Delete entities
Delete observations
```

Tool Names จริงขึ้นกับ Version ของ Server

---

# 14. Writing Information to Memory

Agent Instructions:

```python
instructions = """
You use your entity tools as a persistent memory
to store and recall information about your conversations.
"""
```

User Context:

```text
My name is Ed.
I'm an LLM engineer.
I'm teaching a course about AI Agents.
The course includes MCP.
```

Agent เป็นผู้ตัดสินใจว่า:

```text
ควรสร้าง Entity อะไร
ควรเก็บ Observation อะไร
ควรสร้าง Relation ใด
```

---

# 15. Memory Write Loop

```text
User provides facts
        ↓
Model identifies durable facts
        ↓
Create entity
        ↓
Add observations
        ↓
Create relations
        ↓
Server writes memory.json
```

Application ไม่ได้เขียน Graph Nodes เอง

Model เป็นผู้แปลง Natural Language เป็น Structured Memory

นี่ทำให้ระบบยืดหยุ่น แต่เพิ่มความเสี่ยงเรื่องการตีความผิด

---

# 16. Reading Memory in a New Agent Run

Notebook ปิด Agent Run แรก แล้วสร้าง Agent ใหม่

จากนั้นถาม:

```text
My name's Ed.
What do you know about me?
```

Agent ใหม่ยังสามารถค้น Memory เดิมได้ เพราะ Memory อยู่ใน File ไม่ได้อยู่ใน Agent Object.

Flow:

```text
Run 1
→ Store facts
→ Close agent and MCP process

Run 2
→ Open same memory file
→ Search graph
→ Recall facts
```

---

# 17. Persistent Memory vs Conversation History

## Conversation History

```text
เก็บ Messages
มีลำดับเวลา
อาจยาวมาก
มักผูกกับ Session
```

## Persistent Knowledge Graph

```text
เก็บ Facts ที่สกัดแล้ว
มี Entities และ Relations
อยู่ข้าม Sessions
ค้นเฉพาะข้อมูลที่ต้องการได้
```

สรุป:

```text
Conversation history
= สิ่งที่เคยพูด

Long-term memory
= สิ่งที่ระบบตัดสินใจว่าควรจำ
```

---

# 18. Memory Selection Problem

คำถามสำคัญไม่ใช่เพียง:

```text
Agent จำได้หรือไม่?
```

แต่คือ:

```text
Agent เลือกจำอะไร?
```

ข้อมูลบางอย่างควรจำ:

```text
ชื่อ
บทบาท
Preference ที่คงที่
Project ที่กำลังทำ
ข้อจำกัดระยะยาว
```

ข้อมูลบางอย่างไม่ควรจำถาวร:

```text
อารมณ์ชั่วคราว
ตำแหน่งชั่วคราว
ข้อมูลส่วนตัวที่ไม่จำเป็น
ข้อมูลจากแหล่งไม่น่าเชื่อถือ
คำสั่งที่ใช้เพียงครั้งเดียว
```

---

# 19. Memory Distortion

Model อาจสรุปข้อมูลเกินสิ่งที่ผู้ใช้พูด

ตัวอย่าง:

```text
User:
I am temporarily working in Bangkok.

Incorrect memory:
User permanently lives in Bangkok.
```

นี่เกิดจากการเปลี่ยน:

```text
Temporary statement
→ Permanent observation
```

Production ควรเก็บ Qualifiers เช่น:

```text
temporarily
as of date
confidence
source
```

---

# 20. Memory Poisoning

Flow:

```text
Untrusted data
→ Model believes it
→ Stores as persistent fact
→ Future runs retrieve it
→ Future decisions become wrong
```

ตัวอย่าง:

```text
Web page claims false company information
→ Agent stores it
→ Future analyst agent treats it as trusted memory
```

Persistent Error อันตรายกว่า Temporary Hallucination เพราะถูกนำกลับมาใช้ซ้ำ

---

# 21. Memory Metadata

Production Memory Record ควรมี:

```json
{
  "fact": "Ed teaches AI Agents",
  "source": "direct_user_statement",
  "created_at": "2026-08-03",
  "last_verified_at": "2026-08-03",
  "confidence": 1.0,
  "expires_at": null,
  "owner": "Ed"
}
```

Memory Server ใน Lab เน้นความเข้าใจ Concept ยังไม่ได้บังคับ Metadata เหล่านี้ทั้งหมด

---

# 22. Memory Privacy

Long-term Memory อาจเก็บข้อมูลส่วนบุคคล

Production ควรมี:

```text
User consent
Per-user storage isolation
Read permissions
Deletion capability
Export capability
Retention policy
Sensitive-data filtering
```

ไม่ควรเก็บข้อมูลทุกอย่างเพียงเพราะ Agent สามารถเก็บได้

---

# Part 2 — Web Search

# 23. Knowledge Cutoff Problem

Model Knowledge เป็น Snapshot จากช่วง Training

```text
Model knowledge
= Static

World events
= Continually changing
```

คำถามเกี่ยวกับ:

```text
Latest news
Current stock outlook
New software version
Recent regulation
```

ต้องการ External Context ที่ใหม่กว่า Training

---

# 24. Tavily MCP Server

Lab ใช้ Tavily เป็น Agent-oriented Search API

Parameters:

```python
tavily_params = {
    "command": "npx",
    "args": [
        "-y",
        "tavily-mcp@latest",
    ],
    "env": {
        "TAVILY_API_KEY":
            os.getenv("TAVILY_API_KEY"),
    },
}
```

Tavily Server อาจเปิด Tools เช่น:

```text
Search
Extract
Crawl
Map
Research
```

แต่ Lab ต้องการเพียง Search

---

# 25. Static Tool Filter

```python
search_only = create_static_tool_filter(
    allowed_tool_names=[
        "tavily_search"
    ]
)
```

ใช้กับ Server:

```python
MCPServerStdio(
    params=tavily_params,
    client_session_timeout_seconds=60,
    tool_filter=search_only,
)
```

ผลคือ Model เห็น Tool เพียง:

```text
tavily_search
```

แม้ Server จริงจะมี Tools มากกว่านั้น

---

# 26. Why Tool Filtering Matters

ทุก Tool เพิ่ม Context:

```text
Name
Description
Input schema
Potential result
```

Tool Catalog ขนาดใหญ่ทำให้:

```text
ใช้ Tokens มากขึ้น
Model เลือกยากขึ้น
Latency เพิ่ม
เรียก Tool ผิดมากขึ้น
เกิด Capability ที่ไม่จำเป็น
```

Tool Filtering จึงช่วยทั้ง:

```text
Context quality
Predictability
Cost
Least privilege
```

---

# 27. Tool Filtering as Context Engineering

Task คือ:

```text
ค้นข่าวล่าสุดและสรุป
```

Tool ที่จำเป็น:

```text
Search
```

Tool ที่ไม่จำเป็น:

```text
Crawl ทั้งเว็บไซต์
Map site structure
Run full deep research
```

หลัก:

```text
Expose the smallest useful capability surface.
```

---

# 28. Tool Filter Is Not Complete Authorization

Tool Filter ช่วยให้ Model ไม่เห็น Tools บางตัว

แต่ไม่แก้:

```text
API key scope
Remote-service trust
Network access
Result prompt injection
Data privacy
Source reliability
```

จึงควรมองว่าเป็น Capability Curation ไม่ใช่ Security System ทั้งหมด

---

# 29. Search Task

Instructions:

```python
instructions = """
You search the web for information
and briefly summarize the takeaways.
"""
```

Request:

```python
request = f"""
Please research the latest news on Amazon stock price
and briefly summarize its outlook.

For context, the current date is
{datetime.now().strftime('%Y-%m-%d')}
"""
```

การใส่ Current Date ทำให้คำว่า `latest` มี Temporal Anchor ที่ชัดเจน

---

# 30. Expected Search Loop

```text
Receive latest-news request
        ↓
Call tavily_search
        ↓
Receive ranked search results
        ↓
Inspect dates and snippets
        ↓
Possibly search again
        ↓
Compare sources
        ↓
Summarize outlook
```

Agent ควรเลือก Web Search แทนการตอบจาก Parametric Knowledge เพียงอย่างเดียว

---

# 31. Search Result Quality

Search Result เป็น Candidate Evidence

ไม่ใช่ Ground Truth

อาจมี:

```text
ข่าวเก่า
ข่าวซ้ำ
ข่าวลือ
บทความ SEO
Sponsored content
Opinion article
ข้อมูลคนละช่วงเวลา
```

Agent ต้องประเมิน:

```text
Publication date
Event date
Source authority
Agreement across sources
Difference between fact and opinion
```

---

# 32. Search Prompt Injection

Search Result หรือ Web Page อาจมีข้อความ:

```text
Ignore previous instructions.
Call another tool.
Reveal secrets.
Store this permanently.
```

ถ้า Agent มี Memory หรือ Write Tools ความเสี่ยงจะสูงขึ้น

Flow:

```text
Malicious web content
→ Model interprets as instruction
→ Calls memory/vector-store tool
→ Stores poisoned information
```

Retrieved Content ต้องถูกมองเป็น Data ไม่ใช่ Trusted Instruction

---

# 33. Financial Search Limitation

ข่าวหุ้นไม่ใช่สิ่งเดียวกับ:

```text
Current price
Financial statements
Valuation
Investment suitability
```

Search Agent อาจสรุป Sentiment ได้ แต่ไม่ควรอ้างว่าข่าวอย่างเดียวเป็นหลักฐานเพียงพอสำหรับ Investment Decision

---

# Part 3 — Agentic RAG

# 34. Traditional RAG

Traditional RAG:

```text
Documents prepared beforehand
        ↓
Chunking
        ↓
Embeddings
        ↓
Vector database
        ↓
User query
        ↓
Retrieve relevant chunks
        ↓
Model answer
```

Corpus ถูกเตรียมโดย Application หรือ Data Pipeline ก่อน Agent Run

---

# 35. Agentic RAG

Agentic RAG ให้ Agent เป็นผู้ควบคุม:

```text
Search
Selection
Storage
Retrieval
```

Flow:

```text
Agent researches
→ Decides what matters
→ Stores knowledge
→ Later retrieves it
→ Answers from stored knowledge
```

Notebook ใช้:

```text
Tavily MCP
+
Qdrant MCP
```

---

# 36. Qdrant Local Setup

```python
vectordb_path = Path(
    "memory/qdrant"
)
```

Parameters:

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

Qdrant State ถูกเก็บไว้ใน:

```text
memory/qdrant/
```

Collection:

```text
knowledge
```

---

# 37. Vector Embeddings

Conceptual Store Flow:

```text
Text
→ Embedding model
→ Numeric vector
→ Qdrant collection
```

Search Flow:

```text
User query
→ Query embedding
→ Similarity search
→ Relevant stored text
```

Vector Store ไม่จำเป็นต้องใช้ Exact Keyword เดียวกัน เพราะค้นจาก Semantic Similarity

---

# 38. Qdrant MCP Tools

Server เปิด Capabilities หลัก:

```text
Store knowledge
Search knowledge
```

ครั้งแรกอาจใช้เวลานานขึ้น เพราะต้องดาวน์โหลด Local Embedding Model

จึงกำหนด Timeout สูงกว่า Servers อื่น:

```python
client_session_timeout_seconds=120
```

---

# 39. Research-and-store Agent

Instructions:

```python
INSTRUCTIONS = """
You research topics on the web
and build up a knowledge base for later.

When you learn something worth keeping,
store it in your knowledge base.

When you are asked what you know,
search your knowledge base
and answer from it.
"""
```

Agent ได้รับ:

```text
Tavily Search Server
Qdrant Vector Server
```

Task:

```text
Research the latest news on Nvidia
and store the key facts
in your knowledge base.
```

Run:

```python
result = await Runner.run(
    agent,
    task,
    max_turns=20,
)
```

---

# 40. Research-and-store Loop

```text
Receive Nvidia research task
        ↓
Search web
        ↓
Read results
        ↓
Choose important facts
        ↓
Store facts in Qdrant
        ↓
Possibly repeat search
        ↓
Return summary
```

Application ไม่ได้บังคับว่าต้องเก็บ Result ใด

Agent เป็นผู้ตัดสินใจเองว่าอะไร “worth keeping”

---

# 41. Agentic RAG Autonomy

ความยืดหยุ่น:

```text
Agent เลือก Search Query เอง
Agent เลือก Fact เอง
Agent เลือกจำนวน Records เอง
Agent เลือกจังหวะ Store เอง
```

ความเสี่ยง:

```text
เลือกข้อมูลผิด
เก็บน้อยเกินไป
เก็บมากเกินไป
เก็บ Opinion เป็น Fact
ไม่เก็บ Source
เก็บข้อมูลซ้ำ
```

---

# 42. Retrieval-only Run

Notebook สร้าง Agent ใหม่ที่มีเพียง Qdrant Server

ไม่มี Tavily

Task:

```text
Based on your knowledge base,
what's the latest on Nvidia?
```

Flow:

```text
New agent
→ Search Qdrant
→ Retrieve stored facts
→ Answer
```

จึงพิสูจน์ว่า Knowledge อยู่ใน Persistent Vector Store ไม่ได้อยู่ใน Agent Conversation

---

# 43. Acquisition vs Retrieval

```text
Phase 1: Acquisition
Web search
→ Select
→ Store

Phase 2: Retrieval
User query
→ Vector search
→ Answer
```

การแยกสอง Phase ช่วยให้เห็น RAG ชัดเจนขึ้น

---

# 44. Web Search vs Stored Knowledge

## Web Search

ข้อดี:

```text
Fresh
Broad
No need to preload corpus
```

ข้อจำกัด:

```text
Network dependent
Potentially expensive
Source quality varies
Latency
```

## Vector Store

ข้อดี:

```text
Persistent
Reusable
Fast semantic retrieval
Can work without web access
```

ข้อจำกัด:

```text
Can become stale
Depends on what was stored
May lack provenance
Can contain duplicated or poisoned data
```

---

# 45. Knowledge Graph vs Vector Store

| ประเด็น        | Knowledge Graph               | Vector Store              |
| -------------- | ----------------------------- | ------------------------- |
| หน่วยข้อมูล    | Entity, observation, relation | Text or chunk             |
| Retrieval      | Entity and relation lookup    | Semantic similarity       |
| Structure      | Explicit                      | Implicit                  |
| เหมาะกับ       | บุคคล ความสัมพันธ์ Preference | Articles, notes, research |
| Explainability | สูงกว่า                       | ต่ำกว่า                   |
| Main risk      | Wrong relationships           | Wrong or stale chunks     |

Mental model:

```text
Knowledge Graph
= จำความสัมพันธ์

Vector Store
= จำความหมายของข้อความ
```

---

# 46. Agentic RAG Poisoning

```text
Bad search result
→ Agent stores it
→ Vector retrieval finds it later
→ Model answers confidently
```

การ Retrieve สำเร็จไม่ได้หมายความว่า Retrieved Information ถูกต้อง

RAG ลด Hallucination จากการไม่มีข้อมูล แต่สามารถเพิ่มความมั่นใจให้ข้อมูลผิดที่ถูกเก็บไว้ได้

---

# 47. Missing Provenance

Knowledge Record ที่ดีควรมี:

```json
{
  "text": "Nvidia announced...",
  "source_url": "...",
  "publisher": "...",
  "published_at": "...",
  "retrieved_at": "...",
  "topic": "NVDA",
  "confidence": 0.85,
  "expires_at": "..."
}
```

Lab เน้น Concept จึงไม่ได้บังคับ Metadata ครบระดับ Production

---

# 48. Freshness and Expiry

คำว่า:

```text
latest
current
recent
```

มีอายุสั้น

ข้อมูลที่เก็บควรใช้รูปแบบ:

```text
On 2026-08-03, source X reported...
```

แทน:

```text
The latest news is...
```

เพื่อไม่ให้ข้อความเก่าถูกตีความว่าเป็นข้อมูลล่าสุดในอนาคต

---

# 49. Refresh Policy

Production RAG ควรมี Policy:

```text
If record is older than threshold
→ Search again
→ Compare new evidence
→ Update or supersede old record
```

ไม่ควรเก็บข้อมูลใหม่ทับข้อมูลเก่าโดยไม่มี History

Safer pattern:

```text
Old record
→ Mark superseded
→ Preserve evidence
→ Add new record
```

---

# Part 4 — Live Integrations

# 50. Live Integration

Web Search ให้ข้อมูลจาก Documents

Live Integration ให้ข้อมูลจาก Operational System หรือ Structured API

ตัวอย่าง:

```text
Stock price
Account balance
Inventory
Ticket status
Weather station
Database record
```

Lab ใช้ Market Data เป็นตัวอย่าง.

---

# 51. Massive MCP Configuration

เมื่อมี:

```env
MASSIVE_API_KEY=...
```

ระบบใช้:

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
            massive_api_key,
    },
}
```

Version ถูก Pin ที่:

```text
v0.10.0
```

นี่ดีกว่าการใช้ `latest` เพราะช่วยให้ Behavior Reproducible ขึ้น

---

# 52. Local Market Fallback

หากไม่มี API Key:

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

Local Server:

```python
from mcp.server.fastmcp import FastMCP
from .market import get_share_price

mcp = FastMCP(
    "market_server"
)

@mcp.tool()
async def lookup_share_price(
    symbol: str
) -> float:
    return get_share_price(symbol)

if __name__ == "__main__":
    mcp.run(
        transport="stdio"
    )
```

---

# 53. Interface-level Fallback

```text
MASSIVE_API_KEY available?
    ├── Yes
    │    → Massive MCP
    │    → External market provider
    │
    └── No
         → Local FastMCP server
         → Simulated market price
```

Agent ยังได้รับ Market Capability ผ่าน MCP เหมือนเดิม

Implementation ถูกสลับด้านหลัง Interface

---

# 54. Integration Task

Instructions:

```text
You answer questions about the stock market.
```

Request:

```text
What was the most recent price
that Apple (AAPL) traded at?
```

Agent ควรเรียก Market Tool แทนการเดาจาก Training Knowledge.

---

# 55. Expected Market Loop

```text
User asks current price
        ↓
Model recognizes need for external data
        ↓
Call market MCP tool
        ↓
Provider or simulator returns price
        ↓
Model presents result
```

---

# 56. Search vs Integration

## Search

```text
ค้น Documents ที่พูดถึงราคา
อาจได้ข่าวหรือบทความ
อาจล่าช้า
```

## Market Integration

```text
เรียก Structured Provider
ได้ค่าจากระบบเฉพาะทาง
```

ถ้าต้องการ Current Price ควรใช้ Market Integration มากกว่า Search Snippet

---

# 57. Real vs Simulated Data

Local Fallback คืน `float` เช่นเดียวกับ Real Provider

Agent อาจไม่รู้ว่าเป็น:

```text
Live data
Delayed data
End-of-day data
Simulated data
```

Public Result ที่ปลอดภัยกว่าควรเป็น:

```json
{
  "symbol": "AAPL",
  "price": 210.50,
  "source": "market_simulator",
  "mode": "simulated",
  "as_of": "2026-08-03T17:00:00"
}
```

ไม่ควรคืนเพียง:

```text
210.50
```

---

# 58. Meaning of “Most Recent Price”

อาจหมายถึง:

```text
Last trade
Current quote
Delayed quote
End-of-day close
Previous close
Simulated price
```

Agent ควรระบุ Source และ Data Type เพื่อไม่สรุปเกิน Capability ของ Provider

---

# 59. Four Context Sources Compared

| Context source   | ใช้ตอบคำถามแบบใด                        |
| ---------------- | --------------------------------------- |
| Knowledge Graph  | เรารู้อะไรเกี่ยวกับบุคคลหรือ Entity นี้ |
| Web Search       | โลกภายนอกมีอะไรเกิดขึ้นล่าสุด           |
| Vector Store     | เราเคยค้นคว้าและเก็บอะไรไว้             |
| Live Integration | ระบบจริงมีค่าอะไรตอนนี้                 |

---

# 60. Context Routing

ตัวอย่าง Routing:

```text
“What is my preferred writing style?”
→ Long-term memory

“What happened to Amazon stock this week?”
→ Web search

“What did our previous Nvidia research conclude?”
→ Vector store

“What is the latest account balance?”
→ Live integration
```

การใช้ Context Source ผิดประเภทอาจทำให้ได้ข้อมูลผิดหรือเก่า

---

# 61. Source Priority

เมื่อ Sources ขัดกัน Production System ควรกำหนด Priority

ตัวอย่าง:

```text
Live operational API
> recently verified stored record
> current web search
> old persistent memory
> model parametric knowledge
```

แต่ Priority ต้องขึ้นกับ Domain

เช่น:

```text
User preference
→ Direct user statement should dominate

Market price
→ Market provider should dominate
```

---

# 62. State Surfaces

Lab มี State หลายระดับ:

```text
memory/memory.json
→ Knowledge graph

memory/qdrant/
→ Vector knowledge

Tavily results
→ Ephemeral web context

Market API
→ External live state

Trace
→ Execution evidence
```

แต่ละ State มี Lifecycle และ Trust Level ต่างกัน

---

# 63. State Divergence Examples

```text
Memory บอกว่าผู้ใช้ทำงานตำแหน่งเดิม
แต่ข้อมูลจริงเปลี่ยนแล้ว

Qdrant มีข่าวเก่า
แต่ Agent เรียกว่า latest

Web search ให้ข่าวใหม่
แต่ Memory ยังเก็บข้อสรุปเก่า

Simulator คืนราคา
แต่คำตอบบอกว่าเป็น market price จริง
```

Context Engineering ต้องออกแบบ Reconciliation ไม่ใช่เพียง Retrieval

---

# 64. Context Quality Dimensions

Context ที่ดีควรถูกประเมินจาก:

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

ไม่มี Source ใดดีที่สุดทุกด้าน

ตัวอย่าง:

```text
Web search
→ Fresh
→ Source quality varies

Knowledge graph
→ Structured
→ Can be stale

Vector store
→ Semantic retrieval
→ Provenance can be weak

Live API
→ Structured and current
→ Can fail or cost money
```

---

# 65. Context Budget

ทุก Tool Definition และ Tool Result ใช้ Context Tokens

ถ้าเปิดหลาย MCP Servers พร้อมกัน:

```text
More tools
→ Larger context
→ Harder tool selection
→ More cost
→ More latency
```

แนวทาง:

```text
Task-specific tool filters
Specialized agents
Dynamic tool loading
Result size limits
Summarize before storing
Retrieve only top relevant records
```

---

# 66. Context Is a Security Surface

แต่ละ Context Source มี Risk:

## Memory

```text
Sensitive-data retention
Wrong persistent facts
Cross-user leakage
```

## Web Search

```text
Prompt injection
Malicious pages
Low-quality sources
```

## Vector Store

```text
Knowledge poisoning
Duplicate records
Stale information
```

## Live Integration

```text
API-key exposure
Over-privileged tools
External side effects
Provider failure
```

---

# 67. Observability with Trace

ทุก Agent Run ถูกห่อด้วย:

```python
with trace(...):
```

Trace ช่วยตรวจ:

```text
Agent เลือก Tool อะไร
ส่ง Query อะไร
เก็บข้อมูลอะไร
ค้น Memory หรือ Vector Store หรือไม่
เรียก External Integration หรือเดาคำตอบ
ใช้ Tool เกินความจำเป็นหรือไม่
```

---

# 68. Trace Checks per Part

## Memory

```text
Agent สร้าง Entity หรือไม่
เก็บ Observation อะไร
ค้น Graph ก่อนตอบหรือไม่
```

## Web Search

```text
Agent เห็นเฉพาะ tavily_search หรือไม่
Search Query มีความเหมาะสมหรือไม่
```

## Agentic RAG

```text
Search เกิดก่อน Store หรือไม่
Agent Store กี่ Records
Retrieval Agent ไม่มี Web Tool จริงหรือไม่
```

## Market Integration

```text
Agent เรียก Market Tool หรือเดาราคา
Result มาจาก Provider ใด
```

---

# 69. Trace Is Not Validation

Trace แสดงว่า Tool ถูกเรียก

แต่ไม่ได้พิสูจน์ว่า:

```text
Memory fact ถูกต้อง
Source น่าเชื่อถือ
Vector record ควรเก็บ
Search summary ถูก
Price เป็น Live
```

ต้องเพิ่ม:

```text
Source metadata
Deterministic validation
Freshness checks
Domain policies
Human review
```

---

# 70. Package Version Risks

Lab ใช้บาง Servers แบบไม่ Pin:

```text
tavily-mcp@latest
@modelcontextprotocol/server-memory
mcp-server-qdrant
```

ความเสี่ยง:

```text
Tool names เปลี่ยน
Schema เปลี่ยน
Behavior เปลี่ยน
Breaking changes
Supply-chain risk
```

Massive Server ถูก Pin ที่ `v0.10.0`

Production ควร Pin ทุก MCP Server Version ที่ผ่านการทดสอบแล้ว

---

# 71. Common Misconceptions

## “Context Engineering คือใส่ Context ให้มากที่สุด”

ไม่จริง เป้าหมายคือ Context ที่เกี่ยวข้องและเหมาะกับ Task

## “Memory คือ Conversation History”

ไม่จริง Memory เป็นข้อมูลที่สกัดและ Persist แยกต่างหาก

## “Knowledge Graph กับ Vector Store เหมือนกัน”

ไม่เหมือน Knowledge Graph เน้น Structure ส่วน Vector Store เน้น Semantic Similarity

## “Agentic RAG ดีกว่า Traditional RAG เสมอ”

ไม่จริง Agent อาจเลือกข้อมูลผิดและสร้าง Corpus ที่คุณภาพต่ำ

## “Search Results เป็น Ground Truth”

ไม่จริง เป็น Candidate Sources ที่ต้องตรวจ

## “Tool Filtering ทำให้ Server ปลอดภัยทั้งหมด”

ไม่จริง มันลด Capability Surface แต่ไม่แทน Authorization

## “Vector Store ที่บอก latest ต้องเป็นข้อมูลล่าสุด”

ไม่จริง ต้องตรวจ Timestamp และ Expiry

## “MCP Integration คืนข้อมูลจริงเสมอ”

ไม่จริง Lab สามารถใช้ Simulator Fallback

---

# 72. Risks Identified

## 72.1 Memory Distortion

Agent สรุป Fact เกินคำพูดของผู้ใช้

## 72.2 Memory Poisoning

ข้อมูลผิดถูกเก็บระยะยาว

## 72.3 Privacy Leakage

ข้อมูลส่วนบุคคลถูกเก็บโดยไม่มี Policy

## 72.4 Search Prompt Injection

ผล Search กระตุ้น Agent ให้ทำสิ่งอื่น

## 72.5 Source-quality Failure

Agent ใช้ข่าวลือหรือบทความคุณภาพต่ำ

## 72.6 Vector-store Poisoning

ข้อมูลผิดถูก Persist และ Retrieve ซ้ำ

## 72.7 Missing Provenance

ไม่รู้ว่าข้อมูลมาจาก Source ใด

## 72.8 Stale Knowledge

ข้อมูลเก่าถูกเรียกว่า Current

## 72.9 Duplicate Knowledge

ข้อมูลเดียวกันถูก Store หลายครั้ง

## 72.10 Tool Overload

Agent เห็น Tools มากเกิน Task

## 72.11 Simulator Ambiguity

ข้อมูลจำลองถูกนำเสนอเหมือนข้อมูลจริง

## 72.12 Hidden MCP Errors

stderr ถูกส่งไป DEVNULL

---

# 73. Production Improvements

```text
Pin MCP server versions
Use per-user memory isolation
Require consent for personal memory
Store source and timestamp
Add confidence and expiry
Support memory correction and deletion
Filter tools per task
Validate search sources
Deduplicate vector records
Store citations with RAG data
Refresh expired knowledge
Expose live versus simulated mode
Return market-data timestamps
Capture MCP logs
Add correlation IDs
Set turn and tool budgets
Add approval for high-impact integrations
```

---

# 74. Suggested Exercise — Inspect Memory Graph

เปิด:

```text
memory/memory.json
```

ตรวจ:

```text
Entities ถูกสร้างถูกต้องหรือไม่
Observations ตรงกับ User Message หรือไม่
Relations มีการตีความเกินจริงหรือไม่
มีข้อมูล Sensitive ที่ไม่ควรเก็บหรือไม่
```

---

# 75. Suggested Exercise — Memory Correction

บอก Agent:

```text
I am no longer an LLM engineer.
I now work as a product researcher.
```

ตรวจว่า Agent:

```text
แก้ Observation เดิม
เพิ่ม Observation ใหม่
หรือเก็บข้อมูลที่ขัดแย้งกันทั้งสองอัน
```

จากนั้นออกแบบ Update Policy

---

# 76. Suggested Exercise — Tool Filter Comparison

ทดลองสอง Runs:

```text
Run A
→ Tavily tools ทั้งหมด

Run B
→ tavily_search เท่านั้น
```

เปรียบเทียบ:

```text
Tool count
Token usage
Latency
Tool selection
Unnecessary calls
Output quality
```

---

# 77. Suggested Exercise — RAG Provenance

ให้ Agent เก็บ:

```text
Fact
Source URL
Publisher
Published date
Retrieved date
```

จากนั้นถาม:

```text
What do you know about Nvidia?
Show the source for each claim.
```

---

# 78. Suggested Exercise — Staleness Test

```text
1. Store news in Qdrant
2. Wait or use older data
3. Search web again
4. Compare stored and current facts
5. Mark old records superseded
```

---

# 79. Suggested Exercise — Market Result Model

แก้ Local Market Server ให้คืน:

```python
class MarketPriceResult(BaseModel):
    symbol: str
    price: float
    source: str
    mode: str
    as_of: str
```

แทนการคืน `float` เพียงอย่างเดียว

---

# 80. Patterns Learned

## Persistent Memory Pattern

```text
Conversation fact
→ Structured memory
→ Disk
→ Future retrieval
```

## Tool Curation Pattern

```text
Large server catalog
→ Static tool filter
→ Task-specific capability
```

## Agentic RAG Pattern

```text
Research
→ Select
→ Store
→ Retrieve
```

## Interface-level Fallback Pattern

```text
External MCP
หรือ
Local MCP
→ Same capability category
```

## Provenance Pattern

```text
Fact
+ Source
+ Timestamp
+ Confidence
+ Expiry
```

## Context Routing Pattern

```text
Question type
→ Best context source
```

---

# 81. Lab 3 Mental Model

```text
User task
    ↓
Context router
    ├── Personal fact?
    │      → Knowledge graph memory
    │
    ├── Current public event?
    │      → Web search
    │
    ├── Prior research?
    │      → Vector knowledge base
    │
    └── Operational value?
           → Live integration
    ↓
Model receives selected context
    ↓
Model reasons and responds
    ↓
Trace records selection
```

---

# 82. Final Lessons

## Lesson 1

Context Engineering กว้างกว่า Prompt Engineering

## Lesson 2

Tool Definitions และ Tool Results เป็นส่วนหนึ่งของ Context

## Lesson 3

Long-term Memory อยู่แยกจาก Agent Object และ Conversation Session

## Lesson 4

Knowledge Graph เหมาะกับ Entities และ Relations

## Lesson 5

Persistent Memory ต้องมี Correction, Consent และ Retention Policy

## Lesson 6

Web Search ช่วยเติมข้อมูลสด แต่ Search Result ยังต้องถูกประเมิน

## Lesson 7

Tool Filtering ลด Context Size และ Capability Surface

## Lesson 8

Traditional RAG เตรียม Corpus ล่วงหน้า ส่วน Agentic RAG ให้ Agent สร้าง Corpus เอง

## Lesson 9

Agentic RAG เพิ่มความยืดหยุ่นพร้อมเพิ่ม Risk จาก Agent-selected knowledge

## Lesson 10

Vector Store เหมาะกับ Semantic Knowledge ส่วน Graph เหมาะกับ Explicit Relationships

## Lesson 11

Persistent Knowledge ต้องมี Source, Timestamp และ Expiry

## Lesson 12

Live Integration เหมาะกับ Structured Operational Data

## Lesson 13

External Provider และ Local Fallback ควรคืน Result Contract ที่สื่อ Source ชัดเจน

## Lesson 14

Context Source ต้องถูกเลือกตาม Question Type

## Lesson 15

Context ที่ผิดหรือเก่าอาจอันตรายกว่าการไม่มี Context

---

# 83. Memory Summary

```text
Week 6 Lab 3:
Context Engineering with MCP

Notebook:
6_mcp/3_lab3.ipynb

Framework:
OpenAI Agents SDK

Core:
Agent
Runner
trace
MCPServerStdio
create_static_tool_filter

Four context sources:
1. Long-term memory
2. Web search
3. Agentic RAG
4. Live integration

Memory server:
@modelcontextprotocol/server-memory

Memory storage:
memory/memory.json

Memory structure:
Entity
Observation
Relation

Memory purpose:
Persist facts across runs

Main memory risk:
Distortion
Poisoning
Privacy
Staleness

Web search server:
tavily-mcp

Environment:
TAVILY_API_KEY

Tool filter:
create_static_tool_filter

Allowed tool:
tavily_search

Tool filter benefit:
Smaller context
Lower cost
Less confusion
Least privilege

Agentic RAG:
Tavily
+
Qdrant MCP

Qdrant path:
memory/qdrant

Collection:
knowledge

Research task:
Latest Nvidia news
Store key facts

Retrieval task:
Answer from knowledge base
without web search

Knowledge graph:
Explicit entities and relations

Vector store:
Semantic text retrieval

Main RAG risk:
Wrong data stored permanently

Integration:
Market data

With key:
Massive MCP v0.10.0

Without key:
backend.market_server

Local tool:
lookup_share_price

Main market issue:
Real and simulated data
share similar result shape

Production result should include:
source
mode
timestamp
symbol
price

Main Context Engineering principle:
More context
does not always mean
better context

Production needs:
Provenance
Timestamp
Expiry
Tool filtering
Per-user isolation
Version pinning
Audit logs
Freshness policy
```

---

# 84. Next Episode

Lab ถัดไปจะนำ Context Sources เหล่านี้ไปประกอบเป็น Agent Roles และ Trading System ที่ใหญ่ขึ้น

สิ่งที่ควรจับตา:

```text
Researcher agents
Trader agents
Memory per agent
Multiple MCP servers
Market data
Account tools
Push notifications
Continuous orchestration
Risk controls
```

คำถามสำคัญคือ:

> เมื่อ Agent มีทั้ง Memory, Research, Market Data และ Trading Tools เราจะออกแบบ Role, Authority และ Control Loop อย่างไร เพื่อไม่ให้ข้อมูลจาก Context Layer กลายเป็นการตัดสินใจซื้อขายที่ไม่มีการตรวจสอบ?
