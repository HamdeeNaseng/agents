# Episodic Learning Artifact

## Week 6 — Lab 2: Building a Custom MCP Server with FastMCP

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**โฟลเดอร์:** `6_mcp`
**Notebook:** `2_lab2.ipynb`
**ไฟล์หลัก:** `backend/accounts.py`, `backend/accounts_server.py`, `backend/database.py`, `backend/market.py`, `backend/push_server.py`
**หัวข้อหลัก:** FastMCP, Custom MCP Server, Tools, Resources, Domain Boundaries, stdio Transport, Side Effects, Authorization และ Auditability
**สถานะ:** เรียนและสรุป Lab 2 แล้ว

---

# 1. Context

Week 6 — Lab 1 เรียนรู้การเป็น MCP Client:

```text
Agent application
→ Connect MCP server
→ Discover tools
→ Use external capabilities
```

Lab 2 เปลี่ยนมุมมองมาเป็น MCP Server Developer:

```text
Existing business logic
→ Define public capabilities
→ Expose through MCP
→ Let agents consume them
```

โจทย์หลักคือการนำระบบจำลองบัญชีลงทุนที่เขียนด้วย Python มาห่อเป็น MCP Server

```text
Account domain model
        ↓
FastMCP adapter
        ↓
MCP tools and resources
        ↓
OpenAI Agents SDK client
        ↓
Account manager agent
```

Notebook เริ่มจากทดลอง `Account` โดยตรง ก่อนเปิด Business Operations เหล่านั้นผ่าน `accounts_server.py` เพื่อยืนยันว่า Domain Logic ทำงานได้โดยไม่ขึ้นกับ MCP.

---

# 2. Learning Objectives

หลังจบ Lab 2 สามารถอธิบายได้ว่า:

1. Business Logic แตกต่างจาก MCP Interface อย่างไร
2. FastMCP ใช้สร้าง MCP Server อย่างไร
3. `@mcp.tool()` เปลี่ยน Python Function เป็น MCP Tool อย่างไร
4. `@mcp.resource()` ใช้เปิด Addressable Information อย่างไร
5. Tools และ Resources ต่างกันอย่างไร
6. Type Hints และ Docstrings กลายเป็น Tool Schema อย่างไร
7. `mcp.run(transport="stdio")` เริ่ม Server อย่างไร
8. ทำไม Server จึงถูกรันด้วย `python -m` หรือ `uv run -m`
9. `MCPServerStdio` เชื่อม Custom Server กับ OpenAI Agent อย่างไร
10. Agent เลือก `get_balance` และ `get_holdings` จาก Goal อย่างไร
11. Schema Validation ต่างจาก Domain Validation อย่างไร
12. Read Tools และ Write Tools มี Risk ต่างกันอย่างไร
13. Authentication และ Authorization ควรอยู่ใน Layer ใด
14. Audit Logging ช่วยตรวจ Agent Actions อย่างไร
15. Push Notification MCP Server มี External Side-effect Risk อย่างไร
16. MCP ทำให้ Interface เป็นมาตรฐาน แต่ไม่ได้กำหนด Business Policy อย่างไร
17. Tool Return Type ต้องสอดคล้องกับ Runtime Result อย่างไร
18. Resource ที่ควรเป็น Read Operation อาจมี Hidden Side Effect ได้อย่างไร
19. Concurrent Tool Calls อาจทำให้เกิด Lost Update อย่างไร
20. Production MCP Server ควรเพิ่ม Controls อะไรบ้าง

---

# 3. Prerequisites

ควรเข้าใจ:

```text
Python functions
Classes and methods
Pydantic BaseModel
Async functions
Decorators
SQLite
JSON serialization
Environment variables
OpenAI Agents SDK
MCP client lifecycle
stdio subprocess
```

ควรจำ Pattern จาก Lab 1:

```text
Connect
→ list_tools()
→ inspect schemas
→ expose capabilities to agent
→ run agent
→ inspect trace
```

---

# 4. Lab Structure

```text
6_mcp/
├── 2_lab2.ipynb
└── backend/
    ├── accounts.py
    ├── accounts_server.py
    ├── database.py
    ├── market.py
    ├── market_simulator.py
    └── push_server.py
```

หน้าที่แต่ละไฟล์:

```text
accounts.py
→ Domain logic ของบัญชีลงทุน

accounts_server.py
→ MCP adapter สำหรับ Account operations

database.py
→ SQLite persistence และ audit logs

market.py
→ Market price provider และ simulator fallback

push_server.py
→ ตัวอย่าง MCP Server ที่ส่ง push notification
```

---

# 5. Architecture

```text
User
    ↓
OpenAI Agent
    ↓
OpenAI Agents SDK MCP Client
    ↓ stdio
FastMCP accounts_server
    ↓
Account domain model
    ├── SQLite
    └── Market provider
```

Agent ไม่ต้องรู้ว่า:

```text
Account ถูกเก็บเป็น JSON
Database ใช้ SQLite
ราคาหุ้นมาจาก API หรือ Simulator
Transaction ถูกบันทึกอย่างไร
```

Agent เห็นเพียง Public Capability Contract:

```text
get_balance
get_holdings
buy_shares
sell_shares
change_strategy
```

---

# 6. Domain First

Notebook ทดลอง Domain Logic โดยตรง:

```python
from backend.accounts import Account

account = Account.get("Ed")
account.reset()

account.buy_shares(
    "AMZN",
    3,
    "Because this bookstore website looks promising",
)

account.report()
account.list_transactions()
```

ลำดับนี้สำคัญ:

```text
Test domain behavior
→ Confirm persistence
→ Confirm validation
→ Then expose through MCP
```

ถ้า Domain Logic ผิด การเพิ่ม MCP จะเพียงเพิ่ม Transport Layer รอบ Logic ที่ผิดอยู่แล้ว

---

# 7. Separation of Concerns

## Domain Layer

รับผิดชอบ:

```text
Balances
Holdings
Trading rules
Transaction recording
Portfolio valuation
Persistence
Audit events
```

## MCP Layer

รับผิดชอบ:

```text
Capability names
Descriptions
Input schemas
Protocol messages
Transport
Tool results
```

หลักสำคัญ:

```text
MCP adapter
ไม่ควรเป็น
ที่อยู่หลักของ Business Logic
```

เพราะ Domain Logic ควรถูกใช้และทดสอบได้โดยไม่ต้องเปิด MCP Process

---

# 8. Transaction Model

```python
class Transaction(BaseModel):
    symbol: str
    quantity: int
    price: float
    timestamp: str
    rationale: str
```

Transaction เก็บ:

```text
symbol
→ หุ้นที่ซื้อหรือขาย

quantity
→ จำนวนหุ้น ซื้อเป็นบวก ขายเป็นลบ

price
→ ราคาที่ทำรายการ

timestamp
→ เวลาทำรายการ

rationale
→ เหตุผลของการตัดสินใจ
```

`rationale` มีความสำคัญใน Agentic System เพราะช่วยเก็บ Decision Context ไว้ตรวจย้อนหลัง

---

# 9. Account Model

```python
class Account(BaseModel):
    name: str
    balance: float
    strategy: str
    holdings: dict[str, int]
    transactions: list[Transaction]
    portfolio_value_time_series: list[tuple[str, float]]
```

Account State ประกอบด้วย:

```text
Cash balance
Current strategy
Stock holdings
Transaction history
Portfolio snapshots
```

ค่าเริ่มต้น:

```python
INITIAL_BALANCE = 10_000.0
SPREAD = 0.002
```

---

# 10. Buy and Sell Spread

ซื้อ:

```text
buy price
= market price × 1.002
```

ขาย:

```text
sell price
= market price × 0.998
```

Spread จำลอง:

```text
Transaction cost
Market friction
Brokerage difference
```

จึงไม่ควรคาดว่า Portfolio จะมี Value เท่ากับเงินเริ่มต้นทันทีหลังซื้อและขายคืน

---

# 11. Account Creation

```python
Account.get("Ed")
```

Flow:

```text
Normalize name
→ Read account from SQLite
→ Account exists?
    ├── Yes → Load state
    └── No  → Create account with $10,000
→ Return Account object
```

Account ถูก Persist เป็น JSON ใน SQLite

```text
Python model
→ model_dump()
→ JSON string
→ accounts table
```

---

# 12. Database Structure

Account Table:

```sql
CREATE TABLE IF NOT EXISTS accounts (
    name TEXT PRIMARY KEY,
    account TEXT
)
```

Log Table:

```sql
CREATE TABLE IF NOT EXISTS logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    datetime DATETIME,
    type TEXT,
    message TEXT
)
```

จุดเด่น:

```text
Simple
Easy to understand
Easy to serialize
Suitable for course demo
```

ข้อจำกัด:

```text
Weak relational constraints
Whole-account updates
Harder querying
Potential lost updates
Limited audit detail
```

---

# 13. `buy_shares()` Flow

```text
Receive symbol, quantity and rationale
        ↓
Get market price
        ↓
Apply buy spread
        ↓
Calculate total cost
        ↓
Validate available balance
        ↓
Update holdings
        ↓
Create transaction
        ↓
Deduct balance
        ↓
Save account
        ↓
Write audit log
        ↓
Return latest report
```

---

# 14. `sell_shares()` Flow

```text
Receive symbol, quantity and rationale
        ↓
Check available holdings
        ↓
Get market price
        ↓
Apply sell spread
        ↓
Reduce holdings
        ↓
Delete zero holding
        ↓
Record negative quantity transaction
        ↓
Increase cash
        ↓
Save account
        ↓
Write audit log
```

---

# 15. Market Provider

`market.py` รองรับ:

```text
Live market data
หรือ
Simulated market data
```

เมื่อมี:

```env
MASSIVE_API_KEY=...
```

ระบบพยายามใช้:

```text
Last trade
→ Snapshot
→ Previous close
```

เมื่อ API ใช้งานไม่ได้:

```text
Fallback to simulated price
```

ข้อดี:

```text
Lab ยังทำงานได้โดยไม่มี Paid API
```

ข้อควรระวัง:

```text
Agent หรือ User อาจไม่รู้ว่าใช้ simulated price
```

Result Contract ควรบอก Source:

```json
{
  "price": 120.5,
  "source": "simulator"
}
```

---

# 16. Account Report

```python
account.report()
```

Flow:

```text
Calculate portfolio value
→ Append time-series snapshot
→ Save account
→ Calculate P/L
→ Write audit log
→ Return JSON string
```

จุดที่ต้องสังเกต:

```text
report()
ดูเหมือน Read operation

แต่มี Side Effect:
เพิ่ม snapshot และ save state
```

นี่เป็น Hidden Mutation

ควรแยก:

```text
read_report()
→ pure read

record_snapshot()
→ state mutation
```

---

# 17. Creating a FastMCP Server

```python
from mcp.server.fastmcp import FastMCP
from .accounts import Account

mcp = FastMCP(
    "accounts_server"
)
```

Mental model:

```text
FastMCP instance
= Registry and runtime for MCP capabilities
```

มันรวบรวม:

```text
Tools
Resources
Server metadata
Transport runtime
```

---

# 18. Exposing a Tool

```python
@mcp.tool()
async def get_balance(
    name: str,
) -> float:
    """Get the cash balance of the given account name."""
    return Account.get(name).balance
```

Decorator ทำให้ Function กลายเป็น Public MCP Tool

FastMCP อ่าน:

```text
Function name
Type hints
Docstring
Arguments
Return annotation
```

เพื่อสร้าง Tool Definition

---

# 19. Generated Tool Contract

Conceptual Schema:

```json
{
  "name": "get_balance",
  "description": "Get the cash balance...",
  "inputSchema": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string"
      }
    },
    "required": ["name"]
  }
}
```

Model ใช้ Schema เพื่อ:

```text
เลือก Tool
สร้าง Argument
ลด Invalid Calls
```

แต่ Schema ไม่ได้ตรวจว่า Caller มีสิทธิ์อ่าน Account นั้นหรือไม่

---

# 20. Read Tools

```text
get_balance
get_holdings
```

Read Tools ควร:

```text
ไม่มี Hidden Mutation
ไม่เปลี่ยน Account
คืน Structured Data
มี Low Side-effect Risk
```

ตัวอย่าง:

```python
@mcp.tool()
async def get_holdings(
    name: str,
) -> dict[str, int]:
    return Account.get(name).holdings
```

---

# 21. Write Tools

```text
buy_shares
sell_shares
change_strategy
```

Write Tools เปลี่ยน External State

```text
Tool call
→ Financial/account state mutation
```

จึงต้องการ Controls มากกว่า Read Tools:

```text
Authentication
Authorization
Validation
Approval
Idempotency
Audit logging
Rate limits
```

---

# 22. Return Type Mismatch

Server ประกาศ:

```python
async def buy_shares(...) -> float:
```

แต่ Domain Method คืน:

```python
"Completed. Latest details:\n" + self.report()
```

ซึ่งเป็น `str`

ปัญหาเดียวกันเกิดกับ `sell_shares`.

ผลกระทบ:

```text
Generated schema may be misleading
Clients may expect a number
Evaluators may fail
Tracing becomes confusing
```

ควรแก้เป็น:

```python
async def buy_shares(...) -> str:
```

หรือดีกว่า คืน Structured Result

---

# 23. Structured Trade Result

```python
class TradeResult(BaseModel):
    ok: bool
    action: str
    symbol: str
    quantity: int
    execution_price: float
    remaining_balance: float
    holdings: dict[str, int]
```

ข้อดี:

```text
Machine-readable
Easy to validate
Easy to trace
Less parsing
Clearer error handling
```

หลัก:

```text
Public schema
ต้องตรงกับ
Runtime value
```

---

# 24. Tool vs Resource

## Tool

หมายถึง Action หรือ Operation:

```text
get_balance
buy_shares
change_strategy
```

Mental model:

```text
Tool
= Verb
```

## Resource

หมายถึง Addressable Information:

```text
accounts://accounts_server/ed
accounts://strategy/ed
```

Mental model:

```text
Resource
= Noun with URI
```

---

# 25. Account Resources

```python
@mcp.resource(
    "accounts://accounts_server/{name}"
)
async def read_account_resource(
    name: str,
) -> str:
    return Account.get(name.lower()).report()
```

```python
@mcp.resource(
    "accounts://strategy/{name}"
)
async def read_strategy_resource(
    name: str,
) -> str:
    return Account.get(name.lower()).get_strategy()
```

Resources ทำให้ Client อ้างถึงข้อมูลด้วย URI Pattern

---

# 26. Resource Side-effect Problem

`read_account_resource()` เรียก:

```python
account.report()
```

แต่ `report()`:

```text
เพิ่ม portfolio snapshot
save account
write log
```

ดังนั้น:

```text
Read resource
→ Mutates state
```

นี่ขัดกับความคาดหวังทั่วไปว่า Resource Read ควรไม่มี Side Effect

ควรสร้าง Pure Read Method แยกต่างหาก

---

# 27. Starting the Server

```python
if __name__ == "__main__":
    mcp.run(
        transport="stdio"
    )
```

Server Lifecycle:

```text
Python process starts
→ FastMCP initializes
→ Reads MCP messages from stdin
→ Executes capability
→ Writes protocol results to stdout
```

เมื่อใช้ stdio:

```text
stdout
= Protocol channel
```

จึงไม่ควรใช้ `print()` เพื่อ Debug แบบทั่วไปบน stdout

---

# 28. Starting as a Module

Notebook ใช้:

```python
params = {
    "command": "uv",
    "args": [
        "run",
        "-m",
        "backend.accounts_server",
    ],
}
```

เหตุผลที่ใช้ `-m`:

```python
from .accounts import Account
```

เป็น Relative Import

การรันเป็น Module ทำให้ Python รู้ Parent Package:

```text
backend.accounts_server
```

---

# 29. Custom Server Client Lifecycle

```python
async with MCPServerStdio(
    params=params,
    client_session_timeout_seconds=30,
) as server:

    mcp_tools = await server.list_tools()
```

Flow:

```text
uv starts
→ Python module starts
→ FastMCP stdio opens
→ MCP handshake
→ list_tools()
→ Return catalog
→ Context closes
→ Server stops
```

---

# 30. Tool Discovery

Expected Tools:

```text
get_balance
get_holdings
buy_shares
sell_shares
change_strategy
```

ก่อนให้ Agent ใช้ควรตรวจ:

```text
Tool name
Description
Input schema
Return type
Whether it mutates state
Whether user approval is required
```

หลักจาก Lab 1 ยังคงเดิม:

```text
Discover
→ Inspect
→ Restrict
→ Expose
```

---

# 31. Connecting the Agent

```python
instructions = """
You are able to manage an account for a client,
and answer questions about the account.
"""

request = """
My name is Ed and my account is under the name Ed.
What's my balance and my holdings?
"""
```

Agent:

```python
async with MCPServerStdio(
    params=params,
    client_session_timeout_seconds=30,
) as mcp_server:

    agent = Agent(
        name="account_manager",
        instructions=instructions,
        model="gpt-5.4-mini",
        mcp_servers=[mcp_server],
    )

    with trace("account_manager"):
        result = await Runner.run(
            agent,
            request,
        )
```

---

# 32. Expected Agent Loop

```text
User asks for balance and holdings
        ↓
Model inspects available tools
        ↓
get_balance(name="Ed")
        ↓
Tool returns cash balance
        ↓
get_holdings(name="Ed")
        ↓
Tool returns holdings
        ↓
Model formats answer
```

Agent ไม่ควรเรียก:

```text
buy_shares
sell_shares
change_strategy
```

เพราะ User Request เป็น Read-only

---

# 33. Trace as Evidence

Trace ควรแสดง:

```text
Which tools were selected
Arguments sent
Results returned
Number of model turns
Whether mutation tools were called
```

สำคัญเพราะ Final Response อาจถูกต้อง แต่ Agent อาจเรียก Tool เกินความจำเป็น

ตัวอย่าง Risk:

```text
Agent calls report resource
→ hidden snapshot mutation
→ then calls get_balance
```

Trace ช่วยเห็น Behavior ที่ Final Output ไม่ได้บอก

---

# 34. Excessive Authority

Account Agent ได้รับ Server ทั้งตัว

ดังนั้นแม้ User ถาม Read-only Question Agent ยังเห็น Trading Tools

```text
Read request
+
Write capability
= Excess privilege
```

หลัก Least Privilege แนะนำให้แยก:

```text
Read server
├── get_balance
├── get_holdings
└── get_strategy

Trade server
├── buy_shares
├── sell_shares
└── change_strategy
```

---

# 35. Authentication Problem

Tools รับ:

```python
name: str
```

เป็น Identifier ของ Account

แต่:

```text
Account name
≠ Authentication
```

ใครก็ตามที่เข้าถึง Server สามารถเรียก:

```text
get_balance("Ed")
buy_shares("Ed", ...)
```

ไม่มี:

```text
Login
Token
Session
Ownership check
Role
Approval
```

---

# 36. Trusted Identity Pattern

ไม่ควรให้ Model กำหนด Account Authority ผ่าน Argument

Unsafe:

```python
buy_shares(
    name: str,
    symbol: str,
    quantity: int,
)
```

Safer:

```text
Authenticated session
→ Server resolves account
→ Model supplies trade details only
```

Concept:

```python
async def buy_shares(
    symbol: str,
    quantity: int,
    rationale: str,
    context: AuthContext,
):
    account = Account.get(
        context.account_id
    )
```

---

# 37. Schema Validation vs Domain Validation

## Schema Validation

ตรวจว่า:

```text
name is string
symbol is string
quantity is integer
rationale is string
```

## Domain Validation

ตรวจว่า:

```text
Caller owns account
Quantity is positive
Symbol is recognized
Market is open
Balance is sufficient
Holdings are sufficient
Trade complies with policy
```

หลัก:

```text
Valid JSON
ไม่ได้แปลว่า
Valid business operation
```

---

# 38. Negative Quantity Risk

Domain Method ไม่มี Validation ที่ชัดเจนว่า:

```text
quantity > 0
```

หากเรียก:

```text
buy_shares(quantity=-10)
```

อาจทำให้:

```text
Total cost negative
Balance increases
Holdings decrease
```

ควรใช้:

```python
from pydantic import PositiveInt
```

หรือ:

```python
if quantity <= 0:
    raise ValueError(
        "Quantity must be positive"
    )
```

---

# 39. Market-open Policy

`market.py` มี:

```python
is_market_open()
```

แต่ Trade Methods ไม่ได้เรียกตรวจ Market State.

จึงต้องตัดสินใจใน Domain Policy ว่า:

```text
Reject trade
Queue order
Allow simulated mode
Allow after-hours
```

MCP ไม่ได้กำหนด Policy นี้

---

# 40. Error Handling

Domain อาจ Raise:

```text
Insufficient funds
Unrecognized symbol
Not enough shares
```

MCP Framework จะแปลง Exception เป็น Tool Error

Production ควรใช้ Structured Error:

```json
{
  "ok": false,
  "code": "INSUFFICIENT_FUNDS",
  "message": "The account does not have enough cash.",
  "retryable": false
}
```

ข้อดี:

```text
Agent understands error category
Client handles errors consistently
Logs are searchable
Tests become deterministic
```

---

# 41. Audit Logging

Domain เขียน Logs เช่น:

```text
Bought 3 of AMZN
Sold 2 of AMZN
Changed strategy
Retrieved account details
```

แต่ Production Audit ต้องเพิ่ม:

```text
Authenticated user
Agent name
Model
Request ID
Tool-call ID
Old state
New state
Approval ID
Success or failure
Timestamp
```

---

# 42. Concurrency Risk

Account State ถูกเก็บเป็น JSON ทั้งก้อน

```text
Read JSON
→ Modify object
→ Write JSON
```

ถ้า Calls สองตัวเกิดพร้อมกัน:

```text
Call A reads balance
Call B reads same balance
Call A writes new state
Call B overwrites Call A
```

เรียกว่า:

```text
Lost update
```

MCP ไม่แก้ปัญหานี้

Domain ต้องใช้:

```text
Database transaction
Optimistic locking
Version number
Atomic updates
```

---

# 43. Custom MCP vs Direct Import

## Direct Import

```python
from backend.accounts import Account
```

ข้อดี:

```text
Simple
Low latency
Easy debugging
```

ข้อจำกัด:

```text
Same language
Same process/package environment
Strong coupling
Harder to share externally
```

## MCP

```text
Client
→ Protocol
→ Server
```

ข้อดี:

```text
Framework independence
Language independence
Process isolation
Reusable capability
Clear public contract
```

ต้นทุน:

```text
Transport
Process lifecycle
Schema versioning
Security
Authentication
Observability
```

---

# 44. Exercise — Push Notification Server

Notebook ให้สร้าง MCP Server สำหรับส่ง Push Notification โดยมีตัวอย่างใน:

```text
backend/push_server.py
```

Server:

```python
mcp = FastMCP(
    "push_server"
)
```

Input Model:

```python
class PushModelArgs(BaseModel):
    message: str = Field(
        description=(
            "A brief message to push"
        )
    )
```

---

# 45. Pydantic Input Model

Tool:

```python
@mcp.tool()
def push(
    args: PushModelArgs
):
    ...
```

ข้อดี:

```text
Reusable schema
Field descriptions
Runtime validation
Easy extension
Clearer contracts
```

สามารถเพิ่ม:

```python
message: str = Field(
    min_length=1,
    max_length=200,
)
```

---

# 46. Push Flow

```text
Agent
→ MCP push tool
→ requests.post()
→ Pushover API
→ User device
```

Credentials อยู่ฝั่ง Server:

```env
PUSHOVER_USER=...
PUSHOVER_TOKEN=...
```

นี่เป็นแนวทางที่ถูกกว่าให้ Model ส่ง Credentials เป็น Arguments

---

# 47. Push Server Weakness

Code ปัจจุบัน:

```python
requests.post(
    pushover_url,
    data=payload,
)

return "Push notification sent"
```

ไม่มี:

```text
HTTP timeout
Status validation
Credential validation
Retry policy
Idempotency
```

จึงอาจรายงาน Success แม้ Request ล้มเหลว

Safer:

```python
response = requests.post(
    pushover_url,
    data=payload,
    timeout=10,
)

response.raise_for_status()
```

---

# 48. Push Side-effect Risk

Push Tool สามารถสร้างผลกระทบนอกระบบทันที

Risk:

```text
Duplicate notifications
Spam
Sensitive data exposure
Prompt injection
Wrong message
Repeated agent calls
```

ควรมี:

```text
Explicit user request
One-call budget
Rate limit
Message length limit
Approval
Audit record
Idempotency key
```

---

# 49. Read–Plan–Approve–Execute Pattern

สำหรับ Trade Tools ที่มีความเสี่ยงสูง:

```text
User request
→ Agent analyzes
→ preview_trade()
→ Show expected transaction
→ Human approves
→ execute_trade()
→ Return receipt
```

ไม่ควร:

```text
Agent sees stock recommendation
→ Immediately calls buy_shares
```

Approval Token ควรถูกสร้างโดย Trusted Application ไม่ใช่ Model

---

# 50. MCP Server Development Workflow

```text
1. Implement domain capability

2. Test domain capability directly

3. Create FastMCP instance

4. Expose one read-only tool

5. Start server via stdio

6. Run list_tools()

7. Inspect generated schemas

8. Test MCP call without agent

9. Add agent client

10. Inspect trace

11. Add side-effect tools carefully

12. Add identity and authorization

13. Add deterministic tests

14. Add logs and metrics
```

---

# 51. Testing Layers

## Domain Unit Tests

```text
Account.get()
Account.buy_shares()
Account.sell_shares()
```

## MCP Contract Tests

```text
Server starts
Expected tools exist
Schemas are correct
Return types match
```

## Integration Tests

```text
get_balance result
matches Account.get().balance
```

## Agent Behavior Tests

```text
Read request
→ Read tools only
```

## Security Tests

```text
Unauthorized account
Negative quantity
Duplicate calls
Missing credentials
Side-effect approval
```

---

# 52. Tool Design Principles

MCP Tool ที่ดีควร:

```text
มีชื่อชัดเจน
มี Scope เล็ก
มี Input constraints
คืน Structured Result
อธิบาย Side Effects
มี Stable error codes
ไม่รับ Authority ผ่าน free text
มี Audit Trail
รองรับ Idempotency เมื่อจำเป็น
```

ควรหลีกเลี่ยง:

```text
manage_account(action, payload)
```

เพราะ:

```text
Schema กว้าง
Audit ยาก
Authorization ยาก
Model selection ไม่ชัด
```

---

# 53. Tool Granularity

Lab ใช้ Tools แยกตาม Operation:

```text
get_balance
get_holdings
buy_shares
sell_shares
change_strategy
```

ข้อดี:

```text
Tool selection ชัด
Policy ต่อ Tool ง่าย
Logs อ่านง่าย
Schema เล็ก
```

Production Trading System อาจแยกละเอียดขึ้น:

```text
preview_order
submit_order
approve_order
cancel_order
get_order_status
```

---

# 54. MCP Does Not Define Business Policy

MCP กำหนด:

```text
Tool discovery
Input schemas
Invocation
Results
Transport
```

MCP ไม่ได้กำหนด:

```text
ใครเรียกได้
เรียกได้เมื่อไร
วงเงินสูงสุด
ต้องอนุมัติหรือไม่
ต้องใช้ข้อมูลตลาดแบบใด
ต้องบันทึกอะไร
```

สิ่งเหล่านี้อยู่ใน Domain และ Application Policy

---

# 55. Common Misconceptions

## “ใส่ `@mcp.tool()` แล้ว Function ปลอดภัย”

ไม่จริง Decorator ทำให้ Function เรียกผ่าน MCP ได้เท่านั้น

## “Type Hint ป้องกันข้อมูลผิดทั้งหมด”

ไม่จริง `int` ยังรับค่าติดลบได้

## “Resource เป็น Read-only เสมอ”

ไม่จริง Implementation สามารถเปลี่ยน State ได้

## “Task เป็น Read-only แล้ว Agent ไม่มี Write Authority”

ไม่จริง หาก Server เปิด Write Tools Agent ยังเห็น Capabilities เหล่านั้น

## “Local MCP Server ปลอดภัยกว่าเสมอ”

ไม่จริง Local Server อาจเข้าถึง Database, Environment Variables และ External APIs

## “Tool คืน Success หมายถึง External Action สำเร็จ”

ไม่จริง ต้องตรวจ Response จากระบบปลายทาง

---

# 56. Risks Identified

## 56.1 Missing Authentication

Account name ถูกใช้แทน Identity

## 56.2 Excessive Capability

Read request ได้รับ Trading Tools

## 56.3 Negative Quantity

ไม่มี Positive Quantity Constraint

## 56.4 Return-type Mismatch

Trade Tool ประกาศ `float` แต่คืน `str`

## 56.5 Hidden Resource Mutation

Resource read บันทึก Portfolio Snapshot

## 56.6 Lost Update

Concurrent JSON writes ทับกัน

## 56.7 Simulated Data Ambiguity

ไม่แสดงชัดว่าราคาเป็น Simulator

## 56.8 Weak Error Contract

ใช้ Exception text แทน Error Code

## 56.9 Push False Success

ไม่ตรวจ HTTP Response

## 56.10 Duplicate Side Effects

Agent อาจ Trade หรือ Push ซ้ำ

## 56.11 Weak Audit Context

ไม่มี Agent, Model, Request หรือ Approval IDs

## 56.12 Hidden stderr

Notebook Workaround ซ่อน Server Errors

---

# 57. Production Improvements

```text
Separate read and write servers
Authenticate MCP clients
Resolve account from trusted identity
Use PositiveInt constraints
Return structured result models
Add preview and approval workflow
Use idempotency keys
Add per-tool authorization
Capture stdio server logs
Use database transactions
Add optimistic locking
Separate pure reads from mutations
Expose market-data source
Validate external HTTP responses
Add rate limits
Store trace and audit correlation IDs
```

---

# 58. Suggested Exercise — Inspect Schemas

```python
for tool in mcp_tools:
    print(tool.name)
    print(tool.description)
    print(tool.inputSchema)
```

ตรวจว่า:

```text
Required fields ถูกต้อง
Descriptions ชัดเจน
Types ตรงกับ Runtime
Write tools ถูกระบุชัด
```

---

# 59. Suggested Exercise — Fix Trade Results

สร้าง:

```python
class TradeResult(BaseModel):
    success: bool
    symbol: str
    quantity: int
    remaining_balance: float
    holdings: dict[str, int]
```

แก้ `buy_shares()` และ `sell_shares()` ให้ Return Contract ตรงกันทุก Layer

---

# 60. Suggested Exercise — Read-only MCP Server

สร้าง:

```text
accounts_read_server.py
```

เปิดเฉพาะ:

```text
get_balance
get_holdings
get_strategy
```

จากนั้นตรวจ Trace ว่า Agent ไม่มี Tool สำหรับเปลี่ยน State

---

# 61. Suggested Exercise — Positive Quantity

ทดสอบ:

```text
quantity = 5
quantity = 0
quantity = -5
```

Expected:

```text
5
→ accepted

0, -5
→ rejected before domain mutation
```

---

# 62. Suggested Exercise — Push Reliability

เพิ่ม:

```text
Timeout
raise_for_status()
Credential checks
Rate limit
Audit log
Message length constraint
```

แล้วจำลอง API Failure เพื่อยืนยันว่า Tool ไม่คืน False Success

---

# 63. Suggested Exercise — Human Approval

สร้างสอง Tools:

```text
preview_buy
execute_buy
```

`preview_buy` คืน:

```text
Expected price
Total cost
Remaining cash
Risk note
Approval request ID
```

`execute_buy` ต้องรับ Approval ID ที่ Application ยืนยันแล้ว

---

# 64. Patterns Learned

## Domain Adapter Pattern

```text
Domain model
→ MCP adapter
→ External clients
```

## Tool Decorator Pattern

```text
Typed function
+ docstring
+ @mcp.tool()
→ MCP tool contract
```

## Resource URI Pattern

```text
URI template
→ Addressable domain information
```

## stdio Server Pattern

```text
Client
→ subprocess
→ stdin/stdout
→ FastMCP server
```

## Least-privilege Server Pattern

```text
Task-specific client
→ Only required tools
```

## Approval Pattern

```text
Plan
→ Preview
→ Human approval
→ Execute
```

## Structured Result Pattern

```text
Domain operation
→ Pydantic result
→ Stable client contract
```

---

# 65. Lab 2 Mental Model

```text
Developer
    ↓
Build and test domain logic
    ↓
Select safe public capabilities
    ↓
Create FastMCP server
    ↓
Decorate tools and resources
    ↓
Run server through stdio
    ↓
Client discovers schemas
    ↓
Agent selects capability
    ↓
Server validates and executes
    ↓
Domain state changes
    ↓
Structured result and audit evidence
```

---

# 66. Final Lessons

## Lesson 1

MCP Server ควรห่อ Domain Logic ไม่ควรแทนที่ Domain Logic

## Lesson 2

FastMCP ลดงาน Protocol Boilerplate แต่ไม่เพิ่ม Business Security โดยอัตโนมัติ

## Lesson 3

`@mcp.tool()` เปิด Action ให้ Client และ Agent เรียก

## Lesson 4

`@mcp.resource()` เปิด Addressable Information ผ่าน URI

## Lesson 5

Tools คือ Actions ส่วน Resources คือ Addressable Data

## Lesson 6

Type Hints และ Docstrings เป็นส่วนหนึ่งของ Public Agent Contract

## Lesson 7

Declared Return Type ต้องตรงกับ Runtime Result

## Lesson 8

Read Tools และ Write Tools ไม่ควรถูกให้สิทธิ์แบบเดียวกัน

## Lesson 9

Account Name ไม่ใช่ Authentication Credential

## Lesson 10

Schema Validation ตรวจ Data Shape ส่วน Domain Validation ตรวจ Business Correctness

## Lesson 11

Resource Read ควรหลีกเลี่ยง Hidden Mutation

## Lesson 12

MCP ไม่ได้แก้ Database Concurrency หรือ Lost Update

## Lesson 13

External Side Effects ต้องตรวจผลจากระบบปลายทางจริง

## Lesson 14

Trace และ Audit Log ต้องเชื่อมกันด้วย Correlation IDs

## Lesson 15

Production MCP Server ควรเริ่มจาก Capability ที่เล็ก Read-only และตรวจสอบง่าย

---

# 67. Memory Summary

```text
Week 6 Lab 2:
Build a custom MCP server

Notebook:
6_mcp/2_lab2.ipynb

Domain:
backend/accounts.py

MCP server:
backend/accounts_server.py

Framework:
FastMCP

Server creation:
mcp = FastMCP("accounts_server")

Tool decorator:
@mcp.tool()

Resource decorator:
@mcp.resource(uri_template)

Server transport:
mcp.run(transport="stdio")

Client:
MCPServerStdio

Launch command:
uv run -m backend.accounts_server

Core tools:
get_balance
get_holdings
buy_shares
sell_shares
change_strategy

Resources:
accounts://accounts_server/{name}
accounts://strategy/{name}

Domain persistence:
SQLite

Account storage:
JSON document

Audit storage:
logs table

Market data:
Massive API
or simulator fallback

Main agent:
account_manager

Model:
gpt-5.4-mini

Agent task:
Read balance and holdings

Expected calls:
get_balance
get_holdings

Important:
Read request still receives write tools

Main security issue:
No authentication or ownership verification

Main validation issue:
quantity can be non-positive

Main contract issue:
Trade tools declare float
but return string

Main resource issue:
Account resource read mutates state

Main concurrency issue:
Whole JSON account can suffer lost update

Exercise:
Build push notification MCP server

Push implementation:
FastMCP
Pydantic input model
Pushover HTTP API

Push risk:
False success
duplicate calls
spam
sensitive content

Production requirements:
Read/write separation
Authentication
Authorization
Positive constraints
Structured results
Approval
Idempotency
Transactions
Audit correlation
External response validation
```

---

# 68. Next Episode

Lab ถัดไปควรจับตาว่า MCP Server จะถูกนำไปใช้ใน Architecture ที่ใหญ่ขึ้นอย่างไร โดยเฉพาะ:

```text
Multiple MCP servers
Agent specialization
Trading agents
Market data
Push notifications
Guardrails
Continuous execution
Deployment
```

คำถามสำคัญคือ:

> เมื่อ Business Capabilities ถูกเปิดผ่าน MCP แล้ว เราจะประกอบ Tools จากหลาย Servers ให้กลายเป็น Agentic Application โดยควบคุม Authority, State, Observability และ Side Effects ได้อย่างไร?
