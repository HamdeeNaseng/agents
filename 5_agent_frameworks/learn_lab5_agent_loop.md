# Week 5 — Lab 5: Project 7 — The Agent Loop

ตำแหน่ง Lab:

```text
5_agent_frameworks/
└── 5_agent_loop/
    ├── lab.md
    ├── agent_loop.py
    ├── orchestrator.py
    ├── catalog.py
    ├── board.py
    ├── live_board.py
    ├── prompts.py
    ├── qa_agent.py
    ├── css_agent.py
    ├── config.py
    └── site/
```

Lab 5 ไม่ได้เพิ่ม Agent Framework ใหม่ แต่ประกอบ Workers จากหลาย Framework ให้ทำงานเป็นทีม:

```text
AWS Strands
Pydantic AI
Microsoft Agent Framework
Agno
Mastra
```

ส่วน **Google ADK** เปลี่ยนบทบาทจาก Worker ใน Lab 1 มาเป็น Orchestrator ที่เลือกงาน เรียก Workers ตรวจผลงาน และสั่งแก้ไข จึงเกิดระบบที่มีห้า Builder Frameworks ทำงานภายใต้ Coordinator Framework ตัวที่หก.

---

## Learning Objectives

หลังจบ Lab นี้ควรอธิบายได้ว่า:

1. Inner Agent Loop และ Outer Agent Loop ต่างกันอย่างไร
2. ADK Orchestrator ควบคุม Workers ต่าง Framework อย่างไร
3. Shared SQLite Board ทำหน้าที่เป็น Coordination Substrate อย่างไร
4. Worker Catalog ใช้ค้นหาและ Launch Workers อย่างไร
5. Python และ TypeScript Workers ใช้ Subprocess Contract เดียวกันอย่างไร
6. Workers หลายตัวถูก Launch พร้อมกันอย่างไร
7. `Team` Object เก็บ Working State อะไรบ้าง
8. Orchestrator Tools แบ่งงานระหว่าง LLM Decisions กับ Deterministic Mechanics อย่างไร
9. Live Board แสดง Parallel Work อย่างไร
10. Playwright MCP และ QA Agent ตรวจเกมจริงอย่างไร
11. One-fix Policy ป้องกัน Infinite Repair Loop อย่างไร
12. Timeouts และ Model-call Budgets อยู่ตรงไหน
13. Deterministic Fallbacks ทำให้ Site ยังถูกสร้างได้อย่างไร
14. `is_built()` ตรวจอะไรและยังตรวจอะไรไม่ได้
15. Failure Modes ใดยังคงเกิดได้แม้มี Orchestrator และ QA Agent
16. Agent Team แตกต่างจาก Multi-agent Chat อย่างไร

---

# 1. สิ่งที่สร้างใน Lab นี้

เป้าหมายคือสร้าง **Language Learning Arcade**

```text
site/
├── common.css
├── index.html
├── strands/
│   ├── game.html
│   ├── game.css
│   └── game.js
├── pydantic/
├── maf/
├── agno/
└── mastra/
```

แต่ละ Worker ได้รับ Learning Objective คนละหัวข้อ เช่น:

```text
Greetings
Numbers
Common verbs
Food and drink
Travel phrases
```

Worker เป็นผู้คิดรูปแบบเกมเอง แต่ทุกเกมต้อง:

```text
ใช้ Vanilla HTML/CSS/JavaScript
มีความยากเพิ่มขึ้น
มีคะแนน
มี Feedback ถูก/ผิด
ทำงานผ่าน file://
สร้าง exactly 3 files
เชื่อม common.css
มีลิงก์กลับ index.html
```

ข้อกำหนดเหล่านี้ถูกใส่ใน `GAME_TASK` ซึ่งไม่ขึ้นกับ Framework ตัวใดตัวหนึ่ง.

---

# 2. Inner Loop กับ Outer Loop

ตลอด Labs 1–4 แต่ละ Worker มี Inner Loop:

```text
Read board
→ Plan steps
→ Use file tools
→ Complete steps
→ Complete goal
```

Lab 5 เพิ่ม Outer Loop:

```text
Design curriculum
→ Assign objectives
→ Launch workers
→ Wait for team
→ Test games
→ Repair broken games
→ Build homepage
```

ภาพรวม:

```text
ADK Orchestrator — Outer Loop
│
├── Strands Worker ────── Inner Loop
├── Pydantic Worker ───── Inner Loop
├── MAF Worker ────────── Inner Loop
├── Agno Worker ───────── Inner Loop
└── Mastra Worker ─────── Inner Loop
```

นี่ไม่ใช่ Agents คุยกันโดยตรง แต่เป็น Hierarchical Coordination:

```text
Orchestrator
→ สร้าง Task บน Board
→ เปิด Worker Process
→ Worker อ่าน Task และสร้าง Artifact
→ Orchestrator ตรวจ Artifact
```

---

# 3. Agent vs Mechanics

แนวคิดสำคัญของ `orchestrator.py` คือ:

> Agent owns the decisions; tools own the mechanics.

## Orchestrator Agent ตัดสินใจ

```text
ให้ Worker ไหนสอนอะไร
ควร Launch ใครก่อน
เกมใดควรถูกทดสอบ
เกมใดต้องแก้
เมื่อใดควรสร้างหน้า Hub
```

## Deterministic Tools ลงมือทำ

```text
เขียน CSS
สร้าง Goal ใน SQLite
เปิด Subprocess
รอ Process
ฆ่า Process ที่ Timeout
เปิด Browser
ตรวจ Required Files
เขียน index.html
```

ดังนั้น LLM ไม่ได้รับสิทธิ์เรียก `subprocess.Popen()` เองโดยตรง แต่เรียก Tool ที่ Application ควบคุม Implementation ไว้.

Mental model:

```text
LLM
= Flexible decision maker

Tools
= Controlled operating mechanisms
```

---

# 4. Entry Point — `agent_loop.py`

หน้าที่หลักของไฟล์นี้คือ:

```text
อ่าน Command-line Arguments
โหลด API Keys
ตั้ง Shared Board Path
ตั้ง Worker Model
ค้น Workers ที่มี
เริ่ม Orchestrator
ตรวจผลสุดท้าย
เปิด index.html
```

ส่วนสำคัญ:

```python
SITE = HERE / "site"
BOARD_PATH = SITE / "board.sqlite"

os.environ["BOARD_PATH"] = str(BOARD_PATH)
os.environ["WORKER_MODEL"] = config.WORKER_MODEL
```

Environment Variables ต้องถูกตั้งก่อน Import `board` และ Worker-related Modules เพราะ Board Path ถูกอ่านตอน Import.

---

# 5. Command-line Arguments

รันปกติ:

```powershell
uv run agent_loop.py
```

เปลี่ยนภาษา:

```powershell
uv run agent_loop.py --language French
```

ตัด Workers บางตัวออก:

```powershell
uv run agent_loop.py --skip agno mastra
```

ดูทีมโดยไม่เรียก Model:

```powershell
uv run agent_loop.py --dry-run
```

สร้าง Site แต่ไม่เปิด Browser:

```powershell
uv run agent_loop.py --no-open
```

หากไม่มี Worker Files เลย Program จะหยุดพร้อมข้อความว่าอย่างน้อยต้องมี Worker หนึ่งตัว.

---

# 6. Setup

ต้องมี:

```text
OPENAI_API_KEY
GOOGLE_API_KEY
Node.js 24
Google Chrome
uv
npx
```

ไม่มี Python Package ใหม่สำหรับ Day 5 เพราะใช้ Root `uv` Project เดิม ส่วน Node 24 ใช้สำหรับ Mastra Worker และ Chrome ใช้โดย Playwright MCP ในขั้น QA.

ตรวจ Environment:

```powershell
uv --version
node --version
npx --version
```

ก่อนรันแบบเสียค่าใช้จ่าย ควรเริ่มด้วย:

```powershell
uv run agent_loop.py --dry-run
```

---

# 7. Worker Catalog

`catalog.py` เป็น Worker Manifest:

```python
WORKERS = [
    {
        "key": "strands",
        "name": "AWS Strands",
        "file": "../2_strands_pydantic/strands_worker.py",
        "runner": "python",
    },
    ...
    {
        "key": "mastra",
        "name": "Mastra",
        "file": "../4_mastra/worker.ts",
        "runner": "node",
    },
]
```

แต่ละ Record กำหนด:

```text
key
→ Framework identifier

name
→ Display name

colour
→ สีบน Live Board

file
→ Worker file

runner
→ Python หรือ Node

slug
→ Output folder
```

---

# 8. Worker Discovery

```python
workers = catalog.discover(
    skip=tuple(args.skip)
)
```

Discovery ใช้หลักง่าย ๆ:

```text
Worker file exists
และ
ไม่อยู่ใน --skip
→ ใช้ Worker ตัวนั้น
```

ข้อดี:

```text
ไม่ต้องมี Plugin Registry ซับซ้อน
ข้าม Lab บางวันได้
ทีมมีตั้งแต่ 1–5 Workers
```

ข้อจำกัด:

```text
File exists
≠ Dependencies พร้อม
≠ Worker รันได้
≠ API Key ถูกต้อง
```

Discovery ตรวจเพียง Presence ไม่ได้ตรวจ Health.

---

# 9. Cross-language Worker Contract

Python Worker ถูก Launch แบบ:

```text
uv run <worker.py> <taskId> <boardPath>
```

Mastra Worker ถูก Launch แบบ:

```text
npx tsx <worker.ts> <taskId> <boardPath>
```

Code:

```python
if worker["runner"] == "python":
    return [
        "uv",
        "run",
        path,
        str(task_id),
        str(board_path),
    ]

return [
    "npx",
    "tsx",
    path,
    str(task_id),
    str(board_path),
]
```

Framework และภาษาต่างกัน แต่ Orchestrator ต้องรู้เพียง:

```text
Worker executable
Task ID
Shared Board Path
```

นี่คือ Interface Boundary ที่ทำให้ระบบ Cross-framework ทำงานร่วมกันได้.

---

# 10. Shared Board

Day 5 ใช้ SQLite File เดียว:

```text
site/board.sqlite
```

Workers ทุกตัวอ่านและเขียน:

```text
Goals
Steps
Statuses
Results
```

Board ใช้:

```sql
PRAGMA journal_mode=WAL
PRAGMA busy_timeout=5000
```

เพื่อช่วยให้หลาย Processes เขียนและอ่านพร้อมกันได้ดีขึ้น.

Mental model:

```text
Shared Board
= Coordination channel

Site folders
= Artifact channel

Subprocess exit
= Execution completion signal
```

Workers ไม่จำเป็นต้อง Import Framework ของกันและกัน

---

# 11. Board ไม่ใช่ Message Bus เต็มรูปแบบ

Board มีเพียง Table:

```sql
todos (
    id,
    parent_id,
    task,
    status,
    result
)
```

มันไม่มี:

```text
Lease expiration
Worker identity
Attempt number
Heartbeat
Failure status
Error details
Version field
Event log
```

จึงเป็น Shared Work State ที่เรียบง่าย ไม่ใช่ Production Job Queue

---

# 12. `Team` Object

Orchestrator Tools ใช้ `Team` เป็น Internal Working State:

```python
self.language
self.workers
self.site_dir
self.board_path
self.by_key
self.by_slug
self.objectives
self.registry
self.pending
self.fixed
```

หน้าที่สำคัญ:

```text
objectives
→ แต่ละ Worker ได้หัวข้ออะไร

registry
→ Goal ID เป็นของ Worker ไหน

pending
→ Subprocesses ที่เปิดแล้วแต่ยังไม่ได้ Wait

fixed
→ เกมใดใช้สิทธิ์แก้หนึ่งครั้งแล้ว
```

Agent ไม่เห็น `Team` Object โดยตรง แต่ Tools เป็น Closures ที่อ่านและเปลี่ยน State นี้.

---

# 13. Orchestrator Tools

Orchestrator มี Tools หกตัว:

```text
author_style
launch_worker
wait_for_team
test_game
relaunch_worker
build_hub
```

```python
return [
    author_style,
    launch_worker,
    wait_for_team,
    test_game,
    relaunch_worker,
    build_hub,
]
```

Tools ถูกสร้างเป็น Closures ภายใน `make_tools(team)` จึงเข้าถึง Team State ได้โดยไม่ต้องส่ง State ทั้งหมดให้ Model.

---

# 14. `author_style`

```text
Orchestrator
→ author_style()
→ CSS Agent
→ common.css
```

Art Director เป็น ADK Agent แบบสั้นที่เขียน Shared CSS ให้ทุกเกมใช้

หาก Model Call ล้มเหลวหรือคืนเนื้อหาไม่เพียงพอ ระบบจะเขียน Built-in CSS Template แทน เพื่อไม่ให้ Site ไม่มี Style.

Pattern:

```text
Creative LLM output
+
Deterministic fallback
```

นี่เป็น Production Pattern ที่ดีกว่าการพึ่ง LLM เพียงอย่างเดียว

---

# 15. `launch_worker`

Signature:

```python
launch_worker(
    framework: str,
    objective: str,
)
```

ทำงาน:

```text
หา Worker จาก key
→ ป้องกัน Launch ซ้ำ
→ สร้าง Output folder
→ สร้าง GAME_TASK
→ เพิ่ม Goal ลง Shared Board
→ Register Goal กับ Worker
→ เปิด Worker Subprocess
→ Return ทันที
```

มันไม่รอ Worker จบ จึงทำให้ Orchestrator สามารถ Launch Workers ทุกตัวก่อนเข้าสู่ `wait_for_team()`.

---

# 16. Parallelism

Orchestrator Prompt สั่งว่า:

```text
Launch ทุก Builder ก่อน
แล้วจึง call wait_for_team
```

ดังนั้น:

```text
launch Strands
launch Pydantic
launch MAF
launch Agno
launch Mastra
        ↓
wait_for_team
```

Workers ทำงานพร้อมกันเป็น OS Processes

นี่คือ Process-level Parallelism ไม่ใช่ Async Tool Calls ภายใน Process เดียว.

---

# 17. `_launch()`

Worker ถูกเปิดด้วย:

```python
subprocess.Popen(...)
```

`stdout` และ `stderr` ถูกส่งไป `DEVNULL` เพื่อไม่ให้ Framework Banners ทำลาย Live Board

```text
Worker communication
ไม่ได้ผ่าน stdout

แต่ผ่าน:
SQLite Board
และ Site Files
```

บน POSIX Worker ถูกเปิดใน Process Group ใหม่ ส่วน Windows ใช้ `taskkill /T` เมื่อต้องหยุด เพื่อฆ่าทั้ง Worker, `uv`, `npx` และ MCP Child Processes.

---

# 18. `wait_for_team`

Tool นี้:

```text
นำ Pending Processes ออกมา
→ แสดง Live Board
→ Poll Processes
→ Refresh Board
→ Kill Workers ที่ Timeout
→ Return Team Status
```

Polling Interval:

```text
0.15 seconds
```

Live Board Refresh:

```text
8 times per second
```

Default Worker Timeout:

```text
300 seconds
```

เมื่อ Worker เกินเวลา ระบบหยุด Process Tree เพื่อไม่ให้ Run ค้างตลอดไป.

---

# 19. Live Board

`live_board.py` อ่าน SQLite ซ้ำและ Render:

```text
Goal ของแต่ละ Worker
Steps ใต้ Goal
Status
Result
Framework colour
```

Display Rules:

```text
done
→ dim + strike

in_progress
→ bold

pending
→ normal
```

สีมาจาก Orchestrator Registry ไม่ได้ถูกเก็บใน Board Schema จึงไม่ต้องแก้ Workers หรือ Database เพื่อรองรับ UI.

นี่เป็นแนวคิดที่ดี:

```text
Operational metadata
ไม่จำเป็นต้องปะปน
กับ Business state
```

---

# 20. Artifact Contract

เกมหนึ่งเกมต้องมี:

```text
game.html
game.css
game.js
```

`is_built()` ตรวจว่าไฟล์ทั้งสาม:

```text
มีอยู่
และ
มีขนาดมากกว่า 0
```

```python
GAME_FILES = (
    "game.html",
    "game.css",
    "game.js",
)
```

ข้อดี:

```text
เป็น Deterministic Minimum Gate
```

ข้อจำกัด:

```text
ไฟล์มีอยู่
≠ HTML ถูกต้อง
≠ JavaScript รันได้
≠ เกมโต้ตอบได้
≠ เนื้อหาสอนภาษาถูกต้อง
```

จึงต้องมี Browser QA เพิ่ม

---

# 21. `test_game`

Flow:

```text
ตรวจ Required Files
→ สร้าง file:// URI
→ เปิด Fresh QA Agent
→ ให้ Browser เล่นเกม
→ รับ Verdict
```

QA Timeout ระดับ Orchestrator:

```text
150 seconds
```

หาก Browser หรือ QA ล้มเหลว ระบบไม่ถือว่าเกม Broken โดยอัตโนมัติ แต่รายงานว่าไม่สามารถตรวจได้และปล่อยเกมไว้เป็น Built.

---

# 22. QA Agent

QA Agent เป็น Google ADK Agent แยกใหม่สำหรับแต่ละเกม:

```text
Fresh Agent
Fresh Session
Fresh Browser MCP
```

Tools:

```text
Playwright MCP Browser
report_game
```

`report_game` คืน:

```json
{
  "works": true,
  "note": "The game loads and responds."
}
```

Fresh Agent ต่อเกมช่วยให้ Browser Context สั้นและไม่ให้ Context จากเกมก่อนหน้าปะปนกับเกมถัดไป.

---

# 23. Playwright MCP

Browser Server เริ่มด้วยแนวคิด:

```text
npx @playwright/mcp
--browser chrome
--isolated
--allow-unrestricted-file-access
```

ต้องอนุญาต File Access เพราะเกมเปิดผ่าน:

```text
file://
```

หากตั้ง:

```env
QA_HEADLESS=1
```

จะเพิ่ม `--headless`

QA Agent ถูกสั่งให้:

```text
เปิดเกม
ดูหน้าจอ
คลิกประมาณห้าครั้ง
ตรวจ Console
เรียก report_game
```

---

# 24. QA Budgets

QA Agent มี:

```text
Maximum LLM calls: 25
Browser close timeout: 10 seconds
Outer QA timeout: 150 seconds
```

หากใช้ครบ 25 Calls โดยยังไม่เรียก `report_game` Code จะถือว่าเกมทำงาน เพราะ Agent ยังสามารถโต้ตอบกับเกมตลอด Budget ได้.

นี่เป็น Heuristic:

```text
Agent kept interacting
→ likely responsive
```

แต่ไม่ใช่ Proof เพราะอาจเกิด False Positive ได้

---

# 25. `relaunch_worker`

หาก `test_game()` คืนว่า Broken:

```text
Orchestrator
→ relaunch_worker(framework, problem)
→ New Fix Goal on Board
→ Same Worker Process
→ Same game folder
```

Fix Prompt ระบุ:

```text
เปิด game.html, game.css, game.js
หาสาเหตุจาก Symptom
แก้ให้ใช้ file:// ได้
รักษา common.css และ back link
อ่านไฟล์กลับเพื่อตรวจ
```

แต่ละ Game ได้ Fix Attempt สูงสุดหนึ่งครั้ง เพื่อป้องกัน Repair Loop ไม่จบ.

---

# 26. Bounded Repair Loop

```text
Build
→ Test
→ Broken?
    ├── No → Keep
    └── Yes
         → Fix once
         → Wait
         → Test again
         → Stop
```

ข้อดี:

```text
จำกัด Cost
จำกัดเวลา
ป้องกัน Infinite self-repair
```

ข้อจำกัด:

```text
เกมอาจยัง Broken หลังหนึ่งรอบ
```

Lab เลือกให้ Game ที่ยัง Broken คงปรากฏอยู่ แทนการวนแก้ตลอดไป

---

# 27. `build_hub`

หลัง Games ถูก Build และ Test:

```text
อ่าน Built Games
→ อ่าน <title> จาก game.html
→ สร้างรายการ Links
→ ADK Art Director เขียน index.html
```

หาก Model Call ล้มเหลว ระบบเขียน Template Hub แบบ Deterministic แทน.

---

# 28. Orchestrator Agent

```python
LlmAgent(
    name="orchestrator",
    model=config.ORCHESTRATOR_MODEL,
    instruction=instruction,
    tools=make_tools(team),
)
```

Runner:

```python
InMemoryRunner(
    agent=agent,
    app_name="agent_loop",
)
```

Orchestrator Prompt กำหนดลำดับเชิงเป้าหมาย แต่ Model ยังเป็นผู้ตัดสินใจเรียก Tools จริงระหว่าง Run.

---

# 29. Orchestrator Budget

```python
RunConfig(
    max_llm_calls=80
)
```

Default Outer-loop Budget:

```text
80 Model Calls
```

หากใช้ครบ Budget:

```text
หยุด Orchestrator
เก็บ Artifacts ที่สร้างแล้ว
เรียก Safety Net
สร้าง Hub จากสิ่งที่มี
```

นี่เป็น Graceful Degradation แทนการทิ้งผลทั้งหมด.

---

# 30. Error Handling

หาก Orchestrator เจอ:

```text
Model API error
Network error
Rate limit
Unexpected exception
```

Code จะ:

```text
รายงาน Error Type
หยุด Agent
รักษา Games ที่สร้างแล้ว
สร้าง Fallback Site
```

แต่ไม่แสดง Traceback เต็มโดยค่าเริ่มต้น เพื่อให้ Demo จบอย่างสะอาด.

ข้อเสียคือ Root Cause อาจตรวจยากขึ้นหากไม่มี Structured Logs เพิ่ม

---

# 31. Deterministic Safety Net

หลัง Agent Run:

```python
_ensure_site(team)
```

ตรวจ:

```text
common.css มีหรือไม่
index.html มีหรือไม่
```

ถ้าไม่มี จะเขียน Built-in Templates

ดังนั้นแม้ Orchestrator:

```text
หยุดก่อนเวลา
ใช้ Budget หมด
Model ล้มเหลว
ลืมเรียก build_hub
```

Site ยังมี Style และ Front Door.

---

# 32. Final Deterministic Check

`agent_loop.py` ตรวจทุก Worker Folder หลัง Run:

```text
ok
หรือ
INCOMPLETE
```

โดยเรียก:

```python
orchestrator.is_built(
    SITE / slug
)
```

นี่เป็น Safety Check ที่อยู่นอก Agent Loop จึงยังทำงานแม้ Orchestrator ตัดสินใจผิดหรือหยุดก่อนเวลา.

---

# 33. Model Configuration

Default ปัจจุบัน:

```python
ORCHESTRATOR_MODEL = "gemini-3.5-flash"
WORKER_MODEL = "gpt-5.5"
```

ตัวเลือกประหยัดที่ Comment ไว้:

```text
Orchestrator
→ gemini-3.1-flash-lite

Workers
→ gpt-5.4-mini
```

สามารถ Override ผ่าน Environment Variables:

```powershell
$env:ORCHESTRATOR_MODEL = "gemini-3.1-flash-lite"
$env:WORKER_MODEL = "gpt-5.4-mini"

uv run agent_loop.py
```

---

# 34. State Surfaces

Lab นี้มี State หลายชั้น:

```text
1. Orchestrator ADK Session
   → Messages และ Tool Loop

2. Team Object
   → Objectives, Processes, fixes

3. SQLite Board
   → Goals, Steps, status, results

4. Site Folder
   → HTML, CSS, JavaScript

5. Worker Processes
   → Framework-specific execution

6. MCP Processes
   → Filesystem และ Browser sessions

7. QA Verdict
   → works และ note
```

State เหล่านี้อาจไม่ตรงกัน

ตัวอย่าง:

```text
Worker process จบ
แต่ Goal ยัง in_progress

Goal เป็น done
แต่ Required Files ไม่มี

Files มีครบ
แต่ JavaScript พัง

QA ใช้งานไม่ได้
แต่ Game ถูกปล่อยเป็น built
```

---

# 35. จุดแข็งของ Architecture

## Framework Isolation

Worker แต่ละตัวอยู่คนละ Process

```text
Library conflicts ลดลง
Event loops แยกกัน
Framework crash ไม่ทำให้ Process อื่นพังทันที
```

## Language Isolation

Python และ TypeScript ใช้ Contract เดียวกัน

## Parallel Execution

Workers สร้าง Games พร้อมกัน

## Observable Shared State

Live Board แสดง Steps ที่ Agents สร้างเอง

## Bounded Autonomy

มี Timeouts, Call Budgets และ One-fix Policy

## Graceful Degradation

มี Templates และ Final Checks

---

# 36. Shared Weakness — Non-atomic Claim

Board ยังใช้:

```sql
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
```

ไม่มีเงื่อนไข:

```sql
AND status = 'pending'
```

Workers สองตัวจึงอาจ Claim Goal เดียวกันได้ หาก Orchestrator หรือ External Process Launch ซ้ำ.

Safer pattern:

```sql
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
  AND status = 'pending'
```

แล้วตรวจ:

```text
affected rows == 1
```

---

# 37. Shared Weakness — Exit Codes ไม่ถูกใช้

`wait_for_team()` รอจน Processes จบ แต่ Team Status ตรวจจาก Game Files เป็นหลัก

Code ไม่ได้ใช้ Worker Exit Code เพื่อแยกว่า:

```text
Worker สำเร็จ
Worker crash
Worker ถูกฆ่า
Dependency หาย
```

ผลคือ Worker อาจ Exit ด้วย Error แต่ระบบเห็นเพียง `NOT built`

Production ควรเก็บ:

```text
PID
Exit code
Start time
End time
Failure reason
Log path
```

---

# 38. Shared Weakness — Worker Logs ถูกทิ้ง

```python
stdout=subprocess.DEVNULL
stderr=subprocess.DEVNULL
```

ช่วยให้ Live Board ไม่เสียรูป แต่ซ่อน:

```text
Framework errors
MCP startup errors
Model errors
Dependency problems
Tracebacks
```

Production ควรเขียน Log แยกต่อ Worker:

```text
logs/strands.log
logs/pydantic.log
logs/maf.log
logs/agno.log
logs/mastra.log
```

Live Board ยังสะอาดได้โดยไม่ต้องทิ้ง Evidence

---

# 39. Shared Weakness — Stale Artifacts

ก่อน Run ระบบ Reset Board แต่ไม่ได้ล้างทุก Worker Folder ใน `site/`

ดังนั้นหาก Run ก่อนหน้ามี:

```text
site/agno/game.html
site/agno/game.css
site/agno/game.js
```

แล้ว Run ใหม่ Agno Worker ล้มเหลว `is_built()` อาจยังพบ Files เก่าและรายงานว่า Built

ปัญหาเดียวกันอาจเกิดกับ `common.css` และ `index.html` หาก Orchestrator หยุดก่อนเขียนไฟล์ใหม่

Production ควร:

```text
ล้าง site/ ก่อน Run
หรือ
สร้าง run-specific directory
หรือ
เก็บ run_id ใน Artifact metadata
```

ข้อสังเกตนี้เป็น Inference จากการที่ `run()` สร้าง Directory และ Reset Board แต่ไม่ได้ลบ Artifacts เดิม ขณะที่ `is_built()` ตรวจเพียง File existence และ size.

---

# 40. Shared Weakness — Weak Build Validation

```text
game.html exists
game.css exists
game.js exists
```

ไม่ได้ตรวจ:

```text
HTML parse สำเร็จ
CSS parse สำเร็จ
JavaScript syntax ถูก
Required links มี
No network calls
No external assets
Score system มี
Learning objective ถูกสอนจริง
```

ควรเพิ่ม Deterministic Validators ก่อน Browser QA

---

# 41. Shared Weakness — QA เป็น LLM

QA Agent อาจ:

```text
คลิกไม่ถูกจุด
อ่าน UI ผิด
พลาด Console Error
ตัดสินว่า Responsive ทั้งที่ Logic ผิด
ใช้ Budget หมดแล้วถูกถือว่า Works
```

ดังนั้น:

```text
LLM QA
= Useful exploratory test

ไม่ใช่
= Complete test suite
```

ควรเสริม:

```text
HTML validator
JavaScript syntax check
Required DOM selectors
Automated Playwright assertions
Console-error capture
```

---

# 42. Shared Weakness — Browser Security

QA Browser ใช้:

```text
--allow-unrestricted-file-access
```

เพื่อเปิด Local Games

แต่ Game JavaScript เป็น Code ที่ Worker สร้างขึ้น จึงควรถูกถือเป็น Untrusted Code

แม้ Browser ใช้ Isolated Session แต่ควรพิจารณา:

```text
Network blocking
Temporary browser profile
OS-level sandbox
Disposable working directory
No personal browser session
```

---

# 43. Shared Weakness — Unpinned Playwright MCP

QA ใช้:

```text
@playwright/mcp@latest
```

Version ใหม่อาจเปลี่ยน:

```text
Tool names
Schemas
Browser flags
Behavior
Startup requirements
```

Production ควร Pin Version ที่ผ่านการทดสอบ.

---

# 44. Shared Weakness — In-memory Orchestrator

Orchestrator ใช้:

```text
InMemoryRunner
```

ถ้า Process ปิดกลาง Run:

```text
ADK Session หาย
Team.pending หาย
Objectives mapping หาย
Fix-attempt records หาย
```

แม้ Board และ Files ยังอยู่ แต่ระบบไม่มี Durable Orchestration State สำหรับ Resume

Production ควรเก็บ:

```text
Run ID
Worker assignments
Process status
QA verdicts
Fix attempts
Workflow phase
```

ลง Durable Store

---

# 45. เปรียบเทียบกับ Week 4 Lab 5

## Sidekick

```text
One Worker
→ Evaluator
→ Retry
```

## Agent Loop

```text
Many Workers in parallel
→ Browser QA
→ Selective repair
→ Hub assembly
```

Sidekick เน้น Acceptance Loop ของ Task เดียว

Agent Loop เน้น Orchestration Loop ของ Team และ Artifacts หลายชุด

---

# 46. นี่คือ Multi-agent System แบบใด

ระบบนี้ไม่ใช่ Agents สนทนากันโดยตรง

```text
Strands ไม่คุยกับ Pydantic
Pydantic ไม่คุยกับ Agno
Agno ไม่คุยกับ Mastra
```

ทุกตัวสื่อสารผ่าน:

```text
Shared Board
Shared Files
Orchestrator
```

เรียกว่าแนวคิดแบบ:

```text
Blackboard architecture
หรือ
Shared-state coordination
```

ข้อดีคือ Workers ไม่ต้องรู้ Implementation ของกันและกัน

---

# 47. Agent Interoperability ที่แท้จริงใน Lab

Interoperability ไม่ได้มาจาก Frameworks ใช้ API เดียวกัน

แต่มาจาก Contract กลาง:

```text
Input contract
→ taskId + boardPath

Work-state contract
→ SQLite schema

Artifact contract
→ game.html + game.css + game.js

Completion contract
→ Board status

Execution contract
→ Subprocess exits
```

หลักสำคัญคือ:

> Shared contracts สำคัญกว่า Shared framework

---

# 48. Exercises

## Exercise 1 — Clean Run Directory

ก่อนเริ่ม Run:

```python
if SITE.exists():
    shutil.rmtree(SITE)

SITE.mkdir()
```

หรือสร้าง:

```text
runs/<run-id>/site/
```

เพื่อป้องกัน Stale Artifacts

---

## Exercise 2 — Worker Execution Records

เพิ่ม Table:

```sql
worker_runs (
    id,
    goal_id,
    framework,
    pid,
    started_at,
    finished_at,
    exit_code,
    error_log
)
```

แล้วให้ Live Board แสดง Failure ที่แท้จริง

---

## Exercise 3 — Atomic Claim

แก้ `claim_todo()` ให้คืน Boolean:

```python
def claim_todo(task_id: int) -> bool:
    ...
```

Worker ต้องหยุดทันทีหาก Claim ไม่สำเร็จ

---

## Exercise 4 — Deterministic Game Validator

ตรวจ:

```text
Three files exist
HTML links game.css
HTML links ../common.css
HTML links ../index.html
game.js parses
ไม่มี http:// หรือ https://
มี score element
มี interaction controls
```

---

## Exercise 5 — Preserve Worker Logs

แทน `DEVNULL` ด้วย Log File ต่อ Worker

จากนั้นให้ `wait_for_team()` แสดงบรรทัด Error ล่าสุดเมื่อ Process Exit ไม่เป็นศูนย์

---

## Exercise 6 — Durable Orchestrator State

เก็บ:

```text
phase
objectives
launched workers
completed workers
QA verdicts
fix attempts
```

ใน SQLite เพื่อให้ Restart แล้วทำต่อได้

---

# 49. Checklist

### Orchestrator ใช้ Framework ใด

```text
Google ADK
```

### Builders มีอะไรบ้าง

```text
Strands
Pydantic AI
Microsoft Agent Framework
Agno
Mastra
```

### Worker Discovery ใช้อะไร

```text
File existence
```

### Python Workers เปิดอย่างไร

```text
uv run worker.py taskId boardPath
```

### TypeScript Worker เปิดอย่างไร

```text
npx tsx worker.ts taskId boardPath
```

### Shared State อยู่ที่ไหน

```text
site/board.sqlite
```

### Artifacts อยู่ที่ไหน

```text
site/<framework>/
```

### Workers ทำงานพร้อมกันอย่างไร

```text
subprocess.Popen()
```

### Orchestrator Tools มีอะไร

```text
author_style
launch_worker
wait_for_team
test_game
relaunch_worker
build_hub
```

### Browser QA ใช้อะไร

```text
Playwright MCP
```

### Worker Timeout

```text
300 seconds
```

### QA Timeout

```text
150 seconds
```

### Orchestrator Call Budget

```text
80 calls
```

### QA Call Budget

```text
25 calls per game
```

### Fix Attempts

```text
หนึ่งครั้งต่อเกม
```

### Required Game Files

```text
game.html
game.css
game.js
```

---

# แก่นของ Week 5 — Lab 5

```text
Google ADK Orchestrator
= Outer decision loop

Framework Workers
= Inner execution loops

SQLite Board
= Shared coordination state

Site folders
= Shared artifact state

Subprocesses
= Framework and language isolation

Browser QA
= Environment-based evaluation

Repair task
= Bounded self-correction

Fallback templates
= Deterministic resilience
```

บทเรียนสำคัญที่สุดคือ:

> **Multi-agent interoperability ไม่จำเป็นต้องให้ทุก Agent ใช้ Framework หรือภาษาเดียวกัน หากทุกตัวปฏิบัติตาม Input, State, Artifact และ Completion Contracts เดียวกัน**

อีกบทเรียนคือ:

> **Orchestrator ที่ดีไม่ควรทำงานละเอียดเอง แต่ควรตัดสินใจว่าใครทำอะไร เมื่อใดควรตรวจ และเมื่อใดควรแก้ โดยมอบงานเชิงกลไกที่เปราะบางให้ Deterministic Tools**

และแก่นรวมของ Week 5 คือ:

> **เมื่อมองผ่านชื่อ Class และ Syntax ของแต่ละ Framework จะพบว่า Agent ทุกตัวมี Mental Model เดียวกัน: รับ Goal อ่าน State เลือก Tool สังเกตผล และทำซ้ำ แต่ระบบ Production จะน่าเชื่อถือได้ก็ต่อเมื่อ Application เพิ่ม Task Ownership, Budgets, Timeouts, Artifact Validation, Durable State และ Failure Evidence รอบ Loop นั้น**
