# Week 6 — Lab 4: Autonomous Traders

Notebook:

```text
6_mcp/4_lab4.ipynb
```

Lab 4 คือ **Capstone ของ Week 6** นำสิ่งที่เรียนจากสาม Labs ก่อนหน้ามาประกอบเป็นระบบเดียว:

```text
Custom MCP Servers
+ Persistent Memory
+ Web Research
+ Market Data
+ Account State
+ Push Notifications
+ Multiple Agent Roles
+ Continuous Execution Loop
+ Dashboard
```

> ระบบนี้เป็นการจำลองเพื่อศึกษา Agent Architecture ไม่ควรใช้ตัดสินใจหรือซื้อขายสินทรัพย์จริงตามที่ Notebook เตือนไว้โดยตรง.

---

## Learning Objectives

หลังจบ Lab นี้ควรเข้าใจว่า:

1. Trader Agent และ Researcher Agent แบ่ง Context กันอย่างไร
2. Agent-as-Tool แตกต่างจาก Handoff อย่างไร
3. Trader ใช้ MCP Servers ชุดใด
4. Researcher ใช้ MCP Servers ชุดใด
5. Memory ถูกแยกตาม Trader อย่างไร
6. `AsyncExitStack` จัดการ MCP Lifecycle หลายตัวอย่างไร
7. Trader ใช้ Strategy และ Account State สร้าง Runtime Message อย่างไร
8. Trading กับ Rebalancing สลับกันอย่างไร
9. Traders สี่ตัวรันพร้อมกันผ่าน `asyncio.gather()` อย่างไร
10. Custom Trace Processor ส่งข้อมูลไป Dashboard อย่างไร
11. Gradio Dashboard แยกจาก Trading Engine อย่างไร
12. Multi-model Routing ทำงานอย่างไร
13. จุดใดเป็น Agent Decision และจุดใดเป็น Deterministic Control
14. ระบบยังขาด Risk Controls ใดก่อนนำไปใช้งานจริง

---

# 1. Architecture

ระบบมี Agents หลักสอง Role:

```text
Trader Agent
└── Researcher Agent as a Tool
```

Trader แต่ละตัวมี MCP Servers:

```text
Accounts MCP
Push Notification MCP
Market Data MCP
```

Researcher มี MCP Servers:

```text
Fetch MCP
Tavily Search MCP
Persistent Memory MCP
```

ภาพรวม:

```text
Trader
├── Researcher tool
│   ├── Fetch MCP
│   ├── Tavily Search MCP
│   └── Personal Memory MCP
│
├── Accounts MCP
├── Market MCP
└── Push MCP
```

Notebook อธิบายว่ามี MCP Server หกประเภท และ Trader แต่ละตัวเรียก Researcher เป็น Tool แทนการค้นเว็บด้วยตัวเอง.

---

# 2. ทำไมต้องแยก Trader กับ Researcher

Trader ต้องตัดสินใจเรื่อง:

```text
ควรซื้อหรือขายหรือไม่
ซื้อหุ้นอะไร
ใช้เงินเท่าไร
Portfolio ต้อง Rebalance หรือไม่
```

Researcher รับผิดชอบ:

```text
ค้นข่าว
อ่านเว็บไซต์
รวบรวมข้อมูลบริษัท
จำสิ่งที่ค้นพบ
สรุปโอกาสและความเสี่ยง
```

การแยกนี้ไม่ได้ทำเพียงเพราะบริษัทมนุษย์มีตำแหน่ง “Trader” และ “Researcher”

เหตุผลทาง Architecture คือการจำกัด Context:

```text
Researcher
→ เห็น Search, Fetch และ Memory

Trader
→ เห็น Account, Market, Push และผลวิจัย
```

Model แต่ละตัวจึงเห็นเฉพาะ Tools และ Instructions ที่สัมพันธ์กับหน้าที่ของมัน ซึ่งเป็นหลัก **Context Engineering** ที่ต่อเนื่องจาก Lab 3.

---

# 3. Trader MCP Servers

`trader_mcp_servers()` สร้าง Servers สามตัว:

```python
def trader_mcp_servers():
    params = [
        accounts_server,
        push_server,
        market_server,
    ]

    return [
        MCPServerStdio(...)
        for p in params
    ]
```

รายละเอียด:

```text
Accounts Server
→ อ่านและเปลี่ยน Account State
→ ซื้อและขายหุ้น
→ เปลี่ยน Strategy

Push Server
→ ส่ง Notification หลังทำงาน

Market Server
→ อ่านราคาหรือข้อมูลตลาด
```

หากมี `MASSIVE_API_KEY` ระบบใช้ Massive MCP Server เวอร์ชัน `v0.10.0`; ถ้าไม่มีจะใช้ `backend.market_server` ซึ่งคืนราคาจำลอง.

---

# 4. Researcher MCP Servers

`researcher_mcp_servers(name)` สร้าง Servers สามตัว:

```text
Fetch
Tavily Search
Memory
```

```python
def researcher_mcp_servers(name):
    return [
        fetch,
        search,
        memory,
    ]
```

## Fetch

```text
uvx mcp-server-fetch
```

ใช้ดึงและอ่านหน้าเว็บ

## Tavily

```text
npx -y tavily-mcp@latest
```

แต่เปิดให้ Researcher เห็นเฉพาะ:

```text
tavily_search
```

ผ่าน Static Tool Filter

## Memory

```text
npx -y mcp-memory-libsql
```

เก็บลง:

```text
memory/<Trader name>.db
```

ตัวอย่าง:

```text
memory/Warren.db
memory/George.db
memory/Ray.db
memory/Cathie.db
```

นี่ทำให้ Researcher ของแต่ละ Trader มี Long-term Memory แยกจากกัน.

---

# 5. Per-Trader Memory Isolation

Warren และ Cathie อาจค้นข่าวเดียวกัน แต่ตีความและจำต่างกันตาม Strategy

```text
Warren memory
→ Value investing insights

George memory
→ Macro and geopolitical insights

Ray memory
→ Risk and diversification insights

Cathie memory
→ Technology and crypto ETF insights
```

ข้อดี:

```text
Context ของแต่ละ Trader ไม่ปะปน
แต่ละตัวสะสม Expertise ของตนเอง
ลด Cross-agent contamination
```

ข้อจำกัด:

```text
Research ซ้ำกัน
ใช้ Storage เพิ่ม
ข้อมูลอาจขัดกัน
แต่ละ Memory อาจล้าสมัยคนละเวลา
```

Memory Isolation ใน Code เกิดจากการใช้ชื่อ Trader สร้างไฟล์ LibSQL คนละไฟล์.

---

# 6. Trader Strategies

ระบบมี Traders สี่ตัว:

| Trader          | Style                                 |
| --------------- | ------------------------------------- |
| Warren Patience | Value investing และถือระยะยาว         |
| George Bold     | Aggressive macro และ Contrarian       |
| Ray Systematic  | Diversification และ Risk parity       |
| Cathie Crypto   | Disruptive innovation และ Crypto ETFs |

Strategies ถูกตั้งใน `reset.py` และบันทึกลง Account ของแต่ละ Trader.

Reset ทั้งหมดได้ด้วย:

```powershell
cd 6_mcp
uv run -m backend.reset
```

Notebook Comment `reset_traders()` ไว้ เพื่อไม่ลบสถานะ Portfolio ที่สะสมไว้โดยไม่ตั้งใจ.

---

# 7. Researcher Agent

```python
async def get_researcher(
    mcp_servers,
    model_name,
) -> Agent:

    return Agent(
        name="Researcher",
        instructions=researcher_instructions(),
        model=get_model(model_name),
        mcp_servers=mcp_servers,
    )
```

Researcher ได้รับ:

```text
Instructions
Model
Fetch MCP
Tavily MCP
Memory MCP
```

Researcher Instructions ระบุให้:

```text
ค้นหลายครั้งเพื่อเห็นภาพรวม
ใช้ Fetch หาก Search ติด Rate Limit
อ่านข้อมูลเดิมจาก Knowledge Graph
เก็บข้อมูลบริษัทและ Market Conditions
จำ URL ที่น่าสนใจ
สะสม Expertise ข้าม Runs
```

---

# 8. Researcher Loop

```text
Trader requests research
        ↓
Researcher searches memory
        ↓
Searches Tavily
        ↓
Fetches useful pages
        ↓
Compares information
        ↓
Stores durable facts in memory
        ↓
Returns research summary
```

Researcher อาจใช้ข้อมูลเก่าจาก Memory ร่วมกับข้อมูลใหม่จาก Web

ดังนั้นต้องระวัง:

```text
Old memory
+ new search
→ Conflicting context
```

Prompt ยังไม่ได้กำหนด Policy ชัดว่า Source ใหม่ควรแก้หรือแทน Memory เดิมอย่างไร

---

# 9. Agent-as-Tool

Researcher ถูกเปลี่ยนเป็น Tool ด้วย:

```python
researcher.as_tool(
    tool_name="Researcher",
    tool_description=research_tool(),
)
```

```python
async def get_researcher_tool(...):
    researcher = await get_researcher(...)
    return researcher.as_tool(...)
```

จากมุมมองของ Trader:

```text
Researcher
= Tool หนึ่งตัว
```

Trader ส่ง Request เช่น:

```text
Research recent risks affecting my energy holdings.
```

แล้วรับ Research Summary กลับมา

---

# 10. Agent-as-Tool vs Handoff

## Agent-as-Tool

```text
Trader remains in control
→ Calls Researcher
→ Receives result
→ Continues its own reasoning
```

## Handoff

```text
Trader transfers conversation
→ Researcher becomes active agent
→ Researcher continues interaction
```

Lab ใช้ Agent-as-Tool เพราะ Trader ต้องเป็น Decision Owner และใช้ Researcher เป็น Subroutine ไม่ใช่ส่งความรับผิดชอบทั้งหมดออกไป.

---

# 11. สร้าง Trader Agent

```python
async def create_agent(
    self,
    trader_mcp_servers,
    researcher_mcp_servers,
):
    tool = await get_researcher_tool(
        researcher_mcp_servers,
        self.model_name,
    )

    self.agent = Agent(
        name=self.name,
        instructions=trader_instructions(
            self.name
        ),
        model=get_model(
            self.model_name
        ),
        tools=[tool],
        mcp_servers=trader_mcp_servers,
    )
```

Trader จึงมี:

```text
One local tool
→ Researcher Agent

Three MCP server groups
→ Account
→ Push
→ Market
```

---

# 12. Context Boundary ของ Trader

Trader เห็น:

```text
Investment strategy
Current account report
Current datetime
Researcher tool
Account tools
Market tools
Push tool
```

Trader ไม่เห็น Tool Catalog ของ Tavily, Fetch หรือ Memory โดยตรง

นี่ช่วยลด Context:

```text
Trader ไม่ต้องตัดสินว่า
ควร Search หรือ Fetch อย่างไร

Trader ตัดสินเพียงว่า
ต้องการ Research เรื่องอะไร
```

Researcher เป็นผู้จัดการรายละเอียดการค้นคว้า

---

# 13. Prompt–Capability Mismatch

`trader_instructions()` ระบุว่า Trader:

```text
can use entity tools as persistent memory
```

แต่ `trader_mcp_servers()` ให้เพียง:

```text
Accounts
Push
Market
```

Memory MCP อยู่ฝั่ง Researcher เท่านั้น.

ดังนั้น Trader ไม่มี Entity Tools โดยตรง

ความเป็นจริงคือ:

```text
Trader
→ ขอ Researcher ให้จำหรือค้น Memory
```

ไม่ใช่:

```text
Trader
→ เรียก Memory Tools เอง
```

Production Prompt ควรแก้เป็น:

```text
Use the Researcher tool to recall and store
relevant persistent knowledge.
```

เพื่อให้ Instructions ตรงกับ Capability จริง

---

# 14. อ่าน Account ผ่าน MCP Resource

Trader อ่าน Account ด้วย:

```python
account = await read_accounts_resource(
    self.name
)
```

และ Strategy ด้วย:

```python
strategy = await read_strategy_resource(
    self.name
)
```

`accounts_client.py` เปิด Accounts MCP Server ผ่าน Low-level MCP Client แล้วเรียก URI:

```text
accounts://accounts_server/<name>
accounts://strategy/<name>
```

สิ่งนี้แสดงการใช้ MCP สองรูปแบบในระบบเดียว:

```text
MCP Tools
→ Agent เป็นผู้เรียก

MCP Resources
→ Application Code เป็นผู้เรียก
```

---

# 15. Account Report ถูกลดขนาดก่อนส่ง Model

```python
account_json.pop(
    "portfolio_value_time_series",
    None,
)
```

ก่อนส่ง Account ให้ Trader

เหตุผลเชิง Context Engineering:

```text
Time series อาจยาวขึ้นเรื่อย ๆ
ไม่จำเป็นต่อการตัดสินใจทุกครั้ง
เพิ่ม Tokens
ลด Signal-to-noise ratio
```

นี่เป็นตัวอย่างที่ดีของ:

```text
Retrieve full state
→ Remove irrelevant context
→ Send compact state to Model
```

---

# 16. Hidden Side Effect ของ Account Resource

`read_accounts_resource()` อ่าน Resource ที่ท้ายที่สุดเรียก `Account.report()`

แต่ `report()`:

```text
คำนวณ Portfolio
เพิ่ม Time-series snapshot
บันทึก Account
เขียน Log
```

ดังนั้นการ “อ่าน Account” มี Side Effect คือเพิ่ม Snapshot

```text
Resource read
→ State mutation
```

นี่ช่วย Dashboard ได้ แต่ทำให้ Operation ไม่ใช่ Pure Read และอาจเกิด Snapshot ซ้ำจากการอ่านหลายครั้ง.

Production ควรแยก:

```text
read_account()
record_portfolio_snapshot()
```

ออกจากกัน

---

# 17. Trading Message

เมื่อ `do_trade=True` ระบบใช้:

```python
trade_message(
    name,
    strategy,
    account,
)
```

Prompt สั่งให้ Trader:

```text
ค้นหาโอกาสใหม่
ใช้ Researcher
ตรวจราคาและข้อมูลบริษัท
ทำ Trade ตาม Strategy
ไม่จำเป็นต้อง Rebalance
ส่ง Push Notification
สรุป Portfolio สั้น ๆ
```

Flow:

```text
Research new opportunities
→ Check price
→ Check cash
→ Decide
→ Execute trades
→ Notify
```

---

# 18. Rebalance Message

เมื่อ `do_trade=False` ระบบใช้:

```python
rebalance_message(...)
```

Prompt เน้น:

```text
ตรวจ Holdings เดิม
ค้นข่าวที่กระทบ Portfolio
ไม่ต้องหาโอกาสใหม่
ขายหรือลด Position ตามความจำเป็น
เรียนรู้จากผล Trade เดิม
แก้ Strategy ได้
ส่ง Notification
```

Flow:

```text
Inspect existing portfolio
→ Research related developments
→ Compare with strategy
→ Rebalance
→ Optionally update strategy
→ Notify
```

---

# 19. Trading–Rebalancing Toggle

เริ่มต้น:

```python
self.do_trade = True
```

หลังทุก Run:

```python
self.do_trade = not self.do_trade
```

ดังนั้น:

```text
Run 1 → Seek opportunities
Run 2 → Rebalance
Run 3 → Seek opportunities
Run 4 → Rebalance
```

ข้อดี:

```text
ไม่ซื้อเพิ่มทุก Cycle
มีช่วงตรวจ Portfolio
Behavior เข้าใจง่าย
```

ข้อจำกัด:

```text
Schedule เป็น Boolean Toggle
ไม่ได้อิงสภาพ Portfolio
ไม่ได้อิง Market Event
ไม่ได้อิงความเสี่ยง
```

---

# 20. Failure ยังทำให้ Mode สลับ

```python
async def run(self):
    try:
        await self.run_with_trace()
    except Exception as e:
        print(...)

    self.do_trade = not self.do_trade
```

แม้ Trading Run จะล้มเหลว ระบบก็ยังเปลี่ยนไป Rebalancing รอบถัดไป.

ตัวอย่าง:

```text
Trading Run fails
→ No trade was performed
→ do_trade changes to False
→ Next run attempts rebalance
```

Safer Pattern:

```python
success = False

try:
    await self.run_with_trace()
    success = True
finally:
    if success:
        self.do_trade = not self.do_trade
```

หรือเก็บ Workflow State แบบ Explicit:

```text
READY_TO_RESEARCH
READY_TO_TRADE
READY_TO_REBALANCE
FAILED_RETRYABLE
FAILED_TERMINAL
```

---

# 21. `AsyncExitStack`

Trader ต้องเปิด MCP Servers หกตัว:

```python
async with AsyncExitStack() as stack:
    trader_servers = [...]
    researcher_servers = [...]
    await self.run_agent(...)
```

`AsyncExitStack` ช่วย:

```text
เปิด Context Managers แบบ Dynamic
เก็บ Cleanup callbacks
ปิด Servers ย้อนลำดับเมื่อจบ
ปิดทุกตัวแม้เกิด Exception
```

Mental model:

```text
Enter server A
Enter server B
Enter server C
...
Run agent
...
Close server C
Close server B
Close server A
```

---

# 22. MCP Process Cost

ต่อ Trader หนึ่งรอบ ระบบเปิดชุดหลัก:

```text
3 Trader MCP Servers
+
3 Researcher MCP Servers
=
6 persistent subprocesses during the run
```

เมื่อ Traders สี่ตัวรันพร้อมกัน อาจมี MCP Server Instances เปิดพร้อมกันประมาณ:

```text
4 × 6 = 24 subprocesses
```

นอกจากนี้ การอ่าน Account และ Strategy ใช้ `accounts_client.py` เปิด Accounts Server แยกอีกครั้งต่อ Resource Call จึงมี Process Start เพิ่มอีกสองครั้งต่อ Trader ต่อรอบ

นี่เป็นข้อสรุปเชิง Implementation จาก `run_with_mcp_servers()`, `get_account_report()` และ `read_strategy_resource()`.

เหมาะกับ Demo แต่ Production ควรพิจารณา:

```text
Long-lived MCP services
Connection pooling
Shared remote MCP endpoints
Process supervision
Server health checks
```

---

# 23. Trader Runner

```python
await Runner.run(
    self.agent,
    message,
    max_turns=MAX_TURNS,
)
```

กำหนด:

```python
MAX_TURNS = 30
```

Budget นี้ครอบคลุม Trader Loop เช่น:

```text
Researcher call
Market lookup
Account check
Buy/sell
Push notification
Final response
```

แต่:

```text
Stopped within 30 turns
≠ Trade correct
≠ Risk acceptable
≠ Notification sent
```

ต้องมี Deterministic Post-run Validation

---

# 24. Strategy Evolution

Trader Instructions อนุญาตให้:

```text
ทบทวนผล Trade เก่า
เรียนรู้จาก Performance
เปลี่ยน Strategy
```

ผ่าน `change_strategy` Tool.

แนวคิดนี้คือ Self-improving Policy:

```text
Past outcomes
→ Model interpretation
→ Updated strategy text
→ Future decisions
```

แต่ไม่มี Evaluator แยกว่าการแก้ Strategy ดีขึ้นจริงหรือไม่

ความเสี่ยง:

```text
Recent-loss overreaction
Short-term performance chasing
Strategy drift
Self-confirming feedback loop
```

Production ควรให้ Strategy Changes ผ่าน:

```text
Proposed change
→ Backtest
→ Risk evaluation
→ Human approval
→ Activate new version
```

---

# 25. Multi-model Routing

`get_model()` รองรับหลาย Providers:

```text
OpenAI
OpenRouter
DeepSeek
Gemini
Grok
```

Routing:

```python
if "/" in model_name:
    use OpenRouter
elif "deepseek" in model_name:
    use DeepSeek
elif "grok" in model_name:
    use xAI
elif "gemini" in model_name:
    use Gemini
else:
    use model name directly
```

---

# 26. Four Models Mode

Default:

```env
USE_MANY_MODELS=False
```

Traders ทั้งหมดใช้:

```text
gpt-5.4-mini
```

เมื่อเปิด:

```env
USE_MANY_MODELS=True
```

ระบบตั้ง:

```text
Warren → GPT-5.5
George → DeepSeek V4 Flash
Ray → Gemini 3.5 Flash
Cathie → Grok 4.3
```

ตามค่าปัจจุบันใน Repository.

ประโยชน์:

```text
เปรียบเทียบ Model behavior
ลด correlated decision style
ทดลอง Provider interoperability
```

ข้อจำกัด:

```text
Tools support อาจไม่เท่ากัน
Model reasoning ต่างกัน
Costs ต่างกัน
Provider downtime ต่างกัน
ผลลัพธ์เปรียบเทียบยาก
```

---

# 27. Custom Trace ID

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

ส่วนที่เหลือเป็น Random Characters เพื่อให้ความยาวตามรูปแบบที่ SDK ต้องการ.

จากนั้น:

```python
with trace(
    trace_name,
    trace_id=trace_id,
):
```

ทำให้ Custom Trace Processor แยกได้ว่า Event เป็นของ Trader ใด

---

# 28. `LogTracer`

```python
class LogTracer(
    TracingProcessor
):
```

รับ Events:

```text
on_trace_start
on_trace_end
on_span_start
on_span_end
```

แล้วเขียนลง Database ผ่าน:

```python
write_log(
    name,
    type,
    message,
)
```

Messages เช่น:

```text
Started: Warren-trading
Started response
Started function Researcher
Started mcp accounts_server
Ended function
Ended: Warren-trading
```

---

# 29. Dashboard ไม่ได้เห็น Private Chain of Thought

Notebook ใช้คำว่า Dashboard แสดงสิ่งที่ Agents “คิด”

แต่ `LogTracer` บันทึกเพียง Operational Trace:

```text
Span type
Tool or Agent name
MCP server name
Start/end
Errors
```

มันไม่ได้บันทึก Hidden Chain of Thought หรือ Internal Reasoning Tokens.

ดังนั้นควรเรียกว่า:

```text
Agent activity trace
หรือ
Execution timeline
```

ไม่ใช่:

```text
Full private reasoning
```

---

# 30. One Trader Run

Notebook ทดลอง:

```python
add_trace_processor(
    LogTracer()
)

warren = Trader(
    "Warren",
    "Patience",
    "gpt-5.4-mini",
)

await warren.run()
```

Expected Flow:

```text
Open six MCP servers
        ↓
Build Researcher Agent
        ↓
Wrap Researcher as Tool
        ↓
Build Warren Agent
        ↓
Read Account Resource
        ↓
Read Strategy Resource
        ↓
Research opportunities
        ↓
Look up market information
        ↓
Buy/sell if appropriate
        ↓
Send push notification
        ↓
Write trace events
        ↓
Close all MCP servers
```

---

# 31. ตรวจผลผ่าน Account Resource

หลัง Warren ทำงาน Notebook อ่าน Account Resource:

```python
resources = await read_accounts_resource(
    "Warren"
)

info = json.loads(resources)

print(
    info["transactions"][-1]
)
```

นี่เป็นหลักฐานที่ดีกว่าอ่าน Final Text อย่างเดียว:

```text
Agent says traded
≠ Trade persisted

Account transaction exists
= Environment evidence
```

ควรตรวจเพิ่ม:

```text
Balance changed correctly
Holding quantity matches
Transaction price is valid
No duplicated trade
Audit log exists
```

---

# 32. Full Trading Floor

สร้าง Traders:

```python
names = [
    "Warren",
    "George",
    "Ray",
    "Cathie",
]
```

แล้ว Run:

```python
await asyncio.gather(
    *[
        trader.run()
        for trader in traders
    ]
)
```

Flow:

```text
Scheduler tick
        ↓
Check market state
        ↓
Run four Traders concurrently
        ↓
Wait for all
        ↓
Sleep
        ↓
Repeat
```

---

# 33. Scheduler Settings

```env
RUN_EVERY_N_MINUTES=60
RUN_EVEN_WHEN_MARKET_IS_CLOSED=False
USE_MANY_MODELS=False
```

Defaults:

```text
Run every 60 minutes
Skip when live market is closed
Use one model for all traders
```

---

# 34. Market-closed Check มีข้อควรระวัง

`trading_floor.py` ใช้:

```python
is_market_open()
```

แต่ `market.py` คืน `True` เมื่อ:

```text
ไม่มี MASSIVE_API_KEY
หรือ
Massive API ติดต่อไม่ได้
```

ดังนั้นใน Simulator Mode ระบบจะถือว่า Market เปิดตลอด และ `RUN_EVEN_WHEN_MARKET_IS_CLOSED=False` จะไม่หยุดรอบทำงาน.

นี่เหมาะกับ Demo แต่ควรแสดง Mode ให้ชัดเจน:

```text
LIVE
DELAYED
SIMULATED
FALLBACK
```

---

# 35. Concurrent Database Writes

Traders สี่ตัวทำงานพร้อมกันและใช้ SQLite File ร่วมกันสำหรับ:

```text
Accounts
Transactions
Trace logs
```

แต่ละ Trader ใช้ Account คนละชื่อ จึงลดการชนกันที่ Business Record

อย่างไรก็ตาม Database File และ Logs Table ยังเป็น Shared Resource

จุดเสี่ยง:

```text
database is locked
Write contention
Lost log events
Partial operations
```

โดยเฉพาะเมื่อ MCP Servers และ Trace Processor เขียนพร้อมกันหลาย Process

Production ควรเพิ่ม:

```text
WAL mode
busy timeout
Explicit transactions
Retry policy
Dedicated database service
```

---

# 36. Exceptions ถูกกลืน

```python
try:
    await self.run_with_trace()
except Exception as e:
    print(
        f"Error running trader {self.name}: {e}"
    )
```

ผลคือ Scheduler ไม่ล้มทั้งหมดเมื่อ Trader ตัวหนึ่งผิดพลาด ซึ่งดีสำหรับ Fault Isolation

แต่ข้อเสีย:

```text
ไม่มี Structured failure state
ไม่มี retry classification
ไม่มี alert ที่รับประกัน
ไม่มี stack trace
Scheduler อาจคิดว่ารอบจบปกติ
```

ควรเก็บ:

```text
run_id
trader
phase
exception type
stack trace
retryable
start/end time
```

---

# 37. Dashboard Architecture

Dashboard เริ่มด้วย:

```powershell
cd 6_mcp
uv run app.py
```

Trading Engine เริ่มอีก Terminal:

```powershell
cd 6_mcp
uv run -m backend.trading_floor
```

`app.py` สร้าง Gradio UI และเปิด Browser:

```python
ui = create_ui()

ui.launch(
    theme=...,
    css=css,
    js=js,
    inbrowser=True,
)
```

Architecture:

```text
Trading Engine
→ Writes accounts and logs
→ SQLite

Dashboard
→ Reads accounts and logs
→ Renders UI
```

---

# 38. Execution Plane vs Presentation Plane

## Execution Plane

```text
backend.trading_floor
Trader agents
MCP servers
Model calls
Trades
Notifications
```

## Presentation Plane

```text
app.py
Gradio
Charts
Transactions
Activity logs
```

ข้อดีของการแยก:

```text
UI Restart ไม่ควรหยุด Traders
Trader Loop ไม่ขึ้นกับ Browser UI
ง่ายต่อการเปลี่ยน Frontend
```

Lab 5 จะนำ Backend เดิมไปเชื่อม Production Frontend แยกต่างหากตามคำอธิบายใน Notebook.

---

# 39. จุดที่ Model มี Authority สูง

Trader Instructions สั่งให้ Agent:

```text
ค้นคว้า
ตัดสินใจ
ซื้อหุ้น
ขายหุ้น
เปลี่ยน Strategy
ส่ง Notification
```

นี่เป็น Authority Chain:

```text
Untrusted web information
        ↓
Researcher summary
        ↓
Trader reasoning
        ↓
Market lookup
        ↓
Account mutation
        ↓
External notification
```

ไม่มี Human Approval อยู่ระหว่าง Decision กับ Trade Execution

นี่คือเหตุผลที่ Course เตือนว่าเป็น Simulation เท่านั้น

---

# 40. Missing Risk Controls

ระบบยังไม่มี Hard Limits เช่น:

```text
Maximum position size
Maximum order value
Maximum daily loss
Maximum portfolio concentration
Allowed-symbol list
Trade-count limit
Minimum cash reserve
Stop-loss policy
Human approval
```

Prompt บอกให้ตรวจ Cash และขนาด Position แต่เป็น Soft Instruction ไม่ใช่ Deterministic Rule.

Production ต้องบังคับใน Account/Order Layer เช่น:

```python
if order_value > MAX_ORDER_VALUE:
    reject()

if concentration_after_trade > MAX_CONCENTRATION:
    reject()
```

---

# 41. Researcher Output ไม่ใช่ Verified Evidence

Trader ได้ Researcher Summary กลับมา แต่ไม่มี Structured Schema บังคับว่า Research ต้องมี:

```text
Source URL
Publisher
Event date
Retrieved date
Confidence
Contradicting evidence
```

Trader จึงอาจตัดสินใจจากข้อความสรุปที่ตรวจย้อนหลังยาก

Safer `ResearchResult`:

```json
{
  "thesis": "...",
  "risks": ["..."],
  "sources": [
    {
      "url": "...",
      "published_at": "...",
      "claim": "..."
    }
  ],
  "confidence": 0.72,
  "as_of": "..."
}
```

---

# 42. Researcher Memory Poisoning

Researcher ค้น Web แล้วเก็บข้อมูลลง Memory เดียวกับที่ใช้ในอนาคต

```text
Bad webpage
→ Researcher stores claim
→ Future Researcher recalls claim
→ Trader acts on it
```

Tool Filtering ช่วยจำกัด Search Tool แต่ไม่ได้ตรวจคุณภาพ Content

ควรเพิ่ม:

```text
Source allowlist
Fact verification
Memory provenance
Expiry
Correction workflow
No storage from untrusted instructions
```

---

# 43. Push Notification ไม่ใช่ Trade Confirmation

Push Server ใน Lab ส่งข้อความผ่าน External API

แต่ Implementation เดิมไม่ได้ตรวจ HTTP Response อย่างเข้มงวด

จึงอาจเกิด:

```text
Trader completed trade
→ Push request failed
→ Tool still says sent
```

ดังนั้น Notification ควรเป็น Observability Channel ไม่ใช่ Source of Truth

Source of Truth ควรเป็น:

```text
Account transaction record
Order ID
Execution receipt
Audit log
```

---

# 44. Strategy Drift

Trader สามารถแก้ Strategy ได้เอง

Flow:

```text
Recent trade result
→ Model interprets performance
→ change_strategy()
→ Future policy changes
```

Risk:

```text
เปลี่ยน Strategy เพราะ Noise
เปลี่ยนจาก Long-term เป็น Short-term
Overfit กับเหตุการณ์ล่าสุด
อธิบายเหตุผลย้อนหลังไม่ได้
```

ควร Version Strategy:

```json
{
  "version": 3,
  "previous_version": 2,
  "proposed_by": "Warren",
  "reason": "...",
  "evidence": ["..."],
  "approved": false
}
```

---

# 45. Suggested Production Control Flow

```text
Research
        ↓
Structured evidence validation
        ↓
Trade proposal
        ↓
Deterministic risk engine
        ↓
Human approval
        ↓
Order execution
        ↓
Execution reconciliation
        ↓
Notification
        ↓
Portfolio update
```

แยก Tools เป็น:

```text
research_stock
preview_trade
validate_trade
request_approval
execute_trade
get_execution_status
```

แทนการให้ `buy_shares` เป็น Operation ที่ Model เรียกได้ทันที

---

# 46. Exercises

## Exercise 1 — Prompt–Capability Alignment

แก้ Trader Instructions จาก:

```text
Use your entity tools...
```

เป็น:

```text
Use the Researcher tool to recall and store
persistent research knowledge.
```

ตรวจ Trace ว่า Trader เรียก Researcher เมื่อต้องใช้ Memory

---

## Exercise 2 — Failure-safe Toggle

แก้ `Trader.run()` ให้ `do_trade` สลับเฉพาะเมื่อ Run สำเร็จ

ทดสอบด้วยการทำให้ Tavily Server เริ่มไม่สำเร็จ

---

## Exercise 3 — Structured Research Result

ให้ Researcher คืน:

```text
Summary
Bull case
Bear case
Sources
Dates
Confidence
```

แล้วบังคับ Trader ไม่ให้ Trade เมื่อไม่มี Sources

---

## Exercise 4 — Deterministic Position Limit

เพิ่ม Rule:

```text
หุ้นหนึ่งตัวไม่เกิน 20% ของ Portfolio
Order หนึ่งครั้งไม่เกิน $1,000
Cash reserve อย่างน้อย 10%
```

ให้ Account Server ปฏิเสธ Operation ที่เกิน Limit

---

## Exercise 5 — Process-count Measurement

ระหว่าง Traders สี่ตัวรัน ใช้ Task Manager หรือ Process Explorer ดูจำนวน:

```text
uv
uvx
npx
Python
Node
```

เปรียบเทียบกับจำนวน MCP Server Instances ที่ Architecture สร้าง

---

## Exercise 6 — Add Run Records

สร้าง Table:

```sql
trader_runs (
    run_id,
    trader,
    mode,
    started_at,
    finished_at,
    status,
    error,
    trade_count,
    model
)
```

ให้ Dashboard แยก:

```text
Success
Failed
Timed out
Skipped because market closed
```

---

# Checklist

```text
□ เข้าใจ Trader และ Researcher Roles
□ ตรวจ Trader MCP Servers ทั้งสามตัว
□ ตรวจ Researcher MCP Servers ทั้งสามตัว
□ เข้าใจ Per-trader Memory Database
□ เข้าใจ Agent-as-Tool กับ Handoff
□ เปิด Researcher และดู Trace
□ ตรวจ Researcher Tool name และ description
□ Run Warren หนึ่งรอบ
□ ตรวจ Transaction ผ่าน Account Resource
□ ตรวจ Trace Log ในฐานข้อมูล
□ เข้าใจ Trade/Rebalance Toggle
□ พบว่า Failure ยังทำให้ Toggle
□ เข้าใจ AsyncExitStack
□ เปิด Gradio Dashboard
□ เปิด Trading Engine แยก Terminal
□ ตรวจ Live กับ Simulated Market Mode
□ เข้าใจว่า Dashboard แสดง Trace ไม่ใช่ Private CoT
□ ระบุ Risk Controls ที่ยังขาดได้
```

---

# แก่นของ Week 6 — Lab 4

```text
Researcher
= Search + Fetch + Persistent Memory

Trader
= Strategy + Account + Market + Execution

Researcher as Tool
= Delegated context gathering

MCP
= Common capability boundary

AsyncExitStack
= Multi-server lifecycle

Trace Processor
= Operational observability

Trading Floor
= Concurrent recurring agent loop

Dashboard
= Presentation plane
```

บทเรียนสำคัญที่สุดคือ:

> **Multi-agent Architecture ที่ดีไม่ได้เริ่มจากการเลียนแบบตำแหน่งงานของมนุษย์ แต่เริ่มจากการแยก Context และ Authority ให้แต่ละ Agent เห็นเฉพาะข้อมูลและ Tools ที่จำเป็นต่อหน้าที่**

อีกบทเรียนคือ:

> **Agent-as-Tool ทำให้ Agent หลักยังเป็นเจ้าของการตัดสินใจ ขณะที่มอบงานเฉพาะทางให้ Agent ย่อย โดยไม่ต้องส่ง Conversation Control ทั้งหมดเหมือน Handoff**

และแก่นเชิง Production คือ:

> **เมื่อ Agent สามารถเปลี่ยน Financial State ได้ ความน่าเชื่อถือไม่ควรพึ่ง Prompt เพียงอย่างเดียว ต้องมี Risk Engine, Approval, Structured Evidence, Transaction Reconciliation, Durable Failure State และ Audit Trail อยู่ระหว่าง Agent Decision กับ Side Effect จริง**
