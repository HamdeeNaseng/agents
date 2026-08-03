# Week 6 — Lab 2: สร้าง MCP Server ของตัวเองด้วย FastMCP

ตำแหน่ง Lab:

```text
6_mcp/
├── 2_lab2.ipynb
└── backend/
    ├── accounts.py
    ├── accounts_server.py
    ├── database.py
    ├── market.py
    └── push_server.py
```

Lab 1 ทำให้เราเป็น **MCP Client** ที่นำ Servers ของคนอื่นมาใช้ ส่วน Lab 2 สลับบทบาท:

```text
Lab 1
ใช้ MCP Server ที่มีอยู่แล้ว

Lab 2
นำ Business Logic ของเรา
→ ห่อเป็น MCP Server
→ ให้ Agent เรียกใช้งาน
```

โจทย์หลักคือเปลี่ยนระบบจำลองบัญชีลงทุนให้กลายเป็น MCP Server จากนั้นให้ OpenAI Agent ตรวจยอดเงินและหุ้นที่ถือผ่าน Tools โดยไม่ต้อง Import `Account` Class เข้าไปใน Agent Application โดยตรง.

---

## Learning Objectives

หลังจบ Lab นี้ควรเข้าใจว่า:

1. Business Logic และ MCP Interface ควรแยกกันอย่างไร
2. `FastMCP` ใช้สร้าง MCP Server อย่างไร
3. `@mcp.tool()` เปลี่ยน Python Function เป็น MCP Tool อย่างไร
4. `@mcp.resource()` ต่างจาก Tool อย่างไร
5. Type Hints และ Docstrings กลายเป็น Tool Schema อย่างไร
6. `mcp.run(transport="stdio")` เปิด Server อย่างไร
7. เหตุใดจึงรัน Server ด้วย `python -m` หรือ `uv run -m`
8. OpenAI Agents SDK เชื่อม Custom MCP Server อย่างไร
9. Agent เลือก `get_balance` และ `get_holdings` อย่างไร
10. Business Validation, Authorization และ Audit Logging ควรอยู่ชั้นใด
11. MCP Schema Validation แตกต่างจาก Domain Validation อย่างไร
12. Push Notification Exercise เปิด Side-effect Risk อะไรบ้าง

---

# 1. เริ่มจาก Business Logic ก่อน MCP

Notebook ไม่เริ่มด้วย `FastMCP` ทันที แต่ให้ทดลอง `Account` Class โดยตรง:

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

จุดประสงค์คือยืนยันว่า **Domain Layer ทำงานได้ก่อน** แล้วค่อยสร้าง Protocol Adapter ครอบด้านนอก.

Mental model:

```text
Account class
= Business capability

accounts_server.py
= MCP adapter

OpenAI Agent
= Capability consumer
```

นี่เป็นหลักออกแบบที่สำคัญ:

> อย่าใส่ Business Logic ทั้งหมดไว้ใน MCP Tool Function เพราะจะทำให้ Logic ผูกกับ Protocol และทดสอบแยกยาก

---

# 2. Architecture ของ Lab

```text
OpenAI Agent
      ↓
OpenAI Agents SDK MCP Client
      ↓ stdio
accounts_server.py
      ↓
Account domain model
      ↓
SQLite + market price provider
```

Agent ไม่ได้เปิด SQLite เอง และไม่รู้รายละเอียดการคำนวณ Portfolio

Agent รู้เพียง Tool Contracts เช่น:

```text
get_balance(name)
get_holdings(name)
buy_shares(name, symbol, quantity, rationale)
sell_shares(name, symbol, quantity, rationale)
change_strategy(name, strategy)
```

Server เป็น Boundary ที่แปลง Tool Call จาก MCP ให้กลายเป็น Method Call ใน Domain Layer.

---

# 3. `Account` Domain Model

`accounts.py` ใช้ Pydantic Models สองตัว:

```python
class Transaction(BaseModel):
    symbol: str
    quantity: int
    price: float
    timestamp: str
    rationale: str
```

และ:

```python
class Account(BaseModel):
    name: str
    balance: float
    strategy: str
    holdings: dict[str, int]
    transactions: list[Transaction]
    portfolio_value_time_series: list[tuple[str, float]]
```

บัญชีเริ่มต้นด้วย:

```python
INITIAL_BALANCE = 10_000.0
SPREAD = 0.002
```

`SPREAD = 0.002` หมายถึงการซื้อและขายมีส่วนต่าง 0.2%:

```text
Buy price
= market price × 1.002

Sell price
= market price × 0.998
```

---

# 4. Account Persistence

เมื่อเรียก:

```python
Account.get("Ed")
```

ระบบจะ:

```text
แปลงชื่อเป็น lowercase
→ อ่าน account จาก SQLite
→ ถ้าไม่พบ สร้างบัญชีใหม่
→ มีเงินเริ่มต้น $10,000
→ บันทึกลงฐานข้อมูล
```

บัญชีถูก Serialize เป็น JSON แล้วเก็บใน Table:

```sql
accounts (
    name TEXT PRIMARY KEY,
    account TEXT
)
```

ส่วน Audit Events ถูกเก็บใน Table:

```sql
logs (
    id,
    name,
    datetime,
    type,
    message
)
```

Architecture:

```text
Account object
→ model_dump()
→ JSON
→ SQLite accounts table
```

นี่เหมาะกับ Course Demo เพราะ Schema เรียบง่าย แต่ Production มักแยก:

```text
accounts
holdings
transactions
portfolio_snapshots
audit_logs
```

ออกเป็น Tables ที่มี Constraints ชัดเจนกว่า

---

# 5. Market Price Provider

`Account.buy_shares()` และ `sell_shares()` เรียก:

```python
get_share_price(symbol)
```

`market.py` รองรับสอง Mode:

```text
มี MASSIVE_API_KEY
→ ใช้ข้อมูลตลาดจาก Massive API

ไม่มี MASSIVE_API_KEY
→ ใช้ market simulator
```

ระบบพยายามใช้ราคาตามลำดับ:

```text
Last trade
→ Snapshot
→ Previous close
```

ถ้า API ใช้งานไม่ได้ จะกลับไปใช้ Simulated Price เพื่อให้ Lab ยังรันได้.

นี่คือ Graceful Fallback:

```text
Live dependency fails
→ Demo does not completely stop
```

แต่ต้องระวังว่า Agent อาจไม่รู้ว่าราคาที่ได้รับเป็นข้อมูลจำลอง

---

# 6. Buy Transaction Flow

```python
account.buy_shares(
    symbol,
    quantity,
    rationale,
)
```

Flow:

```text
Get market price
→ Add buy spread
→ Calculate total cost
→ Check balance
→ Update holdings
→ Record transaction
→ Deduct balance
→ Save account
→ Write audit log
→ Return latest account report
```

Transaction เก็บ:

```text
Symbol
Quantity
Executed price
Timestamp
Rationale
```

`rationale` สำคัญต่อ Agentic System เพราะช่วยเก็บเหตุผลของการตัดสินใจ ไม่ใช่เพียงผลลัพธ์สุดท้าย

---

# 7. Sell Transaction Flow

```python
account.sell_shares(
    symbol,
    quantity,
    rationale,
)
```

Flow:

```text
Check available shares
→ Get market price
→ Apply sell spread
→ Reduce holdings
→ Remove zero holding
→ Record negative transaction quantity
→ Increase cash balance
→ Save
→ Write log
```

Sell Transaction ใช้ Quantity ติดลบใน Transaction History:

```python
quantity=-quantity
```

ทำให้การรวม Transaction Flows สามารถแยก Buy และ Sell จากเครื่องหมายได้.

---

# 8. Account Report

```python
account.report()
```

จะ:

```text
คำนวณ Portfolio Value
→ เพิ่ม Snapshot ลง Time Series
→ บันทึก Account
→ คำนวณ Profit/Loss
→ เขียน Audit Log
→ คืน JSON String
```

รายละเอียดที่ควรสังเกตคือ `report()` ไม่ใช่ Read-only Operation อย่างแท้จริง เพราะมันเพิ่ม Portfolio Snapshot และ Save Account ทุกครั้ง

```text
Method ชื่อ report
แต่มี Side Effect
```

Production API ควรตั้งชื่อหรือแยก Operation ให้ชัด เช่น:

```text
get_report()
record_portfolio_snapshot()
```

เพื่อไม่ให้ Caller เข้าใจว่าเป็น Pure Read

---

# 9. สร้าง MCP Server

ไฟล์ `accounts_server.py` เริ่มจาก:

```python
from mcp.server.fastmcp import FastMCP
from .accounts import Account

mcp = FastMCP("accounts_server")
```

`FastMCP` ทำหน้าที่คล้าย Web Framework ขนาดเล็กสำหรับ MCP:

```text
FastAPI
→ สร้าง HTTP API

FastMCP
→ สร้าง MCP capabilities
```

แต่ Transport และ Protocol ต่างกัน

Server Name:

```python
"accounts_server"
```

ใช้เป็น Identity ของ MCP Server.

---

# 10. สร้าง Tool ด้วย `@mcp.tool()`

ตัวอย่าง Read Tool:

```python
@mcp.tool()
async def get_balance(
    name: str,
) -> float:
    """Get the cash balance of the given account name.

    Args:
        name: The name of the account holder
    """
    return Account.get(name).balance
```

FastMCP ใช้:

```text
Function name
Type hints
Docstring
Parameter descriptions
```

เพื่อสร้าง Tool Definition และ Input Schema

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

---

# 11. `get_holdings`

```python
@mcp.tool()
async def get_holdings(
    name: str,
) -> dict[str, int]:
    """Get the holdings of the given account name."""
    return Account.get(name).holdings
```

Expected Result:

```json
{
  "AMZN": 3
}
```

Tool Contract นี้ดีกว่าการคืนข้อความธรรมดา เพราะ Caller สามารถใช้ Structured Data ต่อได้

```text
Structured result
→ Easier downstream processing
→ Less parsing
```

---

# 12. Write Tools

Server ยังเปิด Tools ที่เปลี่ยน State:

```text
buy_shares
sell_shares
change_strategy
```

ตัวอย่าง:

```python
@mcp.tool()
async def buy_shares(
    name: str,
    symbol: str,
    quantity: int,
    rationale: str,
) -> float:
    return Account.get(name).buy_shares(
        symbol,
        quantity,
        rationale,
    )
```

ความแตกต่างสำคัญ:

```text
get_balance
get_holdings
= Read operations

buy_shares
sell_shares
change_strategy
= Side-effect operations
```

การให้ Agent เห็นทั้ง Read และ Write Tools หมายความว่า Agent มี Authority ในการเปลี่ยน Portfolio

---

# 13. จุดผิดปกติของ Return Type

ใน `accounts_server.py`:

```python
async def buy_shares(...) -> float:
```

แต่ `Account.buy_shares()` คืน:

```python
"Completed. Latest details:\n" + self.report()
```

ซึ่งเป็น `str` ไม่ใช่ `float`

` sell_shares()` มีปัญหาแบบเดียวกัน.

ควรแก้เป็น:

```python
async def buy_shares(...) -> str:
```

หรือคืน Structured Model:

```python
class TradeResult(BaseModel):
    status: str
    symbol: str
    quantity: int
    execution_price: float
    remaining_balance: float
    holdings: dict[str, int]
```

หลัก:

```text
Declared contract
ต้องตรงกับ
Runtime result
```

ไม่เช่นนั้น Client, Schema Generator หรือ Evaluation System อาจตีความผลผิด

---

# 14. Tools vs Resources

Server ประกาศ Resources สองตัว:

```python
@mcp.resource(
    "accounts://accounts_server/{name}"
)
async def read_account_resource(
    name: str,
) -> str:
    ...
```

และ:

```python
@mcp.resource(
    "accounts://strategy/{name}"
)
async def read_strategy_resource(
    name: str,
) -> str:
    ...
```

## Tool

Model เรียกเพื่อทำ Operation:

```text
get_balance("Ed")
buy_shares("Ed", "AMZN", 3, ...)
```

## Resource

ข้อมูลที่ระบุด้วย URI และ Client สามารถอ่าน:

```text
accounts://accounts_server/ed
accounts://strategy/ed
```

Mental model:

```text
Tool
= Verb / action

Resource
= Noun / addressable data
```

ตัวอย่าง:

```text
buy_shares
= ทำบางสิ่ง

accounts://accounts_server/ed
= อ่านข้อมูลของบางสิ่ง
```

---

# 15. Resources ไม่ได้ถูกใช้ใน Agent Task นี้โดยตรง

Lab ถาม Agent ว่า:

```text
What's my balance and my holdings?
```

Agent มี Tools:

```text
get_balance
get_holdings
```

จึงสามารถตอบผ่าน Tool Calls โดยไม่ต้องใช้ Resources

Resources ใน `accounts_server.py` แสดงว่า Server สามารถให้ Capability ได้มากกว่า Tools แต่ Notebook นี้เน้น Agent Tool Use เป็นหลัก.

---

# 16. เปิด Server ผ่าน stdio

ท้ายไฟล์:

```python
if __name__ == "__main__":
    mcp.run(
        transport="stdio"
    )
```

Flow:

```text
Python module starts
→ FastMCP initializes
→ Server listens on stdin
→ Client sends MCP messages
→ Server replies on stdout
```

สิ่งสำคัญ:

> เมื่อใช้ stdio อย่า `print()` ข้อมูลทั่วไปไปยัง stdout แบบไม่ควบคุม เพราะ stdout เป็น Channel ของ MCP Protocol

Logs ควรไปที่:

```text
stderr
หรือ
structured log file
```

---

# 17. ทำไมรันด้วย `-m`

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

เทียบกับ:

```text
uv run backend/accounts_server.py
```

การใช้:

```text
-m backend.accounts_server
```

ทำให้ Python รันไฟล์ในฐานะ Module ภายใน Package

จึงรองรับ Relative Import:

```python
from .accounts import Account
```

ถ้ารันไฟล์ตรง ๆ Relative Import อาจล้มเหลวเพราะไม่มี Parent Package Context

---

# 18. Server Process Lifecycle

Notebook เปิด Server:

```python
async with MCPServerStdio(
    params=params,
    client_session_timeout_seconds=30,
) as server:
    mcp_tools = await server.list_tools()
```

Flow:

```text
Enter async context
→ Run uv process
→ uv starts backend.accounts_server
→ FastMCP opens stdio
→ Client initializes session
→ list_tools()
→ Exit context
→ Close process
```

Timeout 30 วินาทีช่วยป้องกัน Notebook รอ Server Startup ไม่สิ้นสุด

---

# 19. Inspect Tool Catalog

```python
mcp_tools = await server.list_tools()
mcp_tools
```

Expected Catalog ประกอบด้วย:

```text
get_balance
get_holdings
buy_shares
sell_shares
change_strategy
```

Resources ไม่ใช่รายการเดียวกับ Tools

จุดที่ต้องตรวจ:

```text
Tool name
Description
Input schema
Read/write behavior
Sensitive side effects
Return contract
```

หลักจาก Lab 1 ยังเหมือนเดิม:

```text
Connect
→ Inspect
→ Approve
→ Expose to Agent
```

---

# 20. เชื่อม Custom Server กับ Agent

Configuration:

```python
instructions = (
    "You are able to manage an account "
    "for a client, and answer questions "
    "about the account."
)

request = (
    "My name is Ed and my account is "
    "under the name Ed. What's my balance "
    "and my holdings?"
)

model = "gpt-5.4-mini"
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
        model=model,
        mcp_servers=[mcp_server],
    )

    with trace("account_manager"):
        result = await Runner.run(
            agent,
            request,
        )
```

---

# 21. Expected Agent Loop

```text
User asks for balance and holdings
        ↓
Model inspects available tools
        ↓
Call get_balance(name="Ed")
        ↓
Receive numeric balance
        ↓
Call get_holdings(name="Ed")
        ↓
Receive holdings dictionary
        ↓
Model formats answer
```

Agent ไม่จำเป็นต้องรู้ว่า:

```text
ข้อมูลอยู่ใน SQLite
Account ถูก Serialize เป็น JSON
Market price มาจาก Massive หรือ Simulator
```

นี่คือ Encapsulation

```text
Consumer knows contract
not implementation
```

---

# 22. Why This Is Better Than Importing `Account`

แบบ Direct Import:

```python
from backend.accounts import Account

account = Account.get("Ed")
```

ข้อจำกัด:

```text
Caller ต้องใช้ Python
ต้องติดตั้ง Domain Package
ต้องอยู่ใน Process เดียวกันหรือแชร์ Library
Implementation coupling สูง
```

แบบ MCP:

```text
Agent client
→ Protocol
→ Account server
```

ข้อดี:

```text
Framework independence
Language independence
Process isolation
Centralized business capability
Reusable by many clients
```

แต่ MCP เพิ่ม:

```text
Process startup
Transport
Schema management
Authorization
Observability
Failure handling
```

ดังนั้น MCP ไม่ได้ทำให้ Architecture ง่ายขึ้นทุกกรณี แต่ทำให้ Boundary ชัดและ Reusable มากขึ้น

---

# 23. Trace

```python
with trace("account_manager"):
```

ช่วยดู:

```text
Tool discovery
Tool selection
Arguments
Tool results
Model turns
Final response
```

ควรตรวจว่า Agent:

```text
เรียกเฉพาะ get_balance และ get_holdings
ไม่เรียก buy_shares
ไม่เรียก change_strategy
```

เพราะคำขอของผู้ใช้เป็น Read-only

---

# 24. Side-effect Tool Risk

Agent ได้รับ Server ทั้งตัว จึงเห็นทั้ง:

```text
Read tools
Write tools
```

แม้ Task ถามเพียงยอดเงิน แต่ Model มี Capability ซื้อขายหุ้นได้

นี่ขัดกับหลัก Least Privilege:

```text
Read request
แต่
Agent receives write authority
```

Production ควรแยก Servers หรือ Filter Tools:

```text
accounts_read_server
├── get_balance
├── get_holdings
└── get_report

accounts_trade_server
├── buy_shares
├── sell_shares
└── change_strategy
```

หรือกำหนด Approval Gate สำหรับ Write Tools

---

# 25. Authentication Problem

Tool รับเพียง:

```python
name: str
```

ดังนั้นใครก็ตามที่เรียก Server และรู้ชื่อบัญชี อาจทำ:

```text
get_balance("Ed")
buy_shares("Ed", ...)
change_strategy("Ed", ...)
```

ไม่มี:

```text
User authentication
Account ownership verification
Role
Session
Access token
Approval
```

ชื่อบัญชีไม่ใช่ Security Credential

Production ต้องแยก:

```text
Identity
Authorization
Tool argument
```

ไม่ควรให้ Model เป็นผู้กำหนด Identity ที่มี Authority เอง

---

# 26. Safer Identity Pattern

แทน:

```python
buy_shares(
    name: str,
    ...
)
```

ควรให้ Trusted Server Context กำหนด Account:

```text
Authenticated user
→ Server resolves account ID
→ Model supplies only trade details
```

Concept:

```python
async def buy_shares(
    symbol: str,
    quantity: int,
    rationale: str,
    context: AuthenticatedContext,
):
    account = Account.get(
        context.account_id
    )
```

Model ไม่ควรเลือก Account Owner ผ่าน Free-text Argument

---

# 27. Missing Quantity Validation

`buy_shares()` และ `sell_shares()` รับ:

```python
quantity: int
```

แต่ Domain Method ไม่ได้ตรวจชัดเจนว่า:

```text
quantity > 0
```

Negative Quantity อาจทำให้ Logic ผิด เช่น:

```text
buy negative shares
→ total cost becomes negative
→ balance may increase
→ holdings may decrease
```

Tool Schema ตรวจเพียงว่า Quantity เป็น Integer

ไม่ตรวจว่าเป็น Positive Integer

ควรใช้ Domain Validation:

```python
if quantity <= 0:
    raise ValueError(
        "Quantity must be positive"
    )
```

และอาจกำหนด Schema Constraint:

```python
from pydantic import PositiveInt
```

---

# 28. Tool Schema vs Domain Validation

## MCP Schema

ตรวจ:

```text
name เป็น string
symbol เป็น string
quantity เป็น integer
rationale เป็น string
```

## Domain Validation

ต้องตรวจ:

```text
Account exists
User owns account
Symbol is supported
Quantity is positive
Funds are sufficient
Holdings are sufficient
Market is open
Trade complies with policy
```

ดังนั้น:

```text
Schema validation
= รูปแบบข้อมูล

Domain validation
= ความถูกต้องของ Operation
```

---

# 29. Market State ไม่ได้ถูกบังคับใน Trade

`market.py` มี:

```python
is_market_open()
```

แต่ `Account.buy_shares()` และ `sell_shares()` ไม่ได้เรียก Function นี้.

จึงสามารถซื้อขายได้แม้ Market Provider รายงานว่าตลาดปิด

ใน Demo อาจตั้งใจให้ใช้ง่าย แต่ Production ต้องตัดสินใจว่า:

```text
Reject order
Queue order
Use after-hours policy
Use simulator mode
```

---

# 30. Error Handling

Domain Methods อาจ Raise:

```text
Insufficient funds
Unrecognized symbol
Not enough shares
```

MCP Server ปล่อย Exception ให้ Framework แปลงเป็น Tool Error

ข้อดี:

```text
Agent เห็นว่าการเรียกล้มเหลว
สามารถอธิบายหรือแก้ Argument
```

แต่ Production ควรคืน Error Codes ที่แยกประเภท:

```json
{
  "ok": false,
  "code": "INSUFFICIENT_FUNDS",
  "message": "Available balance is...",
  "retryable": false
}
```

แทนการพึ่งข้อความ Exception เพียงอย่างเดียว

---

# 31. Audit Logging

Account Operations เรียก:

```python
write_log(
    name,
    "account",
    message,
)
```

Events เช่น:

```text
Bought 3 of AMZN
Sold 2 of AMZN
Changed strategy
Retrieved account details
```

ถูกบันทึกใน SQLite `logs` Table.

นี่เป็นจุดเริ่มต้นของ Audit Trail

แต่ยังขาด:

```text
Agent identity
Request ID
Tool call ID
Old state
New state
Model
Prompt
Approval record
Success/failure status
```

---

# 32. Concurrency Risk

Account Data ทั้งก้อนถูก:

```text
Read JSON
→ Modify in memory
→ Write JSON back
```

ถ้า Workers สองตัวทำ Trade พร้อมกัน:

```text
Worker A reads balance 10,000
Worker B reads balance 10,000
Worker A saves
Worker B saves
```

Update ของ Worker A อาจถูกเขียนทับ

เรียกว่า:

```text
Lost update
```

Production ต้องมี:

```text
Database transaction
Version column
Optimistic locking
Atomic SQL updates
```

MCP ไม่ได้แก้ Concurrency Problem ให้ Domain Layer

---

# 33. Resource Side Effect

Resource:

```python
@mcp.resource(
    "accounts://accounts_server/{name}"
)
async def read_account_resource(name):
    return Account.get(name).report()
```

แต่ `report()` เพิ่ม Portfolio Snapshot และ Save Account.

ดังนั้นการ “อ่าน Resource” มี Side Effect

นี่ขัดกับ Mental Model ทั่วไปที่ Resource ควรคล้าย Read Operation

ควรแยก:

```python
def snapshot_report():
    # mutates state
```

ออกจาก:

```python
def read_report():
    # pure read
```

---

# 34. Exercise — Push Notification MCP Server

Notebook ให้สร้าง MCP Server ที่ส่ง Push Notification และมี Solution ใน:

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
class PushModelArgs(
    BaseModel
):
    message: str = Field(
        description=(
            "A brief message to push"
        )
    )
```

Tool:

```python
@mcp.tool()
def push(
    args: PushModelArgs
):
    ...
```

---

# 35. Pydantic Model as Tool Input

แทนที่จะประกาศ:

```python
def push(
    message: str
):
```

Solution ใช้:

```python
def push(
    args: PushModelArgs
):
```

ข้อดี:

```text
รวม Arguments เป็น Domain Input Model
เพิ่ม Field descriptions
เพิ่ม Validation ได้ง่าย
Reuse Model ได้
```

สามารถเพิ่ม:

```python
class PushModelArgs(BaseModel):
    message: str = Field(
        min_length=1,
        max_length=200,
    )
    priority: int = Field(
        ge=-2,
        le=2,
    )
```

---

# 36. Push Notification Flow

```text
Agent
→ push MCP tool
→ requests.post()
→ Pushover API
→ Mobile notification
```

Credentials:

```env
PUSHOVER_USER=...
PUSHOVER_TOKEN=...
```

Server โหลดค่าจาก Environment และส่ง Request ไปยัง Pushover Messages API.

---

# 37. ปัญหาใน Push Server Solution

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
Timeout
raise_for_status()
Response validation
Credential validation
Retry policy
Idempotency
```

จึงอาจคืน:

```text
Push notification sent
```

แม้ HTTP Request ล้มเหลว

Safer version:

```python
response = requests.post(
    pushover_url,
    data=payload,
    timeout=10,
)

response.raise_for_status()

return {
    "ok": True,
    "status_code": response.status_code,
}
```

---

# 38. Push Tool เป็น Side Effect

Push Notification ส่งผลออกนอกระบบทันที

ความเสี่ยง:

```text
Spam
ข้อมูลลับใน Notification
ส่งผิดคน
Repeated tool calls
Prompt injection
Agent loop ส่งซ้ำ
```

ควรมี:

```text
Message length limit
Rate limit
Recipient fixed by server
Human approval
Idempotency key
Audit log
```

อย่าให้ Model ส่ง:

```text
user token
recipient ID
API endpoint
```

เป็น Free-text Arguments

ข้อมูล Authority ควรถูกกำหนดฝั่ง Server

---

# 39. การรัน Push Server

โครงสร้างท้ายไฟล์:

```python
if __name__ == "__main__":
    mcp.run(
        transport="stdio"
    )
```

Client Parameters ควรเป็น:

```python
push_params = {
    "command": "uv",
    "args": [
        "run",
        "-m",
        "backend.push_server",
    ],
}
```

แล้วตรวจ Tools:

```python
async with MCPServerStdio(
    params=push_params,
    client_session_timeout_seconds=30,
) as server:
    tools = await server.list_tools()
```

---

# 40. Suggested Push Agent

```python
instructions = """
You can send a brief push notification.
Only send one notification when explicitly requested.
Never include passwords, API keys, or private data.
"""

request = (
    "Send me a brief notification saying "
    "that Week 6 Lab 2 is complete."
)

async with MCPServerStdio(
    params=push_params,
    client_session_timeout_seconds=30,
) as server:

    agent = Agent(
        name="notifier",
        instructions=instructions,
        model="gpt-5.4-mini",
        mcp_servers=[server],
    )

    result = await Runner.run(
        agent,
        request,
        max_turns=5,
    )
```

ควรจำกัด `max_turns` เพราะ Tool มี External Side Effect

---

# 41. MCP Server Development Workflow

ลำดับที่ควรใช้:

```text
1. Implement domain function

2. Test domain function directly

3. Create FastMCP server

4. Expose one read-only tool

5. Run server with stdio

6. Call list_tools()

7. Inspect generated schema

8. Test simple MCP client

9. Connect one Agent

10. Add write tools carefully

11. Add authentication and approval

12. Add logs and deterministic tests
```

Lab ใช้ Pattern นี้โดยเริ่มจาก `Account` ก่อน `accounts_server`

---

# 42. Testing Layers

## Unit Test

```text
Account.buy_shares()
Account.get()
Account.report()
```

## MCP Contract Test

```text
Server starts
list_tools() returns expected tools
Tool schemas match expectations
```

## Integration Test

```text
MCP Client calls get_balance
Result matches Account.get().balance
```

## Agent Test

```text
Agent selects correct tool
Agent does not call mutation tool
Final response is accurate
```

## Security Test

```text
Unauthorized account name
Negative quantity
Repeated buy call
Prompt injection
Missing credentials
```

---

# 43. Tool Design Principles

MCP Tool ที่ดีควร:

```text
มีชื่อชัด
Description บอก Use Case
Arguments มี Type และ Constraints
ผลลัพธ์เป็น Structured Data
Error Codes สม่ำเสมอ
Side Effects ระบุชัด
Operation มีขอบเขตเล็ก
ไม่ให้ Model ส่ง Authority Data
```

ตัวอย่าง Tool ที่กว้างเกินไป:

```text
manage_account(action, data)
```

ดีกว่าคือ:

```text
get_balance
get_holdings
place_buy_order
place_sell_order
```

เพราะ Tool Selection และ Audit ชัดกว่า

---

# 44. Tool Granularity

Tools ใน Lab แบ่งตาม Business Operation

ข้อดี:

```text
Model เข้าใจง่าย
Schema เล็ก
Logs อ่านง่าย
Policy ต่อ Tool ได้
```

แต่ `buy_shares` ทำหลายอย่าง:

```text
Price lookup
Validation
Execution
Persistence
Audit
Report generation
```

Production อาจแยก:

```text
preview_order
approve_order
execute_order
get_order_status
```

เพื่อเพิ่ม Human-in-the-loop และลด Irreversible Action

---

# 45. Read–Plan–Approve–Execute Pattern

ระบบลงทุนที่ปลอดภัยกว่า:

```text
User request
→ Agent analyzes
→ preview_trade()
→ Show expected cost
→ Human approves
→ execute_trade()
→ Return receipt
```

ไม่ควร:

```text
User mentions a stock
→ Agent directly calls buy_shares
```

โดยเฉพาะเมื่อ Agent อ่านข้อมูลจาก Web หรือ External Sources

---

# 46. MCP Does Not Define Business Policy

MCP กำหนด:

```text
How to describe a tool
How to call it
How to return a result
```

MCP ไม่ได้กำหนด:

```text
ใครมีสิทธิ์ซื้อหุ้น
ซื้อได้สูงสุดเท่าไร
ต้องอนุมัติหรือไม่
ต้องบันทึก Audit อะไร
ตลาดต้องเปิดหรือไม่
```

สิ่งเหล่านี้เป็นความรับผิดชอบของ Domain และ Application Policy

---

# 47. Common Misconceptions

### “ใส่ `@mcp.tool()` แล้ว Function ปลอดภัย”

ไม่จริง Decorator ทำให้ Function ถูกเรียกผ่าน MCP ได้ แต่ไม่ได้เพิ่ม Authorization หรือ Business Validation

### “Type Hint ป้องกันค่าผิดทั้งหมด”

Type Hint ตรวจรูปแบบ เช่น Integer แต่ไม่ป้องกัน `quantity=-100`

### “Resource เป็น Read-only เสมอ”

ไม่จำเป็น Code ของ Resource ยังสามารถมี Side Effect ได้ และ Resource ใน Lab เรียก `report()` ที่บันทึก Snapshot

### “Agent ถามข้อมูลอย่างเดียว จึงไม่มีสิทธิ์เขียน”

ไม่จริง ถ้า Server เปิด Write Tools Agent ยังเห็น Tools เหล่านั้น

### “Server อยู่ Local จึงปลอดภัย”

Local Server ยังเข้าถึง Database, Credentials และ External APIs ได้

### “ส่ง Notification สำเร็จเพราะ Tool คืน Success”

ไม่จริง ต้องตรวจ HTTP Status และ Response

---

# 48. Risks Identified

## 48.1 Missing Authentication

ชื่อ Account ถูกใช้แทน Identity

## 48.2 Excessive Authority

Read-only Agent ได้รับ Trading Tools

## 48.3 Negative Quantity

Domain ไม่บังคับ Positive Quantity

## 48.4 Return-type Mismatch

Trade Tools ประกาศ `float` แต่คืน `str`

## 48.5 Resource Side Effect

การอ่าน Account Resource บันทึก Portfolio Snapshot

## 48.6 Lost Updates

Account JSON ถูกอ่านและเขียนกลับทั้งก้อน

## 48.7 Simulated-price Ambiguity

Agent อาจไม่รู้ว่าราคาเป็นข้อมูลจำลอง

## 48.8 Weak Error Contract

ใช้ Exception Message แทน Structured Error

## 48.9 Push False Success

Push Server ไม่ตรวจ HTTP Response

## 48.10 Notification Spam

Agent อาจเรียก Push Tool หลายครั้ง

## 48.11 Hidden stdio Errors

Windows Workaround ส่ง `stderr` ไป DEVNULL

## 48.12 Unbounded Side Effects

ไม่มี Approval Gate สำหรับ Trade หรือ Notification

---

# 49. Production Improvements

```text
Separate read and write servers
Authenticate callers
Resolve account from trusted context
Use positive-number constraints
Return structured result models
Add preview/approval/execute workflow
Add idempotency keys
Capture server logs
Use database transactions
Add optimistic locking
Separate pure reads from state mutation
Validate external API responses
Add rate limits
Add per-tool authorization
Store request and tool-call IDs
```

---

# 50. Exercises

## Exercise 1 — Inspect Tools

```python
for tool in mcp_tools:
    print(tool.name)
    print(tool.description)
    print(tool.inputSchema)
```

ตรวจว่า Schema ตรงกับ Domain Contract หรือไม่

---

## Exercise 2 — Fix Return Types

แก้:

```python
buy_shares(...) -> float
sell_shares(...) -> float
```

ให้ตรงกับ Runtime Result หรือสร้าง `TradeResult`

---

## Exercise 3 — Positive Quantity

ใช้:

```python
from pydantic import PositiveInt
```

หรือ Validate ใน Domain Method

ทดสอบ:

```text
quantity = 0
quantity = -5
```

---

## Exercise 4 — Read-only Server

สร้าง `accounts_read_server.py` ที่เปิดเฉพาะ:

```text
get_balance
get_holdings
get_strategy
```

แล้วตรวจ Trace ว่า Agent ไม่มี Trading Capability

---

## Exercise 5 — Push Server

สร้าง Tool ส่ง Notification แล้วเพิ่ม:

```text
Timeout
Status checking
Message length
Rate limiting
Audit log
```

---

## Exercise 6 — Human Approval

สร้างสอง Tools:

```text
preview_buy
execute_buy
```

ให้ `execute_buy` ต้องรับ Approval Token ที่ Application สร้าง ไม่ใช่ Model สร้างเอง

---

# Checklist

```text
□ Account Domain ทำงานโดยไม่ผ่าน MCP
□ Account reset เป็น $10,000
□ Buy transaction ถูกบันทึก
□ accounts_server รันผ่าน uv run -m
□ list_tools() แสดง Tools ที่คาดหวัง
□ get_balance schema ถูกต้อง
□ get_holdings schema ถูกต้อง
□ Agent เรียก Read Tools ถูกตัว
□ Trace ไม่มี Write Tool Call ที่ไม่จำเป็น
□ เข้าใจ Tool กับ Resource ต่างกัน
□ พบ Return-type mismatch ของ Trade Tools
□ เข้าใจ Authentication Risk
□ Push Server เปิดและค้น Tool ได้
□ Push HTTP Response ถูกตรวจ
```

---

# แก่นของ Week 6 — Lab 2

```text
Account
= Domain capability

FastMCP
= MCP server framework

@mcp.tool()
= Action exposed through MCP

@mcp.resource()
= Addressable information

mcp.run(stdio)
= Local server transport

MCPServerStdio
= Client-side process connection

OpenAI Agent
= Tool-selection layer

SQLite
= Persistent domain state

Trace
= Execution evidence
```

บทเรียนสำคัญที่สุดคือ:

> **การสร้าง MCP Server ไม่ใช่เพียงนำ Decorator ไปวางบน Function แต่คือการออกแบบ Public Capability Contract ระหว่าง Agent กับ Business System**

อีกบทเรียนคือ:

> **MCP ช่วยแยก Agent ออกจาก Implementation แต่ไม่ได้จัดการ Authentication, Authorization, Validation, Concurrency หรือ Approval ให้ สิ่งเหล่านี้ยังต้องถูกบังคับใน Domain และ Application Layer**

และแก่นเชิง Production คือ:

> **เริ่มจาก Read-only Tools ที่เล็กและตรวจสอบง่าย จากนั้นค่อยเพิ่ม Side-effect Tools พร้อม Identity, Approval, Structured Errors, Audit Logs และ Deterministic Business Rules**
