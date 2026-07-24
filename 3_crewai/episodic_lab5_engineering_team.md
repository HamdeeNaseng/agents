# Episodic Learning Artifact

## Week 3 — Lab 5: CrewAI Engineering Team

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**โปรเจกต์:** `3_crewai/reference/engineering_team/`
**หัวข้อหลัก:** Multi-Agent Software Engineering, Shared Artifacts, Task Dependencies, MCP Documentation, Testing และ Integration Quality Gates
**สถานะ:** เรียนและสรุป Week 3 แล้ว

---

# 1. Context

Week 3 Lab 4 สร้าง Coder Agent ตัวเดียวที่ทำงานผ่านวงจร:

```text
Assignment
→ Write code
→ Execute
→ Observe
→ Fix
→ Summarize
```

Lab 5 ขยายจาก Coding Agent ตัวเดียวไปเป็นทีม Software Engineering หลายบทบาท:

```text
Requirements
→ Engineering Design
→ Backend
→ Frontend
→ Tests
```

แนวคิดใหม่ที่เพิ่มเข้ามา:

```text
Specialized Engineering Agents
Shared Mutable Filesystem
Design as a Contract
Task Dependencies
MCP Documentation
Multi-file Software Artifacts
Unit Testing
Integration Risks
```

ระบบมี Agents สี่ตัว:

```text
Engineering Lead
Backend Engineer
Frontend Engineer
Test Engineer
```

ทุก Agent มีขอบเขตงานต่างกัน แต่ร่วมกันสร้าง Application เดียวภายใน Shared Sandbox

---

# 2. Learning Objectives

หลังจบ Lab 5 สามารถอธิบายได้ว่า:

1. Engineering Lead, Backend Engineer, Frontend Engineer และ Test Engineer แบ่งงานกันอย่างไร
2. Design Artifact ทำหน้าที่เป็น Contract ระหว่าง Agents อย่างไร
3. Task Context แตกต่างจาก Shared Filesystem อย่างไร
4. Sequential Process กำหนดลำดับการพัฒนาอย่างไร
5. MCP และ Context7 ช่วย Agent เข้าถึง Documentation ล่าสุดอย่างไร
6. Shared Sandbox ช่วยให้ Agents ร่วมกันสร้างหลายไฟล์อย่างไร
7. Interface Drift เกิดขึ้นได้อย่างไร
8. ทำไม Unit Tests ผ่านจึงยังไม่รับประกัน System Correctness
9. ทำไมต้องมี Final Integration Gate
10. Tool Permission และ File Ownership ควรถูกควบคุมอย่างไร
11. Mutable Artifacts แตกต่างจาก CrewAI Memory อย่างไร
12. Security Risks จาก Coder Lab ยังคงอยู่ใน Engineering Team อย่างไร
13. Structured Design และ Contract Tests ช่วยลดความขัดแย้งระหว่าง Agents อย่างไร
14. Multi-Agent Coding เพิ่มคุณค่าและ Complexity พร้อมกันอย่างไร

---

# 3. Architecture Overview

```text
High-level Requirements
        ↓
Engineering Lead
        ↓
sandbox/design.md
        ↓
Backend Engineer
        ↓
backend.py
        ↓
Frontend Engineer
        ↓
app.py
_validate.py
        ↓
Test Engineer
        ↓
test_backend.py
test_summary.md
```

Task Sequence:

```text
design_task
→ code_task
→ frontend_task
→ test_task
```

Crew ใช้:

```python
process=Process.sequential
```

ดังนั้น Agent แต่ละบทบาททำงานตามลำดับที่กำหนด

---

# 4. Project Structure

```text
engineering_team/
├── output/
├── sandbox/
├── sandbox_gpt/
├── sandbox_claude/
├── sandbox_mixed/
├── src/
│   └── engineering_team/
│       ├── config/
│       │   ├── agents.yaml
│       │   └── tasks.yaml
│       ├── tools/
│       │   └── sandbox_tools.py
│       ├── crew.py
│       ├── main.py
│       └── patch.py
├── pyproject.toml
└── README.md
```

หน้าที่ของส่วนสำคัญ:

```text
agents.yaml
→ กำหนด Engineering Roles

tasks.yaml
→ กำหนดลำดับและงานของแต่ละ Role

crew.py
→ ประกอบ Agents, Tools, MCP และ Crew

main.py
→ กำหนด Requirements และเริ่ม Workflow

sandbox/
→ Active workspace ของ Run ปัจจุบัน

sandbox_gpt/
sandbox_claude/
sandbox_mixed/
→ Reference outputs จาก Model configurations ต่างกัน
```

---

# 5. Engineering Agents

ระบบแบ่งงานตามความเชี่ยวชาญ:

```text
Engineering Lead
→ Architecture and contracts

Backend Engineer
→ Business logic and backend API

Frontend Engineer
→ Gradio user interface

Test Engineer
→ Backend verification and defect repair
```

Mental Model:

```text
Lead
= Architect

Backend Engineer
= Core system builder

Frontend Engineer
= User experience builder

Test Engineer
= Independent verifier
```

---

# 6. Engineering Lead

Engineering Lead รับ High-level Requirements แล้วสร้าง Detailed Design

หน้าที่:

```text
กำหนด Modules
กำหนด Classes
กำหนด Functions
กำหนด Method Signatures
กำหนด File Structure
กำหนด Responsibilities
กำหนด Agent Ownership
```

Lead ถูกสั่งว่า:

```text
ห้ามเขียน Implementation Code
```

ผลลัพธ์หลัก:

```text
sandbox/design.md
```

Mental Model:

```text
Requirements
→ Engineering interpretation
→ Design contract
```

---

# 7. ทำไม Lead ไม่ควรเขียน Implementation

การแยก Architecture ออกจาก Implementation ช่วยให้:

```text
คิดภาพรวมก่อนเขียน
ลดการตัดสินใจเฉพาะหน้า
กำหนด Interface ก่อน
แบ่งงานให้ Agents ชัดเจน
เพิ่ม Testability
```

หาก Lead เขียน Implementation ด้วย อาจเกิด:

```text
Architecture ผูกติดกับ Code มากเกินไป
Worker Agents มีพื้นที่ตัดสินใจน้อย
Lead กลายเป็น Bottleneck
บทบาททับซ้อนกัน
```

---

# 8. Design Artifact

Design ถูกบันทึกที่:

```text
sandbox/design.md
```

ควรระบุ:

```text
Required files
Class names
Function names
Method signatures
Input types
Return types
Exceptions
Business rules
Agent ownership
```

ตัวอย่าง Contract:

```python
def deposit(
    account_id: str,
    amount: Decimal
) -> Transaction:
    ...
```

Design ช่วยให้ Backend, Frontend และ Test Engineer ใช้ Interface เดียวกัน

---

# 9. Design เป็น Contract

Mental Model:

```text
design.md
= ข้อตกลงร่วมกันของทีม
```

Backend ใช้ Design เพื่อ Implement

Frontend ใช้ Design เพื่อเรียก Backend

Test Engineer ใช้ Design เพื่อสร้าง Expected Behavior

```text
Design
├── guides Backend
├── guides Frontend
└── guides Tests
```

ถ้า Design ไม่ชัด ทุก Agent อาจสร้างความเข้าใจคนละแบบ

---

# 10. Markdown Contract Limitation

Design ปัจจุบันเป็น Markdown

ข้อดี:

```text
มนุษย์อ่านง่าย
อธิบายเหตุผลได้ละเอียด
แก้ไขได้สะดวก
```

ข้อจำกัด:

```text
ตรวจชื่อ Function แบบ Deterministic ยาก
ตรวจ Parameter Types ยาก
สร้าง Contract Tests อัตโนมัติยาก
Agent อาจตีความต่างกัน
```

ดังนั้น:

```text
Readable Contract
≠
Machine-enforceable Contract
```

---

# 11. Structured Design

ระบบที่แข็งแรงกว่าควรใช้ Pydantic เช่น:

```python
class ParameterContract(BaseModel):
    name: str
    type: str
    required: bool


class FunctionContract(BaseModel):
    name: str
    parameters: list[ParameterContract]
    return_type: str
    exceptions: list[str]


class ModuleDesign(BaseModel):
    filename: str
    owner: str
    responsibilities: list[str]
    functions: list[FunctionContract]
```

ประโยชน์:

```text
ตรวจ File Names ได้
ตรวจ Function Signatures ได้
สร้าง Tests ได้
สร้าง Frontend bindings ได้
ตรวจ Interface Drift ได้
```

---

# 12. Backend Engineer

Backend Engineer รับ:

```text
Requirements
+
Design Context
```

แล้วสร้าง Core Application Logic

ข้อกำหนด:

```text
ใช้ Python Standard Library
ไม่สร้าง Frontend
ไม่ใช้ Gradio
เขียน Code ตาม Design
รันและตรวจ Code
```

Mental Model:

```text
Design Contract
→ Executable Backend
```

---

# 13. Backend Responsibilities

ตัวอย่าง Trading Simulation System ต้องรองรับ:

```text
Create account
Deposit cash
Withdraw cash
Buy shares
Sell shares
Calculate portfolio value
Calculate profit/loss
Retrieve holdings
Retrieve transaction history
```

Business Rules:

```text
ห้ามเงินสดติดลบ
ห้ามซื้อเกินเงินที่มี
ห้ามขายเกินหุ้นที่ถือ
```

Backend จึงต้องจัดการทั้ง:

```text
Domain state
Validation
Transactions
Historical records
Financial calculations
```

---

# 14. Monetary Precision

Financial calculations ควรใช้:

```python
decimal.Decimal
```

มากกว่า:

```python
float
```

เพราะ Floating-point อาจให้ผลเช่น:

```python
0.1 + 0.2 != 0.3
```

สำหรับ:

```text
Cash
Share prices
Profit/loss
Portfolio values
```

ควรมี Precision Policy ที่ชัดเจน

---

# 15. Backend Public API

Frontend และ Tests ต้องพึ่ง Backend Public API

ตัวอย่าง:

```text
create_account()
deposit()
withdraw()
buy_shares()
sell_shares()
get_portfolio_value()
get_profit_loss()
get_transactions()
```

Public API ต้องมี:

```text
Stable names
Stable parameters
Stable return types
Predictable exceptions
```

การเปลี่ยน Interface หลัง Frontend ถูกสร้างแล้วอาจทำให้ระบบรวมพัง

---

# 16. Frontend Engineer

Frontend Engineer รับ:

```text
Design Context
+
Backend Task Context
+
Shared Backend Files
```

แล้วสร้าง:

```text
app.py
_validate.py
```

หน้าที่:

```text
สร้าง Gradio UI
Import Backend
เชื่อม Components กับ Backend Functions
สร้าง User flows
ตรวจ Construction ของ UI
```

---

# 17. Frontend Uses Actual Backend

Frontend ไม่ควรเขียนตาม Design เพียงอย่างเดียว

ควร:

```text
อ่าน backend.py
ตรวจชื่อ Functions จริง
ตรวจ Parameters จริง
ตรวจ Return Types จริง
```

Flow ที่เหมาะสม:

```text
Read design
→ Read backend
→ Build UI
→ Import and validate
```

เพราะ Backend Implementation อาจแตกต่างจาก Design

---

# 18. Context7 MCP

Frontend Engineer และ Engineering Lead ใช้:

```text
Context7 MCP
```

เป้าหมาย:

```text
ค้น Gradio Documentation ล่าสุด
ตรวจ API signatures
ตรวจ Components
ตรวจ Arguments
ตรวจ Breaking changes
```

MCP ช่วยลดปัญหา:

```text
Model จำ API รุ่นเก่า
ใช้ kwargs ที่ถูกยกเลิก
เรียก Method ที่เปลี่ยนชื่อ
```

---

# 19. MCP Mental Model

```text
MCP Server
= External capability provider

Context7
= Documentation provider

Agent
= Consumer of documentation tools
```

Flow:

```text
Agent needs current API
→ Calls MCP tool
→ Receives documentation
→ Implements code
```

---

# 20. MCP ไม่รับประกัน Correctness

แม้ Agent ค้น Documentation แล้ว ยังอาจ:

```text
อ่านผิด
เลือก Version ผิด
ผสม API หลายรุ่น
ใช้ Argument ผิดบริบท
สร้าง Code ที่ Import ผ่านแต่ใช้งานจริงพัง
```

ดังนั้น:

```text
Current documentation
≠
Correct implementation
```

ยังต้องมี Runtime Validation

---

# 21. MCP Monkey Patch

โปรเจกต์มี:

```text
patch.py
```

และ Import แบบ Side Effect:

```python
import engineering_team.patch
```

Patch แก้ปัญหา Tool Names ของ MCP ใน CrewAI เวอร์ชันที่ Course ใช้

Mental Model:

```text
CrewAI internal behavior
→ incompatible with MCP tool names
→ monkey patch adjusts behavior
```

---

# 22. Monkey Patch Risks

Monkey Patch:

```text
แก้ Internal Library Behavior
โดยไม่เปลี่ยน Source Package โดยตรง
```

ข้อดี:

```text
แก้ปัญหาได้เร็ว
ไม่ต้อง Fork Library
ใช้กับ Course Version ได้
```

ความเสี่ยง:

```text
ผูกกับ Internal Implementation
Upgrade แล้วอาจพัง
Debug ยาก
ผู้พัฒนารุ่นหลังอาจไม่รู้ว่ามี Patch
```

ควรบันทึก:

```text
เหตุผลของ Patch
CrewAI Version
Issue ที่เกี่ยวข้อง
เงื่อนไขในการลบ Patch
```

---

# 23. Frontend Validation

Frontend ต้องสร้าง:

```text
_validate.py
```

เพื่อ:

```text
Import app.py
Construct Gradio Blocks
Exit successfully
```

ไม่ควรเรียก:

```python
demo.launch()
```

เพราะ `.launch()` จะเปิด Server และ Block Process

---

# 24. Construction Validation

`_validate.py` ช่วยจับ:

```text
Syntax errors
Import errors
Missing backend symbols
Invalid Gradio arguments
Failure during Blocks construction
```

แต่ยังไม่ตรวจ:

```text
Buttons ทำงานจริง
Callbacks ถูกต้อง
State updates ถูกต้อง
Layout ถูกต้อง
User workflow สำเร็จ
```

ดังนั้น:

```text
UI constructs successfully
≠
UI functions correctly
```

---

# 25. Test Engineer

Test Engineer รับ:

```text
Design Context
+
Backend Task Context
+
Shared Files
```

แล้วสร้าง:

```text
test_backend.py
test_summary.md
```

ใช้:

```python
unittest
```

และห้ามใช้ Third-party Test Framework

---

# 26. Test Engineer Responsibilities

```text
อ่าน Design
อ่าน Backend
สร้าง Test Cases
รัน Tests
วิเคราะห์ Failures
แก้ Backend เมื่อพบ Defect
รัน Tests ซ้ำ
สรุปผล
```

Mental Model:

```text
Implementation
→ Independent verification
→ Repair
```

---

# 27. Test Engineer Can Modify Backend

ข้อดี:

```text
แก้ Bug ได้ทันที
ลด Handoff กลับไป Backend Engineer
Workflow สั้นลง
```

ความเสี่ยง:

```text
Test Engineer เปลี่ยน Implementation
Tests และ Code ถูกแก้โดย Agent เดียวกัน
Frontend อาจไม่เข้ากับ Backend ใหม่
บทบาท Reviewer กับ Implementer ทับซ้อนกัน
```

---

# 28. Self-validating Test Risk

ถ้า Test Engineer แก้ทั้ง:

```text
Backend
+
Tests
```

อาจเกิด:

```text
ทำ Tests ให้ง่ายลง
แทนการแก้ Code ให้ตรง Requirements
```

Mental Model:

```text
ผู้สอบ
แก้ทั้งข้อสอบและคำตอบ
```

วิธีลดความเสี่ยง:

```text
Hidden tests
Developer-owned acceptance tests
Test file ownership
Independent validator
Contract checks
```

---

# 29. Sequential Process

Tasks ทำตามลำดับ:

```text
Design
→ Backend
→ Frontend
→ Tests
```

ข้อดี:

```text
Dependency ชัดเจน
ไม่มี Concurrent file writes
Debug ลำดับง่าย
```

ข้อจำกัด:

```text
Latency สูง
Agent ท้าย Workflow สามารถแก้ Artifact ก่อนหน้า
ไม่มี Automatic rollback
ไม่มี Final revalidation ทุก Artifact
```

---

# 30. Task Context

Explicit Context:

```text
design_task
→ code_task

design_task + code_task
→ frontend_task

design_task + code_task
→ test_task
```

Task Context ส่ง:

```text
คำอธิบาย
รายงาน
Task outputs
```

ไม่จำเป็นต้องส่ง Source Files ทั้งหมด เพราะ Agents สามารถอ่าน Shared Sandbox ได้

---

# 31. Shared Filesystem

Agents ใช้:

```text
engineering_team/sandbox/
```

ร่วมกัน

Artifacts เช่น:

```text
design.md
backend.py
app.py
_validate.py
test_backend.py
test_summary.md
```

Shared Sandbox เป็น Source of Runtime Truth ของ Code

---

# 32. Task Context กับ Shared Files ต่างกันอย่างไร

## Task Context

```text
ส่งข้อมูลที่ Task ก่อนหน้ารายงาน
```

เช่น:

```text
Backend implementation completed
Public API includes deposit and withdraw
```

## Shared Files

```text
เก็บ Implementation จริง
```

เช่น:

```text
backend.py
```

ดังนั้น:

```text
Task Context
= Narrative

Shared Files
= Executable artifacts
```

---

# 33. Shared Filesystem ไม่ใช่ Memory

## Shared Filesystem

```text
Mutable
Explicit
Readable
Executable
```

## CrewAI Memory

```text
Retrieved by relevance
May be summarized
May not always be recalled
```

ดังนั้น:

```text
sandbox/backend.py
≠
CrewAI memory record
```

---

# 34. Collaboration Channels

ระบบนี้มี Collaboration สามประเภท:

```text
Task Context
Shared Files
MCP Documentation
```

Mental Model:

```text
Task Context
= รายงานจากเพื่อนร่วมทีม

Shared Files
= ผลงานจริงบนโต๊ะกลาง

MCP
= คู่มือจากภายนอก
```

แต่ละช่องทางมีบทบาทต่างกัน

---

# 35. Shared Mutable State

Sandbox เป็น Mutable State

Agent แต่ละตัวสามารถ:

```text
อ่านไฟล์
สร้างไฟล์
แก้ไฟล์
เขียนทับไฟล์
รันไฟล์
```

ข้อดี:

```text
ร่วมมือผ่าน Artifacts จริง
ไม่ต้องส่ง Code ทั้งหมดใน Prompt
```

ความเสี่ยง:

```text
Overwrite
Interface drift
Stale assumptions
Hidden changes
No ownership enforcement
```

---

# 36. Logical Overwrite Risk

แม้ Sequential Process ป้องกัน Concurrent Writes

แต่ Agent ที่ทำงานทีหลังอาจเขียนทับงานก่อนหน้า

ตัวอย่าง:

```text
Backend Engineer creates backend.py
Frontend works with API
Test Engineer rewrites backend.py
Frontend now breaks
```

นี่ไม่ใช่ Race Condition เชิงเวลา

แต่เป็น:

```text
Logical Overwrite
```

---

# 37. Interface Drift

Interface Drift เกิดเมื่อ Design, Implementation และ Consumer ไม่ตรงกัน

ตัวอย่าง:

```text
Design:
deposit(account_id, amount)

Backend:
deposit(user_id, value)

Frontend:
deposit(account_id, amount)
```

ผล:

```text
Import ผ่าน
แต่ Callback เรียกผิด Parameter
```

---

# 38. Sources of Interface Drift

```text
Design ไม่ละเอียด
Backend ไม่ทำตาม Design
Frontend ใช้ Design เก่า
Test Engineer เปลี่ยน Backend
Return Type เปลี่ยน
Exception Behavior เปลี่ยน
```

ต้องมี:

```text
Contract Tests
Signature Validation
Schema Validation
Final Integration Run
```

---

# 39. Contract Tests

Contract Tests ตรวจว่า Backend API ตรงตาม Contract

ตัวอย่าง:

```python
import inspect

signature = inspect.signature(Account.deposit)

assert list(signature.parameters) == [
    "self",
    "amount"
]
```

หรือทดสอบ:

```text
Method exists
Parameter names
Return types
Expected exceptions
```

---

# 40. Quality Layers

ระบบ Software Engineering ควรมี:

```text
Design validation
Static analysis
Unit tests
Contract tests
Integration tests
Acceptance tests
Security checks
```

แต่ Lab ปัจจุบันเน้น:

```text
Design
Implementation
Frontend construction
Backend unit tests
```

ยังขาด Gates หลายระดับ

---

# 41. Unit Tests

Unit Tests ตรวจ:

```text
Backend function behavior
Business rules
Edge cases
Exceptions
State changes
```

ตัวอย่าง:

```text
Deposit increases cash
Withdraw rejects insufficient funds
Buy rejects insufficient balance
Sell rejects excessive shares
Portfolio value is correct
```

---

# 42. Tests Passing ไม่เท่ากับ System Correctness

Unit Tests ผ่านไม่ได้พิสูจน์ว่า:

```text
Frontend เรียก Backend ถูก
Gradio callbacks ทำงาน
Design ครบ Requirements
Integration ถูก
Security ปลอดภัย
```

ดังนั้น:

```text
Unit Correctness
≠
System Correctness
```

---

# 43. Missing Final Integration Gate

Workflow ปัจจุบัน:

```text
Frontend validates
→ Test Engineer may modify backend
→ Workflow ends
```

ความเสี่ยง:

```text
Tests pass
แต่
app.py ใช้ Backend API รุ่นเก่า
```

Safer Flow:

```text
Design
→ Backend
→ Frontend
→ Unit Tests
→ Final Frontend Validation
→ Integration Tests
```

---

# 44. Final Integration Validator

ควรเพิ่ม Agent หรือ Deterministic Task:

```text
integration_validation_task
```

หน้าที่:

```text
ตรวจ Required Files
รัน Backend Tests
รัน _validate.py
Import Frontend and Backend
ตรวจ Public API Contract
ตรวจ Application start-up
```

---

# 45. Deterministic Final Gate

ตัวอย่าง:

```python
def validate_project() -> tuple[bool, str]:
    required = [
        "design.md",
        "backend.py",
        "app.py",
        "_validate.py",
        "test_backend.py"
    ]

    for filename in required:
        if not exists(filename):
            return False, f"Missing {filename}"

    tests = run("test_backend.py")
    if tests.exit_code != 0:
        return False, tests.stderr

    frontend = run("_validate.py")
    if frontend.exit_code != 0:
        return False, frontend.stderr

    return True, "Project validation passed"
```

---

# 46. Sandbox Reset

ก่อนเริ่ม Crew:

```python
reset_sandbox()
```

Flow:

```text
Delete active sandbox
→ Create new sandbox
→ Initialize UV project
→ Add Gradio dependency
```

ข้อดี:

```text
Clean environment
No stale artifacts
Reproducible starting point
```

ข้อเสีย:

```text
Previous output lost
Dependency setup repeated
Requires network
Debug history removed
```

---

# 47. Active Sandbox กับ Reference Sandboxes

## Active Sandbox

```text
sandbox/
```

ถูกสร้างใหม่ตอน Run

## Reference Sandboxes

```text
sandbox_gpt/
sandbox_claude/
sandbox_mixed/
```

ใช้เปรียบเทียบผลจาก Model configurations

ไม่ควรสับสนว่า Reference Sandboxes คือ Current Run State

---

# 48. Artifact Versioning

ก่อน Reset ควร Archive:

```text
sandbox/
→ runs/<run_id>/
```

ตัวอย่าง:

```text
runs/2026-07-24-001/
├── design.md
├── backend.py
├── app.py
├── test_backend.py
└── validation.json
```

ช่วย:

```text
เปรียบเทียบ Runs
ตรวจ Repair History
กู้ Code
Audit
```

---

# 49. Sandbox Tools

Engineers ใช้:

```text
List Sandbox Files
Read Sandbox File
Write Sandbox File
Run Sandbox Python File
```

Coding Loop:

```text
Inspect
→ Read
→ Write
→ Execute
→ Observe
→ Fix
```

---

# 50. Tool Cache

Tool Cache ถูกปิด เพราะ Shared Files เปลี่ยนตลอด

ตัวอย่าง:

```text
Read backend.py
→ Version 1

Test Engineer modifies backend.py

Read backend.py
→ Must return Version 2
```

ถ้า Cache:

```text
Agent อาจเห็น Version 1
```

หลัก:

```text
Mutable state
→ avoid stale caching
```

---

# 51. Code Execution Runtime

Run Tool ใช้ Docker และ UV Runtime

Conceptual Command:

```bash
docker run --rm \
  -v <sandbox>:/workspace \
  -w /workspace \
  ghcr.io/astral-sh/uv:python3.13-bookworm-slim \
  uv run <filename>
```

Runtime นี้ช่วย:

```text
ใช้ Python 3.13
ใช้ Project Dependencies
แยก Process จาก Crew Runtime
```

---

# 52. Security Risks from Lab 4

ความเสี่ยงเดิมยังอยู่:

```text
Path traversal
Network access
Writable bind mount
No CPU limit
No memory limit
No PID limit
stdout-only feedback
Missing exit code
```

Lab นี้มี Agents หลายตัวใช้ Tools เดียวกัน

จึงเพิ่มจำนวน Actors ที่สามารถเปลี่ยน Shared State

---

# 53. Path Traversal

หาก Write Tool รับ:

```text
../source.py
```

โดยไม่ตรวจ Resolved Path อาจเขียนนอก Sandbox

ต้องใช้:

```python
def safe_path(filename: str) -> Path:
    root = SANDBOX_DIR.resolve()
    candidate = (root / filename).resolve()

    if candidate != root and root not in candidate.parents:
        raise ValueError("Path escapes sandbox")

    return candidate
```

---

# 54. File Ownership

ปัจจุบัน Agent ที่มี Write Tool สามารถเขียนไฟล์ใดก็ได้

Prompt กำหนดหน้าที่ แต่ Tool ไม่ได้บังคับ

ควรกำหนด:

```text
Backend Engineer
→ backend.py

Frontend Engineer
→ app.py, _validate.py

Test Engineer
→ test_backend.py, test_summary.md
```

---

# 55. Controlled Backend Repair

Test Engineer อาจต้องแก้ Backend จริง

แทนให้ Write ได้ทุกอย่าง ควรใช้:

```text
Propose patch
→ Validate patch
→ Run backend tests
→ Run frontend validation
→ Apply patch
```

ช่วยรักษา:

```text
Change history
Reviewability
Integration safety
```

---

# 56. Patch-based Editing

แทนการเขียนทับไฟล์ทั้งหมด:

```text
Write complete backend.py
```

ควรใช้:

```text
Apply targeted diff
```

ข้อดี:

```text
เห็นสิ่งที่เปลี่ยน
ลด accidental overwrite
Review ง่าย
Rollback ง่าย
```

---

# 57. Structured Execution Result

Run Tool ควรคืน:

```python
class ExecutionResult(BaseModel):
    success: bool
    exit_code: int | None
    stdout: str
    stderr: str
    timed_out: bool
    duration_seconds: float
```

ช่วย Engineers แยก:

```text
No output
จาก
Execution failure
```

---

# 58. Requirements Coverage

Design และ Tests ต้องครอบคลุม Requirements ทั้งหมด

ควรสร้าง Matrix:

```text
Requirement
→ Design component
→ Backend function
→ Test case
→ Frontend flow
```

ตัวอย่าง:

```text
Deposit cash
→ Account.deposit
→ test_deposit
→ Deposit tab
```

---

# 59. Traceability Matrix

```text
| Requirement | Design | Code | Test | UI |
|-------------|--------|------|------|----|
| Deposit | Account.deposit | backend.py | test_deposit | Deposit form |
| Buy shares | Account.buy | backend.py | test_buy | Buy form |
```

ช่วยตรวจว่า Feature ไม่ตกหล่น

---

# 60. Design Review Gate

ก่อน Backend เริ่มควรตรวจ:

```text
ทุก Requirement ถูก Mapping
Public API ชัดเจน
Data model ชัดเจน
Error behavior ชัดเจน
Test strategy ชัดเจน
Security assumptions ชัดเจน
```

หาก Design ไม่ผ่านให้ Lead แก้ก่อนเริ่ม Implementation

---

# 61. Error Propagation

```text
Incorrect requirements interpretation
        ↓
Incorrect design
        ↓
Incorrect backend
        ↓
Frontend implements wrong workflow
        ↓
Tests validate wrong behavior
```

Multi-Agent ไม่ได้หยุด Error Propagation

มันอาจขยาย Error ผ่านหลาย Artifacts

---

# 62. Independent Quality Gates

ควรใช้ Gates ที่ไม่ได้พึ่งความคิดเห็นของ Agent เพียงอย่างเดียว:

```text
Schema validation
File existence checks
AST checks
Unit tests
Contract tests
Integration tests
Security scan
```

Mental Model:

```text
Agents produce
Systems verify
```

---

# 63. Multi-Agent Value

การแยก Agents มีประโยชน์เมื่อ:

```text
บทบาทมี Context ต่างกัน
Tools ต่างกัน
Output ต่างกัน
Quality criteria ต่างกัน
```

ตัวอย่าง:

```text
Lead
→ Architecture reasoning

Backend
→ Domain implementation

Frontend
→ UI framework knowledge

Tester
→ Adversarial verification
```

---

# 64. Multi-Agent Costs

เพิ่ม Agents ทำให้เพิ่ม:

```text
Model calls
Latency
Cost
Context transfer
Artifacts
Failure points
Debug complexity
```

ดังนั้น:

```text
More agents
≠
Better software automatically
```

---

# 65. When Single Agent May Be Better

Single Agent + Deterministic Harness อาจเหมาะเมื่อ:

```text
Project เล็ก
ไฟล์น้อย
Interface ง่าย
Token budget จำกัด
Workflow ชัด
```

Engineering Team เหมาะขึ้นเมื่อ:

```text
มีหลาย Components
บทบาทเฉพาะทาง
หลาย Tool sets
ต้องมี Independent Review
```

---

# 66. Prompt Instructions vs Runtime Enforcement

Prompt อาจบอก:

```text
Backend Engineer must not edit app.py
```

แต่หาก Write Tool อนุญาต Agent ยังสามารถแก้ได้

ดังนั้น:

```text
Prompt policy
≠
Security boundary
```

Runtime Tool ต้องตรวจ Ownership

---

# 67. Tools and Authority

Agents เสนอ:

```text
สร้างไฟล์
แก้ไฟล์
รันไฟล์
```

Application ต้องถือ Authority เหนือ:

```text
Allowed paths
Allowed file types
Allowed owners
Runtime limits
Network access
Artifact publication
```

หลักจาก Week 2 ยังคงใช้:

```text
LLM proposes actions
Application authorizes actions
```

---

# 68. Testing Strategy

## Test 1: Interface Drift

เปลี่ยน Method Signature ใน Backend แล้วตรวจว่า Contract Test จับได้หรือไม่

## Test 2: Backend Repair Breaks Frontend

ให้ Test Engineer แก้ Backend แล้วรัน `_validate.py` ซ้ำ

## Test 3: Missing Requirement

ลบ Feature หนึ่งจาก Design แล้วตรวจ Requirements Coverage Gate

## Test 4: Invalid Gradio API

ให้ Frontend ใช้ Argument ที่ล้าสมัย แล้วดูว่า Context7 และ `_validate.py` ตรวจพบหรือไม่

## Test 5: File Ownership

ให้ Frontend Engineer พยายามแก้ `backend.py` แล้วตรวจ Tool ปฏิเสธ

## Test 6: Path Traversal

ลองเขียน `../escape.py`

## Test 7: Backend Tests Pass, UI Fails

สร้างสถานการณ์ที่ Unit Tests ผ่านแต่ Frontend Import พัง

## Test 8: Sandbox Reset

ตรวจว่า Run เดิมถูก Archive ก่อน Reset

## Test 9: MCP Failure

ปิด Network หรือ MCP Server แล้วดูว่า Agent มี Fallback หรือไม่

## Test 10: Docker Unavailable

ตรวจ Error Handling เมื่อ Docker Daemon ไม่พร้อม

---

# 69. Suggested Exercise: Structured Design

สร้าง Pydantic Contract สำหรับ:

```text
Files
Owners
Classes
Functions
Parameters
Return types
Exceptions
```

แล้วให้ Lead คืน Structured Output แทน Markdown อย่างเดียว

---

# 70. Suggested Exercise: Contract Generator

ใช้ Structured Design สร้าง:

```text
backend interface stubs
frontend binding checklist
test skeletons
```

แบบ Deterministic

---

# 71. Suggested Exercise: Final Integration Task

เพิ่ม:

```text
integration_task
```

หลัง `test_task`

ต้องรัน:

```text
Backend unit tests
Frontend construction validation
Contract tests
Application import test
```

---

# 72. Suggested Exercise: File Ownership Tool

สร้าง Write Tools แยก:

```text
Write Backend File
Write Frontend File
Write Test File
```

แต่ละ Tool อนุญาต Path ต่างกัน

---

# 73. Suggested Exercise: Change Log

ทุกครั้งที่ Agent แก้ไฟล์ให้บันทึก:

```text
Agent
Timestamp
File
Previous hash
New hash
Reason
```

ช่วย Audit และ Rollback

---

# 74. Suggested Exercise: Independent Acceptance Tests

เตรียม Tests ที่ Agents มองไม่เห็นล่วงหน้า

ตรวจ:

```text
Business rules
Historical holdings
Portfolio calculations
Invalid operations
Integration behavior
```

---

# 75. Suggested Exercise: Final Reviewer

เพิ่ม Reviewer ที่ไม่มี Write Tool

มีสิทธิ์:

```text
Read files
Run tests
Run validation
Report defects
```

แต่ไม่มีสิทธิ์แก้ Code

ช่วยแยก:

```text
Builder
ออกจาก
Approver
```

---

# 76. Patterns Learned

## Specialist Engineering Agents

```text
Lead
Backend
Frontend
Tester
```

## Design-first Development

```text
Requirements
→ Design
→ Implementation
```

## Shared Artifact Collaboration

```text
Agent A writes file
→ Agent B consumes file
```

## Task Context Transfer

```text
Task output
→ Next task instructions
```

## External Documentation

```text
MCP
→ Current API knowledge
```

## Independent Testing

```text
Implementation
→ Test Engineer
```

## Integration Gate Pattern

```text
Backend tests
+
Frontend validation
+
Contract tests
```

---

# 77. Misconceptions Corrected

## ความเข้าใจคลาดเคลื่อนที่ 1

> หลาย Coding Agents ทำให้ระบบถูกต้องขึ้นโดยอัตโนมัติ

**ข้อเท็จจริง:**
หลาย Agents เพิ่มความเชี่ยวชาญ แต่เพิ่ม Interface และ Coordination Risks

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> Design.md เป็น Contract ที่บังคับได้

**ข้อเท็จจริง:**
Markdown เป็น Human-readable Guidance ไม่ใช่ Machine-enforced Contract

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> Shared Sandbox คือ Shared Memory

**ข้อเท็จจริง:**
Sandbox เป็น Mutable Filesystem ส่วน Memory เป็น Context Retrieval

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> Unit Tests ผ่านแปลว่า Application เสร็จ

**ข้อเท็จจริง:**
ยังต้องตรวจ Frontend, Integration, Requirements และ Security

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> MCP ทำให้ Agent ใช้ API ได้ถูกต้อง

**ข้อเท็จจริง:**
MCP ให้ข้อมูลล่าสุด แต่ Agent ยังตีความหรือ Implement ผิดได้

---

## ความเข้าใจคลาดเคลื่อนที่ 6

> Sequential Process ป้องกัน Agent เขียนทับกัน

**ข้อเท็จจริง:**
ป้องกัน Concurrent Race แต่ไม่ป้องกัน Logical Overwrite

---

## ความเข้าใจคลาดเคลื่อนที่ 7

> Test Engineer เป็น Reviewer อิสระเต็มรูปแบบ

**ข้อเท็จจริง:**
Test Engineer มี Write Tool และแก้ Backend ได้ จึงยังมี Self-validation Risk

---

## ความเข้าใจคลาดเคลื่อนที่ 8

> Prompt บอก File Ownership ก็เพียงพอ

**ข้อเท็จจริง:**
Ownership ต้องถูกบังคับใน Tool Code

---

# 78. Risks Identified

## 78.1 Interface Drift

Design, Backend, Frontend และ Tests ใช้ Interface ไม่ตรงกัน

## 78.2 Logical Overwrite

Agent หลังแก้ Artifact ที่ Agent ก่อนสร้าง

## 78.3 Missing Final Integration

Backend เปลี่ยนหลัง Frontend Validation

## 78.4 Test Self-validation

Test Engineer แก้ทั้ง Code และ Tests

## 78.5 Poor Design Propagation

Design ผิดส่งผลต่อทุก Stage

## 78.6 MCP Dependency

Documentation Server อาจล่มหรือเปลี่ยน

## 78.7 Monkey Patch Fragility

Patch อาจพังเมื่อ Upgrade CrewAI

## 78.8 Sandbox Security

Path, Network และ Resource Risks ยังคงอยู่

## 78.9 Artifact Loss

Sandbox ถูก Reset และลบผลเดิม

## 78.10 Tool Permission Overreach

หลาย Agents เขียนไฟล์ใดก็ได้

---

# 79. Production Improvements

```text
Structured system design
Requirements traceability matrix
Design review gate
File ownership enforcement
Patch-based editing
Contract tests
Independent hidden tests
Final integration validation
Structured execution results
Path validation
Network isolation
Resource limits
Artifact versioning
Change logs
Read-only reviewer
Security scanning
Human approval
```

---

# 80. Safer Engineering Architecture

```text
Requirements
    ↓
Requirements Validator
    ↓
Structured Engineering Design
    ↓
Design Review Gate
    ↓
Backend Engineer
    ↓
Backend Static Checks
    ↓
Frontend Engineer
    ↓
Frontend Construction Test
    ↓
Test Engineer
    ↓
Unit and Contract Tests
    ↓
Integration Validator
    ↓
Security Scan
    ↓
Read-only Reviewer
    ↓
Human Approval
    ↓
Versioned Release Artifact
```

---

# 81. Connection to Week 3 Lab 4

Lab 4:

```text
One Coder Agent
→ Write
→ Run
→ Fix
```

Lab 5:

```text
Multiple Engineering Agents
→ Design
→ Build
→ Integrate
→ Test
```

Lab 4 เน้น Environment Feedback Loop

Lab 5 เพิ่ม:

```text
Role specialization
Shared artifacts
Interface contracts
Cross-agent integration
```

---

# 82. Connection to Week 3 Lab 3

Stock Picker ใช้:

```text
Structured Intermediate Outputs
Task Context
Hierarchical coordination
```

Engineering Team ใช้:

```text
Task Context
Shared Files
Sequential process
MCP documentation
```

ทั้งสองแสดงว่า Agents ร่วมมือผ่าน Artifacts มากกว่าการสนทนาโดยตรง

---

# 83. Connection to Week 2

Week 2 สอน:

```text
Code controls invariants
Structured outputs control boundaries
Guardrails control risk
```

Engineering Team ยืนยันหลักเดียวกัน:

```text
Agents write code
System validates interfaces
Tests validate behavior
Tools enforce permissions
```

---

# 84. Week 3 Complete Mental Model

```text
Agent
= Specialist worker

Task
= Assignment

Crew
= Team runtime

Process
= Work sequence

Context
= Explicit task dependency

Memory
= Retrieved historical context

Tool
= External capability

Sandbox
= Shared mutable working state

MCP
= External tool/documentation source

Structured Output
= Machine-readable artifact contract
```

---

# 85. Week 3 Progression

```text
Lab 1 — Debate
Agent–Task separation

Lab 2 — Financial Researcher
External tools and task context

Lab 3 — Stock Picker
Manager, hierarchical process, memory and structured outputs

Lab 4 — Coder
External working state and code execution

Lab 5 — Engineering Team
Multi-agent software development and shared artifacts
```

---

# 86. Final Lessons

## Lesson 1

Multi-Agent Software Engineering ต้องออกแบบ Interfaces ชัดเจนกว่าการใช้ Agent ตัวเดียว

## Lesson 2

Design Artifact เป็นจุดเริ่มต้นของ Collaboration แต่ต้องทำให้ Machine-checkable หากต้องการบังคับ Contract

## Lesson 3

Task Context และ Shared Filesystem ทำหน้าที่ต่างกัน

## Lesson 4

Shared Files ช่วยลด Context Size แต่สร้าง Mutable State Risks

## Lesson 5

Sequential Process ป้องกัน Concurrent Writes แต่ไม่ป้องกัน Logical Overwrite

## Lesson 6

MCP ช่วยลดความล้าสมัยของ Documentation แต่ไม่แทน Runtime Validation

## Lesson 7

Unit Tests ตรวจ Component ไม่ได้ตรวจ System Integration ทั้งหมด

## Lesson 8

Backend Change หลัง Frontend Validation ต้องทำให้ Frontend ถูก Validate ซ้ำ

## Lesson 9

Test Engineer ที่แก้ทั้ง Code และ Tests ยังไม่ใช่ Reviewer อิสระอย่างสมบูรณ์

## Lesson 10

File Ownership ต้องบังคับใน Tool Implementation ไม่ใช่ Prompt อย่างเดียว

## Lesson 11

Final Integration Gate เป็นสิ่งจำเป็นก่อนถือว่า Software Artifact เสร็จ

## Lesson 12

การเพิ่ม Agents ควรทำเมื่อ Specialization ให้คุณค่ามากกว่า Coordination Cost

---

# 87. Memory Summary

```text
Week 3 Lab 5 สร้าง CrewAI Engineering Team

Agents:
Engineering Lead
Backend Engineer
Frontend Engineer
Test Engineer

Workflow:
Requirements
→ Design
→ Backend
→ Frontend
→ Tests

Process:
Process.sequential

Engineering Lead:
สร้าง design.md
กำหนด files, classes,
functions และ signatures

Backend Engineer:
สร้าง business logic
ใช้ Python standard library

Frontend Engineer:
สร้าง Gradio app.py
และ _validate.py

Test Engineer:
สร้าง unittest
ทดสอบและแก้ Backend

Collaboration channels:
Task Context
Shared Filesystem
MCP Documentation

Task Context:
ส่งรายงานและผลของ Task ก่อนหน้า

Shared Files:
เก็บ Implementation จริง

MCP:
ให้ Documentation ล่าสุด

Context7:
ใช้ตรวจ Gradio APIs

Shared Sandbox:
เป็น Mutable Working State

ไม่ใช่:
CrewAI Memory

Risks:
Interface drift
Logical overwrite
Missing integration gate
Self-validating tests
File permission overreach
Sandbox security

Frontend Validation:
ตรวจ Import และ Blocks construction

แต่ไม่ตรวจ:
User interaction
Callbacks
End-to-end behavior

Unit Tests:
ตรวจ Backend

แต่ไม่รับประกัน:
Frontend
Integration
Security
Requirement coverage

จุดอ่อนสำคัญ:
Test Engineer อาจแก้ Backend
หลัง Frontend ถูก Validate

ควรเพิ่ม:
Final integration task

Structured Design:
ควรใช้ Pydantic Contract

File Ownership:
ต้องบังคับใน Tool Code

Production ควรเพิ่ม:
Design review
Contract tests
Integration tests
Hidden tests
Artifact versioning
Change logs
Read-only reviewer
Human approval
```

---

# 88. Week 3 Completion Summary

```text
CrewAI ช่วยจัดโครงสร้าง:
Agents
Tasks
Crews
Processes
Context
Memory
Tools
Structured Outputs

แต่ระบบจริงยังต้องควบคุม:
Requirements
Data quality
Contracts
Permissions
Side effects
Code execution
Testing
Integration
Security
Observability
```

แก่นรวมของ Week 3 คือ:

> CrewAI ช่วยให้เราอธิบายระบบในรูปของ “ทีม คน และงาน” ได้ง่ายขึ้น แต่ความน่าเชื่อถือของระบบไม่ได้เกิดจากการเพิ่ม Agent จำนวนมาก ความน่าเชื่อถือเกิดจาก Contract ที่ชัดเจน, Tool Permissions ที่บังคับได้, Artifacts ที่ตรวจสอบได้ และ Quality Gates ที่ไม่พึ่งคำยืนยันของ Agent เพียงอย่างเดียว

คำถามสำคัญก่อนเข้าสู่ Week ถัดไปคือ:

> เมื่อ Workflow เริ่มซับซ้อนเกินกว่าลำดับ Task แบบตรงไปตรงมา เราจะใช้ Graph, State และ Conditional Routing เพื่อควบคุมเส้นทางของ Agent อย่างชัดเจน ตรวจสอบได้ และสามารถย้อนกลับหรือแก้ไขเฉพาะขั้นตอนได้อย่างไร?
