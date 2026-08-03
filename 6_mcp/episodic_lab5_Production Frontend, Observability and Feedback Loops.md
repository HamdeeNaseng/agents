# Episodic Learning Artifact

## Week 6 — Lab 5: Production Frontend, Observability and Feedback Loops

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**โฟลเดอร์:** `6_mcp`
**Notebook:** `5_lab5.ipynb`
**หัวข้อหลัก:** FastAPI, Vite, TypeScript, Agent Observability, Strategy Feedback, Frontend Polling, Read Models และ Operational Controls
**สถานะ:** เรียนและสรุป Lab 5 แล้ว
**หมายเหตุ:** ระบบเป็น Autonomous Trading Simulation เพื่อศึกษา Architecture ไม่ควรใช้ซื้อขายสินทรัพย์จริง

---

# 1. Context

Lab 4 สร้าง Autonomous Trading Floor ที่มี:

```text
Four Trader Agents
+ Researcher Agent
+ Six MCP Servers
+ Persistent Memory
+ Trading Scheduler
+ SQLite
+ Gradio Dashboard
```

Lab 5 ไม่ได้เปลี่ยน Logic หลักของ Traders

สิ่งที่เพิ่มคือสาม Improvements:

```text
1. Observability
2. Evaluation and Feedback
3. Production-style Frontend
```

Trading Engine และ Database ยังคงเดิม แต่ Frontend ไม่อ่าน SQLite โดยตรงอีกต่อไป

Architecture ใหม่:

```text
Trading Engine
        ↓ writes
SQLite
        ↑ reads
FastAPI
        ↓ JSON over HTTP
Vite + TypeScript Frontend
```

---

# 2. Learning Objectives

หลังจบ Lab 5 สามารถอธิบายได้ว่า:

1. Gradio Prototype ต่างจาก Decoupled Web Frontend อย่างไร
2. Trading Engine, FastAPI และ Frontend แยก Process กันอย่างไร
3. FastAPI ทำหน้าที่เป็น Read-only API อย่างไร
4. Frontend ใช้ TypeScript Interfaces แทน Python Objects อย่างไร
5. Vite Proxy ช่วยให้ Development ไม่ต้องตั้ง CORS อย่างไร
6. Portfolio Data และ Logs ถูก Poll ด้วยความถี่ต่างกันอย่างไร
7. Custom Trace Processor ส่ง Agent Activity ไปยัง UI อย่างไร
8. Observability ต่างจากการเปิดเผย Chain of Thought อย่างไร
9. Portfolio Performance ถูกใช้เป็น Feedback Signal อย่างไร
10. Strategy Self-modification มีความเสี่ยงอะไร
11. API Read Model คำนวณข้อมูลสำหรับ Frontend อย่างไร
12. Client State ต่างจาก Persisted Backend State อย่างไร
13. Chart History ใน Browser ต่างจาก Time Series ใน Database อย่างไร
14. Read-only API ลด Capability Surface อย่างไร
15. Production System ยังต้องเพิ่ม Control Plane อะไรบ้าง

---

# 3. Prerequisites

ควรเข้าใจเนื้อหาจาก Week 6 Labs 1–4:

```text
MCP Client
MCP Server
FastMCP
MCP Tools
MCP Resources
Persistent Memory
Context Engineering
Agent-as-Tool
Trader and Researcher Agents
Custom Trace Processor
SQLite
Autonomous Scheduler
```

ควรเข้าใจพื้นฐาน:

```text
HTTP
REST API
FastAPI
JSON
TypeScript
Vite
Frontend state
Polling
CORS
Async processes
Observability
```

---

# 4. Project Structure

```text
6_mcp/
├── 5_lab5.ipynb
├── backend/
│   ├── api.py
│   ├── trading_floor.py
│   ├── traders.py
│   ├── tracers.py
│   ├── database.py
│   ├── accounts.py
│   └── market.py
└── frontend/
    ├── package.json
    ├── vite.config.ts
    ├── index.html
    └── src/
        ├── main.ts
        ├── api.ts
        ├── state.ts
        ├── panel.ts
        ├── chart.ts
        ├── heatmap.ts
        ├── log.ts
        ├── transactions.ts
        └── styles.css
```

หน้าที่หลัก:

```text
backend/api.py
→ เปิดข้อมูล Trading Floor ผ่าน HTTP

frontend/src/api.ts
→ Client สำหรับเรียก FastAPI

frontend/src/state.ts
→ เก็บ State ของ Trader ใน Browser

frontend/src/main.ts
→ Poll API และ Update UI

frontend/src/panel.ts
→ Render Trader panel

frontend/src/chart.ts
→ Portfolio chart

frontend/src/heatmap.ts
→ Holdings visualization

frontend/src/log.ts
→ Operational trace log
```

---

# 5. Three-process Architecture

ระบบต้องเปิดสาม Process แยกกัน

## Process 1 — FastAPI

```powershell
cd 6_mcp
uv run uvicorn backend.api:app --port 8000
```

## Process 2 — Frontend

```powershell
cd 6_mcp/frontend
npm run dev
```

## Process 3 — Trading Engine

```powershell
cd 6_mcp
uv run -m backend.trading_floor
```

แต่ละ Process มีหน้าที่ต่างกัน:

```text
Trading Engine
= ตัดสินใจและเปลี่ยน State

FastAPI
= อ่านและจัดรูปข้อมูล

Frontend
= แสดงข้อมูลให้ผู้ใช้
```

---

# 6. Execution, Read and Presentation Planes

## Execution Plane

```text
Trader Agents
Researcher Agents
MCP Servers
Trading Scheduler
Account mutations
Strategy changes
Notifications
```

## Read Plane

```text
FastAPI
Portfolio calculations
Holdings enrichment
Log retrieval
Market status
```

## Presentation Plane

```text
Charts
Heatmaps
Activity logs
Recent trades
Returns ranking
Theme controls
```

Mental model:

```text
Execution decides
Read layer explains
Frontend presents
```

---

# 7. Improvement 1 — Observability

Lab 4 มี `LogTracer` ที่ Register ด้วย:

```python
add_trace_processor(
    LogTracer()
)
```

มันรับ Trace และ Span Events จาก OpenAI Agents SDK แล้วเขียนลง SQLite

ตัวอย่าง Events:

```text
Started Warren-trading
Started agent
Started function Researcher
Started MCP accounts_server
Ended generation
Ended response
Changed strategy
Bought shares
```

Lab 5 นำข้อมูลเหล่านี้มาแสดงใน Frontend ผ่าน FastAPI.

---

# 8. Observability Flow

```text
Agent execution
        ↓
OpenAI Agents SDK trace stream
        ↓
LogTracer
        ↓
SQLite logs table
        ↓
GET /api/traders/{name}/logs
        ↓
TypeScript client
        ↓
Activity panel
```

Observability ไม่ได้เป็น Feature สำหรับ Developer เท่านั้น

มันกลายเป็นส่วนหนึ่งของ User Experience:

```text
ผู้ใช้เห็น Agent กำลังทำอะไร
Agent เรียก Tool ใด
ขั้นตอนไหนกำลังทำงาน
มี Error หรือไม่
```

---

# 9. Operational Trace Is Not Chain of Thought

Frontend แสดง:

```text
Agent name
Tool name
MCP server
Span type
Start and end
Errors
Account events
```

มันไม่ได้แสดง:

```text
Hidden reasoning tokens
Private chain of thought
Internal deliberation
```

ควรเรียกว่า:

```text
Execution trace
Agent activity log
Operational timeline
```

ไม่ควรเรียกว่า Full Reasoning

---

# 10. Log Categories

Backend กำหนดสีตามประเภท Event:

```text
trace
agent
function
generation
response
account
```

FastAPI คืนทั้ง:

```json
{
  "datetime": "...",
  "type": "function",
  "message": "Started Researcher",
  "color": "#00dd00"
}
```

Frontend จึงไม่ต้องมี Mapping Logic ซ้ำ.

ข้อดี:

```text
Presentation rules ถูกกำหนดจาก Backend
Gradio และ Web UI ใช้ Semantic เดียวกัน
```

ข้อจำกัด:

```text
API ผูกกับสีของ UI
เปลี่ยน Theme ยากขึ้น
```

Production อาจคืน Semantic Level:

```text
info
success
warning
error
agent
tool
```

แล้วให้ Frontend เลือกสีเอง

---

# 11. Improvement 2 — Evaluation and Feedback

Trader สามารถเปลี่ยน Strategy ผ่าน Tool:

```text
change_strategy
```

Feedback Loop:

```text
Current strategy
        ↓
Trading decisions
        ↓
Portfolio performance
        ↓
Agent evaluates result
        ↓
Strategy is rewritten
        ↓
Future decisions use new strategy
```

Notebook มองว่านี่เป็นแนวคิดสำคัญของ Agentic AI:

```text
Real-world outcome
→ Feedback
→ Future policy changes
```

---

# 12. Feedback Is Not Automatically Learning

Portfolio กำไรไม่ได้พิสูจน์ว่า Strategy ดี

```text
Market may rise
Agent may be lucky
Position may be excessively risky
Evaluation period may be too short
```

Portfolio ขาดทุนก็ไม่ได้พิสูจน์ว่า Strategy แย่

```text
Short-term volatility
Temporary drawdown
Market-wide decline
Insufficient observation period
```

ดังนั้น:

```text
Outcome
≠ Correct causal evaluation
```

---

# 13. Self-evaluation Problem

Agent ตัวเดียวกันทำหน้าที่:

```text
ตัดสินใจ
ดำเนินการ
ประเมินตัวเอง
แก้ Strategy ตัวเอง
```

ไม่มี Independent Evaluator

อาจเกิด:

```text
Confirmation bias
Overfitting
Recent-event bias
Loss chasing
Reward hacking
Strategy drift
```

Safer pattern:

```text
Trader proposes change
        ↓
Independent evaluator
        ↓
Backtest
        ↓
Risk checks
        ↓
Human review
        ↓
Activate strategy
```

---

# 14. Strategy Versioning

Strategy ไม่ควรถูกเขียนทับโดยไม่มี History

ควรเก็บ:

```json
{
  "trader": "Warren",
  "version": 4,
  "previous_version": 3,
  "strategy": "...",
  "reason": "...",
  "evidence": ["run-102", "run-108"],
  "created_at": "...",
  "status": "pending",
  "approved_by": null
}
```

Benefits:

```text
Compare versions
Rollback
Audit changes
Measure performance by version
Require approval
```

---

# 15. Improvement 3 — Production-style Frontend

Lab 4 ใช้ Gradio:

```text
Python process
→ UI
→ Database access directly
```

ข้อดี:

```text
สร้างเร็ว
เหมาะกับ Demo
เหมาะกับ Internal Tool
```

Lab 5 ใช้:

```text
FastAPI
+
Vite
+
TypeScript
```

ข้อดี:

```text
Frontend ไม่ต้องรู้ SQLite
Backend และ Frontend Deploy แยกได้
สามารถสร้าง Client อื่นใช้ API เดิม
API Contract ชัดขึ้น
Frontend Technology เปลี่ยนได้
```

---

# 16. “Production” Means Production-style

Lab มี Production-style Separation

แต่ยังไม่ใช่ Production-complete System

ยังขาด:

```text
Authentication
Authorization
TLS
Deployment manifests
API versioning
Rate limiting
Health checks
Caching
Backups
Operational controls
Approval workflow
```

จึงควรตีความว่า:

```text
Prototype architecture
→ Closer to production structure
```

ไม่ใช่:

```text
Ready for real financial production
```

---

# 17. FastAPI Application

ไฟล์:

```text
backend/api.py
```

Application:

```python
app = FastAPI(
    title="Trading Floor"
)
```

API นี้เป็น Read-only

ไม่มี Endpoints สำหรับ:

```text
Buy
Sell
Change strategy
Start trader
Stop trader
```

นี่ลด Authority ของ Browser:

```text
Browser compromise
→ Cannot directly call trading mutations
```

อย่างไรก็ตาม Read-only Data ยังอาจเป็น Sensitive Data จึงยังต้องมี Authentication เมื่อ Deploy จริง

---

# 18. API Endpoints

```http
GET /api/traders
```

คืน Trader roster

```http
GET /api/market
```

คืน Market source และ Market-open status

```http
GET /api/traders/{name}
```

คืน Trader state

```http
GET /api/traders/{name}/logs
```

คืน Recent logs

---

# 19. Trader Detail Response

`GET /api/traders/{name}` คืน:

```text
Name
Lastname
Model name
Cash balance
Strategy
Portfolio value
Profit/loss
Holdings
Transactions
Time series
```

Holdings ถูก Enrich ด้วย:

```text
Current price
Average cost
Market value
Unrealized P/L
```

API จึงไม่ได้คืน Raw Account State เท่านั้น แต่สร้าง Frontend Read Model ด้วย.

---

# 20. Read Model Pattern

```text
Stored account
        +
Current market prices
        +
Derived calculations
        ↓
TraderDetail response
```

ข้อดี:

```text
Frontend ไม่คำนวณ Business Logic ซ้ำ
Client หลายตัวเห็นตัวเลขแบบเดียวกัน
```

ข้อควรระวัง:

```text
Read request may be expensive
Current prices may come from different timestamps
Result may not be a consistent market snapshot
```

---

# 21. Market Request Amplification

Frontend Poll Trader Data ทุก 6 วินาที

Trader Detail Endpoint เรียก Market Price ต่อทุก Holding

สมมติ:

```text
4 Traders
× 5 Holdings
× 10 Polls per minute
= 200 price lookups per minute
```

ถ้าใช้ External Provider อาจเกิด:

```text
Rate limits
Higher latency
Higher cost
Inconsistent timestamps
```

ควรเพิ่ม:

```text
Quote cache
Batch lookup
Shared market snapshot
As-of timestamp
```

---

# 22. Market Response Should Include Provenance

ปัจจุบัน Market Endpoint คืน:

```json
{
  "source": "massive",
  "is_market_open": true
}
```

แต่ Holding Price คืนเพียงตัวเลข

Production Result ควรมี:

```json
{
  "symbol": "AAPL",
  "price": 210.50,
  "source": "massive",
  "mode": "live",
  "as_of": "2026-08-03T11:00:00Z",
  "delayed": false
}
```

เพื่อป้องกัน Simulated หรือ Delayed Data ถูกตีความเป็น Live Data

---

# 23. Average-cost Calculation Limitation

API คำนวณ Average Cost จาก Buy Transactions ทั้งหมดของ Symbol.

ปัญหา:

```text
Buy 10 AAPL
Sell all 10
Buy 2 AAPL again
```

Average Cost อาจรวม Position รุ่นเก่า

Production ควรใช้:

```text
Tax lots
Position lifecycle
FIFO
LIFO
Weighted average reset
Realized P/L ledger
```

---

# 24. API Validation

`require_trader()` ตรวจ Trader Name

ถ้าไม่พบจะคืน:

```http
404 Unknown trader
```

แต่ Query:

```http
?last_n=...
```

ยังไม่มี Hard Bounds

ควรกำหนด:

```python
last_n: int = Query(
    default=13,
    ge=1,
    le=100,
)
```

เพื่อป้องกัน Request ใหญ่เกินไป

---

# 25. Missing Response Models

Routes ปัจจุบันคืน:

```python
dict
list[dict]
```

แม้ FastAPI จะสร้าง OpenAPI ได้ แต่ Contract ยังไม่แข็งแรง

ควรสร้าง:

```text
TraderInfo
MarketInfo
HoldingDetail
TransactionInfo
TraderDetail
LogRow
```

เป็น Pydantic Models

Benefits:

```text
Runtime validation
Clear API documentation
Generated frontend types
Stable schema
Better tests
```

---

# 26. TypeScript API Client

ไฟล์:

```text
frontend/src/api.ts
```

ประกาศ Interfaces:

```text
TraderInfo
Holding
Transaction
TimePoint
TraderDetail
LogRow
MarketInfo
```

และ Generic Function:

```typescript
async function get<T>(
  path: string
): Promise<T>
```

ข้อดี:

```text
Autocomplete
Compile-time checks
Reusable API client
Components ไม่ต้องใช้ fetch โดยตรง
```

---

# 27. Manual Schema Drift

Backend และ Frontend มี Schema คนละชุด:

```text
Python dictionaries
TypeScript interfaces
```

ถ้า Backend เปลี่ยน Field:

```text
portfolio_value
→ total_value
```

TypeScript Compiler จะไม่ตรวจพบจาก Runtime JSON

Safer:

```text
Pydantic Models
→ OpenAPI
→ Generated TypeScript Types
```

เช่นใช้:

```text
openapi-typescript
Orval
```

---

# 28. Vite Development Proxy

`vite.config.ts` กำหนด:

```typescript
proxy: {
  "/api": {
    target:
      "http://127.0.0.1:8000"
  }
}
```

Browser เรียก:

```text
http://localhost:5173/api/traders
```

Vite ส่งต่อไป:

```text
http://127.0.0.1:8000/api/traders
```

ข้อดี:

```text
Browser เห็น Single Origin
Development ไม่ต้องตั้ง CORS
Frontend ใช้ Relative URLs
```

---

# 29. Development Proxy Is Not Production Configuration

Vite Proxy ทำงานเฉพาะ Development Server

เมื่อ Build จริงต้องใช้:

```text
Reverse proxy
หรือ
CORS configuration
หรือ
Same-domain deployment
```

Example:

```text
https://traders.example.com/
→ Frontend

https://traders.example.com/api/
→ FastAPI
```

---

# 30. Frontend Technology

Frontend ใช้ Vanilla TypeScript

ไม่ได้ใช้:

```text
React
Vue
Angular
```

Runtime Dependency หลักคือ:

```text
uPlot
```

Scripts:

```json
{
  "dev": "vite",
  "build": "tsc && vite build",
  "preview": "vite preview"
}
```

---

# 31. Frontend Startup Flow

`main.ts` ทำงานตามลำดับ:

```text
Initialize theme
        ↓
Load market status
        ↓
Load trader roster
        ↓
Create TraderState objects
        ↓
Create TraderPanel objects
        ↓
Fetch portfolio data
        ↓
Fetch logs
        ↓
Start polling
```

---

# 32. Polling

```typescript
const DATA_POLL_MS = 6000;
const LOG_POLL_MS = 2000;
```

Portfolio:

```text
Poll every 6 seconds
```

Logs:

```text
Poll every 2 seconds
```

ข้อดี:

```text
Simple
Easy to debug
No persistent connection
Automatic recovery on next poll
```

ข้อจำกัด:

```text
Repeated requests
Stale data between polls
Load increases per viewer
No immediate event delivery
```

---

# 33. Polling Overlap

ใช้:

```typescript
setInterval(
  pollData,
  DATA_POLL_MS
);
```

แต่ `pollData()` เป็น Async

ถ้า Request ใช้เวลานานกว่า Interval:

```text
Poll 1 begins
Poll 2 begins before Poll 1 ends
```

อาจเกิด:

```text
Overlapping requests
Response reordering
Duplicate load
Old response overwrites new state
```

Safer:

```typescript
async function pollLoop() {
  await pollData();
  setTimeout(
    pollLoop,
    DATA_POLL_MS
  );
}
```

---

# 34. Error Visibility

Current Frontend ทำ:

```typescript
console.error(...)
```

เมื่อ Fetch ล้มเหลว.

ผู้ใช้อาจยังเห็นข้อมูลเก่าโดยไม่รู้ว่า Connection ขาด

ควรแสดง:

```text
Connected
Refreshing
Stale
Disconnected
Last updated
Retrying
```

หลัก:

```text
Old data must not look like live data.
```

---

# 35. Market Badge Loaded Once

`loadMarket()` ถูกเรียกตอน Startup เท่านั้น

ไม่ได้ถูก Poll ต่อ.

ดังนั้นสถานะ:

```text
Market open
Market closed
```

อาจไม่เปลี่ยนใน UI หลังเปิด Page

ควร Poll Market Status หรือใช้ Event Stream

---

# 36. TraderState

แต่ละ Trader มี Client State:

```text
Static trader info
Latest detail
Chart data
Previous prices
Chart seeded flag
```

`recordDetail()` ทำ:

```text
Update latest detail
Seed chart from backend once
Append current portfolio value
Limit chart size
```

---

# 37. Persisted State vs Browser State

Backend State:

```text
Accounts
Transactions
Strategy
Portfolio time series
Logs
```

Browser State:

```text
Current screen data
Previous poll prices
Chart points collected while page is open
Theme
Rendered panels
```

Browser State ไม่ใช่ Source of Truth

Reload Page ทำให้ Browser State หาย

---

# 38. Chart History

ครั้งแรก Chart ใช้:

```text
Backend time_series
```

หลังจากนั้นเติมจุดทุก Poll:

```typescript
{
  t: Date.now() / 1000,
  value:
    detail.portfolio_value
}
```

ดังนั้น Chart เป็นส่วนผสม:

```text
Persisted historical points
+
Browser-observed live points
```

ผู้ใช้สองคนอาจเห็น Chart History ไม่เหมือนกัน

---

# 39. Better Time-series Design

ถ้าต้องการ Chart เป็น Source of Truth:

```text
Trading Engine
→ Records portfolio snapshots
→ Database
→ Time-series API
→ Frontend
```

Frontend ไม่ควรสร้าง Historical Records เอง

ทางเลือก:

```text
GET /api/traders/{name}/timeseries
```

พร้อม Parameters:

```text
from
to
interval
limit
```

---

# 40. Portfolio Chart

`chart.ts` ใช้ uPlot

Features:

```text
Responsive resize
Time axis
Portfolio value axis
Green/red trend
Gradient fill
Theme refresh
```

Chart ถูกสร้างหลัง Element อยู่ใน DOM เพราะต้องใช้ Container Dimensions ที่ถูกต้อง

---

# 41. Holdings Heatmap

Heatmap แสดง:

```text
Tile size
→ Position market value

Tile color
→ Unrealized profit or loss

Flash
→ Price direction from previous poll
```

ช่วยมองเห็น:

```text
Portfolio concentration
Large positions
Winning positions
Losing positions
Recent price movement
```

---

# 42. Potential DOM Injection

Heatmap สร้าง Symbol ด้วย `innerHTML`.

แม้ Symbol ปกติควรเป็น:

```text
AAPL
MSFT
SPY
```

แต่ค่ามาจาก System ที่ Agent สามารถส่ง Symbol เข้า Trading Tool ได้

ควรใช้:

```typescript
element.textContent = symbol;
```

และ Backend Validation:

```text
Ticker allowlist
Ticker pattern
Supported-symbol lookup
```

---

# 43. Trader Panel

แต่ละ Panel แสดง:

```text
Trader name
Model
Strategy
Portfolio value
P/L
Portfolio chart
Holdings heatmap
Activity log
Recent transactions
```

Frontend ยังเลือก Trader ที่ Portfolio Value สูงสุดเป็น Leader

นี่เป็น Visualization Rule ไม่ใช่ Risk-adjusted Performance Evaluation

---

# 44. Returns Ranking Limitation

Frontend คำนวณ Return จาก:

```text
P/L ÷ initial capital
```

แต่ Absolute Return ไม่ได้สะท้อน:

```text
Volatility
Maximum drawdown
Concentration
Risk exposure
Sharpe ratio
Benchmark performance
```

Trader ที่กำไรมากที่สุดอาจรับ Risk สูงที่สุด

---

# 45. Transaction View

Recent Trades แสดง:

```text
Date
BUY or SELL
Quantity
Symbol
Execution price
```

และจำกัด 12 รายการล่าสุด.

มันไม่แสดง:

```text
Rationale
Order ID
Execution source
Status
Realized profit
Approval
```

Production Audit View ควรแสดงข้อมูลเหล่านี้เพิ่มเติม

---

# 46. Log View

Activity Log แสดง:

```text
Time
Event type
Message
Color
```

และเลื่อนไปด้านล่างอัตโนมัติ.

Current API คืนเพียง Recent Events

ยังไม่มี:

```text
Pagination
Event IDs
Filtering
Search
Run grouping
Trace links
```

---

# 47. Timestamp Problem

Database ใช้:

```sql
datetime('now')
```

ซึ่งเป็น UTC แต่ String ไม่มี Timezone Marker.

Frontend อาจตีความว่าเป็น Local Time

จึงอาจเกิด Time Shift

ควรใช้:

```text
2026-08-03T11:00:00Z
```

แล้วแปลงเป็น User Timezone ใน Browser

---

# 48. Relative Database Path

Database:

```python
DB = "accounts.db"
```

Path ขึ้นกับ Working Directory

จึงต้อง Run API และ Engine จาก:

```text
6_mcp/
```

หากรันจาก Directory อื่นอาจสร้าง Database คนละไฟล์

Safer:

```python
DB = (
    Path(__file__).resolve()
    .parent.parent
    / "accounts.db"
)
```

หรือใช้:

```env
DATABASE_URL=...
```

---

# 49. API Coupling

`api.py` Import Trader roster จาก:

```text
backend.trading_floor
```

ทำให้ Read API ผูกกับ Execution Module

ควรแยก Config:

```text
backend/config.py
backend/roster.py
```

ให้ API และ Engine ใช้ Shared Configuration โดยไม่ Import กันโดยตรง

---

# 50. Read-only API Is Not Public-safe

แม้ API ไม่มี Write Operations แต่ข้อมูลประกอบด้วย:

```text
Balances
Holdings
Strategies
Transactions
Agent activities
Models
Market source
```

เมื่อ Deploy จริงยังต้องมี:

```text
Authentication
Authorization
Tenant isolation
TLS
Access logging
Rate limits
```

---

# 51. Polling vs Event Streaming

Current:

```text
REST polling
```

Alternative:

## Server-sent Events

เหมาะกับ:

```text
Live logs
Portfolio update events
Read-only stream
```

## WebSocket

เหมาะกับ:

```text
Bidirectional controls
Approval requests
Pause and resume
Interactive commands
```

สำหรับ Dashboard ปัจจุบัน:

```text
REST for snapshots
+
SSE for activity logs
```

อาจเหมาะกว่า Poll Logs ทุก 2 วินาที

---

# 52. Missing Control Plane

Frontend ปัจจุบันทำได้เพียง Monitor

ยังทำไม่ได้:

```text
Pause engine
Resume engine
Run trader now
Stop trader
Approve trade
Reject trade
Emergency stop
Rollback strategy
Reset account
```

นี่เป็นข้อจำกัดที่เหมาะกับ Lab เพราะ API ตั้งใจเป็น Read-only

แต่ Production Autonomous System ต้องมี Control Plane

---

# 53. Suggested Control API

```http
POST /api/control/pause
POST /api/control/resume
POST /api/control/emergency-stop

POST /api/traders/{name}/run
POST /api/traders/{name}/stop

GET  /api/approvals
POST /api/approvals/{id}/approve
POST /api/approvals/{id}/reject
```

ทุก Write Endpoint ต้องมี:

```text
Authentication
Authorization
Idempotency
CSRF protection
Audit log
Approval ownership
```

---

# 54. Durable System Status

ควรเก็บ Engine State:

```text
RUNNING
PAUSED
STOPPING
STOPPED
ERROR
```

และ Trader State:

```text
IDLE
RESEARCHING
PROPOSING
WAITING_APPROVAL
EXECUTING
COMPLETED
FAILED
```

Frontend ควรแสดง State จริง ไม่ใช่อนุมานจาก Logs เพียงอย่างเดียว

---

# 55. Health and Readiness

Production API ควรมี:

```http
GET /health
GET /ready
```

`/health` ตรวจ:

```text
Process alive
```

`/ready` ตรวจ:

```text
Database accessible
Market provider available
Trading engine heartbeat recent
Log writer operational
```

---

# 56. Engine Heartbeat

ปัจจุบัน API ไม่รู้ว่า Trading Engine กำลังทำงานหรือหยุดไปแล้ว

Frontend อาจยังแสดง Portfolio เดิมโดยไม่มี Warning

ควรมี:

```text
engine_heartbeat
last_cycle_started
last_cycle_completed
next_run_at
current_status
```

ถ้า Heartbeat เก่า:

```text
Frontend shows Engine offline
```

---

# 57. Cost Governance

Trading Engine ทำงานเป็น Loop

แต่ปิด Browser ไม่ได้หยุด Engine

Costs อาจมาจาก:

```text
Model calls
Researcher calls
Search API
Market API
MCP startups
Push notifications
```

Notebook เตือนให้ตรวจ API Usage และหยุด Engine เมื่อทดลองเพียงพอ.

ควรมี:

```text
Daily cost limit
Cycle cost limit
Search limit
Model-turn limit
Emergency shutdown
```

---

# 58. Production Read Architecture

Safer Architecture:

```text
Trading agents
        ↓ commands
Execution service
        ↓ domain events
Transactional database
        ↓
Read-model projector
        ↓
Read database or cache
        ↓
FastAPI
        ↓
REST + SSE
        ↓
Web frontend
```

ข้อดี:

```text
Read load ไม่รบกวน trading writes
Frontend queries เร็ว
Events Replay ได้
State มี Version
Audit ชัดขึ้น
```

---

# 59. Common Misconceptions

## “แยก Frontend แล้วเป็น Production ทันที”

ไม่จริง ยังต้องมี Security, Deployment และ Operations

## “Observability คือ Chain of Thought”

ไม่จริง Observability คือ Execution Evidence

## “Strategy เปลี่ยนแล้วแปลว่า Agent เรียนรู้ดีขึ้น”

ไม่จริง ต้องประเมินผลของ Strategy ใหม่

## “Read-only API ไม่ต้อง Authentication”

ไม่จริง Read Data อาจ Sensitive

## “TypeScript Interface รับประกัน Backend Contract”

ไม่จริง Interfaces ไม่มี Runtime Validation

## “Polling คือ Real-time”

ไม่จริงเป็น Near-real-time ตาม Poll Interval

## “Chart เป็น Backend History ทั้งหมด”

ไม่จริง ส่วนหนึ่งเป็น Browser-local History

## “ปิด Frontend แล้ว Agent หยุด”

ไม่จริง Trading Engine เป็นอีก Process

---

# 60. Risks Identified

## 60.1 Missing Authentication

API เปิด Account และ Agent Data

## 60.2 Contract Drift

Backend Dicts กับ TypeScript Interfaces เขียนแยกกัน

## 60.3 Poll Overlap

Async Poll รอบใหม่อาจเริ่มก่อนรอบเก่าจบ

## 60.4 Silent Stale Data

Frontend แสดงข้อมูลเก่าโดยไม่มี Warning

## 60.5 Market Status Staleness

Market Badge โหลดครั้งเดียว

## 60.6 Request Amplification

ทุก Poll เรียก Market Price ต่อ Holding

## 60.7 Browser-local History

Charts ของผู้ใช้แต่ละคนไม่เหมือนกัน

## 60.8 Timestamp Ambiguity

UTC ถูกตีความเป็น Local Time

## 60.9 Relative DB Path

API และ Engine อาจเปิด Database คนละไฟล์

## 60.10 DOM Injection

Ticker ถูกใส่ด้วย `innerHTML`

## 60.11 No Operational Controls

ไม่มี Pause, Approval หรือ Emergency Stop

## 60.12 Weak Evaluation

Agent ประเมินและแก้ Strategy ของตัวเอง

---

# 61. Production Improvements

```text
Add authentication and authorization
Use Pydantic response models
Generate TypeScript types from OpenAPI
Add market-data cache
Return price provenance and timestamps
Prevent overlapping polls
Show stale and disconnected states
Refresh market status
Move chart history to backend
Use ISO 8601 UTC timestamps
Use absolute database configuration
Separate roster config from execution module
Add health and readiness endpoints
Add engine heartbeat
Add SSE for activity logs
Add operational control plane
Version strategy changes
Add approval workflow
Add cost limits
```

---

# 62. Suggested Exercise — Typed API

สร้าง Pydantic Models:

```text
TraderInfo
HoldingDetail
TraderDetail
MarketInfo
LogRow
```

จากนั้น Generate TypeScript Types จาก OpenAPI

เปรียบเทียบกับ Manual Interfaces

---

# 63. Suggested Exercise — Safe Polling

แทน:

```typescript
setInterval(
  pollData,
  6000
)
```

ใช้:

```typescript
async function loop() {
  await pollData();
  setTimeout(loop, 6000);
}
```

เพิ่ม:

```text
Timeout
AbortController
Backoff
Last-updated timestamp
```

---

# 64. Suggested Exercise — Market Cache

สร้าง Market Snapshot:

```json
{
  "as_of": "...",
  "source": "massive",
  "prices": {
    "AAPL": 210.5,
    "MSFT": 430.1
  }
}
```

ให้ Trader Detail Endpoints ใช้ Snapshot เดียวกัน

---

# 65. Suggested Exercise — SSE Logs

สร้าง:

```http
GET /api/events
```

คืน:

```text
Content-Type: text/event-stream
```

Frontend รับ Logs ใหม่ทันทีโดยไม่ Poll ทุก 2 วินาที

---

# 66. Suggested Exercise — Engine Heartbeat

สร้าง Table หรือ File:

```text
engine_status
last_heartbeat
last_cycle
next_cycle
state
```

Frontend แสดง:

```text
Engine online
Engine paused
Engine offline
```

---

# 67. Suggested Exercise — Strategy Approval

เปลี่ยน:

```text
change_strategy
```

เป็น:

```text
propose_strategy_change
```

Frontend แสดง:

```text
Old strategy
Proposed strategy
Reason
Evidence
Approve
Reject
```

---

# 68. Suggested Exercise — Emergency Stop

เพิ่ม Durable State:

```text
RUNNING
PAUSED
STOP_REQUESTED
STOPPED
```

Trading Scheduler ตรวจ State ก่อน:

```text
เริ่ม Cycle
เรียก Trader
Execute Trade
```

---

# 69. Patterns Learned

## Decoupled Frontend Pattern

```text
Backend state
→ HTTP API
→ Independent web client
```

## Read-model Pattern

```text
Domain state
+ derived values
→ frontend-ready JSON
```

## Observability Pipeline Pattern

```text
Agent trace
→ processor
→ database
→ API
→ UI
```

## Feedback Loop Pattern

```text
Action
→ outcome
→ evaluation
→ policy update
```

## Polling Pattern

```text
Snapshot API
→ periodic refresh
```

## Control-plane Pattern

```text
Autonomous engine
+ human operational controls
```

---

# 70. Lab 5 Mental Model

```text
Trading agents act
        ↓
Accounts and logs persist
        ↓
FastAPI reads shared state
        ↓
Frontend polls JSON
        ↓
Panels, charts and logs update
        ↓
User observes outcomes
        ↓
Portfolio performance feeds back
        ↓
Agent may propose or apply strategy changes
```

Production extension:

```text
Observation
        ↓
Evaluation
        ↓
Human approval
        ↓
Safe policy update
```

---

# 71. Final Lessons

## Lesson 1

Agentic System ต้องมี Observability ที่ผู้ใช้และทีมงานเข้าใจได้

## Lesson 2

Operational Trace ไม่จำเป็นต้องเปิดเผย Hidden Reasoning

## Lesson 3

Feedback Loop ต้องแยก Outcome ออกจาก Causal Evaluation

## Lesson 4

Agent ไม่ควรเปลี่ยน Strategy โดยไม่มี Versioning และ Review

## Lesson 5

FastAPI สร้าง Boundary ระหว่าง Domain State กับ Frontend

## Lesson 6

Read-only API ลด Write Authority แต่ยังต้องมี Security

## Lesson 7

Frontend Types ควรถูกสร้างจาก API Contract เดียวกัน

## Lesson 8

Polling เหมาะกับ Demo แต่ต้องควบคุม Overlap และ Staleness

## Lesson 9

Browser State ไม่ใช่ Source of Truth

## Lesson 10

Market Data ต้องมี Source และ Timestamp

## Lesson 11

Frontend ต้องแสดง Connection และ Freshness Status

## Lesson 12

Production Autonomous System ต้องมี Control Plane

## Lesson 13

Pause, Approval และ Emergency Stop สำคัญพอ ๆ กับ Agent Intelligence

## Lesson 14

ปิด Dashboard ไม่ได้หยุด Autonomous Engine

## Lesson 15

Production Readiness คือทั้ง Architecture, Security, Reliability และ Operations

---

# 72. Memory Summary

```text
Week 6 Lab 5:
Production-style trading frontend

Notebook:
6_mcp/5_lab5.ipynb

Three improvements:
Observability
Evaluation and feedback
Production frontend

Processes:
Trading engine
FastAPI
Vite frontend

Trading engine:
backend.trading_floor

API:
backend.api

API port:
8000

Frontend:
Vite + TypeScript

Frontend port:
5173

Development proxy:
/api
→ 127.0.0.1:8000

API endpoints:
/api/traders
/api/market
/api/traders/{name}
/api/traders/{name}/logs

API mode:
Read-only

Observability:
LogTracer
→ SQLite
→ FastAPI
→ Frontend

Activity log:
Operational trace
not private chain of thought

Feedback:
Portfolio performance
→ Agent changes strategy

Main feedback risk:
Self-evaluation
Strategy drift
Overfitting

Frontend polling:
Portfolio every 6 seconds
Logs every 2 seconds

Market badge:
Loaded once

Client state:
TraderState

Chart:
Seed backend time series once
Then append browser-local points

Chart library:
uPlot

Holdings:
Heatmap
Tile size by market value
Color by unrealized P/L

Frontend dependencies:
TypeScript
Vite
uPlot

Main API risks:
No auth
No response models
Relative database path
Repeated market calls

Main frontend risks:
Polling overlap
Silent stale data
Manual schema drift
Browser-local history

Missing controls:
Pause
Resume
Approval
Emergency stop
Engine heartbeat

Production needs:
Authentication
Typed contracts
Caching
SSE
Health endpoints
Durable control state
Strategy versioning
Human approval
Cost limits
```

---

# 73. Week 6 Final Mental Model

```text
MCP
→ Connects agents to capabilities

Custom MCP
→ Exposes business operations

Context Engineering
→ Selects memory, search and integrations

Agent-as-Tool
→ Separates research from execution

Trading Floor
→ Runs agents autonomously

Tracing
→ Makes execution observable

FastAPI
→ Exposes state through HTTP

TypeScript Frontend
→ Presents live system state

Control Plane
→ Keeps autonomy governable
```

---

# 74. Completion Summary

Week 6 เริ่มจาก Agent ใช้ MCP Server หนึ่งตัว และจบด้วยระบบที่มี:

```text
Multiple agents
Multiple MCP servers
Persistent memory
Web research
Live integrations
Custom business tools
Autonomous execution
Tracing
Shared persistence
HTTP API
Independent frontend
Feedback loops
```

บทเรียนที่สำคัญที่สุดไม่ใช่เพียงการเชื่อม Tool ได้

แต่คือ:

> **เมื่อ Agent สามารถจำ ค้นคว้า ตัดสินใจ และเปลี่ยน State ได้ ระบบรอบ Agent ต้องออกแบบ Context, Authority, Evidence, Observability และ Human Control ให้แข็งแรงพอ ๆ กับตัว Agent**

และบทสรุปเชิง Production คือ:

> **Autonomy ที่ไม่มี Monitoring, Approval, Limits และ Emergency Controls ไม่ใช่ความฉลาดของระบบ แต่เป็นความเสี่ยงที่ทำงานได้อัตโนมัติ**
