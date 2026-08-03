# Episodic Learning Artifact

## Week 6 — Lab 4: Autonomous Traders

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**โฟลเดอร์:** `6_mcp`
**Notebook:** `4_lab4.ipynb`
**โปรเจกต์:** Autonomous Traders
**หัวข้อหลัก:** Agent-as-Tool, Context Separation, Multiple MCP Servers, Persistent Research Memory, Autonomous Trading Loop, Custom Tracing และ Dashboard
**สถานะ:** เรียนและสรุป Lab 4 แล้ว
**ข้อจำกัดสำคัญ:** เป็นระบบจำลองเพื่อการศึกษา ไม่ควรใช้ตัดสินใจหรือซื้อขายสินทรัพย์จริง

---

# 1. Context

Labs ก่อนหน้าปูพื้นฐานตามลำดับ:

```text
Lab 1
→ ใช้ MCP Servers ที่ผู้อื่นสร้าง

Lab 2
→ สร้าง Custom MCP Server ด้วย FastMCP

Lab 3
→ ใช้ MCP ทำ Context Engineering
   ผ่าน Memory, Search, RAG และ Integration
```

Lab 4 นำทุกอย่างมาประกอบเป็นระบบเดียว:

```text
Custom Accounts MCP
+ Push Notification MCP
+ Market Data MCP
+ Fetch MCP
+ Tavily MCP
+ Persistent Memory MCP
+ Trader Agents
+ Researcher Agents
+ Recurring Scheduler
+ Live Dashboard
```

ระบบจำลอง Trading Floor ที่มี Traders สี่ตัว โดยแต่ละ Trader มี Strategy, Account, Research Memory และ Model ของตนเอง

```text
Warren Patience
George Bold
Ray Systematic
Cathie Crypto
```

Trader ไม่ค้นเว็บด้วยตนเอง แต่เรียก Researcher Agent ในฐานะ Tool จากนั้นใช้ Market, Account และ Push Tools เพื่อดำเนินงานต่อ.

---

# 2. Learning Objectives

หลังจบ Lab 4 สามารถอธิบายได้ว่า:

1. Autonomous Traders Architecture ประกอบด้วยอะไรบ้าง
2. Trader และ Researcher แบ่ง Context กันอย่างไร
3. Agent-as-Tool แตกต่างจาก Handoff อย่างไร
4. Trader ใช้ MCP Servers ชุดใด
5. Researcher ใช้ MCP Servers ชุดใด
6. Persistent Memory ถูกแยกต่อ Trader อย่างไร
7. `AsyncExitStack` จัดการ MCP Server Lifecycle อย่างไร
8. Account และ Strategy ถูกอ่านผ่าน MCP Resources อย่างไร
9. Account State ถูกย่อก่อนส่งเข้า Model อย่างไร
10. Trader Prompt ถูกประกอบจาก Strategy, Account และ Current Time อย่างไร
11. Trading Mode กับ Rebalancing Mode สลับกันอย่างไร
12. Traders สี่ตัวรันพร้อมกันด้วย `asyncio.gather()` อย่างไร
13. Model Routing รองรับหลาย Providers อย่างไร
14. Custom Trace Processor แปลง Agent Events เป็น Dashboard Logs อย่างไร
15. Dashboard แยกออกจาก Trading Engine อย่างไร
16. Agent Decision แตกต่างจาก Deterministic Application Control อย่างไร
17. Prompt–Capability Mismatch เกิดขึ้นตรงไหน
18. Failure Handling ปัจจุบันมีข้อจำกัดอย่างไร
19. การให้ Agent ซื้อขายโดยตรงมี Risk อะไร
20. Production Version ต้องเพิ่ม Risk Engine และ Approval อย่างไร

---

# 3. Prerequisites

ควรเข้าใจเนื้อหาจาก Week 6 Labs 1–3:

```text
MCP Client
MCP Server
MCPServerStdio
FastMCP
MCP Tools
MCP Resources
Tool filtering
Persistent memory
Web search
Context Engineering
Agentic RAG
Live integrations
```

ควรเข้าใจพื้นฐาน:

```text
OpenAI Agents SDK
Agent
Runner
Agent-as-Tool
Async context manager
AsyncExitStack
asyncio.gather()
SQLite
Environment variables
Tracing
Subprocess
```

Environment Variables ที่เกี่ยวข้อง:

```env
OPENAI_API_KEY=...
TAVILY_API_KEY=...
MASSIVE_API_KEY=...

RUN_EVERY_N_MINUTES=60
RUN_EVEN_WHEN_MARKET_IS_CLOSED=False
USE_MANY_MODELS=False

PUSHOVER_USER=...
PUSHOVER_TOKEN=...
```

บาง Keys เป็น Optional เพราะระบบมี Local Fallback หรือสามารถปิด Feature บางส่วนได้

---

# 4. Project Structure

```text
6_mcp/
├── 4_lab4.ipynb
├── app.py
├── backend/
│   ├── accounts.py
│   ├── accounts_client.py
│   ├── accounts_server.py
│   ├── database.py
│   ├── market.py
│   ├── market_server.py
│   ├── mcp_servers.py
│   ├── push_server.py
│   ├── reset.py
│   ├── templates.py
│   ├── tracers.py
│   ├── traders.py
│   └── trading_floor.py
└── demo/
    ├── ui.py
    └── util.py
```

หน้าที่หลัก:

```text
traders.py
→ สร้าง Researcher และ Trader Agents

mcp_servers.py
→ สร้าง MCP Server Connections

templates.py
→ เก็บ Agent Instructions และ Runtime Messages

trading_floor.py
→ Scheduler และ Concurrent Trading Loop

tracers.py
→ แปลง Trace Events เป็น Database Logs

accounts_client.py
→ อ่าน Account และ Strategy ผ่าน MCP Resources

app.py
→ เปิด Gradio Dashboard
```

---

# 5. High-level Architecture

```text
Trading Floor Scheduler
        │
        ├── Warren Trader
        ├── George Trader
        ├── Ray Trader
        └── Cathie Trader
```

Trader แต่ละตัวมี:

```text
Trader Agent
├── Researcher Agent as Tool
├── Accounts MCP
├── Market MCP
└── Push Notification MCP
```

Researcher มี:

```text
Researcher Agent
├── Fetch MCP
├── Tavily Search MCP
└── Persistent Memory MCP
```

ภาพรวม:

```text
Trader
│
├── Researcher Tool
│   ├── Search
│   ├── Fetch
│   └── Memory
│
├── Market Data
├── Account Operations
└── Push Notification
```

สีและตำแหน่งใน Architecture Diagram ใช้แยก Agent กับ MCP Server แต่หลักสำคัญคือการแยก Context และ Authority ไม่ใช่การเลียนแบบตำแหน่งงานมนุษย์.

---

# 6. Why Separate Trader and Researcher?

Trader ต้องรับผิดชอบ:

```text
อ่าน Strategy
อ่าน Portfolio
กำหนด Research Question
ประเมินผล Research
ตรวจ Market Data
ซื้อหรือขาย
เปลี่ยน Strategy
ส่ง Notification
```

Researcher รับผิดชอบ:

```text
ค้นข่าว
อ่านเว็บไซต์
รวบรวมข้อมูลบริษัท
เปรียบเทียบข้อมูลหลายแหล่ง
จำสิ่งที่ค้นพบ
สรุป Opportunities และ Risks
```

การแบ่งนี้ทำให้ Context ของแต่ละ Agent เล็กลง:

```text
Researcher
→ ไม่เห็น Account Trading Tools

Trader
→ ไม่เห็น Tavily, Fetch และ Memory Tools โดยตรง
```

Mental model:

```text
Researcher
= Context acquisition specialist

Trader
= Decision and execution owner
```

---

# 7. Context Separation

หากรวมทุกอย่างใน Agent เดียว:

```text
Search tools
Fetch tools
Memory tools
Account tools
Market tools
Push tools
```

Model ต้องเลือกจาก Tool Catalog ขนาดใหญ่ และต้องจัดการทั้ง Research กับ Execution ภายใน Loop เดียว

ผลที่อาจเกิด:

```text
Tool selection ยากขึ้น
Prompt ยาวขึ้น
Context ใหญ่ขึ้น
Authority มากเกินไป
Debug ยาก
Research กับ Execution ปะปนกัน
```

Lab จึงใช้ Architecture:

```text
Trader
→ ขอข้อมูลจาก Researcher

Researcher
→ จัดการ Search, Fetch และ Memory

Trader
→ ใช้ข้อมูลเพื่อ Execute
```

นี่เป็นการออกแบบตาม Context Boundary มากกว่าตาม Organizational Chart

---

# 8. Trader MCP Servers

`trader_mcp_servers()` เปิด MCP Servers สามตัว:

```text
Accounts Server
Push Server
Market Server
```

Conceptual code:

```python
def trader_mcp_servers():
    params = [
        accounts_params,
        push_params,
        market_params,
    ]

    return [
        MCPServerStdio(
            params,
            client_session_timeout_seconds=TIMEOUT,
        )
        for params in params
    ]
```

---

# 9. Accounts MCP Server

Trader ใช้ Accounts Server เพื่อ:

```text
อ่านยอดเงิน
อ่าน Holdings
ซื้อหุ้น
ขายหุ้น
เปลี่ยน Strategy
```

Account Server เป็น Custom MCP Server ที่สร้างใน Lab 2

```text
Trader Agent
→ Accounts MCP
→ Account Domain Model
→ SQLite
```

Write Tools มี Side Effects ต่อ Portfolio State จริงภายใน Simulation

---

# 10. Push Notification MCP Server

Trader ถูกสั่งให้ส่ง Push Notification หลังจบรอบ

```text
Trader finishes activity
→ Call push tool
→ External notification service
```

จุดประสงค์:

```text
แจ้ง Trade Activity
แจ้ง Portfolio Health
ทำให้ผู้ใช้เห็น Agent Action
```

แต่ Notification ไม่ควรถูกใช้เป็น Source of Truth

Source of Truth ควรเป็น:

```text
Transaction record
Account state
Execution receipt
Audit log
```

---

# 11. Market MCP Server

ระบบเลือก Market Server ตาม Environment

เมื่อมี `MASSIVE_API_KEY`:

```text
Massive MCP Server
→ Market provider
```

เมื่อไม่มี:

```text
backend.market_server
→ Simulated prices
```

Code ปัจจุบัน Pin Massive Server ที่ Version:

```text
v0.10.0
```

และใช้ Local FastMCP Server เป็น Fallback.

---

# 12. Market Interface-level Fallback

```text
MASSIVE_API_KEY available?
    │
    ├── Yes
    │   → Massive MCP
    │   → External market data
    │
    └── No
        → Local market MCP
        → Simulated price
```

Agent ยังคงเห็น Capability ประเภท Market Data

Implementation ด้านหลังเปลี่ยนได้โดยไม่ต้องเปลี่ยน Trader Architecture

นี่คือ Pattern:

```text
Stable capability interface
+
Replaceable implementation
```

---

# 13. Researcher MCP Servers

`researcher_mcp_servers(name)` สร้าง:

```text
Fetch MCP
Tavily Search MCP
Memory MCP
```

Concept:

```python
def researcher_mcp_servers(name):
    return [
        fetch_server,
        search_server,
        memory_server_for(name),
    ]
```

แต่ละ Researcher จึงมี Research Stack ของตนเอง

---

# 14. Fetch MCP

Fetch MCP ใช้:

```text
uvx mcp-server-fetch
```

หน้าที่:

```text
เปิด URL
อ่านหน้าเว็บ
คืนข้อความหรือ Markdown
```

Researcher Instructions กำหนดให้ใช้ Fetch เป็นทางเลือก หาก Tavily Search เจอ Rate Limit หรือพบ URL ที่ควรอ่านเจาะลึก

---

# 15. Tavily Search MCP

Tavily Server เปิดหลาย Tools แต่ระบบจำกัดให้ Researcher เห็นเฉพาะ:

```text
tavily_search
```

ผ่าน:

```python
create_static_tool_filter(
    allowed_tool_names=[
        "tavily_search"
    ]
)
```

เหตุผล:

```text
Researcher ต้องการ Search
ไม่จำเป็นต้องใช้ Crawl หรือ Deep Research ทุกครั้ง
```

นี่เป็นทั้ง:

```text
Context curation
Tool-budget control
Capability restriction
```

---

# 16. Persistent Memory MCP

Researcher ใช้:

```text
mcp-memory-libsql
```

และกำหนด Database แยกตาม Trader:

```text
memory/Warren.db
memory/George.db
memory/Ray.db
memory/Cathie.db
```

ผ่าน Environment:

```python
{
    "LIBSQL_URL":
        f"file:./memory/{name}.db"
}
```

---

# 17. Per-Trader Memory

แต่ละ Trader มี Research Memory ของตนเอง

```text
Warren
→ สะสม Value Investing knowledge

George
→ สะสม Macro and Contrarian knowledge

Ray
→ สะสม Diversification and Risk knowledge

Cathie
→ สะสม Innovation and Crypto ETF knowledge
```

ข้อดี:

```text
ลด Context ปะปน
สร้าง Expertise ตาม Strategy
ลด Cross-agent contamination
```

ข้อจำกัด:

```text
Research ซ้ำ
ข้อมูลขัดแย้งกัน
แต่ละ Memory มี Freshness ต่างกัน
แก้ข้อมูลพร้อมกันยาก
```

---

# 18. Trader Strategies

Strategies ถูกกำหนดใน `reset.py`

## Warren Patience

```text
Value-oriented
Long-term
High-quality companies
Intrinsic value
Cash flow
Management quality
Competitive advantage
```

## George Bold

```text
Aggressive macro
Contrarian
Geopolitical events
Large market mispricing
Decisive timing
```

## Ray Systematic

```text
Systematic
Diversified
Risk parity
Economic cycles
Central-bank policy
Capital preservation
```

## Cathie Crypto

```text
Disruptive innovation
Crypto ETFs
High volatility
Technology breakthroughs
Regulatory developments
```

---

# 19. Resetting Traders

```python
reset_traders()
```

จะ Reset:

```text
Balance
Holdings
Transactions
Strategy
Portfolio history
```

ของ Traders ทั้งสี่

Notebook Comment บรรทัดนี้ไว้ เพื่อไม่ลบสถานะที่สะสมโดยไม่ตั้งใจ

```python
# reset_traders()
```

นี่เป็น Safety Choice ใน Learning Environment

---

# 20. Researcher Agent Creation

```python
async def get_researcher(
    mcp_servers,
    model_name,
) -> Agent:

    return Agent(
        name="Researcher",
        instructions=
            researcher_instructions(),
        model=get_model(model_name),
        mcp_servers=mcp_servers,
    )
```

Researcher ได้รับ:

```text
Financial research instructions
Model
Fetch server
Search server
Persistent memory server
```

---

# 21. Researcher Instructions

Researcher ถูกสั่งให้:

```text
ค้นหลายแหล่ง
สรุปภาพรวม
ใช้ Fetch เมื่อ Search มีปัญหา
ค้นข้อมูลเดิมจาก Memory
เก็บ Company Facts
เก็บ Market Conditions
เก็บ URL ที่น่าสนใจ
สร้าง Expertise ข้าม Runs
```

Prompt ยังใส่ Current Date และ Time เพื่อให้คำว่า `latest` มี Temporal Context.

---

# 22. Researcher Loop

```text
Receive research request
        ↓
Search persistent memory
        ↓
Search Tavily
        ↓
Fetch useful sources
        ↓
Compare evidence
        ↓
Store useful facts
        ↓
Return summary
```

Researcher เป็น Agentic Research Pipeline ไม่ใช่ Search Function ธรรมดา

---

# 23. Researcher as a Tool

Researcher ถูกเปลี่ยนเป็น Tool:

```python
researcher.as_tool(
    tool_name="Researcher",
    tool_description=
        research_tool(),
)
```

จากมุมมอง Trader:

```text
Researcher
= Callable tool
```

Trader ส่ง Natural-language Research Request และรับ Final Research Output กลับมา

---

# 24. Agent-as-Tool Mental Model

```text
Parent Agent
→ Calls Child Agent
→ Child Agent runs its own loop
→ Returns result
→ Parent Agent continues
```

Trader ยังคงเป็นเจ้าของ Workflow

Researcher ทำหน้าที่เหมือน Agentic Function ที่มี:

```text
Model
Instructions
Tools
Memory
Multiple steps
```

---

# 25. Agent-as-Tool vs Handoff

## Agent-as-Tool

```text
Trader retains control
→ Calls Researcher
→ Receives summary
→ Decides what to do
```

## Handoff

```text
Trader transfers conversation
→ Researcher becomes active owner
→ User interaction continues with Researcher
```

Lab ใช้ Agent-as-Tool เพราะ Trader ต้องรับผิดชอบ Decision และ Execution ต่อไป.

---

# 26. Creating Trader Agent

```python
async def create_agent(
    self,
    trader_mcp_servers,
    researcher_mcp_servers,
):
    researcher_tool = (
        await get_researcher_tool(
            researcher_mcp_servers,
            self.model_name,
        )
    )

    self.agent = Agent(
        name=self.name,
        instructions=
            trader_instructions(self.name),
        model=get_model(self.model_name),
        tools=[researcher_tool],
        mcp_servers=trader_mcp_servers,
    )
```

Trader จึงได้รับ:

```text
Researcher Agent Tool
Accounts MCP
Market MCP
Push MCP
```

---

# 27. Trader Context

ก่อน Run Trader ได้รับ:

```text
Trader identity
Investment strategy
Current account report
Current datetime
Market capability description
Researcher capability
Account tools
Push tool
```

Trader ไม่เห็น:

```text
Tavily tool definitions
Fetch tool definitions
Memory tool definitions
```

เพราะรายละเอียด Research ถูกซ่อนอยู่หลัง Researcher Tool

---

# 28. Prompt–Capability Mismatch

Trader Instructions ระบุว่า:

```text
You can use your entity tools
as a persistent memory...
```

แต่ Trader MCP Servers มีเพียง:

```text
Accounts
Push
Market
```

Memory MCP อยู่ฝั่ง Researcher เท่านั้น.

ดังนั้น Trader ไม่มี Entity Tools โดยตรง

ควรแก้ Instruction เป็น:

```text
Use the Researcher tool to recall and store
persistent research knowledge.
```

หลัก:

```text
Prompt promises
ต้องตรงกับ
Actual capability surface
```

---

# 29. Reading Account via MCP Resource

Trader อ่าน Account ผ่าน:

```python
read_accounts_resource(
    self.name
)
```

และ Strategy ผ่าน:

```python
read_strategy_resource(
    self.name
)
```

`accounts_client.py` เปิด Accounts MCP Server แล้วอ่าน URIs:

```text
accounts://accounts_server/<name>
accounts://strategy/<name>
```

---

# 30. Tools vs Resources in the Same System

ระบบใช้ MCP สองรูปแบบ:

## MCP Tools

Model เป็นผู้เลือกเรียก:

```text
buy_shares
sell_shares
lookup_share_price
push
```

## MCP Resources

Application Code อ่านโดยตรง:

```text
Account report
Investment strategy
```

Mental model:

```text
Tool
= Model-directed operation

Resource
= Application-retrieved context
```

---

# 31. Account Context Reduction

ก่อนส่ง Account เข้า Model:

```python
account_json.pop(
    "portfolio_value_time_series",
    None,
)
```

เหตุผล:

```text
Portfolio history ยาวขึ้นทุก Run
เพิ่ม Tokens
อาจไม่จำเป็นต่อ Current Decision
ลด Signal-to-noise ratio
```

Pattern:

```text
Retrieve full state
→ Remove low-value fields
→ Send compact context
```

นี่เป็น Context Engineering ระดับ Application

---

# 32. Account Resource Hidden Side Effect

Account Resource เรียก `report()`

แต่ `report()`:

```text
คำนวณ Portfolio Value
เพิ่ม Snapshot
Save Account
Write Log
```

ดังนั้น:

```text
Read resource
→ Mutates state
```

ผลที่อาจเกิด:

```text
Dashboard read creates snapshot
Trader read creates snapshot
Repeated reads inflate time series
```

Production ควรแยก:

```text
read_account_report()
record_portfolio_snapshot()
```

---

# 33. Trading Mode

เมื่อ:

```python
self.do_trade = True
```

ระบบใช้:

```python
trade_message(
    name,
    strategy,
    account,
)
```

Prompt สั่งให้ Trader:

```text
ค้นโอกาสใหม่
ใช้ Researcher
ตรวจราคาหุ้น
ตรวจเงินสด
เลือกหุ้นตาม Strategy
ซื้อหรือขายตามความเหมาะสม
ไม่ต้อง Rebalance
ส่ง Push Notification
สรุป Portfolio
```

---

# 34. Trading Loop

```text
Read strategy
        ↓
Read account
        ↓
Research new opportunities
        ↓
Check market information
        ↓
Check available cash
        ↓
Decide whether to trade
        ↓
Execute account tools
        ↓
Send notification
        ↓
Return short appraisal
```

---

# 35. Rebalancing Mode

เมื่อ:

```python
self.do_trade = False
```

ระบบใช้:

```python
rebalance_message(...)
```

Prompt เน้น:

```text
ตรวจ Portfolio เดิม
ค้นข่าวที่กระทบ Holdings
ไม่ต้องหา Opportunity ใหม่
ลดหรือเพิ่ม Position ตาม Strategy
ทบทวนผล Trade เดิม
เปลี่ยน Strategy ได้
ส่ง Notification
```

---

# 36. Rebalancing Loop

```text
Read current holdings
        ↓
Research affected companies
        ↓
Review performance
        ↓
Compare with strategy
        ↓
Rebalance if required
        ↓
Possibly update strategy
        ↓
Send notification
```

---

# 37. Trading–Rebalancing Toggle

Initial State:

```python
self.do_trade = True
```

หลังจบ Run:

```python
self.do_trade = (
    not self.do_trade
)
```

Sequence:

```text
Run 1
→ Trading

Run 2
→ Rebalancing

Run 3
→ Trading

Run 4
→ Rebalancing
```

---

# 38. Why Toggle?

ข้อดี:

```text
ไม่หา Opportunity ใหม่ทุกรอบ
มีรอบตรวจ Portfolio
Control Flow เข้าใจง่าย
Demo แสดง Behavior สองแบบ
```

ข้อจำกัด:

```text
ไม่อิง Market Conditions
ไม่อิง Portfolio Risk
ไม่อิง Trade Outcome
ไม่อิง Event Trigger
```

เป็น Simple Scheduling Policy ไม่ใช่ Intelligent Workflow State

---

# 39. Failure-safe Toggle Problem

Current Code:

```python
async def run(self):
    try:
        await self.run_with_trace()
    except Exception as e:
        print(...)

    self.do_trade = not self.do_trade
```

แม้ Run ล้มเหลว Mode ก็ยังสลับ.

ตัวอย่าง:

```text
Trading run fails
→ No trade occurs
→ State changes to Rebalance
→ Next run assumes previous phase completed
```

Safer:

```python
async def run(self):
    success = False

    try:
        await self.run_with_trace()
        success = True
    except Exception as exc:
        record_failure(exc)

    if success:
        self.do_trade = (
            not self.do_trade
        )
```

---

# 40. Better Workflow State

แทน Boolean:

```text
do_trade = True / False
```

ควรใช้ Explicit State:

```text
READY_TO_RESEARCH
READY_TO_TRADE
READY_TO_REBALANCE
WAITING_FOR_APPROVAL
FAILED_RETRYABLE
FAILED_TERMINAL
COMPLETED
```

ข้อดี:

```text
Resume ง่าย
ตรวจสอบง่าย
Failure ไม่ทำให้ Phase สูญหาย
เพิ่ม Approval ได้
เพิ่ม Retry Policy ได้
```

---

# 41. AsyncExitStack

Trader ต้องเปิด MCP Servers หลายตัวแบบ Dynamic

```python
async with AsyncExitStack() as stack:
    trader_servers = [
        await stack.enter_async_context(server)
        for server
        in trader_mcp_servers()
    ]

    researcher_servers = [
        await stack.enter_async_context(server)
        for server
        in researcher_mcp_servers(
            self.name
        )
    ]

    await self.run_agent(
        trader_servers,
        researcher_servers,
    )
```

---

# 42. AsyncExitStack Mental Model

```text
Open server 1
Open server 2
Open server 3
Open server 4
Open server 5
Open server 6
        ↓
Run agents
        ↓
Close server 6
Close server 5
Close server 4
Close server 3
Close server 2
Close server 1
```

ช่วยรับประกัน Cleanup แม้เกิด Exception ภายใน Context

---

# 43. MCP Process Count

ต่อ Trader หนึ่งรอบ:

```text
Trader servers
= 3

Researcher servers
= 3

Total
= 6 MCP processes
```

Traders สี่ตัวรันพร้อมกันอาจเปิด:

```text
4 × 6
= 24 MCP server processes
```

นอกจากนี้ Account และ Strategy Resource Calls ยังเปิด Accounts Server เพิ่มแยกตาม Operation

ผล:

```text
High process startup cost
More memory usage
More package startup latency
Harder process supervision
```

Demo นี้เหมาะกับ Local Learning แต่ Production ควรใช้ Long-lived Services หรือ Shared Remote MCP Endpoints

---

# 44. Runner Budget

Trader Run ใช้:

```python
MAX_TURNS = 30
```

และ:

```python
await Runner.run(
    self.agent,
    message,
    max_turns=MAX_TURNS,
)
```

Budget ครอบคลุม:

```text
Researcher call
Market lookups
Account operations
Strategy update
Notification
Final answer
```

แต่:

```text
Run stopped within budget
≠ Trade was valid
≠ Risk was acceptable
≠ Notification succeeded
```

---

# 45. Model Routing

`get_model()` รองรับ:

```text
Native OpenAI model
OpenRouter
DeepSeek
Gemini
Grok
```

Routing Logic:

```text
Model name contains "/"
→ OpenRouter

Contains "deepseek"
→ DeepSeek endpoint

Contains "grok"
→ xAI endpoint

Contains "gemini"
→ Google OpenAI-compatible endpoint

Otherwise
→ Native model identifier
```

---

# 46. Multi-model Mode

Default:

```env
USE_MANY_MODELS=False
```

Traders ทั้งสี่ใช้:

```text
gpt-5.4-mini
```

เมื่อเปิด:

```env
USE_MANY_MODELS=True
```

ระบบกำหนด Models ต่างกัน:

```text
Warren
→ GPT-5.5

George
→ DeepSeek V4 Flash

Ray
→ Gemini 3.5 Flash

Cathie
→ Grok 4.3
```

---

# 47. Why Multiple Models?

ข้อดี:

```text
เปรียบเทียบ Behavior
ลด Decision Correlation
ทดลอง Provider Interoperability
ดูความต่างด้าน Tool Use
```

ข้อจำกัด:

```text
Tool-call behavior ต่างกัน
Costs ต่างกัน
Latency ต่างกัน
Provider reliability ต่างกัน
Prompt adherence ต่างกัน
ผลลัพธ์เปรียบเทียบยาก
```

---

# 48. Strategy Evolution

Trader Instructions อนุญาตให้:

```text
ทบทวน Trade Performance
เรียนรู้จากผลลัพธ์
เปลี่ยน Strategy
```

ผ่าน `change_strategy` Tool.

Concept:

```text
Past performance
→ Model interpretation
→ Strategy rewrite
→ Future actions
```

นี่คือ Self-modifying Policy

---

# 49. Strategy Drift Risk

Model อาจเปลี่ยน Strategy จาก:

```text
Long-term value investing
```

ไปเป็น:

```text
Short-term momentum trading
```

เพราะผลลัพธ์ไม่กี่รอบ

Risks:

```text
Recent-event overreaction
Overfitting
Loss chasing
Policy instability
Loss of original objective
```

Production ควรใช้:

```text
Propose strategy change
→ Backtest
→ Risk review
→ Human approval
→ Versioned activation
```

---

# 50. Strategy Versioning

ตัวอย่าง:

```json
{
  "strategy_id": "warren",
  "version": 4,
  "previous_version": 3,
  "proposal": "...",
  "reason": "...",
  "evidence": ["..."],
  "created_by": "agent",
  "approved_by": null,
  "status": "pending"
}
```

ไม่ควรเขียนทับ Strategy เดิมโดยไม่มี History

---

# 51. Custom Tracing

ระบบสร้าง Custom Trace ID:

```python
trace_id = make_trace_id(
    self.name.lower()
)
```

Trace ID ฝังชื่อ Trader:

```text
trace_warren0...
trace_george0...
```

แล้วใช้:

```python
with trace(
    trace_name,
    trace_id=trace_id,
):
```

---

# 52. LogTracer

```python
class LogTracer(
    TracingProcessor
):
```

รับ Events:

```text
Trace start
Trace end
Span start
Span end
```

จากนั้นเขียน Logs ลง Database:

```python
write_log(
    name,
    type,
    message,
)
```

---

# 53. Trace Events

ตัวอย่างข้อมูลที่บันทึก:

```text
Started: Warren-trading
Started response
Started function Researcher
Started MCP accounts_server
Ended MCP accounts_server
Ended function Researcher
Ended: Warren-trading
```

Dashboard ใช้ Events เหล่านี้แสดง Agent Activity Timeline

---

# 54. Trace Is Not Private Chain of Thought

LogTracer บันทึก:

```text
Span type
Tool name
Agent name
MCP server
Start/end
Error
```

ไม่ได้บันทึก:

```text
Hidden model reasoning
Private chain of thought
Internal token-level deliberation
```

จึงควรเรียกว่า:

```text
Execution trace
Agent activity
Operational timeline
```

ไม่ใช่ Full Reasoning

---

# 55. Running One Trader

Notebook Register Processor:

```python
add_trace_processor(
    LogTracer()
)
```

จากนั้น:

```python
warren = Trader(
    "Warren",
    "Patience",
    "gpt-5.4-mini",
)

await warren.run()
```

---

# 56. One Trader End-to-end Flow

```text
Open Trader MCP Servers
        ↓
Open Researcher MCP Servers
        ↓
Build Researcher Agent
        ↓
Wrap Researcher as Tool
        ↓
Build Trader Agent
        ↓
Read Account Resource
        ↓
Read Strategy Resource
        ↓
Build Runtime Message
        ↓
Research
        ↓
Market lookup
        ↓
Trade or no trade
        ↓
Push notification
        ↓
Write traces
        ↓
Close MCP processes
```

---

# 57. Environment Evidence

หลัง Trader Run Notebook อ่าน Account Resource:

```python
resources = (
    await read_accounts_resource(
        "Warren"
    )
)

info = json.loads(resources)

print(
    info["transactions"][-1]
)
```

หลัก:

```text
Agent says trade completed
≠ Trade persisted

Transaction in Account
= Environment evidence
```

---

# 58. Post-run Validation

ควรตรวจ:

```text
Transaction exists
Balance changed correctly
Holdings changed correctly
Price is plausible
Quantity is positive
No duplicate transaction
Audit log exists
Push result exists
```

ไม่ควรตรวจเพียง Final Response

---

# 59. Trading Floor

`trading_floor.py` สร้าง Traders ทั้งสี่:

```python
names = [
    "Warren",
    "George",
    "Ray",
    "Cathie",
]
```

จากนั้น:

```python
await asyncio.gather(
    *[
        trader.run()
        for trader in traders
    ]
)
```

Traders จึงทำงานพร้อมกันในหนึ่ง Scheduler Tick

---

# 60. Trading Floor Loop

```python
while True:
    if market_is_open:
        await asyncio.gather(
            *runs
        )

    await asyncio.sleep(
        interval
    )
```

Mental model:

```text
Check market
        ↓
Run four traders concurrently
        ↓
Wait for all
        ↓
Sleep
        ↓
Repeat
```

---

# 61. Scheduler Configuration

```env
RUN_EVERY_N_MINUTES=60
```

กำหนดความถี่

```env
RUN_EVEN_WHEN_MARKET_IS_CLOSED=False
```

กำหนดว่าจะข้ามรอบเมื่อ Market ปิดหรือไม่

```env
USE_MANY_MODELS=False
```

กำหนด Model Configuration

---

# 62. Market State in Simulator Mode

เมื่อไม่มี Live Market Provider ระบบอาจใช้ Simulator และถือว่า Market พร้อมใช้งาน

ผลคือ:

```text
RUN_EVEN_WHEN_MARKET_IS_CLOSED=False
```

อาจไม่ได้หยุด Loop แบบเดียวกับ Live Market Mode

Production ควรแสดง Mode:

```text
LIVE
DELAYED
SIMULATED
FALLBACK
```

ให้ชัดเจนใน Logs และ Dashboard

---

# 63. Parallelism

```python
asyncio.gather(
    trader1.run(),
    trader2.run(),
    trader3.run(),
    trader4.run(),
)
```

ให้ Traders ทำงาน Concurrently

ข้อดี:

```text
ลดเวลารวม
จำลองหลาย Actors
Dashboard เห็นหลายกิจกรรมพร้อมกัน
```

ข้อเสีย:

```text
Database contention
API rate limits
High process count
High token usage
Harder debugging
```

---

# 64. Shared Database Risk

Traders ใช้ Shared SQLite Database สำหรับ:

```text
Accounts
Transactions
Logs
Portfolio snapshots
```

แม้ Account Names ต่างกัน แต่ Database File เดียวกันอาจเจอ:

```text
database is locked
Write contention
Log ordering problems
Partial writes
```

Production ควรเพิ่ม:

```text
WAL mode
busy timeout
transactions
retry policy
database service
```

---

# 65. Error Handling

Current Code:

```python
try:
    await self.run_with_trace()
except Exception as e:
    print(
        f"Error running trader "
        f"{self.name}: {e}"
    )
```

ข้อดี:

```text
Trader ตัวหนึ่งล้ม
ไม่ทำให้ทั้งทีมล้มทันที
```

ข้อเสีย:

```text
Error ถูกกลืน
ไม่มี Durable failure state
ไม่มี Retry classification
ไม่มี Stack trace
Scheduler ไม่รู้ว่า Run ไม่สำเร็จ
```

---

# 66. Recommended Run Record

```sql
trader_runs (
    run_id,
    trader,
    mode,
    model,
    started_at,
    finished_at,
    status,
    error_type,
    error_message,
    trade_count
)
```

Statuses:

```text
RUNNING
SUCCESS
FAILED_RETRYABLE
FAILED_TERMINAL
TIMED_OUT
SKIPPED_MARKET_CLOSED
WAITING_APPROVAL
```

---

# 67. Dashboard

เปิด Dashboard:

```powershell
cd 6_mcp
uv run app.py
```

เปิด Trading Engine แยก Terminal:

```powershell
cd 6_mcp
uv run -m backend.trading_floor
```

`app.py` สร้าง Gradio UI และเปิด Browser.

---

# 68. Execution Plane vs Presentation Plane

## Execution Plane

```text
Trading scheduler
Trader agents
Researcher agents
MCP servers
Model calls
Account mutations
```

## Presentation Plane

```text
Gradio dashboard
Charts
Transactions
Logs
Agent activities
```

Architecture:

```text
Trading engine
→ Writes SQLite

Dashboard
→ Reads SQLite
→ Renders state
```

---

# 69. Benefits of Separation

```text
Dashboard restart
ไม่ควรหยุด Trading Engine

Trading Engine
ไม่ขึ้นกับ Browser UI

Frontend
สามารถเปลี่ยนได้

Backend
สามารถถูกใช้จากหลาย Clients
```

Lab ถัดไปจะใช้ Backend เดิมกับ Frontend ที่แยกเป็น Web Application มากขึ้น.

---

# 70. Authority Chain

```text
Untrusted web information
        ↓
Researcher summary
        ↓
Trader reasoning
        ↓
Market lookup
        ↓
buy_shares / sell_shares
        ↓
Portfolio state mutation
        ↓
Push notification
```

Chain นี้ไม่มี Human Approval ก่อน Trade

จึงเหมาะเฉพาะ Simulation

---

# 71. Prompt Is Not a Risk Engine

Prompt สั่งให้ Trader:

```text
ตรวจเงินสด
ขนาด Position ไม่เกิน Balance
ทำตาม Strategy
```

แต่เป็น Soft Instructions

Model ยังอาจ:

```text
ซื้อ Position ใหญ่เกินไป
กระจุกตัวมากเกินไป
Trade บ่อย
ตัดสินใจจาก Source ต่ำคุณภาพ
```

Production Rules ต้องอยู่ใน Deterministic Code

---

# 72. Missing Deterministic Limits

ระบบยังไม่มี Hard Rules เช่น:

```text
Maximum order value
Maximum position percentage
Maximum sector exposure
Maximum daily loss
Maximum trade count
Minimum cash reserve
Allowed-symbol list
Stop-loss limits
```

Account Server ควรเป็นผู้บังคับ ไม่ใช่ Trader Prompt

---

# 73. Deterministic Risk Engine

```text
Trade proposal
        ↓
Check symbol allowlist
        ↓
Check quantity
        ↓
Check available cash
        ↓
Check position concentration
        ↓
Check portfolio exposure
        ↓
Check daily loss
        ↓
Approve or reject
```

Pseudo-code:

```python
if order_value > MAX_ORDER_VALUE:
    reject()

if new_position_ratio > MAX_POSITION_RATIO:
    reject()

if projected_cash < MIN_CASH:
    reject()
```

---

# 74. Human Approval Pattern

Safer Flow:

```text
Research
        ↓
Trade proposal
        ↓
Risk validation
        ↓
Human review
        ↓
Approval token
        ↓
Execution
        ↓
Reconciliation
```

Tools:

```text
research_stock
preview_trade
validate_trade
request_approval
execute_trade
get_trade_status
```

Trader ไม่ควรเรียก `buy_shares` โดยตรงในระบบที่มีเงินจริง

---

# 75. Research Evidence

Researcher ปัจจุบันคืนข้อความสรุป

แต่ไม่มี Schema บังคับ:

```text
Source URL
Publisher
Event date
Retrieved date
Bull case
Bear case
Confidence
Contradictions
```

Trader อาจตัดสินใจจาก Summary ที่ตรวจย้อนหลังยาก

---

# 76. Structured Research Result

```json
{
  "topic": "AMZN",
  "summary": "...",
  "bull_case": ["..."],
  "bear_case": ["..."],
  "sources": [
    {
      "url": "...",
      "publisher": "...",
      "published_at": "...",
      "claim": "..."
    }
  ],
  "confidence": 0.73,
  "as_of": "2026-08-03"
}
```

Trader ควรถูกห้าม Trade หากไม่มี Evidence ขั้นต่ำ

---

# 77. Memory Poisoning Risk

```text
Bad webpage
→ Researcher stores fact
→ Fact persists
→ Future Researcher retrieves it
→ Trader acts on it
```

Controls:

```text
Source allowlist
Provenance
Expiry
Confidence
Verification
Correction workflow
```

Tool Filtering ลดจำนวน Tools แต่ไม่ได้ตรวจความน่าเชื่อถือของ Content

---

# 78. Push Notification Reliability

Notification Tool อาจล้มเหลว แม้ Trade สำเร็จ

```text
Trade persisted
→ Push API fails
→ Notification missing
```

หรือ:

```text
Trade fails
→ Agent still sends success message
```

Notification ต้องใช้ Structured Event จาก Execution Layer ไม่ควรพึ่ง Model Summary เพียงอย่างเดียว

---

# 79. Transaction Reconciliation

หลัง Execute Trade ควรตรวจ:

```text
Order accepted?
Order executed?
Execution price?
Executed quantity?
Balance updated?
Holdings updated?
Transaction ID?
```

Simulation ปัจจุบันบันทึก Account โดยตรง แต่ Production Broker Integration ต้องแยก:

```text
Order submission
Order acknowledgment
Partial fill
Full fill
Cancel
Reject
Settlement
```

---

# 80. Cost Surfaces

หนึ่ง Trading Cycle อาจใช้:

```text
4 Trader model runs
4 Researcher model runs
Multiple web searches
Multiple fetch requests
Multiple market calls
Account tools
Push calls
MCP process startups
```

เมื่อใช้หลาย Models และทำทุกชั่วโมง Cost อาจเพิ่มเร็ว

ควรติดตาม:

```text
Tokens per trader
Tokens per researcher
Tool calls
Search calls
MCP startup time
Provider costs
Cycle duration
```

---

# 81. Global Budget

ควรกำหนด:

```text
Maximum model calls per cycle
Maximum search calls
Maximum research duration
Maximum trade proposals
Maximum MCP process lifetime
Maximum total cycle cost
```

`MAX_TURNS=30` จำกัดแต่ละ Trader Run แต่ไม่ใช่ Global System Budget

---

# 82. Security Boundaries

## Researcher

เข้าถึง:

```text
Web
Search API
Persistent memory
```

Risk:

```text
Prompt injection
Memory poisoning
Sensitive-query leakage
```

## Trader

เข้าถึง:

```text
Account mutations
Market data
External notifications
```

Risk:

```text
Unauthorized trades
Excessive authority
External side effects
```

การแยก Agent ลด Cross-capability Surface แต่ Researcher Output ยังสามารถมีอิทธิพลต่อ Trader ได้

---

# 83. Common Misconceptions

## “Multi-agent System ต้องให้ Agents คุยกันเอง”

ไม่จริง Agent หนึ่งสามารถเรียก Agent อื่นเป็น Tool ได้

## “แยก Trader กับ Researcher เพราะเหมือนทีมมนุษย์”

ไม่ใช่เหตุผลหลัก จุดสำคัญคือ Context และ Capability Separation

## “Researcher as Tool เหมือน Handoff”

ไม่เหมือน Trader ยังคงเป็น Workflow Owner

## “Dashboard แสดง Chain of Thought”

ไม่จริง Dashboard แสดง Operational Traces

## “Prompt จำกัด Risk ได้”

ไม่พอ Hard Limits ต้องอยู่ใน Code

## “Run สำเร็จเพราะไม่มี Exception”

ไม่จริง ต้องตรวจ Transactions และ Account State

## “Push Notification ยืนยัน Trade”

ไม่จริง Transaction Record คือ Source of Truth

## “หลาย Models ทำให้ระบบฉลาดขึ้นอัตโนมัติ”

ไม่จริง เพิ่ม Complexity, Cost และ Variance ด้วย

---

# 84. Risks Identified

## 84.1 Excessive Trading Authority

Trader เรียก Write Tools ได้โดยตรง

## 84.2 No Human Approval

Decision เชื่อมกับ Execution ทันที

## 84.3 Prompt–Capability Mismatch

Trader Prompt กล่าวถึง Memory Tools ที่ Trader ไม่มี

## 84.4 Failure Toggles Mode

Run ล้มเหลวแต่ Trading Mode ยังเปลี่ยน

## 84.5 Hidden Resource Mutation

Account Resource Read เพิ่ม Snapshot

## 84.6 Memory Poisoning

Web Data ผิดถูกเก็บระยะยาว

## 84.7 Weak Research Provenance

Researcher ไม่คืน Sources แบบ Structured

## 84.8 Strategy Drift

Trader เปลี่ยน Strategy ได้โดยไม่มี Review

## 84.9 Process Explosion

MCP Server Processes จำนวนมากต่อ Cycle

## 84.10 SQLite Contention

หลาย Agents และ Processes เขียนพร้อมกัน

## 84.11 Hidden Errors

Exception ถูก Print และกลืน

## 84.12 Simulator Ambiguity

ข้อมูลจำลองอาจถูกนำเสนอเหมือน Live Data

---

# 85. Production Improvements

```text
Add deterministic risk engine
Separate proposal from execution
Add human approval
Use structured research output
Store source provenance
Add confidence and expiry
Version strategies
Toggle phase only on success
Persist run states
Use long-lived MCP services
Add process supervision
Use transactional database
Add per-tool authorization
Add global cost budget
Reconcile trade execution
Expose live/simulated market mode
Use structured notifications
```

---

# 86. Suggested Exercise — Prompt Alignment

แก้:

```text
You can use your entity tools...
```

เป็น:

```text
Use the Researcher tool to recall and store
persistent market knowledge.
```

จากนั้นตรวจ Trace ว่า Trader ใช้ Researcher เมื่อจำเป็น

---

# 87. Suggested Exercise — Failure-safe Mode

ทำให้ Search Server ล้มเหลวชั่วคราว

ตรวจว่า Current Code:

```text
Run fails
→ Mode still changes
```

จากนั้นแก้ให้ Mode สลับเฉพาะ Success

---

# 88. Suggested Exercise — Research Schema

สร้าง Pydantic Model:

```python
class ResearchResult(BaseModel):
    summary: str
    bull_case: list[str]
    bear_case: list[str]
    sources: list[Source]
    confidence: float
    as_of: str
```

ห้าม Trader Trade หาก:

```text
sources is empty
confidence below threshold
```

---

# 89. Suggested Exercise — Risk Rules

เพิ่ม Rules:

```text
Order ≤ $1,000
Position ≤ 20% of portfolio
Cash reserve ≥ 10%
Maximum three trades per cycle
```

บังคับ Rules ใน Account Server ไม่ใช่ Prompt

---

# 90. Suggested Exercise — Run Records

สร้าง Table:

```sql
trader_runs (
    run_id,
    trader,
    mode,
    model,
    status,
    started_at,
    finished_at,
    error,
    trade_count
)
```

ให้ Dashboard แสดง:

```text
Success
Failed
Timed out
Skipped
Awaiting approval
```

---

# 91. Suggested Exercise — Process Measurement

ขณะรัน Traders สี่ตัว ตรวจ Process Count:

```text
python
node
npx
uv
uvx
```

เปรียบเทียบ:

```text
Short-lived local MCP
vs
Long-lived shared MCP service
```

---

# 92. Patterns Learned

## Context-specialized Agent Pattern

```text
Research context
→ Researcher

Execution context
→ Trader
```

## Agent-as-Tool Pattern

```text
Parent agent
→ Child agent as callable capability
```

## Per-agent Memory Pattern

```text
Agent identity
→ Separate persistent database
```

## Dynamic MCP Lifecycle Pattern

```text
AsyncExitStack
→ Open multiple servers
→ Guaranteed cleanup
```

## Concurrent Agent Pattern

```text
asyncio.gather()
→ Run independent agents together
```

## Custom Observability Pattern

```text
Agent trace
→ Custom processor
→ Database
→ Dashboard
```

## Execution/Presentation Separation Pattern

```text
Backend engine
→ Shared state
← Frontend dashboard
```

---

# 93. Lab 4 Mental Model

```text
Scheduler tick
        ↓
Create four Trader runs
        ↓
Each Trader opens MCP servers
        ↓
Read strategy and account
        ↓
Call Researcher tool
        ↓
Researcher searches web and memory
        ↓
Trader evaluates research
        ↓
Market lookup
        ↓
Trade or rebalance
        ↓
Account mutation
        ↓
Push notification
        ↓
Trace events written
        ↓
Dashboard reads state
        ↓
Sleep and repeat
```

---

# 94. Final Lessons

## Lesson 1

Agent Roles ควรถูกออกแบบจาก Context และ Authority Boundaries

## Lesson 2

Researcher as Tool ทำให้ Trader ยังคงเป็น Workflow Owner

## Lesson 3

Trader ไม่จำเป็นต้องเห็น Search และ Memory Tool Catalog ทั้งหมด

## Lesson 4

Persistent Memory แยกตาม Trader ช่วยสร้าง Expertise เฉพาะทาง

## Lesson 5

MCP Resources สามารถถูก Application Code อ่านโดยตรง ไม่จำเป็นต้องผ่าน Model

## Lesson 6

Account Context ควรถูกย่อก่อนส่งเข้า Model

## Lesson 7

`AsyncExitStack` เหมาะกับการจัดการ MCP Context Managers จำนวนไม่แน่นอน

## Lesson 8

Boolean Toggle เป็น Workflow Policy ที่ง่ายแต่เปราะบางต่อ Failure

## Lesson 9

Concurrent Agents เพิ่มทั้งความเร็วและ Resource Contention

## Lesson 10

Custom Trace Processor ทำให้ Agent Activity ถูกแสดงใน Product UI ได้

## Lesson 11

Operational Trace ไม่ใช่ Private Chain of Thought

## Lesson 12

Strategy Self-modification ต้องมี Versioning และ Review

## Lesson 13

Prompt ไม่ควรเป็น Control หลักสำหรับ Financial Risk

## Lesson 14

Agent Decision ต้องถูกคั่นด้วย Risk Validation และ Approval ก่อน Side Effect จริง

## Lesson 15

Source of Truth ต้องเป็น Persisted Account และ Transaction State ไม่ใช่ Final Agent Message

---

# 95. Memory Summary

```text
Week 6 Lab 4:
Autonomous Traders

Notebook:
6_mcp/4_lab4.ipynb

Project:
Four simulated traders
and one researcher role

Traders:
Warren Patience
George Bold
Ray Systematic
Cathie Crypto

Trader capabilities:
Researcher tool
Accounts MCP
Market MCP
Push MCP

Researcher capabilities:
Fetch MCP
Tavily Search MCP
Persistent Memory MCP

Researcher memory:
memory/<trader>.db

Search filter:
tavily_search only

Researcher form:
Agent wrapped with as_tool()

Agent-as-tool:
Parent retains control

Trader context:
Strategy
Account report
Datetime
Research result
Market tools
Account tools

Account resources:
accounts://accounts_server/{name}
accounts://strategy/{name}

Context reduction:
Remove portfolio_value_time_series

Trader modes:
Trading
Rebalancing

Mode state:
self.do_trade

Mode toggle:
Changes after every run

Known issue:
Mode changes even when run fails

MCP lifecycle:
AsyncExitStack

Maximum turns:
30

Concurrent loop:
asyncio.gather()

Default interval:
60 minutes

Market control:
RUN_EVEN_WHEN_MARKET_IS_CLOSED

Multi-model mode:
USE_MANY_MODELS

Tracing:
LogTracer

Trace storage:
SQLite logs

Dashboard:
Gradio app.py

Trading engine:
backend.trading_floor

Architecture split:
Execution plane
Presentation plane

Major security issue:
No human approval before trade

Major reliability issue:
Exceptions are printed and swallowed

Major context issue:
Research provenance is unstructured

Major policy issue:
Prompt rules are not deterministic

Production needs:
Risk engine
Approval
Structured evidence
Strategy versions
Durable run state
Transactions
Reconciliation
Cost budgets
```

---

# 96. Next Episode

Lab ถัดไปจะนำ Backend เดิมไปเชื่อมกับ Frontend Application ที่แยกจาก Trading Engine

สิ่งที่ควรจับตา:

```text
API boundary
Backend/frontend separation
Deployment
Persistent processes
Authentication
Frontend state
Production observability
Operational controls
```

คำถามสำคัญคือ:

> เมื่อ Agentic Backend สามารถทำงานต่อเนื่องและเปลี่ยน State ได้แล้ว Frontend ควรเป็นเพียง Dashboard หรือควรเพิ่ม Approval, Pause, Resume, Audit และ Emergency-stop Controls เพื่อให้มนุษย์สามารถควบคุมระบบได้จริง?
