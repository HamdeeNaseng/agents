# Episodic Learning Artifact

## Week 5 — Lab 5: Project 7 — The Agent Loop

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**โฟลเดอร์:** `5_agent_frameworks/5_agent_loop`
**เอกสารหลัก:** `lab.md`
**ไฟล์สำคัญ:** `agent_loop.py`, `orchestrator.py`, `catalog.py`, `board.py`, `live_board.py`, `prompts.py`, `qa_agent.py`, `css_agent.py`, `config.py`
**หัวข้อหลัก:** Multi-framework Orchestration, Shared Blackboard, Parallel Workers, Browser QA, Bounded Repair Loop และ Deterministic Fallbacks
**สถานะ:** เรียนและสรุป Week 5 แล้ว

---

# 1. Context

Week 5 ใช้ Business Contract เดียวกันเพื่อสร้าง Worker ด้วยหลาย Framework:

```text
Lab 1
Google ADK

Lab 2
AWS Strands
Pydantic AI

Lab 3
Microsoft Agent Framework
Agno

Lab 4
Mastra
```

Workers ทุกตัวเรียนรู้ Pattern เดียวกัน:

```text
Read board
→ Plan steps
→ Use tools
→ Update board
→ Repeat
→ Complete goal
```

Lab 5 ไม่ได้สร้าง Worker Framework ใหม่ แต่ยกระดับ Workers ที่มีอยู่ให้กลายเป็น Team

```text
Google ADK
→ Orchestrator

Strands
Pydantic AI
MAF
Agno
Mastra
→ Builders
```

ผลลัพธ์คือ Language Learning Arcade ที่แต่ละ Framework รับผิดชอบสร้างเกมหนึ่งเกม

---

# 2. Learning Objectives

หลังจบ Lab 5 สามารถอธิบายได้ว่า:

1. Inner Loop และ Outer Loop ต่างกันอย่างไร
2. Google ADK Agent ทำหน้าที่เป็น Orchestrator อย่างไร
3. Workers ต่างภาษาและ Framework ทำงานร่วมกันอย่างไร
4. Shared SQLite Board ทำหน้าที่เป็น Blackboard อย่างไร
5. Worker Catalog ใช้ค้นหาและ Launch Workers อย่างไร
6. Subprocess Contract ทำให้ Python และ TypeScript Interoperate อย่างไร
7. Workers หลายตัวถูกเปิดให้ทำงานพร้อมกันอย่างไร
8. `Team` Object เก็บ Orchestration State อะไรบ้าง
9. Agent Decisions แยกจาก Deterministic Mechanics อย่างไร
10. Live Board แสดง Parallel Agent Work อย่างไร
11. Artifact Contract ใช้ตรวจขั้นต่ำอย่างไร
12. Playwright MCP และ QA Agent ตรวจเกมจริงอย่างไร
13. Repair Loop ถูกจำกัดไม่ให้วนตลอดไปอย่างไร
14. Worker Timeout, QA Timeout และ LLM-call Budget ทำงานอย่างไร
15. Deterministic Fallbacks ช่วยให้ระบบ Gracefully Degrade อย่างไร
16. Agent Team นี้แตกต่างจาก Multi-agent Conversation อย่างไร
17. Shared Contract สำคัญกว่า Shared Framework อย่างไร
18. State Surfaces ของระบบมีอะไรบ้าง
19. Failure Evidence ใดที่ระบบยังขาด
20. Production Version ควรเพิ่ม Controls อะไร

---

# 3. Prerequisites

ควรเข้าใจ Episodic Artifacts ของ Week 5 Labs 1–4:

```text
Google ADK
Strands
Pydantic AI
Microsoft Agent Framework
Agno
Mastra
```

และแนวคิด:

```text
Agent loop
Tool call
MCP
SQLite board
Subprocess
External artifact
Runtime validation
Timeout
Deterministic validation
```

Environment:

```text
Python >= 3.12
uv
Node.js 24
npx
Google Chrome
Google API key
OpenAI API key
```

Environment Variables:

```env
GOOGLE_API_KEY=...
OPENAI_API_KEY=...
```

---

# 4. Project Goal

ระบบสร้างเว็บไซต์:

```text
site/
├── common.css
├── index.html
├── board.sqlite
├── strands/
│   ├── game.html
│   ├── game.css
│   └── game.js
├── pydantic/
├── maf/
├── agno/
└── mastra/
```

แต่ละ Worker ได้รับ Learning Objective ที่แตกต่างกัน เช่น:

```text
Greetings
Numbers
Food and drink
Common verbs
Travel phrases
```

Worker เป็นผู้คิดเองว่าเกมจะเล่นอย่างไร

แต่ทุกเกมต้องมี:

```text
Progression
Score
Correct/wrong feedback
Language-learning content
Local file execution
No network calls
No external assets
```

---

# 5. Week 5 Inner Loop

Workers แต่ละตัวมี Inner Loop:

```text
Receive one goal
        ↓
Read shared board
        ↓
Create steps
        ↓
Use filesystem tools
        ↓
Create game files
        ↓
Read files back
        ↓
Complete steps
        ↓
Complete goal
```

Inner Loop ถูกควบคุมโดย Framework ของ Worker แต่ละตัว

```text
Strands
→ invoke_async()

Pydantic AI
→ run()

MAF
→ run()

Agno
→ arun()

Mastra
→ generate()
```

---

# 6. Lab 5 Outer Loop

Google ADK Orchestrator มี Outer Loop:

```text
Choose curriculum
        ↓
Author common style
        ↓
Assign objectives
        ↓
Launch all workers
        ↓
Wait for team
        ↓
Test each game
        ↓
Repair broken games
        ↓
Build homepage
        ↓
Finish
```

ภาพรวม:

```text
Outer Orchestrator Loop
├── Inner Strands Loop
├── Inner Pydantic Loop
├── Inner MAF Loop
├── Inner Agno Loop
└── Inner Mastra Loop
```

---

# 7. Hierarchical Coordination

ระบบนี้เป็น Hierarchical Multi-agent System

```text
Orchestrator
→ Assigns and evaluates

Workers
→ Build assigned artifacts
```

Workers ไม่ได้คุยกันโดยตรง

```text
Strands
× ไม่คุยกับ Pydantic

Pydantic
× ไม่คุยกับ MAF

Agno
× ไม่คุยกับ Mastra
```

การประสานงานเกิดผ่าน:

```text
Shared Board
Shared Site Folder
Orchestrator
```

---

# 8. Blackboard Architecture

SQLite Board ทำหน้าที่คล้าย Blackboard

```text
Orchestrator
→ วาง Goals

Workers
→ อ่าน Goals
→ เขียน Steps
→ อัปเดต Status

Live UI
→ อ่าน Board

Orchestrator
→ อ่าน Artifact State
```

Blackboard ช่วยให้ Workers ไม่ต้องรู้จัก APIs ของกันและกัน

```text
Worker A ไม่ต้อง Import Worker B
Worker B ไม่ต้องเข้าใจ Framework A
```

---

# 9. Shared Contracts

Interoperability เกิดจาก Contract กลาง

## Input Contract

```text
taskId
boardPath
```

## Work-state Contract

```text
SQLite todos table
```

## Status Contract

```text
pending
in_progress
done
```

## Artifact Contract

```text
game.html
game.css
game.js
```

## Execution Contract

```text
Worker process exits
```

หลัก:

```text
Shared framework
ไม่จำเป็น

Shared contracts
จำเป็น
```

---

# 10. Entry Point — `agent_loop.py`

หน้าที่:

```text
Load environment
Parse CLI arguments
Set shared board path
Set worker model
Discover workers
Start orchestrator
Run final deterministic check
Open finished website
```

ส่วนสำคัญ:

```python
SITE = HERE / "site"
BOARD_PATH = SITE / "board.sqlite"

os.environ["BOARD_PATH"] = str(
    BOARD_PATH
)

os.environ["WORKER_MODEL"] = (
    config.WORKER_MODEL
)
```

Environment ต้องถูกตั้งก่อน Import Board-related Modules

---

# 11. Command-line Usage

รันปกติ:

```powershell
uv run agent_loop.py
```

เปลี่ยนภาษา:

```powershell
uv run agent_loop.py `
  --language French
```

ข้าม Workers:

```powershell
uv run agent_loop.py `
  --skip agno mastra
```

ดูทีมโดยไม่เรียก Model:

```powershell
uv run agent_loop.py `
  --dry-run
```

ไม่เปิด Browser หลังจบ:

```powershell
uv run agent_loop.py `
  --no-open
```

---

# 12. `--dry-run`

`--dry-run` ใช้ตรวจ:

```text
Worker files ถูกค้นพบหรือไม่
Worker ไหนจะถูกรัน
Framework ไหนถูก Skip
Node/Mastra worker พร้อมหรือไม่ในระดับไฟล์
```

แต่ไม่ได้ตรวจ:

```text
API keys
Dependencies
MCP startup
Chrome
Model access
Worker health
```

ดังนั้น Dry Run คือ Discovery Check ไม่ใช่ Full Readiness Check

---

# 13. Worker Catalog

`catalog.py` เก็บ Manifest:

```python
WORKERS = [
    {
        "key": "strands",
        "name": "AWS Strands",
        "file": "../2_strands_pydantic/"
                "strands_worker.py",
        "runner": "python",
    },
    ...
]
```

ข้อมูลต่อ Worker:

```text
key
→ Framework identifier

name
→ Human-readable name

colour
→ Live Board colour

file
→ Worker source

runner
→ Python หรือ Node

slug
→ Output folder
```

---

# 14. Worker Discovery

```python
catalog.discover(
    skip=tuple(args.skip)
)
```

Logic:

```text
Worker file exists
+
Worker not skipped
        ↓
Add to team
```

ข้อดี:

```text
Simple
Flexible
Students can skip labs
No plugin installation required
```

ข้อจำกัด:

```text
File exists
≠ Worker is healthy

File exists
≠ Dependencies installed

File exists
≠ API keys valid
```

---

# 15. Cross-language Worker Launch

Python Worker:

```text
uv run worker.py taskId boardPath
```

TypeScript Worker:

```text
npx tsx worker.ts taskId boardPath
```

Catalog สร้าง Argument Vector:

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

Orchestrator ไม่ต้องรู้รายละเอียดภายใน Framework

---

# 16. Subprocess Isolation

Workers ถูกเปิดด้วย:

```python
subprocess.Popen(...)
```

แต่ละ Worker มี:

```text
Own Python/Node runtime
Own Framework dependencies
Own Agent loop
Own MCP child process
```

ข้อดี:

```text
Library conflicts ลดลง
Event loops แยกกัน
Framework crash ถูกจำกัดใน Process
Python และ TypeScript ร่วมกันได้
```

ข้อเสีย:

```text
Process management ซับซ้อน
Logs กระจาย
Cancellation ต้องจัดการ Process Tree
State synchronization ยากขึ้น
```

---

# 17. Shared Board

Board อยู่ที่:

```text
site/board.sqlite
```

Schema:

```sql
CREATE TABLE todos (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    parent_id INTEGER,
    task TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    result TEXT NOT NULL DEFAULT ''
)
```

Goal:

```text
parent_id = NULL
```

Step:

```text
parent_id = goal.id
```

---

# 18. SQLite Concurrency

Board ใช้:

```sql
PRAGMA journal_mode=WAL
PRAGMA busy_timeout=5000
```

ช่วยให้:

```text
Readers หลายตัวอ่านพร้อมกัน
Writers รอ Lock ชั่วคราว
ลด database locked errors
```

แต่:

```text
WAL
≠ Job ownership

Busy timeout
≠ Distributed locking
```

---

# 19. Board Is Not a Production Queue

Board ยังไม่มี:

```text
worker_id
attempt
lease_expiry
heartbeat
error_status
created_at
started_at
finished_at
exit_code
```

Status มีเพียง:

```text
pending
in_progress
done
```

ดังนั้น Worker ที่ Crash อาจทิ้ง Goal เป็น:

```text
in_progress
```

โดยไม่มีหลักฐานว่าทำไม

---

# 20. `Team` Object

`Team` เป็น Orchestration State ใน Memory

เก็บ:

```text
language
workers
site_dir
board_path
by_key
by_slug
objectives
registry
pending
fixed
```

รายละเอียด:

```text
objectives
→ Worker แต่ละตัวได้รับหัวข้ออะไร

registry
→ Goal ID เป็นของ Worker ไหน

pending
→ Processes ที่ยังไม่ Wait

fixed
→ Game ที่ใช้ Fix Attempt แล้ว
```

---

# 21. Team State Is Not Durable

`Team` อยู่ใน Memory ของ Orchestrator

ถ้า Process ปิด:

```text
pending processes mapping หาย
objectives mapping หาย
fix records หาย
QA results หาย
```

แม้ Board และ Files ยังอยู่

Production ต้อง Persist:

```text
run_id
phase
assignments
worker attempts
QA verdicts
repair history
```

---

# 22. Orchestrator Tools

Orchestrator มี Tools:

```text
author_style
launch_worker
wait_for_team
test_game
relaunch_worker
build_hub
```

Tools ถูกสร้างเป็น Closures รอบ `Team`

```python
def make_tools(team):
    ...
```

Agent เห็นเพียง Tool Interfaces

ส่วน Internal State ยังคงอยู่ใน Application

---

# 23. Agent Owns Decisions

Orchestrator Agent ตัดสินใจ:

```text
แต่ละ Worker ควรสอนอะไร
Objectives ควรแตกต่างกันอย่างไร
ควร Launch ใครบ้าง
ควร Test เกมใด
เกมใดควรแก้
เมื่อใดควร Build Hub
```

---

# 24. Tools Own Mechanics

Tools รับผิดชอบ:

```text
เขียนไฟล์
สร้าง Goal
เปิด Process
รอ Process
ฆ่า Process
เปิด Browser
ตรวจ Files
สร้าง Index
```

หลัก:

```text
LLM
→ Flexible judgment

Deterministic tool
→ Mechanical execution
```

นี่ช่วยลดการให้ Model มี Authority ต่ำเกินไปหรือสูงเกินไป

---

# 25. `author_style`

Flow:

```text
Orchestrator
→ author_style()
→ ADK Art Director
→ common.css
```

Art Director ใช้ Model เพื่อสร้าง:

```text
Shared colours
Typography
Cards
Buttons
Correct/wrong classes
```

หาก Model ล้มเหลว:

```text
Use built-in CSS template
```

Pattern:

```text
Creative generation
+
Deterministic fallback
```

---

# 26. `launch_worker`

Signature:

```python
launch_worker(
    framework: str,
    objective: str,
)
```

Flow:

```text
Validate framework
→ Prevent duplicate launch
→ Save objective
→ Create output folder
→ Format GAME_TASK
→ Add goal to board
→ Register goal ownership
→ Start subprocess
→ Return immediately
```

การ Return ทันทีทำให้ Launch Workers พร้อมกันได้

---

# 27. Worker Task Contract

`GAME_TASK` บอก Worker ให้:

```text
Invent one browser game
Teach assigned objective
Use vanilla HTML/CSS/JavaScript
No frameworks
No network calls
No external assets
Add progression
Keep score
Give feedback
Create exactly three files
Use common.css
Link back to index.html
Run through file://
Read files back before completion
```

Task เน้น Outcome มากกว่า Step-by-step Procedure

Worker จึงมีพื้นที่ออกแบบเกมเอง

---

# 28. Parallel Launch

Prompt สั่ง Orchestrator:

```text
Launch all builders
before waiting
```

Expected Flow:

```text
launch_worker("strands", ...)
launch_worker("pydantic", ...)
launch_worker("maf", ...)
launch_worker("agno", ...)
launch_worker("mastra", ...)
        ↓
wait_for_team()
```

Workers ทำงานแบบ Process-level Parallelism

---

# 29. Why Not Await Each Worker Immediately?

หากทำ:

```text
Launch Strands
→ Wait Strands
→ Launch Pydantic
→ Wait Pydantic
```

งานจะเป็น Sequential

แต่ Lab ใช้:

```text
Launch all
→ Wait all
```

เพื่อลดเวลารวมและแสดง Shared Board ที่อัปเดตพร้อมกัน

---

# 30. `_launch()`

Worker Process ใช้:

```python
subprocess.Popen(
    argv,
    stdout=subprocess.DEVNULL,
    stderr=subprocess.DEVNULL,
    cwd=cwd,
)
```

Worker ไม่สื่อสารผ่าน Standard Output

แต่ผ่าน:

```text
Board
Files
Process exit
```

---

# 31. Process Groups

บน POSIX:

```text
start_new_session=True
```

ทำให้ Worker และ Child Processes อยู่ใน Process Group ของตนเอง

บน Windows เมื่อ Timeout ใช้:

```text
taskkill /F /T /PID
```

เป้าหมายคือฆ่าทั้ง:

```text
Worker
uv/npx
MCP server
child process
```

ไม่ใช่ฆ่าเพียง Parent Process

---

# 32. `wait_for_team`

Flow:

```text
Copy pending processes
→ Clear pending list
→ Start live board
→ Poll process status
→ Refresh board
→ Check timeout
→ Terminate hung processes
→ Return team status
```

Defaults:

```text
Worker timeout
= 300 seconds

Polling interval
= 0.15 seconds

Live refresh
= 8 FPS
```

---

# 33. Live Board

Live Board อ่าน SQLite และแสดง:

```text
Framework
Learning objective
Goal
Steps
Status
Result
```

Display:

```text
done
→ dim + strike

in_progress
→ bold

pending
→ normal
```

แต่ละ Framework มีสีของตนเอง

---

# 34. Operational Metadata Separation

สีของ Worker ไม่ถูกเก็บใน SQLite

มันอยู่ใน:

```text
Team registry
```

Board เก็บเฉพาะ Business State

```text
Task
Status
Result
```

Live UI รวมสอง Source ตอน Render

หลัก:

```text
Operational presentation metadata
ไม่จำเป็นต้องอยู่ใน business schema
```

---

# 35. Artifact Contract

เกมที่สำเร็จขั้นต่ำต้องมี:

```text
game.html
game.css
game.js
```

และไฟล์ต้อง:

```text
exist
non-empty
```

ตรวจผ่าน:

```python
is_built(folder)
```

---

# 36. Minimum Gate

`is_built()` เป็น Deterministic Minimum Gate

มันตอบคำถามว่า:

```text
มี Files ครบหรือไม่
Files ว่างหรือไม่
```

แต่ไม่ตอบว่า:

```text
HTML ถูกต้องหรือไม่
CSS โหลดหรือไม่
JavaScript รันได้หรือไม่
Game เล่นได้หรือไม่
เนื้อหาภาษาถูกหรือไม่
```

จึงต้องมี Browser QA

---

# 37. `test_game`

Flow:

```text
Check required files
→ Create file:// URI
→ Start QA Agent
→ Open game
→ Interact
→ Check browser console
→ Record verdict
```

QA Agent ไม่ได้อ่านเพียง Code แต่ตรวจ Runtime Environment

---

# 38. QA Agent

ต่อหนึ่งเกมจะสร้าง:

```text
Fresh ADK Agent
Fresh Runner
Fresh Session
Fresh Browser MCP
```

Tools:

```text
Playwright browser
report_game
```

ผลลัพธ์:

```json
{
  "works": true,
  "note": "The game loads and responds."
}
```

---

# 39. Why Fresh Agent per Game?

ช่วยลด:

```text
Context contamination
Browser state contamination
Long conversation history
Unbounded QA loop
```

แต่ละเกมมี Evaluation Context ของตนเอง

---

# 40. Playwright MCP

Browser Process เริ่มด้วย:

```text
@playwright/mcp
Chrome
Isolated session
Local file access
```

ต้องใช้:

```text
--allow-unrestricted-file-access
```

เพราะเกมเปิดจาก:

```text
file://
```

สามารถเปิด Headless Mode ด้วย:

```env
QA_HEADLESS=1
```

---

# 41. QA Prompt

QA Agent ถูกสั่งให้:

```text
Open the game
Look at the screen
Click a few controls
Check console errors
Decide if it works
Call report_game
```

ควรใช้ประมาณไม่กี่ Actions เพื่อไม่ให้ QA ใช้เวลามากเกินไป

---

# 42. QA Budgets

QA มี Limits:

```text
Maximum LLM calls
= 25

Orchestrator QA timeout
= 150 seconds

Browser close timeout
= 10 seconds
```

ช่วยป้องกัน Browser Agent ค้าง

---

# 43. QA Heuristic

หาก QA Agent ใช้ครบ Call Budget โดยไม่ได้ Report:

```text
Code treats game as working
```

เหตุผล:

```text
Agent interacted throughout the budget
→ Game likely responsive
```

แต่เป็น Heuristic

อาจเกิด:

```text
False positive
```

เช่น Agent วนบนหน้า Error โดยยังเรียก Browser Tools ได้

---

# 44. QA Failure Policy

หาก Browser หรือ QA ใช้งานไม่ได้:

```text
Do not automatically mark game broken
```

ระบบจะรายงานว่า:

```text
Could not check
```

และปล่อย Artifact ไว้

นี่เป็น Policy เลือกระหว่าง:

```text
Fail closed
หรือ
Fail open
```

Lab เลือกแนวโน้ม Fail Open เพื่อให้ Demo สร้าง Site ต่อได้

---

# 45. `relaunch_worker`

หากเกม Broken:

```text
Create new fix task
→ Add to board
→ Launch same framework
→ Same game folder
→ Wait again
→ Test again
```

Fix Task มี:

```text
Language
Folder
Learning objective
Symptom
Required files
file:// requirement
```

---

# 46. Bounded Repair Loop

```text
Build
→ Test
→ Broken?
    ├── No → accept
    └── Yes
         → fix once
         → retest
         → stop
```

แต่ละเกมใช้ Fix ได้สูงสุด:

```text
1 attempt
```

ช่วยควบคุม:

```text
Cost
Latency
Infinite loops
Repeated regressions
```

---

# 47. One Fix Is a Policy

`fixed` Set เก็บ Slugs ที่ใช้ Repair Attempt แล้ว

```text
slug in fixed
→ Do not relaunch again
```

นี่คือ Application Policy ไม่ใช่ Model Decision

Model อาจต้องการลองอีก แต่ Tool ปฏิเสธ

หลัก:

```text
Hard budget
ควรถูกบังคับด้วย Code
```

---

# 48. `build_hub`

เมื่อ Games สร้างและตรวจแล้ว:

```text
Read finished games
→ Extract <title>
→ Build links
→ Generate index.html
```

Labels ใช้ลำดับ Fallback:

```text
HTML title
→ Assigned objective
→ Framework name
```

---

# 49. Hub Fallback

หาก Art Director Model ล้มเหลว:

```text
Write deterministic HTML template
```

ระบบจึงมี Front Door แม้ Agent ไม่สามารถสร้างหน้า Hub ได้

---

# 50. Orchestrator Agent

สร้างด้วย:

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

Orchestrator ใช้ Google ADK เพราะเป็น Framework ที่เรียนใน Lab 1

---

# 51. Orchestrator Prompt

Prompt ระบุ Workflow:

```text
1. Author style
2. Give distinct objectives
3. Launch all builders
4. Wait
5. Test each game
6. Repair broken games
7. Build hub
8. Summarize
```

แต่ไม่ได้เขียน Python Loop แบบตายตัวเพื่อเรียกทุก Tool

Model เป็นผู้เลือก Tool Calls ตาม Goal

---

# 52. Prompt-guided Workflow vs Code Workflow

Orchestrator Flow อยู่ใน Prompt

ไม่ใช่ Graph ที่บังคับด้วย Code

ผล:

```text
Agent อาจเรียก Tools ผิดลำดับ
ลืม Worker
ลืม Test
ลืม Hub
เรียก Wait เร็วเกินไป
```

Code ลดความเสี่ยงด้วย:

```text
Tool guards
Budgets
Final checks
Fallback templates
```

แต่ยังไม่เข้มเท่า Deterministic Workflow Graph

---

# 53. Orchestrator Budget

Outer Loop จำกัด:

```text
Maximum LLM calls
= 80
```

หากถึง Limit:

```text
Stop orchestrator
Keep built files
Run deterministic safety net
```

นี่คือ Bounded Autonomy

---

# 54. Orchestrator Error Handling

หากเกิด:

```text
Network error
Model error
Rate limit
Unexpected exception
```

ระบบ:

```text
Stops agent cleanly
Reports exception type
Keeps artifacts
Builds fallback site
```

ไม่ทิ้งสิ่งที่ Workers สร้างเสร็จแล้ว

---

# 55. Graceful Degradation

ระบบถูกออกแบบให้ลดคุณภาพแทนการล้มทั้งหมด

ตัวอย่าง:

```text
CSS Agent fails
→ Built-in CSS

Hub Agent fails
→ Built-in index.html

Orchestrator stops
→ Keep completed games

QA unavailable
→ Leave game built but unchecked

Worker timeout
→ Stop worker and continue
```

---

# 56. Deterministic Safety Net

หลัง Agent Loop:

```text
Check common.css
Check index.html
```

หากไม่มี:

```text
Write templates
```

Safety Net อยู่นอก Agent Loop จึงไม่ขึ้นกับ Model Decision

---

# 57. Final Check

หลัง Run:

```text
For each worker
→ Check output folder
→ Print ok or INCOMPLETE
```

ตรวจผ่าน Required Files

ทำให้ผู้ใช้เห็นว่า Worker ใดสร้าง Artifact ไม่ครบ แม้ Orchestrator จะรายงานว่าจบแล้ว

---

# 58. Model Configuration

`config.py` แยก Models ออกจาก Logic

```text
ORCHESTRATOR_MODEL
WORKER_MODEL
```

Default ใน Course:

```text
Orchestrator
→ Gemini model

Workers
→ OpenAI model
```

สามารถ Override ผ่าน Environment

---

# 59. Why Separate Orchestrator and Worker Models?

ช่วยให้:

```text
ใช้ Model ใหญ่กับ Planning
ใช้ Model เล็กกับ Builders
ควบคุม Cost
Benchmark Models
เปลี่ยน Provider โดยไม่แก้ Logic
```

แต่ Worker Frameworks อาจรองรับ Model IDs แตกต่างกัน จึงต้องรักษา Model Contract ให้สอดคล้องกัน

---

# 60. State Surfaces

ระบบมี State หลาย Layer:

```text
ADK orchestrator session
Team object
SQLite board
Worker subprocesses
MCP subprocesses
Site files
QA verdicts
Browser state
```

แต่ละ Surface มี Lifecycle ต่างกัน

---

# 61. State Divergence Examples

```text
Worker process exited
แต่ Goal ยัง in_progress

Goal เป็น done
แต่ Files ไม่มี

Files มีครบ
แต่ JavaScript พัง

QA says works
แต่ Learning content ผิด

Agent stopped
แต่ Workers ยังทำงาน

Board reset
แต่ Old files ยังอยู่
```

ระบบที่น่าเชื่อถือจำเป็นต้อง Reconcile State เหล่านี้

---

# 62. Stale Artifact Risk

ก่อน Run ระบบ Reset Board

แต่ไม่ได้รับประกันว่าจะล้างทุก Artifact จาก Run ก่อนหน้า

กรณี:

```text
Old game files remain
New worker crashes
is_built() sees old files
→ reports built
```

Safer Pattern:

```text
Delete site directory before run
หรือ
Use run-specific directory
```

ตัวอย่าง:

```text
runs/
└── 2026-08-03T1610/
    └── site/
```

---

# 63. Worker Exit-code Risk

`wait_for_team()` ตรวจว่า Process จบหรือไม่

แต่ยังไม่ได้ใช้ Exit Code เป็นหลักฐานเต็มรูปแบบ

ควรแยก:

```text
exit_code = 0
→ Process claims success

exit_code != 0
→ Worker failure

terminated
→ Timeout/cancellation
```

และเก็บไว้ใน Worker Run Record

---

# 64. Worker Logs

ปัจจุบัน Worker Output ถูกส่งไป:

```text
DEVNULL
```

ข้อดี:

```text
Live board ไม่เสียรูป
Console ไม่เต็ม
```

ข้อเสีย:

```text
Traceback หาย
MCP error หาย
Dependency error หาย
Model failure หาย
```

Production ควรเก็บ:

```text
logs/strands.log
logs/pydantic.log
logs/maf.log
logs/agno.log
logs/mastra.log
```

---

# 65. Non-atomic Claim Risk

Board ใช้:

```sql
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
```

ไม่มี:

```sql
AND status = 'pending'
```

ถ้า Worker ถูก Launch ซ้ำ:

```text
Worker A claims Task
Worker B claims same Task
```

ทั้งสองอาจทำงานพร้อมกัน

---

# 66. Atomic Claim Improvement

```sql
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
  AND status = 'pending'
```

ตรวจจำนวน Rows:

```text
1 row
→ Claim successful

0 rows
→ Already claimed
```

Worker ต้องหยุดหาก Claim ไม่สำเร็จ

---

# 67. Weak Artifact Validation

Minimum Gate ตรวจเพียง:

```text
Files exist
Files non-empty
```

ควรเพิ่ม:

```text
HTML parses
CSS links correctly
JavaScript syntax valid
common.css link exists
index.html back link exists
No external URLs
No build dependencies
Required interaction exists
Score element exists
```

---

# 68. Browser QA Limitations

QA Agent อาจ:

```text
คลิก Controls ไม่ครบ
เข้าใจ Game ผิด
พลาด Console Error
ตัดสิน Language Content ไม่ถูก
รายงาน works จาก UI ที่แค่เปิดได้
```

ดังนั้น Browser Agent เป็น Exploratory QA ไม่ใช่ Complete Test Suite

---

# 69. Better Validation Stack

```text
Artifact files
        ↓
Deterministic static checks
├── Required files
├── HTML structure
├── JS syntax
├── Required links
└── No network dependencies
        ↓
Automated browser assertions
├── Page loads
├── Controls respond
├── Score changes
└── Console has no errors
        ↓
LLM QA
├── Game makes sense
├── Feedback is understandable
└── Language-learning quality
        ↓
Human review
```

---

# 70. Browser Security

QA Browser ใช้ Local File Access เพื่อเปิด Agent-generated JavaScript

ต้องถือ Generated Game เป็น Untrusted Code

Safer Production Setup:

```text
Disposable container
Temporary browser profile
No personal cookies
Network disabled
Read-only host filesystem
Run-specific workspace
Resource limits
```

---

# 71. MCP Version Risk

Browser MCP ใช้ Package Version ที่อาจเปลี่ยนตามเวลา

MCP Server Version Drift อาจทำให้:

```text
Tool names เปลี่ยน
Schema เปลี่ยน
Flags เปลี่ยน
Startup behavior เปลี่ยน
```

ควร Pin Version ที่ทดสอบแล้ว

---

# 72. Multi-agent System Type

ระบบนี้เป็น:

```text
Hierarchical orchestration
+
Blackboard coordination
+
Artifact-based collaboration
```

ไม่ใช่:

```text
Free-form group chat
Peer-to-peer negotiation
Debate agents
```

---

# 73. Agent Team vs Multi-agent Chat

## Multi-agent Chat

```text
Agent A sends message to Agent B
Agent B replies
Agents negotiate
```

## Agent Loop Team

```text
Orchestrator creates task
Worker reads task
Worker writes artifact
Orchestrator evaluates artifact
```

Lab นี้ให้ความสำคัญกับ Work Products มากกว่า Conversation

---

# 74. Why Artifacts Matter

Workers ส่งผลลัพธ์หลักผ่าน:

```text
HTML
CSS
JavaScript
Board status
```

ไม่ใช่เพียงข้อความว่า:

```text
I finished the task
```

Artifact-based Systems ตรวจสอบได้มากกว่า Chat-based Claims

แต่ยังต้องมี Validators เพื่อยืนยัน Artifact Quality

---

# 75. Framework Isolation

แต่ละ Framework รันใน Process ของตัวเอง

จึงสามารถ:

```text
ใช้ Event Loop ของตนเอง
ใช้ MCP Integration ของตนเอง
ใช้ Agent APIs ของตนเอง
ใช้ TypeScript หรือ Python
```

Orchestrator ไม่ต้องจัดการ Framework-specific Runtime ภายใน Process เดียว

---

# 76. Event-loop Design

Orchestrator ใช้ Event Loop เดียวสำหรับ:

```text
ADK agent
Async tools
Browser QA
Style generation
Hub generation
```

Worker Processes ใช้ `subprocess.Popen()` แทน Async Transport เพื่อไม่ให้ Worker Child Processes ผูกกับ Orchestrator Event Loop

ช่วยลด Error เช่น:

```text
event loop is closed
```

ระหว่าง Cleanup

---

# 77. Cleanup

QA Agent ปิด:

```text
Browser MCP
Runner
```

ภายใน Event Loop

Workers ที่ Timeout ถูกฆ่าทั้ง Process Tree

แต่ Production ยังต้องตรวจ:

```text
Zombie processes
Orphaned npx processes
Open SQLite locks
Temporary browser profiles
```

---

# 78. Cost Surfaces

Cost ไม่ได้เกิดจาก Orchestrator Model เพียงตัวเดียว

```text
Orchestrator calls
Workers × model calls
CSS agent
Hub agent
QA agents × games
Repair workers
```

Worst-case:

```text
Initial build
+ QA for every game
+ Repair one round
+ QA again
```

ควรมี Global Run Budget

---

# 79. Existing Budgets

```text
Orchestrator
→ 80 LLM calls

Worker
→ Framework-specific limit

Worker timeout
→ 300 seconds

QA
→ 25 LLM calls per game

QA timeout
→ 150 seconds

Fix
→ 1 attempt per game
```

แต่ยังไม่มี Single Global Monetary Budget

---

# 80. Recommended Global Budget

ควรเก็บ:

```text
Total model calls
Total tool calls
Total tokens
Total elapsed time
Total worker launches
Total QA sessions
Estimated cost
```

จากนั้นหยุดทั้ง Run เมื่อถึง Limit

---

# 81. Failure Categories

## Worker Failure

```text
Dependency missing
Model error
MCP error
Timeout
Crash
```

## Artifact Failure

```text
Missing files
Empty files
Syntax error
Broken links
```

## QA Failure

```text
Browser unavailable
Chrome missing
MCP unavailable
QA timeout
```

## Orchestrator Failure

```text
Model call limit
Network error
Tool-order error
```

## State Failure

```text
Duplicate claim
Stale artifact
Board mismatch
Lost in-memory state
```

---

# 82. Failure Evidence

Production System ควรบันทึก:

```text
run_id
goal_id
framework
pid
start_time
end_time
exit_code
timeout
log_path
artifact_hashes
QA verdict
repair_attempt
```

Board `result` เพียง String เดียวไม่เพียงพอสำหรับ Audit

---

# 83. Durable Orchestrator Improvement

เพิ่ม Tables:

```sql
runs
workers
worker_runs
qa_results
artifacts
```

ตัวอย่าง:

```text
runs
→ Overall phase and budget

workers
→ Framework registry

worker_runs
→ Attempts and exit codes

qa_results
→ Browser verdicts

artifacts
→ Paths, hashes and validation
```

---

# 84. Workflow Phases

Durable Workflow ควรมี:

```text
INITIALIZING
STYLING
BUILDING
WAITING
VALIDATING
REPAIRING
ASSEMBLING
COMPLETED
FAILED
```

หาก Process Restart ระบบสามารถอ่าน Phase แล้วทำต่อได้

---

# 85. Connection to Week 4 Lab 5

## Sidekick

```text
One worker
→ Evaluator
→ Feedback
→ Retry
```

## Agent Loop

```text
Multiple workers
→ QA evaluator
→ Selective repair
→ Site assembly
```

Sidekick คือ Acceptance Loop ของ Task เดียว

Agent Loop คือ Orchestration Loop ของ Team

---

# 86. Connection to Week 5 Lab 1

Google ADK จาก Worker กลายเป็น Orchestrator

```text
Lab 1
ADK builds one artifact

Lab 5
ADK coordinates many artifact builders
```

Concept เดิม:

```text
LlmAgent
Runner
Tools
MCP
```

แต่ Authority สูงขึ้น

---

# 87. Connection to Week 5 Lab 2

Strands และ Pydantic AI Workers ไม่ต้องเปลี่ยน Agent Loop หลัก

เพียงรับ:

```text
taskId
boardPath
```

แล้วเข้าร่วม Team

นี่พิสูจน์คุณค่าของ Stable Worker Contract

---

# 88. Connection to Week 5 Lab 3

MAF และ Agno Workers รันเป็น Subprocesses เหมือนกัน

Orchestrator ไม่สนใจว่า Framework เน้น:

```text
Durable workflow
หรือ
Lightweight runtime
```

ตราบใดที่ Worker ทำตาม Contract

---

# 89. Connection to Week 5 Lab 4

Mastra เป็น TypeScript Worker

แต่สามารถเข้าร่วม Team ผ่าน:

```text
npx tsx
taskId
boardPath
Shared SQLite
Shared site directory
```

แสดงว่า Cross-language Interoperability ไม่จำเป็นต้องใช้ RPC เสมอไป

---

# 90. Connection to MCP

MCP ถูกใช้สองระดับ:

```text
Workers
→ Filesystem MCP

QA Agent
→ Browser MCP
```

Filesystem MCP ช่วยสร้าง Artifact

Browser MCP ช่วยตรวจ Artifact

```text
Tool for action
+
Tool for verification
```

---

# 91. Misconceptions Corrected

## ความเข้าใจคลาดเคลื่อนที่ 1

> Multi-agent System ต้องให้ Agents คุยกัน

**ข้อเท็จจริง:**
สามารถประสานงานผ่าน Shared State และ Artifacts ได้

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> ทุก Worker ต้องใช้ Framework เดียวกัน

**ข้อเท็จจริง:**
ต้องใช้ Contract เดียวกัน ไม่ใช่ Framework เดียวกัน

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> Worker Process จบหมายถึง Task สำเร็จ

**ข้อเท็จจริง:**
ต้องตรวจ Board, Exit Code และ Artifacts

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> Required Files ครบหมายถึงเกมใช้งานได้

**ข้อเท็จจริง:**
ต้องตรวจ Runtime ผ่าน Browser

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> QA Agent เป็น Ground Truth

**ข้อเท็จจริง:**
ยังเป็น LLM และอาจตัดสินผิด

---

## ความเข้าใจคลาดเคลื่อนที่ 6

> Timeout ทำให้ระบบ Reliable

**ข้อเท็จจริง:**
Timeout ป้องกันค้าง แต่ไม่ได้ Recovery หรือพิสูจน์ Success

---

## ความเข้าใจคลาดเคลื่อนที่ 7

> SQLite WAL ป้องกัน Duplicate Work

**ข้อเท็จจริง:**
ช่วย Database Concurrency ไม่ได้บังคับ Task Ownership

---

## ความเข้าใจคลาดเคลื่อนที่ 8

> Fallback Template ทำให้ระบบสำเร็จสมบูรณ์

**ข้อเท็จจริง:**
ทำให้ระบบมี Output ขั้นต่ำ แต่คุณภาพอาจลดลง

---

## ความเข้าใจคลาดเคลื่อนที่ 9

> Agent Orchestrator คือ Python Loop ที่มี LLM Call

**ข้อเท็จจริง:**
ตัว Agent เป็นผู้ตัดสินใจ Tool Order ภายใต้ Goal และ Prompt

---

## ความเข้าใจคลาดเคลื่อนที่ 10

> Shared Board คือ Durable Orchestrator

**ข้อเท็จจริง:**
Board ไม่มี Phase, Attempts, Process Records หรือ QA State

---

# 92. Risks Identified

## 92.1 Duplicate Claim

Workers ทำ Task เดียวกัน

## 92.2 Worker Crash

Process Exit แต่ Board ไม่สะท้อน Failure

## 92.3 Hidden Logs

Errors ถูกส่งไป DEVNULL

## 92.4 Stale Artifacts

Files จาก Run ก่อนหน้าถูกนับเป็นของ Run ใหม่

## 92.5 Weak Artifact Gate

Files ครบแต่ Code พัง

## 92.6 QA False Positive

Browser Agent ตัดสินผิด

## 92.7 QA Unavailable

เกมถูกปล่อยโดยไม่มี Runtime Test

## 92.8 Untrusted JavaScript

Browser เปิด Code ที่ Agent สร้าง

## 92.9 Model-call Explosion

Orchestrator, Workers, QA และ Repairs ใช้ Calls ซ้อนกัน

## 92.10 Process Leakage

Worker หรือ MCP Child Process ไม่ถูกปิด

## 92.11 In-memory State Loss

Orchestrator Restart แล้ว Resume ไม่ได้

## 92.12 Version Drift

MCP Packages หรือ Framework APIs เปลี่ยน

---

# 93. Production Improvements

```text
Run-specific workspace
Atomic task claim
Worker identity and lease
Process exit-code tracking
Per-worker log files
Deterministic static validation
Automated Playwright assertions
LLM QA as qualitative layer
Durable orchestration state
Global cost budget
Pinned dependencies
Container isolation
Artifact hashes
Audit trail
Human review
```

---

# 94. Suggested Exercise — Clean Run

ก่อน Run:

```python
if SITE.exists():
    shutil.rmtree(SITE)

SITE.mkdir(
    parents=True
)
```

หรือใช้:

```text
runs/<run-id>/site/
```

เพื่อตัด Stale Artifacts

---

# 95. Suggested Exercise — Worker Run Records

เพิ่ม Table:

```sql
CREATE TABLE worker_runs (
    id INTEGER PRIMARY KEY,
    goal_id INTEGER,
    framework TEXT,
    pid INTEGER,
    started_at TEXT,
    finished_at TEXT,
    exit_code INTEGER,
    status TEXT,
    log_path TEXT
);
```

---

# 96. Suggested Exercise — Atomic Claim

แก้:

```python
def claim_todo(
    task_id: int
) -> bool:
    ...
```

ให้คืน `False` หาก Task ถูก Claim แล้ว

---

# 97. Suggested Exercise — Static Validator

ตรวจ:

```text
Required files
HTML structure
JavaScript syntax
Shared CSS link
Home link
No external URLs
No empty scripts
```

ก่อนเรียก Browser QA

---

# 98. Suggested Exercise — Preserve Logs

เปิด Worker ด้วย:

```python
stdout=log_file
stderr=subprocess.STDOUT
```

เมื่อ Worker Fail ให้ Live Board แสดง Error Summary จาก Log

---

# 99. Suggested Exercise — Durable Resume

จำลอง:

```text
Launch workers
→ Stop orchestrator process
→ Restart
→ Read durable state
→ Continue waiting/testing
```

---

# 100. Patterns Learned

## Blackboard Pattern

```text
Many workers
→ Shared state
```

## Hierarchical Orchestration Pattern

```text
Coordinator
→ Specialized workers
```

## Process Isolation Pattern

```text
One framework
→ One subprocess
```

## Artifact Collaboration Pattern

```text
Worker communication
→ Files and board
```

## Bounded Autonomy Pattern

```text
Calls
Timeouts
Repair attempts
```

## Environment Evaluation Pattern

```text
Generated artifact
→ Real browser
→ Verdict
```

## Graceful Degradation Pattern

```text
LLM failure
→ Deterministic fallback
```

## Contract-driven Interoperability Pattern

```text
Different runtimes
→ Same input/state/output contract
```

---

# 101. Week 5 Framework Comparison

| Framework                 | Role in Week 5                                |
| ------------------------- | --------------------------------------------- |
| Google ADK                | Worker in Lab 1, Orchestrator and QA in Lab 5 |
| AWS Strands               | Python game-building Worker                   |
| Pydantic AI               | Typed Python game-building Worker             |
| Microsoft Agent Framework | Enterprise-oriented Python Worker             |
| Agno                      | Lightweight Python Worker                     |
| Mastra                    | TypeScript game-building Worker               |

Agent Pattern:

```text
Goal
→ Model
→ Tool
→ Observation
→ Repeat
```

ยังเหมือนกันทั้งหมด

---

# 102. Week 5 Mental Model

```text
Framework
= Agent-loop implementation

Worker contract
= How a worker joins the system

SQLite board
= Shared work state

Site folder
= Shared artifact state

Orchestrator
= Decides assignments and evaluation

QA agent
= Checks runtime behavior

Application code
= Budgets, timeouts and fallbacks
```

---

# 103. Final Lessons

## Lesson 1

Multi-agent Interoperability ไม่ต้องใช้ Framework เดียวกัน

## Lesson 2

Stable Contracts สำคัญกว่า Framework-specific APIs

## Lesson 3

Inner Loops ทำงานเฉพาะ Task ส่วน Outer Loop ควบคุม Team

## Lesson 4

Shared Board ช่วยประสาน Agents โดยไม่ต้องให้ Agents คุยกันโดยตรง

## Lesson 5

Subprocess Isolation ทำให้ Python และ TypeScript Workers อยู่ร่วมกันได้

## Lesson 6

Orchestrator ควรตัดสินใจ ส่วน Tools ควรจัดการ Mechanics

## Lesson 7

Parallel Launch ลดเวลารวม แต่เพิ่มความซับซ้อนของ State และ Failure

## Lesson 8

Artifact Existence เป็น Minimum Gate ไม่ใช่ Proof of Correctness

## Lesson 9

Browser QA ตรวจ Runtime ได้มากกว่า Static File Check

## Lesson 10

LLM QA ยังไม่ใช่ Ground Truth

## Lesson 11

Repair Loop ต้องมี Budget

## Lesson 12

Fallbacks ทำให้ระบบลดคุณภาพได้โดยไม่ล้มทั้งหมด

## Lesson 13

Board Status, Process Status และ Artifact Status ต้องถูก Reconcile

## Lesson 14

Worker Logs และ Exit Codes เป็น Failure Evidence ที่จำเป็น

## Lesson 15

Agentic Production System ต้องเพิ่ม Deterministic Controls รอบทุก Loop

---

# 104. Memory Summary

```text
Week 5 Lab 5:
Project 7 — The Agent Loop

Folder:
5_agent_frameworks/5_agent_loop

Entry point:
agent_loop.py

Orchestrator:
Google ADK LlmAgent

Builders:
Strands
Pydantic AI
MAF
Agno
Mastra

Goal:
Build a language-learning arcade

Outer loop:
Style
Assign
Launch
Wait
Test
Repair
Build hub

Inner loop:
Read board
Plan
Use tools
Complete steps
Complete goal

Coordination:
Shared SQLite board

Artifacts:
site/<framework>/
game.html
game.css
game.js

Shared style:
common.css

Hub:
index.html

Worker catalog:
catalog.py

Worker launch:
Python
→ uv run

TypeScript
→ npx tsx

Worker input:
taskId
boardPath

Team state:
objectives
registry
pending
fixed

Orchestrator tools:
author_style
launch_worker
wait_for_team
test_game
relaunch_worker
build_hub

Parallelism:
subprocess.Popen

Live board:
Rich Live
reads SQLite

Worker timeout:
300 seconds

QA timeout:
150 seconds

Orchestrator call budget:
80

QA call budget:
25 per game

Repair budget:
1 fix per game

Browser QA:
Playwright MCP
Chrome
file:// access

Minimum artifact gate:
Three files exist and are non-empty

Deterministic fallbacks:
Built-in common.css
Built-in index.html

Final deterministic check:
ok / INCOMPLETE

Shared weakness:
claim_todo not atomic

Shared weakness:
Worker logs discarded

Shared weakness:
Exit codes underused

Shared weakness:
Stale artifacts possible

Shared weakness:
In-memory orchestration state

Production needs:
Run ID
Atomic claims
Worker records
Logs
Static validators
Browser assertions
Durable state
Global budgets
Pinned versions
Isolation
```

---

# 105. Week 5 Completion Summary

```text
Lab 1
Google ADK and A2A

Lab 2
Strands and Pydantic AI

Lab 3
Microsoft Agent Framework and Agno

Lab 4
Mastra and TypeScript

Lab 5
Cross-framework agent team
```

สิ่งที่ Week 5 พิสูจน์:

```text
Framework syntax
อาจแตกต่างกัน

แต่
Agent mental model
ยังเหมือนกัน
```

และ:

```text
Agent Framework
ไม่ได้กำหนด Architecture ทั้งระบบ

Application Contracts
State
Artifacts
Budgets
Validation
เป็นสิ่งที่กำหนดความน่าเชื่อถือ
```

---

# 106. Final Mental Model

```text
User goal
        ↓
ADK Orchestrator
        ↓
Shared task contracts
        ↓
Framework workers in parallel
        ↓
Shared SQLite board
        ↓
Shared site artifacts
        ↓
Deterministic file checks
        ↓
Browser QA agent
        ↓
Bounded repair
        ↓
Finished arcade
```

แก่นสำคัญที่สุดของ Lab คือ:

> **การสร้าง Team of Agents ไม่ได้เริ่มจากการให้ Agents สนทนากัน แต่เริ่มจากการออกแบบ Contract ที่ทำให้แต่ละ Agent รับงาน สร้าง Artifact รายงานสถานะ และถูกตรวจสอบได้โดยไม่ต้องรู้รายละเอียดภายในของกันและกัน**

แก่นรวมของ Week 5 คือ:

> **เมื่อถอดชื่อ Framework ออก Agent ทุกตัวทำสิ่งคล้ายกัน คือรับ Goal อ่าน State เลือก Tool สังเกตผล และทำซ้ำ ความแตกต่างของระบบ Production จึงไม่ได้อยู่เพียงที่ Agent API แต่อยู่ที่การจัดการ State, Task Ownership, Failure Evidence, Artifact Validation, Budgets และ Security รอบ Agent Loop**

และข้อสรุปสุดท้ายคือ:

> **ความสามารถของ Model ทำให้ระบบยืดหยุ่น แต่ความน่าเชื่อถือของระบบมาจาก Application Code ที่กำหนดขอบเขต ตรวจผลงาน เก็บหลักฐาน และหยุด Loop เมื่อถึงจุดที่เหมาะสม**
