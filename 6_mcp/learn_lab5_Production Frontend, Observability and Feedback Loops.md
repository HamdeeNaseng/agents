# Week 6 — Lab 5: Production Frontend, Observability และ Feedback Loop

Notebook:

```text
6_mcp/5_lab5.ipynb
```

Lab 5 เป็นบทสรุปของ Week 6 และทั้งหลักสูตร โดยไม่ได้เพิ่ม Agent Framework ใหม่ แต่ปรับ Autonomous Trading Floor จาก Lab 4 ในสามด้าน:

```text
1. Observability
2. Evaluation and Feedback
3. Production-style Frontend
```

Trading Engine และ Database เดิมยังทำงานเหมือนเดิม สิ่งที่เพิ่มคือ FastAPI สำหรับเปิดข้อมูลผ่าน HTTP และ Vite + TypeScript Frontend สำหรับแสดงผลโดยไม่อ่าน SQLite โดยตรง.

> ระบบนี้ยังเป็น Trading Simulation เพื่อเรียน Agent Architecture ไม่ใช่ระบบสำหรับซื้อขายเงินจริง

---

## Learning Objectives

หลังจบ Lab นี้ควรเข้าใจว่า:

1. Production-style Frontend ต่างจาก Gradio Dashboard อย่างไร
2. Trading Engine, API และ Frontend แยก Process กันอย่างไร
3. FastAPI ทำหน้าที่เป็น Read Model ของระบบอย่างไร
4. Frontend Polling แตกต่างจาก Event Streaming อย่างไร
5. Custom Trace Processor ส่งข้อมูลไปแสดงใน UI อย่างไร
6. Operational Trace ต่างจาก Private Chain of Thought อย่างไร
7. Portfolio Performance ถูกนำกลับมาเป็น Feedback อย่างไร
8. Strategy Self-modification มีข้อจำกัดอะไร
9. TypeScript Interfaces เชื่อมกับ Backend JSON Contract อย่างไร
10. Vite Proxy ช่วยหลีกเลี่ยง CORS ใน Development อย่างไร
11. Client-side State และ Backend State ต่างกันอย่างไร
12. Chart History ใน Browser แตกต่างจาก Persisted Time Series อย่างไร
13. Read-only API ลด Authority Surface อย่างไร
14. ระบบยังขาด Authentication, API Schema และ Operational Controls อะไร
15. Production Version ควรเพิ่ม Pause, Approval, Audit และ Emergency Stop อย่างไร

---

# 1. ภาพรวม Architecture

Lab 4:

```text
Trading Engine
    ↓
SQLite
    ↓
Gradio Dashboard อ่าน Database โดยตรง
```

Lab 5:

```text
Trading Engine
    ↓ writes
SQLite
    ↑ reads
FastAPI
    ↓ JSON over HTTP
Vite + TypeScript Frontend
```

มีสาม Process แยกจากกัน:

```text
Process 1
Trading Engine

Process 2
FastAPI Server

Process 3
Vite Development Server
```

Trading Engine ไม่เรียกผ่าน FastAPI แต่ยังเขียน Account และ Logs ลง Database โดยตรง ส่วน API ทำหน้าที่อ่าน State แล้วแปลงเป็น JSON ให้ Frontend.

---

# 2. Improvement 1 — Observability

จาก Lab 4 ระบบมี Custom Trace Processor:

```python
add_trace_processor(
    LogTracer()
)
```

`LogTracer` รับ Events เช่น:

```text
Trace started
Agent span started
Function call started
MCP call started
Generation ended
Response ended
Account event
```

จากนั้นเขียนลง `logs` Table ใน SQLite.

Flow:

```text
OpenAI Agents SDK
        ↓ trace stream
LogTracer
        ↓
SQLite logs table
        ↓
FastAPI
        ↓
Frontend activity panel
```

Lab 5 ทำให้ Trace ไม่ได้อยู่เฉพาะ OpenAI Trace Viewer แต่กลายเป็นส่วนหนึ่งของ Product UI.

---

# 3. Dashboard ไม่ได้แสดง Private Chain of Thought

สิ่งที่ Frontend แสดงคือ:

```text
Agent activity
Tool calls
MCP server activity
Generation spans
Response spans
Account events
Errors
```

มันไม่ได้แสดง:

```text
Hidden reasoning tokens
Private chain of thought
Model deliberation แบบคำต่อคำ
```

`LogTracer` บันทึกเพียงชื่อ Span, Type, Server, Start, End และ Error.

ชื่อที่เหมาะสมคือ:

```text
Execution timeline
Operational trace
Agent activity log
```

Observability ที่ดีควรตอบว่า:

```text
เกิดอะไรขึ้น
เกิดเมื่อใด
Agent ไหนทำ
Tool ไหนถูกเรียก
Operation สำเร็จหรือไม่
```

ไม่จำเป็นต้องเปิดเผย Internal Reasoning ของ Model

---

# 4. Improvement 2 — Evaluation and Feedback

Trader ไม่ได้ถูกล็อกไว้กับ Strategy เริ่มต้น

มันมี Tool:

```text
change_strategy
```

และ Instructions บอกให้พิจารณาผลการซื้อขายที่ผ่านมา แล้วปรับ Strategy สำหรับรอบถัดไป

Flow:

```text
Strategy version A
        ↓
Trading decisions
        ↓
Portfolio result
        ↓
Agent evaluates result
        ↓
change_strategy()
        ↓
Strategy version B
```

Lab เรียกสิ่งนี้ว่า Feedback Loop เพราะ Outcome ในโลกภายนอกถูกส่งกลับไปเปลี่ยน Decision Policy ของ Agent.

---

# 5. นี่เป็น Feedback แต่ยังไม่ใช่ Evaluation ที่แข็งแรง

Portfolio Return เป็น Outcome Signal แต่ไม่ได้บอกโดยตรงว่า:

```text
กำไรมาจาก Strategy ที่ดี
หรือมาจากตลาดขึ้นทั้งตลาด

ขาดทุนมาจาก Strategy ที่แย่
หรือเป็น Noise ระยะสั้น

การเปลี่ยน Strategy ใหม่
จะทำให้ผลลัพธ์ในอนาคตดีขึ้นจริงหรือไม่
```

ระบบปัจจุบันให้ Agent คนเดียวกัน:

```text
ตัดสินใจ
→ ลงมือ
→ ประเมินผลงานตัวเอง
→ แก้ Strategy ของตัวเอง
```

ยังไม่มี Independent Evaluator, Benchmark, Backtest หรือ Approval Gate

จึงอาจเกิด:

```text
Strategy drift
Overfitting
Loss chasing
Recent-event bias
Reward hacking
```

Production Flow ที่ปลอดภัยกว่า:

```text
Agent proposes strategy change
        ↓
Independent evaluation
        ↓
Backtest
        ↓
Risk review
        ↓
Human approval
        ↓
Activate new version
```

---

# 6. Strategy ควรถูก Version

ไม่ควรเขียนทับ Strategy เก่าโดยไม่มี History

ตัวอย่างข้อมูล:

```json
{
  "strategy_id": "warren",
  "version": 4,
  "previous_version": 3,
  "proposal": "Increase minimum cash reserve",
  "reason": "Recent concentration created excessive volatility",
  "evidence": ["run-128", "run-132"],
  "created_by": "Warren Agent",
  "status": "pending_review"
}
```

ควรสามารถ:

```text
เปรียบเทียบ Strategy versions
Rollback
ดูผู้อนุมัติ
ดู Evidence
ดูผลหลังเปลี่ยน
```

นี่เปลี่ยน Self-modification จากการแก้ Prompt ธรรมดาให้กลายเป็น Managed Policy Evolution

---

# 7. Improvement 3 — Production-style Frontend

Lab 4 ใช้ Gradio:

```text
Python UI
→ อ่าน SQLite โดยตรง
```

เหมาะกับ:

```text
Prototype
Internal tool
Notebook demonstration
Rapid development
```

Lab 5 ใช้:

```text
FastAPI Backend
+
Vite TypeScript Frontend
```

เหมาะกับ Architecture ที่ Frontend และ Backend แยก Deploy หรือเปลี่ยน Client ได้ง่ายกว่า.

อย่างไรก็ตาม คำว่า “Production Frontend” ใน Lab หมายถึง **Production-style separation**

ยังไม่ใช่ Production-complete System เพราะยังขาด:

```text
Authentication
Authorization
Deployment configuration
TLS
API versioning
Rate limiting
Health checks
Operational control endpoints
```

---

# 8. FastAPI Layer

ไฟล์:

```text
backend/api.py
```

สร้าง Application:

```python
app = FastAPI(
    title="Trading Floor"
)
```

API นี้เป็น Read-only โดยออกแบบมาเพื่อ:

```text
อ่าน Account
อ่าน Portfolio
อ่าน Holdings
อ่าน Transactions
อ่าน Time Series
อ่าน Logs
อ่าน Market Mode
```

มันไม่เริ่ม Trader และไม่มี Endpoint สำหรับซื้อหรือขายหุ้น.

นี่เป็น Boundary ที่ดี:

```text
Frontend
→ Read capability only

Trading Engine
→ Write capability
```

ลดโอกาสที่ UI Bug หรือ Browser Request จะเปลี่ยน Financial State โดยตรง

---

# 9. API Endpoints

## Trader roster

```http
GET /api/traders
```

คืน:

```json
[
  {
    "name": "Warren",
    "lastname": "Patience",
    "model_name": "GPT 5.4 mini"
  }
]
```

## Market information

```http
GET /api/market
```

คืน:

```json
{
  "source": "simulator",
  "is_market_open": true
}
```

## Trader detail

```http
GET /api/traders/Warren
```

คืน:

```text
Balance
Strategy
Portfolio value
Profit/loss
Holdings
Transactions
Time series
```

## Logs

```http
GET /api/traders/Warren/logs?last_n=13
```

คืน Activity Logs พร้อมสีของแต่ละ Event Type.

---

# 10. API เป็น Read Model

API ไม่ได้คืน Account JSON เดิมอย่างเดียว แต่คำนวณข้อมูลเพิ่ม:

```text
Current share price
Market value per holding
Average cost
Unrealized profit
Total portfolio value
Total P/L
```

`holdings_detail()` เรียก Market Provider ต่อ Symbol แล้วสร้างข้อมูลที่ Frontend พร้อม Render.

Pattern นี้เรียกว่า Read Model:

```text
Raw domain state
+
Current market data
+
Derived calculations
        ↓
Frontend-ready response
```

ข้อดีคือ Frontend ไม่ต้องทำ Business Calculation ซ้ำ

---

# 11. Market Lookup Cost

ทุกครั้งที่ Frontend เรียก:

```http
GET /api/traders/{name}
```

API จะเรียก `get_share_price()` สำหรับทุก Holding ของ Trader นั้น.

Frontend Poll Trader สี่ตัวทุก 6 วินาที.

ถ้าใช้ Live Provider อาจเกิด:

```text
API calls จำนวนมาก
Rate limit
Latency
ข้อมูลแต่ละหุ้นมาจากคนละเวลา
ค่าใช้จ่ายเพิ่ม
```

Production ควรใช้:

```text
Market-data cache
Batch quote API
Price snapshot
Shared timestamp
Refresh interval ที่เหมาะสม
```

Response ควรมี:

```json
{
  "price": 210.5,
  "source": "massive",
  "as_of": "2026-08-03T17:55:00Z",
  "delayed": true
}
```

---

# 12. Average-cost Limitation

API คำนวณ Average Cost จากทุก Buy Transaction ของ Symbol:

```python
spend = sum(
    t.price * t.quantity
    for t in account.transactions
    if t.symbol == symbol
    and t.quantity > 0
)
```

กรณีถือหุ้นต่อเนื่อง วิธีนี้พอใช้กับ Average-cost Model

แต่ถ้า:

```text
ซื้อ AAPL
→ ขายออกทั้งหมด
→ ซื้อ AAPL ใหม่ภายหลัง
```

Average Cost อาจรวม Purchases จาก Position รอบเก่าด้วย

Production ต้องใช้:

```text
Tax lots
Position lifecycle
Weighted average with resets
FIFO/LIFO policy
Realized P/L records
```

---

# 13. TypeScript API Client

ไฟล์:

```text
frontend/src/api.ts
```

ประกาศ Interfaces:

```typescript
TraderInfo
Holding
Transaction
TimePoint
TraderDetail
LogRow
MarketInfo
```

และ Generic GET Function:

```typescript
async function get<T>(
  path: string
): Promise<T>
```

ทุก Request ใช้ Relative URL เช่น:

```typescript
get("/api/traders")
get("/api/market")
```

ข้อดี:

```text
Frontend มี Type Checking
API Calls อยู่ในจุดเดียว
Components ไม่ต้องรู้ Fetch details
```

---

# 14. Manual Contract Drift

Backend คืน `dict` และ Frontend เขียน TypeScript Interfaces ด้วยมือ

จึงมี Schema สองชุด:

```text
Python response structure
TypeScript interface
```

ถ้า Backend เปลี่ยน:

```text
portfolio_value
→ total_value
```

แต่ Frontend ไม่เปลี่ยน TypeScript Compiler จะไม่รู้ เพราะข้อมูลมาจาก Runtime JSON

Safer Pattern:

```text
Pydantic response models
→ OpenAPI schema
→ Generate TypeScript client/types
```

เช่น:

```text
openapi-typescript
Orval
NSwag
```

FastAPI มี `/docs` และ OpenAPI อยู่แล้ว แต่ควรเพิ่ม Response Models เพื่อให้ Contract ชัดกว่าการใช้ `list[dict]` หรือ `dict`.

---

# 15. Vite Development Proxy

`vite.config.ts` กำหนด:

```typescript
proxy: {
  "/api": {
    target:
      "http://127.0.0.1:8000"
  }
}
```

Browser เปิด:

```text
http://localhost:5173
```

เมื่อเรียก:

```text
/api/traders
```

Vite ส่งต่อไป:

```text
http://127.0.0.1:8000/api/traders
```

Browser จึงมองว่า Request มาจาก Origin เดียวกัน และไม่ต้องตั้ง CORS ระหว่าง Development.

---

# 16. Production Deployment ยังต้องจัดการ Origin

Development Proxy มีผลเฉพาะตอนใช้:

```powershell
npm run dev
```

เมื่อ Build และ Deploy จริง ต้องเลือก:

```text
Option A
Reverse proxy Frontend และ API
ให้อยู่ Domain เดียวกัน

Option B
Frontend และ API อยู่คนละ Domain
แล้วกำหนด CORS

Option C
FastAPI serve built frontend files
```

ตัวอย่าง Production Boundary:

```text
https://traders.example.com/
→ Static frontend

https://traders.example.com/api/
→ FastAPI
```

การที่ Frontend และ Backend “แยก Host ได้” ไม่ได้แปลว่าจะไม่ต้องตั้ง CORS หรือ Reverse Proxy ใน Production

---

# 17. Frontend Entry Point

ไฟล์:

```text
frontend/src/main.ts
```

เมื่อ Page Load:

```text
Initialize theme
→ Load market information
→ Load trader roster
→ Build four panels
→ Fetch portfolio data
→ Fetch logs
→ Start polling loops
```

Frontend เก็บ State ใน Maps:

```typescript
const states =
  new Map<string, TraderState>();

const panels =
  new Map<string, TraderPanel>();
```

แต่ละ Trader มี State และ UI Panel ของตนเอง

---

# 18. Polling Intervals

```typescript
const DATA_POLL_MS = 6000;
const LOG_POLL_MS = 2000;
```

ดังนั้น:

```text
Portfolio data
→ ทุก 6 วินาที

Activity logs
→ ทุก 2 วินาที
```

ข้อดี:

```text
Implementation ง่าย
ไม่ต้องใช้ WebSocket
Reconnect ง่าย
เหมาะกับ Demo
```

ข้อจำกัด:

```text
Requests เกิดแม้ไม่มีข้อมูลเปลี่ยน
Latency สูงสุดตาม Poll interval
ใช้ API และ Market quota
Server รับ Requests ซ้ำ
หลาย Browser เพิ่ม Load ตามจำนวนผู้ชม
```

---

# 19. Polling Overlap Risk

Code ใช้:

```typescript
setInterval(
  pollData,
  DATA_POLL_MS
);
```

แต่ `pollData()` เป็น Async Function

ถ้า Request รอบก่อนใช้เวลานานกว่า 6 วินาที รอบใหม่อาจเริ่มก่อนรอบเก่าจบ

ผลที่อาจเกิด:

```text
Overlapping requests
Responses กลับผิดลำดับ
State ใหม่ถูกเขียนทับด้วย Response เก่า
API load เพิ่ม
```

Safer Poll Loop:

```typescript
async function loop(): Promise<void> {
  await pollData();
  setTimeout(loop, DATA_POLL_MS);
}
```

หรือใช้:

```text
AbortController
Request sequence number
Backoff
Visibility API
```

---

# 20. Market Badge ถูกโหลดเพียงครั้งเดียว

`loadMarket()` ถูกเรียกตอน `main()` เริ่มต้น แต่ไม่ได้อยู่ใน Polling Interval.

ดังนั้น:

```text
Page เปิดตอน Market ปิด
→ Badge แสดง Market closed

Market เปิดภายหลัง
→ Badge อาจยังแสดง Market closed
```

ควร Refresh:

```text
Market status
Market source
Last update
Connection state
```

ตามช่วงเวลาหรือใช้ Server Event

---

# 21. Frontend Trader State

`TraderState` เก็บ:

```text
Trader information
Latest detail
Chart points
Previous holding prices
Whether chart was seeded
```

เมื่อรับ Detail ใหม่:

```typescript
recordDetail(detail)
```

จะ:

```text
Set current detail
Seed chart from backend time_series ครั้งแรก
Add current portfolio value เป็นจุดใหม่
Trim chart เมื่อยาวเกิน limit
```

---

# 22. Chart เป็น Browser-observed History

Chart ถูก Seed จาก Backend Time Series เพียงครั้งแรก

หลังจากนั้น Frontend เติม:

```typescript
{
  t: Date.now() / 1000,
  value:
    detail.portfolio_value
}
```

ทุกครั้งที่ Poll.

ดังนั้น Chart มีข้อมูลสองชนิด:

```text
Persisted history
→ มาจาก Backend

Viewer history
→ Browser เติมเองทุก 6 วินาที
```

ผลคือ:

```text
Browser A เปิด 1 ชั่วโมง
→ มี Chart history 1 ชั่วโมง

Browser B เพิ่งเปิด
→ มี Chart history สั้นกว่า
```

Reload Page ก็ทำให้ Local Polling History หาย เหลือเฉพาะ Persisted Time Series ที่ Seed ใหม่

ถ้าต้องการ Source of Truth กลาง ทุกจุดควรมาจาก Backend Time-series Endpoint

---

# 23. Portfolio Chart

Frontend ใช้:

```text
uPlot
```

เป็น Chart Library เพียง Runtime Dependency หลักของ Frontend.

`PortfolioChart`:

```text
Resize ตาม Container
ซ่อน Cursor และ Legend
แสดงแกนเวลา
ใช้สีเขียวหรือแดงตาม Trend
สร้าง Gradient fill
รองรับ Theme change
```

Chart ถูกสร้างหลัง Panel ถูกเพิ่มลง DOM เพราะ Library ต้องรู้ขนาด Container จริง

---

# 24. Holdings Heatmap

แต่ละ Holding ถูกแสดงเป็น Tile:

```text
Tile size
→ สัดส่วน Market Value

Tile color
→ Unrealized P/L

Flash direction
→ ราคาขึ้นหรือลงจาก Poll ก่อนหน้า
```

นี่ช่วยให้เห็น Portfolio Concentration ได้เร็วกว่าอ่านตารางอย่างเดียว

ตัวอย่าง:

```text
Tile AAPL ใหญ่มาก
→ Portfolio กระจุกตัวใน AAPL

Tile เป็นสีแดง
→ Unrealized loss
```

---

# 25. Potential DOM Injection Point

Heatmap สร้าง Tile ด้วย:

```typescript
tile.innerHTML = `
  <span>${symbol}</span>
`;
```

ถ้า Symbol มาจากข้อมูลที่ไม่ผ่าน Validation อาจกลายเป็น DOM Injection Point

แม้ Symbol ปกติควรเป็นค่าเช่น `AAPL` แต่ใน Agentic System Model เป็นผู้ส่ง Argument เข้า Trading Tool

Safer:

```typescript
const ticker =
  document.createElement("span");

ticker.textContent = symbol;
```

และ Backend ควร Validate Symbol ด้วย Allowlist หรือ Pattern เช่น:

```text
^[A-Z][A-Z0-9.-]{0,9}$
```

Transactions และ Logs ใช้ `textContent` อยู่แล้ว จึงปลอดภัยกว่าการ Interpolate เข้า `innerHTML`.

---

# 26. Trader Panel

แต่ละ Panel แสดง:

```text
Trader name
Model name
Strategy
Portfolio value
P/L
Portfolio chart
Holdings heatmap
Activity log
Recent trades
```

และสามารถระบุ Trader ที่ Portfolio Value สูงที่สุดเป็น Leader.

Sidebar แสดง:

```text
Market source
Market status
Returns ranking
Theme toggle
```

---

# 27. Returns Ranking

Frontend คำนวณ Return Percentage จาก:

```typescript
initial =
  portfolio_value - pnl;

percentage =
  pnl / initial;
```

จากนั้น Sort จากมากไปน้อย.

นี่ใช้ได้กับ Simulation ที่ Profit/Loss นิยามจาก Initial Balance เดียวกัน

Production Analytics ควรแยก:

```text
Time-weighted return
Money-weighted return
Realized P/L
Unrealized P/L
Cash flows
Benchmark return
Risk-adjusted return
```

ไม่ควรตัดสิน Agent จาก Absolute Return เพียงตัวเดียว

---

# 28. Logs Endpoint

```http
GET /api/traders/{name}/logs
```

อ่าน Logs ล่าสุดจาก SQLite แล้วคืนจากเก่าไปใหม่ พร้อม Color ที่ Backend กำหนด.

Database Query ใช้:

```sql
ORDER BY datetime DESC
LIMIT ?
```

จากนั้น Reverse ก่อนคืน.

จุดที่ควรปรับ:

```text
last_n ยังไม่มี minimum/maximum constraint
ไม่มี pagination
ไม่มี cursor
ไม่มี event ID
```

Safer FastAPI Parameter:

```python
last_n: int = Query(
    default=13,
    ge=1,
    le=100,
)
```

---

# 29. Timestamp Risk

SQLite Logs ใช้:

```sql
datetime('now')
```

ซึ่งเป็น UTC แต่ Timestamp ไม่มี `Z` หรือ Offset.

Frontend แปลง Timestamp แบบไม่มี Timezone เป็น Browser Local Time.

อาจเกิด:

```text
Backend thinks UTC
Frontend interprets local
→ Time shifted
```

Production ควรใช้ ISO 8601 ที่มี Timezone:

```text
2026-08-03T10:59:00Z
```

หรือ:

```text
2026-08-03T17:59:00+07:00
```

และเก็บทุก Event เป็น UTC ก่อนแปลงแสดงตาม User Timezone

---

# 30. Relative Database Path

Database กำหนด:

```python
DB = "accounts.db"
```

จึงขึ้นกับ Current Working Directory

Notebook ย้ำให้เริ่ม API จาก:

```powershell
cd 6_mcp
```

ก่อน Run Uvicorn เพื่อให้ API และ Trading Engine อ่าน File เดียวกัน.

หากเริ่มจาก Directory อื่น:

```text
API
→ อาจสร้าง accounts.db คนละไฟล์

Trading Engine
→ เขียนอีก accounts.db
```

ผลคือ Frontend อาจเห็นข้อมูลว่าง

Production ควรใช้ Absolute Configured Path:

```python
DB = Path(__file__).resolve().parent.parent / "accounts.db"
```

หรือใช้ Database URL จาก Environment

---

# 31. API Coupling กับ Trading Engine

`api.py` Import:

```python
from backend.trading_floor import (
    names,
    lastnames,
    short_model_names,
)
```

ทำให้ Read-only API ผูกกับ Execution Module ซึ่งยัง Import Trader, Model Providers และ Environment Configs

Safer:

```text
backend/config.py
หรือ
backend/roster.py
```

เก็บ:

```text
Trader names
Display names
Model names
Market configuration
```

แล้วให้ทั้ง API และ Trading Engine Import จาก Config กลาง

---

# 32. API ยังไม่มี Response Models

Routes ปัจจุบันประกาศ:

```python
def get_traders() -> list[dict]
def get_market() -> dict
def get_trader() -> dict
```

ควรสร้าง Pydantic Models:

```python
class MarketInfo(BaseModel):
    source: Literal[
        "massive",
        "simulator"
    ]
    is_market_open: bool
    as_of: datetime

class TraderDetail(BaseModel):
    name: str
    balance: float
    portfolio_value: float
    pnl: float
    holdings: list[Holding]
```

ข้อดี:

```text
Runtime response validation
Better OpenAPI
Generated TypeScript types
Clear backward compatibility
```

---

# 33. Read-only ไม่ได้แปลว่าไม่ต้อง Authentication

API ไม่มี Write Endpoints ซึ่งลด Risk มาก

แต่ข้อมูลอาจประกอบด้วย:

```text
Account balance
Portfolio holdings
Strategies
Transactions
Agent logs
Research activity
Model names
```

จึงยังควรมี:

```text
Authentication
Tenant isolation
Authorization
TLS
Audit access logs
Rate limiting
```

โดยเฉพาะเมื่อ Deploy นอก Localhost

---

# 34. Polling vs Server-sent Events

Current Architecture:

```text
Browser
→ Poll API every 2 or 6 seconds
```

ทางเลือก:

## Server-sent Events

เหมาะกับ:

```text
One-way live logs
Portfolio update events
Simple reconnect
```

## WebSocket

เหมาะกับ:

```text
Two-way operational controls
Live approval requests
Pause/resume commands
Interactive status
```

สำหรับ Dashboard แบบ Read-only:

```text
REST
+
SSE สำหรับ Logs
```

อาจเรียบง่ายกว่า WebSocket

---

# 35. API Error Handling ใน Frontend

เมื่อ Fetch ล้มเหลว Frontend ทำเพียง:

```typescript
console.error(...)
```

ผู้ใช้ยังเห็นข้อมูลเก่าโดยไม่มีคำเตือนชัดเจน

ควรมี UI State:

```text
Connected
Refreshing
Stale
Disconnected
Partial failure
```

พร้อม:

```text
Last successful update
Retry countdown
Error banner
```

เพื่อไม่ให้ข้อมูลเก่าดูเหมือนข้อมูลปัจจุบัน

---

# 36. Operational Controls ที่ยังไม่มี

API ปัจจุบันเป็น Read-only และ Frontend เป็น Dashboard เท่านั้น

ยังไม่มี:

```text
Pause trading floor
Resume trading floor
Run one trader
Stop one trader
Approve proposed trade
Reject proposed trade
Emergency stop
Reset trader
Rollback strategy
```

นี่ไม่ใช่ข้อผิดพลาดของ Lab เพราะตั้งใจสร้าง Monitoring Frontend

แต่ Production Agent System ที่เปลี่ยน External State ควรมี Control Plane แยก

---

# 37. Recommended Control Plane

```text
POST /api/control/pause
POST /api/control/resume

POST /api/traders/{name}/run
POST /api/traders/{name}/stop

GET  /api/approvals
POST /api/approvals/{id}/approve
POST /api/approvals/{id}/reject

POST /api/emergency-stop
```

Endpoints เหล่านี้ต้องมี:

```text
Strong authentication
Role-based authorization
CSRF protection
Idempotency
Audit trail
Approval ownership
```

ไม่ควรรวม Write Controls ลง API โดยไม่มี Security Layer

---

# 38. Three-terminal Setup

## One-time frontend setup

```powershell
cd 6_mcp/frontend
npm install
```

## Terminal 1 — API

```powershell
cd 6_mcp
uv run uvicorn backend.api:app --port 8000
```

API docs:

```text
http://localhost:8000/docs
```

## Terminal 2 — Frontend

```powershell
cd 6_mcp/frontend
npm run dev
```

เปิด:

```text
http://localhost:5173
```

## Terminal 3 — Trading Engine

```powershell
cd 6_mcp
uv run -m backend.trading_floor
```

---

# 39. Frontend Build

`package.json` มี Scripts:

```json
{
  "dev": "vite",
  "build": "tsc && vite build",
  "preview": "vite preview"
}
```

Production Build:

```powershell
npm run build
```

จะได้:

```text
frontend/dist/
```

แต่ยังต้องจัดเตรียม:

```text
Static hosting
API URL
Reverse proxy
Caching headers
Compression
TLS
Environment configuration
```

---

# 40. API Usage Warning

Trading Engine ทำงานเป็น Loop ตาม:

```env
RUN_EVERY_N_MINUTES=60
```

หรือค่าที่กำหนดไว้

ถ้าปล่อย Engine รันต่อเนื่องจะใช้:

```text
Model API
Tavily
Market provider
MCP process startups
Push notifications
```

Notebook เตือนให้ตรวจ API Usage และหยุด Engine เมื่อทดลองเพียงพอ.

Dashboard ปิดไม่ได้ทำให้ Trading Engine หยุด เพราะเป็นคนละ Process

ต้องหยุด Terminal ที่รัน:

```text
backend.trading_floor
```

โดยตรง

---

# 41. Production Read Architecture ที่แนะนำ

```text
Trading Agents
        ↓ commands
Execution service
        ↓ events
Transactional database
        ↓
Read-model projector
        ↓
Read database / cache
        ↓
FastAPI
        ↓
REST + SSE
        ↓
Web frontend
```

ข้อดี:

```text
Agents ไม่เขียน Dashboard tables โดยตรง
Read Queries ไม่รบกวน Trading Transactions
Frontend อ่านข้อมูลที่เหมาะกับการแสดงผล
Events มี ID และ Replay ได้
```

นี่ใกล้กับ CQRS มากกว่า Architecture ที่ API อ่าน JSON Account โดยตรง

---

# 42. Risks ที่พบใน Lab 5

## API Risks

```text
No authentication
No response models
No versioning
Unbounded log limit
Relative database path
Direct dependency on trading_floor
Repeated market calls
No health endpoint
```

## Frontend Risks

```text
Polling overlap
Silent stale data
Market status loaded once
Manual TypeScript contract
Browser-local chart history
Potential innerHTML injection
No request timeout
```

## Evaluation Risks

```text
Self-evaluation by same agent
No benchmark
No independent evaluator
No backtest
No strategy version approval
```

## Operational Risks

```text
No pause/resume
No approval queue
No emergency stop
No durable scheduler status
No global cost budget
```

---

# 43. Suggested Exercises

## Exercise 1 — Refresh Market Status

เพิ่ม Market Poll:

```typescript
setInterval(
  loadMarket,
  30_000
);
```

แสดง Last Updated Time และ Stale State

---

## Exercise 2 — Prevent Overlapping Polls

เปลี่ยน `setInterval()` เป็น Recursive `setTimeout()` หลัง Request จบ

เพิ่ม:

```text
AbortController
Timeout
Exponential backoff
```

---

## Exercise 3 — Add Pydantic Response Models

สร้าง:

```text
TraderInfo
HoldingDetail
TraderDetail
LogRow
MarketInfo
```

กำหนด `response_model` ทุก Route แล้ว Generate TypeScript Types จาก OpenAPI

---

## Exercise 4 — Centralize Market Snapshot

สร้าง Service:

```text
MarketSnapshotCache
```

Refresh ราคาเป็นรอบ แล้วให้ Trader APIs ทุกตัวอ่าน Snapshot เดียวกัน

Response ต้องมี `as_of`

---

## Exercise 5 — SSE Activity Stream

เปลี่ยน Log Polling ทุก 2 วินาทีเป็น:

```text
GET /api/events
Content-Type: text/event-stream
```

ส่ง Log Event ใหม่ทันทีเมื่อถูกเขียน

---

## Exercise 6 — Strategy Approval

เปลี่ยน `change_strategy` เป็น:

```text
propose_strategy_change
```

ให้ Frontend มี Approval Queue

```text
Approve
Reject
Compare previous version
View evidence
```

---

## Exercise 7 — Emergency Stop

เพิ่ม Durable Control State:

```text
RUNNING
PAUSED
STOP_REQUESTED
STOPPED
```

Scheduler ต้องอ่าน State ก่อนเริ่มทุก Cycle

---

# Checklist

```text
□ เข้าใจ Observability, Feedback และ Frontend Improvements
□ เปิด backend/api.py และดู Routes
□ เปิด FastAPI /docs
□ ติดตั้ง Frontend Dependencies
□ เปิด API บน Port 8000
□ เปิด Vite บน Port 5173
□ เปิด Trading Engine แยก Terminal
□ เห็น Traders สี่ตัว
□ เห็น Market Mode Badge
□ เห็น Portfolio Chart
□ เห็น Holdings Heatmap
□ เห็น Activity Logs
□ เห็น Recent Trades
□ เข้าใจ Data Poll ทุก 6 วินาที
□ เข้าใจ Log Poll ทุก 2 วินาที
□ เข้าใจ Chart Local History
□ เข้าใจ Vite Proxy
□ เข้าใจว่า API เป็น Read-only
□ เข้าใจว่า Trace ไม่ใช่ Private CoT
□ ระบุ Missing Production Controls ได้
```

---

# แก่นของ Week 6 — Lab 5

```text
LogTracer
= Agent observability

Portfolio results
= Feedback signal

change_strategy
= Self-modifying policy

FastAPI
= Read-only HTTP boundary

Vite + TypeScript
= Decoupled web client

Polling
= Simple live-update mechanism

SQLite
= Shared state between processes

Trading Engine
= Execution plane

FastAPI
= Read plane

Frontend
= Presentation plane
```

บทเรียนสำคัญที่สุดคือ:

> **Agentic Application ไม่ได้จบเมื่อ Agent เรียก Tool สำเร็จ แต่ต้องทำให้ผู้ใช้มองเห็น State, Actions, Outcomes และ Failures ผ่านระบบ Observability ที่ตรวจสอบได้**

อีกบทเรียนคือ:

> **Feedback Loop จะมีคุณค่าเมื่อ Evaluation เชื่อถือได้ การให้ Agent ดูผลลัพธ์แล้วแก้ Strategy เองเป็นจุดเริ่มต้น แต่ยังต้องมี Benchmark, Versioning, Independent Evaluation และ Approval ก่อนเรียกว่าเรียนรู้ดีขึ้นจริง**

และบทสรุปเชิง Production คือ:

> **การแยก Trading Engine, HTTP API และ Web Frontend ทำให้ Architecture ยืดหยุ่นขึ้น แต่ Production Readiness ยังต้องเพิ่ม Authentication, Typed Contracts, Market-data Caching, Event Streaming, Durable Control State, Approval Workflow และ Emergency Stop รอบ Autonomous Loop**
