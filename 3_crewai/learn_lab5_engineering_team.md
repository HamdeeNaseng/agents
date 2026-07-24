# Week 3 — Lab 5: CrewAI Engineering Team

โปรเจกต์:

```text
3_crewai/reference/engineering_team/
```

Lab นี้ต่อยอดจาก Coder Lab จาก Agent ตัวเดียว ไปเป็น **ทีมพัฒนาซอฟต์แวร์หลายบทบาท** ที่ร่วมกันสร้าง Backend, Frontend และ Unit Tests ภายใน Sandbox เดียวกัน โดยมี Engineering Lead ออกแบบระบบก่อนเริ่มเขียนโค้ด. ([GitHub][1])

โปรเจกต์ปัจจุบันล็อก:

```text
CrewAI 1.14.4
Python >=3.10,<3.14
Gradio >=6.14.0
```

เอกสาร CrewAI ออนไลน์ปัจจุบันเป็นเวอร์ชัน 1.15.5 จึงต้องยึด Source Code ของหลักสูตรเป็นหลัก โดยเฉพาะส่วน MCP monkey patch ซึ่งเขียนมาแก้ปัญหาเฉพาะ CrewAI 1.14.4. ([GitHub][2])

---

## Learning Objectives

หลังจบ Lab นี้ควรอธิบายได้ว่า:

1. Engineering Lead, Backend Engineer, Frontend Engineer และ Test Engineer แบ่งงานกันอย่างไร
2. Shared Sandbox แตกต่างจาก Task Context อย่างไร
3. Design Artifact ทำหน้าที่เป็น Contract ระหว่าง Agents อย่างไร
4. Sequential Process ควบคุมลำดับการพัฒนาอย่างไร
5. MCP และ Context7 ช่วยตรวจ API Documentation อย่างไร
6. Backend, Frontend และ Tests ถูกสร้างเป็นหลายไฟล์ร่วมกันอย่างไร
7. ทำไมการที่ Unit Tests ผ่านยังไม่รับประกันว่า UI และระบบรวมทำงานถูกต้อง
8. Mutable Shared Files สร้างความเสี่ยงเรื่อง Interface Drift อย่างไร
9. Quality Gate ควรถูกเพิ่มตรงไหน
10. Secure Code Execution จาก Lab 4 ยังเกี่ยวข้องกับ Lab นี้อย่างไร

---

# 1. Architecture Overview

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

Crew ใช้:

```python
process=Process.sequential
```

ดังนั้น Tasks ทำตามลำดับ:

```text
design_task
→ code_task
→ frontend_task
→ test_task
```

และเปิดทั้ง `verbose=True` กับ `tracing=True` เพื่อดู Agent Runs, Tasks และ Tool Calls. ([GitHub][3])

---

# 2. Four-Agent Engineering Team

ระบบมี Agents สี่บทบาท:

```text
Engineering Lead
Backend Engineer
Frontend Engineer
Test Engineer
```

แต่ละ Agent ใช้ Tools และ Models ต่างกันตามหน้าที่ โดย Engineering Lead ใช้ `openai/gpt-5.5` ส่วน Engineers ใช้ `openai/gpt-5.4-mini` ใน Configuration ปัจจุบัน. ([GitHub][4])

---

## Engineering Lead

Engineering Lead รับ Requirements ระดับสูงแล้วออกแบบ:

```text
Modules
Classes
Functions
Method signatures
Agent assignments
Interfaces ระหว่างไฟล์
```

Lead ถูกสั่งว่า **ห้ามเขียน Implementation Code** แต่ต้องสร้าง Detailed Design และระบุให้ชัดว่า Backend, Frontend และ Test Engineer ต้องทำส่วนใด. ([GitHub][4])

Mental Model:

```text
Engineering Lead
= System Architect

design.md
= Architecture Contract
```

Lead จึงไม่ได้สร้าง Product โดยตรง แต่ลดความกำกวมก่อน Engineers เริ่มแก้ Shared Files

---

## Backend Engineer

Backend Engineer มีหน้าที่เขียน Python Backend ตาม Design โดยใช้เฉพาะ Python Standard Library:

```text
No Gradio
No third-party backend packages
No frontend code
```

Agent ได้รับ Sandbox Tools เพื่ออ่าน เขียน และรันไฟล์ และต้องตรวจ Code ของตัวเองก่อนส่งต่อ. ([GitHub][4])

Mental Model:

```text
Design Contract
→ Backend implementation
→ Stable public API
```

สิ่งสำคัญคือ Backend ต้องสร้าง Interface ที่ Frontend และ Tests สามารถ Import ใช้ได้ตรงกัน

---

## Frontend Engineer

Frontend Engineer สร้าง:

```text
app.py
```

ซึ่งเป็น Gradio UI สำหรับ Backend และต้องสร้าง Validation Script เช่น:

```text
_validate.py
```

เพื่อ Import และ Construct Gradio `Blocks` โดยไม่เรียก `.launch()` เพราะ `.launch()` จะ Block Process จนชน Timeout. ([GitHub][4])

Frontend Agent ได้ทั้ง:

```text
Sandbox Tools
Context7 MCP
```

เพื่ออ่าน Backend Files, เขียน UI, รัน Validation และค้น API ของ Gradio รุ่นล่าสุด. ([GitHub][5])

---

## Test Engineer

Test Engineer สร้าง Unit Tests สำหรับ Backend โดยใช้:

```python
unittest
```

เท่านั้น ห้ามใช้ `pytest` หรือ Third-party Packages และมีสิทธิ์แก้ Backend หากพบ Defect จาก Tests. ([GitHub][4])

Mental Model:

```text
Backend
→ Independent tests
→ Detect defect
→ Repair backend
→ Rerun tests
```

อย่างไรก็ตาม Test Engineer ถูกสั่งว่าอย่าแก้สิ่งที่ทำให้ `app.py` พัง แต่หลัง Test Engineer แก้ Backend แล้วไม่มี Final Frontend Validation Task อีกครั้ง นี่เป็นช่องว่างด้าน Integration Quality ของ Workflow ปัจจุบัน. ([GitHub][3])

---

# 3. Four Tasks

## Task 1 — `design_task`

```text
Requirements
→ Engineering design
→ sandbox/design.md
```

Design ต้องอธิบาย:

```text
File structure
Modules
Classes
Functions
Method signatures
Responsibilities
Agent assignments
```

และห้ามเขียน Implementation. ([GitHub][3])

---

## Task 2 — `code_task`

```text
design.md
+ requirements
→ Backend Engineer
→ Backend Python files
```

Task ใช้:

```yaml
context:
  - design_task
```

จึงได้รับผลของ Design Task อย่างชัดเจนก่อนเริ่มเขียน Code. ([GitHub][3])

---

## Task 3 — `frontend_task`

```text
design_task
+ code_task
→ Frontend Engineer
→ app.py
→ _validate.py
```

Frontend ได้ Context ทั้ง Design และ Backend Task เพื่อให้ UI ยึดตาม Architecture และ Interface ที่ Backend สร้างจริง. ([GitHub][3])

---

## Task 4 — `test_task`

```text
design_task
+ code_task
→ Test Engineer
→ Unit tests
→ Backend fixes
→ test_summary.md
```

Task ไม่ได้ระบุ `frontend_task` ใน Explicit Context แต่ Engineers ใช้ Filesystem เดียวกัน Test Engineer จึงยังสามารถอ่าน `app.py` ผ่าน Sandbox Tools ได้. อย่างไรก็ดี การส่ง Dependency ผ่าน Shared Files แบบนี้ไม่ชัดเจนเท่าการระบุ Task Context โดยตรง. ([GitHub][3])

---

# 4. สามช่องทางที่ Agents ใช้ร่วมมือกัน

Lab นี้มี Collaboration สามประเภทซ้อนกัน

## 4.1 Task Context

```text
Task Output
→ Context ของ Task ถัดไป
```

ใช้ส่งความหมายและคำอธิบาย เช่น Design หรือผลการทำงานก่อนหน้า

## 4.2 Shared Filesystem

```text
backend.py
app.py
test_backend.py
```

เป็น Artifact จริงที่ Agents ทุกตัวที่มี Sandbox Tools สามารถอ่านหรือแก้ไขได้

## 4.3 MCP Documentation

```text
Context7 MCP
→ Current library/API information
```

ใช้ช่วย Engineering Lead และ Frontend Engineer ตรวจ API ซึ่งต่างจาก Task Context และ Filesystem เพราะ MCP เป็น External Tool Source. CrewAI รองรับการกำหนด MCP Servers ผ่าน `mcps` บน Agent และค้นพบ Tools จาก Server ให้อัตโนมัติ. ([GitHub][5])

Mental Model:

```text
Task Context
= สิ่งที่ทีมก่อนหน้ารายงาน

Shared Files
= สิ่งที่ทีมก่อนหน้าสร้างจริง

MCP
= คู่มือหรือแหล่งความรู้ภายนอก
```

---

# 5. Design Artifact เป็น Interface Contract

Engineering Lead บันทึก:

```text
sandbox/design.md
```

Reference Design ปัจจุบันกำหนดไฟล์หลัก:

```text
backend.py
app.py
test_backend.py
```

พร้อม Interface เช่น `get_share_price()` และแนะนำให้ใช้ `decimal.Decimal` สำหรับเงินและราคาหุ้น เพื่อหลีกเลี่ยงปัญหา Floating-point Precision. ([GitHub][6])

Design ที่ดีควรทำให้ทั้งสาม Engineer เห็นตรงกันว่า:

```text
ชื่อ Class คืออะไร
ชื่อ Method คืออะไร
Parameters คืออะไร
Return Type คืออะไร
Exceptions อะไรที่ต้องเกิด
```

หาก Design ระบุเพียงว่า “สร้างระบบบัญชี” แต่ไม่กำหนด Interface Agents แต่ละตัวอาจสร้าง Code ที่ Import และเรียกกันไม่ได้

---

# 6. Requirements ของ Lab

`main.py` กำหนดโจทย์เป็น Trading Simulation Account Management System ที่ต้องรองรับ:

```text
Create account
Deposit
Withdraw
Buy shares
Sell shares
Portfolio value
Profit/loss
Holdings at a point in time
Transaction history
```

พร้อม Business Rules:

```text
ห้ามถอนจนยอดเงินติดลบ
ห้ามซื้อเกินเงินสดที่มี
ห้ามขายมากกว่าจำนวนหุ้นที่ถือ
```

และมี Price Provider แบบ Fixed Price สำหรับ:

```text
AAPL
TSLA
GOOGL
```

Requirements ถูกส่งผ่าน `{requirements}` ไปยัง Agents และ Tasks ตอน `kickoff()`. ([GitHub][7])

---

# 7. Why the Sandbox Is Reset

ก่อนเริ่ม Crew:

```python
reset_sandbox()
```

Function นี้จะ:

```text
ลบ Sandbox เดิมทั้งหมด
สร้าง Sandbox ใหม่
รัน uv init --bare --python 3.13
รัน uv add gradio
```

จึงทำให้แต่ละ Run เริ่มจาก Project สะอาดและมี Gradio พร้อมใช้งาน. ([GitHub][8])

ข้อดี:

```text
ไม่มีไฟล์จาก Run ก่อนปะปน
ลดความสับสนของ Agent
ทดลองซ้ำได้สะอาดขึ้น
```

ข้อเสีย:

```text
Artifacts เดิมถูกลบ
Debug history หาย
ต้องติดตั้ง/Resolve Gradio ใหม่
ใช้เวลาและ Network เพิ่ม
```

หากต้องการเก็บผลแต่ละ Run ควร Copy หรือ Version Sandbox ก่อน Reset

---

# 8. Sandbox Tools

Backend, Frontend และ Test Engineer ได้ Tools สี่ตัว:

```text
List Sandbox Files
Read Sandbox File
Write Sandbox File
Run Sandbox Python File
```

Tools ใช้ Shared Directory เดียว:

```text
engineering_team/sandbox/
```

และ Tool Cache ถูกปิดเพื่อป้องกันการคืน File State เก่าหลังมีการแก้ไข. ([GitHub][8])

Coding Loop ของแต่ละ Engineer:

```text
List
→ Read
→ Write
→ Run
→ Inspect
→ Fix
```

ต่างจาก Coder Lab ตรงที่ครั้งนี้หลาย Agents ใช้ Mutable Filesystem เดียวกัน

---

# 9. Shared Filesystem: ข้อดีและความเสี่ยง

ข้อดี:

```text
Frontend Import Backend ได้ทันที
Tests Import Backend ได้ทันที
ทุก Agent เห็น Artifacts ล่าสุด
ไม่ต้องส่ง Source Code ทั้งหมดผ่าน Prompt
```

ความเสี่ยง:

```text
Agent หลังเขียนทับงาน Agent ก่อน
ชื่อไฟล์ชนกัน
Interface ถูกเปลี่ยนโดยไม่แจ้ง
Test Engineer แก้ Backend แล้ว UI พัง
Artifact ที่เคยผ่าน Validation ถูกแทนด้วยเวอร์ชันใหม่
```

เพราะ Process เป็น Sequential จึงไม่มี Concurrent Write Race แต่ยังมี **Logical Overwrite Risk** จาก Agent ที่ทำงานภายหลัง. ([GitHub][5])

---

# 10. Interface Drift

ตัวอย่าง Design ระบุ:

```python
def deposit(account_id: str, amount: Decimal) -> Transaction:
    ...
```

แต่ Backend Engineer อาจสร้าง:

```python
def deposit(user_id: str, value: float) -> None:
    ...
```

Frontend อาจเขียนตาม Design เดิม ขณะที่ Backend จริงใช้ Signature ใหม่

ผล:

```text
Design
≠ Backend
≠ Frontend expectation
≠ Test expectation
```

นี่เรียกว่า **Interface Drift**

การส่ง Context เพียงอย่างเดียวไม่ป้องกันปัญหานี้ ต้องมี Runtime Import Tests หรือ Contract Tests

---

# 11. Context7 MCP

Engineering Lead และ Frontend Engineer เชื่อม:

```python
mcps=["https://mcp.context7.com/mcp"]
```

CrewAI รองรับ MCP ผ่าน `mcps` บน Agent โดย Framework จะค้นพบ Tools จาก MCP Server แล้วทำให้ Agent เรียกใช้ได้เหมือน Tool อื่น. ([GitHub][5])

ใน Lab นี้ Context7 มีเป้าหมายช่วยตรวจ:

```text
Gradio 6 API
Current kwargs
Method signatures
Breaking changes
```

MCP จึงลดความเสี่ยงจากการเขียน Code ตาม API ที่อยู่ใน Model Training Data แต่ล้าสมัย

อย่างไรก็ตาม:

```text
MCP documentation retrieved
≠ Code is correct
```

Agent ยังอาจตีความ Docs ผิดหรือใช้ API หลาย Version ปะปนได้

---

# 12. CrewAI MCP Monkey Patch

`main.py` Import:

```python
import engineering_team.patch
```

เพื่อใช้ Side Effect Patch แก้ Bug ใน CrewAI 1.14.4 ที่ Sanitize ชื่อ MCP Tools ที่มีเครื่องหมาย `-` แล้วส่งชื่อที่ถูกแก้กลับไปยัง Server ทำให้ Tool จริง เช่น `resolve-library-id` ถูกเรียกไม่เจอ. ([GitHub][7])

Patch เก็บชื่อดั้งเดิมของ Tool ไว้สำหรับ Call ฝั่ง Server

ประเด็นสำคัญ:

> Monkey patch เป็น Technical Debt แบบตั้งใจใช้ชั่วคราว

เมื่อ Upgrade CrewAI ต้องตรวจใหม่ว่า Bug ถูกแก้แล้วหรือไม่ เพราะ Patch ที่แก้ Internal Class ของเวอร์ชันเก่าอาจทำให้เวอร์ชันใหม่พัง. ไฟล์ Patch เองระบุให้ Re-verify เมื่อ Upgrade. ([GitHub][9])

---

# 13. Code Execution Runtime

Run Tool ใช้ Container:

```text
ghcr.io/astral-sh/uv:python3.13-bookworm-slim
```

แล้วรัน:

```bash
uv run <filename>
```

ภายใน `/workspace` ซึ่ง Mount จาก Shared Sandbox และมี Timeout 300 วินาที. ([GitHub][8])

การใช้ `uv run` ทำให้ Script ทำงานภายใน Project Environment ที่ `reset_sandbox()` เตรียมไว้ และ Gradio ถูกประกาศเป็น Dependency ของ Sandbox

---

# 14. Security Issues จาก Coder Lab ยังอยู่

Sandbox Tools ของ Engineering Team มีรูปแบบใกล้กับ Coder Lab และยังมีข้อจำกัดสำคัญ:

```text
Filename ไม่มี Safe-path Validation
Run Tool คืน stdout เท่านั้น
ไม่มี stderr
ไม่มี Exit Code
ไม่มี --network none
ไม่มี CPU limit
ไม่มี Memory limit
ไม่มี PID limit
Writable bind mount
```

ข้อจำกัดเหล่านี้เห็นได้จาก Implementation ของ `read`, `write` และ `docker run` ปัจจุบัน. ([GitHub][8])

Lab นี้เสี่ยงเพิ่มขึ้นเพราะมี Agents สามตัวที่สามารถแก้และรันไฟล์ได้ ไม่ใช่ Agent เดียว

---

# 15. Frontend Validation

Frontend Task ต้องสร้าง `_validate.py` ที่:

```text
Import app.py
Construct Gradio Blocks
Exit without calling launch()
```

นี่เป็น Validation ที่ดีระดับหนึ่ง เพราะจับปัญหาเช่น:

```text
Import error
Invalid Gradio argument
Missing Backend symbol
Blocks construction failure
```

แต่ไม่ได้ตรวจ:

```text
Click handlers ทำงานจริงหรือไม่
Layout แสดงถูกต้องหรือไม่
Dark mode ใช้งานดีหรือไม่
User flow ครบหรือไม่
Backend state ถูกต้องหรือไม่
```

Task จึงตรวจได้เพียง **Construction Validation** ไม่ใช่ End-to-End UI Test. ([GitHub][3])

---

# 16. Unit Testing

Reference Output ตัวอย่างหนึ่งรายงานว่า Backend ผ่าน 32 Unit Tests ซึ่งครอบคลุม Account Creation, Deposits, Withdrawals, Buying, Selling, Holdings, Historical Snapshots, Profit/Loss และ Helper Functions. ([GitHub][10])

นี่แสดง Pattern:

```text
Requirements
→ Design
→ Implementation
→ Tests derived from behavior
```

แต่ต้องระวังว่า Test Engineer สามารถแก้ทั้ง Tests และ Backend ได้ จึงมีโอกาสทำให้ Tests ง่ายลงเพื่อให้ผ่านแทนการแก้ Implementation ให้ถูก

---

# 17. Tests Passing ไม่เท่ากับ System Correctness

```text
Backend unit tests pass
```

ไม่ได้พิสูจน์ว่า:

```text
Frontend ยัง Import Backend ได้
Gradio callbacks ใช้ Parameters ถูก
UI เรียก Business Methods ถูก
Design Requirements ครบทั้งหมด
ไม่มี Security Defect
Point-in-time valuation semantics ถูกต้อง
```

จึงควรแยก Quality Layers:

```text
Unit Tests
→ Backend functions

Contract Tests
→ Backend public API

Frontend Validation
→ Gradio construction

Integration Tests
→ Frontend + Backend

Acceptance Tests
→ User requirements
```

---

# 18. จุดอ่อนสำคัญ: ไม่มี Final Integration Gate

Workflow ปัจจุบัน:

```text
Frontend validates app
        ↓
Test Engineer may change backend
        ↓
Workflow ends
```

หาก Test Engineer เปลี่ยนชื่อ Method หรือ Return Type เพื่อแก้ Unit Tests:

```text
Backend tests pass
แต่
app.py อาจพัง
```

Safer Flow ควรเป็น:

```text
Design
→ Backend
→ Frontend
→ Backend Tests
→ Final Integration Validation
```

หรือ:

```text
Test Engineer fixes backend
→ Frontend Engineer reruns _validate.py
→ Acceptance Gate
```

---

# 19. Design Review ยังไม่มี

Engineering Lead สร้าง Design แล้ว Backend เริ่ม Implement ทันที

ยังไม่มี Task สำหรับ:

```text
ตรวจ Requirement Coverage
ตรวจ Interface Consistency
ตรวจ Security Design
ตรวจ Testability
อนุมัติ Design
```

Design ที่ผิดจะส่งผลต่อ Agents ทุกตัว:

```text
Poor design
→ Wrong backend
→ Wrong frontend
→ Tests validate wrong behavior
```

นี่คือ Error Propagation ผ่าน Design Artifact

---

# 20. Shared Artifact Contract

ควรกำหนดให้ `design.md` มีส่วนที่ Machine-checkable เช่น:

```yaml
files:
  - backend.py
  - app.py
  - test_backend.py

backend_api:
  - name: create_account
    parameters:
      - account_id
      - name
      - initial_deposit
    returns: Account
```

หรือใช้ Pydantic Structured Output สำหรับ Design

ข้อดี:

```text
ชื่อไฟล์ตรวจได้
Function signatures ตรวจได้
Agent assignments อ่านได้
Contract Tests สร้างอัตโนมัติได้
```

Lab ปัจจุบันใช้ Markdown Design ซึ่งอ่านง่ายแต่ตรวจแบบ Deterministic ยาก

---

# 21. Tool Permissions

```text
Engineering Lead
→ Context7 MCP

Backend Engineer
→ Sandbox Tools

Frontend Engineer
→ Sandbox Tools + Context7 MCP

Test Engineer
→ Sandbox Tools
```

นี่เป็น Principle of Least Privilege ในระดับหนึ่ง เพราะ Lead ไม่จำเป็นต้องเขียนไฟล์ผ่าน Tool และ Backend/Test Engineer ไม่จำเป็นต้องใช้ Documentation MCP สำหรับ Gradio. ([GitHub][5])

แต่ Backend, Frontend และ Test Engineer ได้ Write Tool แบบเดียวกันทั้งหมด จึงไม่มี File Ownership Enforcement เช่น:

```text
Backend Engineer เขียนได้เฉพาะ backend.py
Frontend Engineer เขียนได้เฉพาะ app.py
Test Engineer เขียนได้เฉพาะ test_*.py
```

Prompt บอกหน้าที่ แต่ Tool Code ไม่บังคับ

---

# 22. File Ownership ควรอยู่ใน Code

ตัวอย่าง Policy:

```python
ALLOWED_FILES = {
    "backend_engineer": {"backend.py"},
    "frontend_engineer": {"app.py", "_validate.py"},
    "test_engineer": {"test_backend.py", "test_summary.md"},
}
```

หาก Test Engineer ต้องแก้ Backend อาจใช้ Controlled Patch Tool:

```text
propose_backend_patch
→ validate tests
→ validate frontend
→ apply patch
```

ดีกว่าให้ทุก Agentเขียนทับทุกไฟล์โดยตรง

---

# 23. วิธีรัน Lab

จาก Project Root:

```powershell
cd 3_crewai\reference\engineering_team
crewai install
docker version
crewai run
```

ต้องมี:

```text
OPENAI_API_KEY
Docker Engine/Desktop
uv
Internet สำหรับ Context7 MCP
```

`reset_sandbox()` จะสร้าง UV Project ใหม่และเพิ่ม Gradio ก่อนเริ่ม Crew. Project Script เชื่อม `crewai run` เข้ากับ `engineering_team.main:run`. ([GitHub][1])

หลังจบตรวจ:

```text
sandbox/design.md
sandbox/backend.py
sandbox/app.py
sandbox/_validate.py
sandbox/test_backend.py
sandbox/test_summary.md
```

ชื่อ Test File จริงอาจต่างไปตามสิ่งที่ Agent สร้าง แต่ Task กำหนดให้มี Test File เดียวและ Summary. ([GitHub][3])

---

# 24. สิ่งที่ควรสังเกตใน Trace

ดูว่า:

```text
Lead เรียก Context7 หรือไม่
Design ระบุ Function Signatures ชัดหรือไม่
Backend เขียนไฟล์กี่รอบ
Backend รัน Code จริงหรือไม่
Frontend อ่าน Backend ก่อนสร้าง UI หรือไม่
Frontend รัน _validate.py หรือไม่
Test Engineer สร้าง Test ครบหรือไม่
Test Engineerแก้ Backend หรือไม่
หลังแก้ Backend มีการ Validate UI ซ้ำหรือไม่
```

คำถามสุดท้ายสำคัญที่สุด เพราะเป็นจุดที่ Workflow ปัจจุบันยังไม่มี Deterministic Final Gate

---

# 25. แบบฝึกหัดแนะนำ

## Exercise 1 — ตรวจ Interface Drift

แก้ Design ให้ Method ชื่อ:

```python
get_portfolio_value()
```

แล้วดูว่า Backend, Frontend และ Tests ใช้ชื่อเดียวกันครบหรือไม่

---

## Exercise 2 — ทำให้ Backend Test ผ่านแต่ UI พัง

ให้ Test Engineer เปลี่ยน Backend Method Signature แล้วตรวจว่า:

```text
Unit tests pass
_validate.py fails
```

จากนั้นเพิ่ม Final Integration Task

---

## Exercise 3 — เพิ่ม Structured Design

สร้าง Pydantic Models:

```python
class FunctionContract(BaseModel):
    name: str
    parameters: list[str]
    returns: str

class ModuleDesign(BaseModel):
    filename: str
    owner: str
    functions: list[FunctionContract]
```

ใช้ Output นี้เป็น Contract ก่อนเขียน Code

---

## Exercise 4 — เพิ่ม Final Validator Agent

```text
Design
→ Backend
→ Frontend
→ Tests
→ Integration Validator
```

Validator ต้องรัน:

```bash
uv run test_backend.py
uv run _validate.py
```

และตรวจว่า Required Files มีครบ

---

## Exercise 5 — File Ownership Enforcement

ปรับ Write Tool ให้รับ Agent Role แล้วปฏิเสธการเขียนไฟล์นอก Ownership

---

## Exercise 6 — Compare Reference Sandboxes

Repository มี:

```text
sandbox_gpt
sandbox_claude
sandbox_mixed
```

ใช้เปรียบเทียบ Design, Backend, UI และ Test Quality จาก Model Configurations ต่างกัน โดยอย่าสับสนกับ Active `sandbox/` ที่ถูกสร้างใหม่ตอนรัน. ([GitHub][1])

---

# 26. Safer Engineering Pipeline

```text
Requirements
    ↓
Requirements Validation
    ↓
Structured System Design
    ↓
Design Review Gate
    ↓
Backend Implementation
    ↓
Backend Static Checks
    ↓
Frontend Implementation
    ↓
Frontend Construction Test
    ↓
Independent Backend Tests
    ↓
Contract Tests
    ↓
Integration Tests
    ↓
Security Scan
    ↓
Human Review
    ↓
Versioned Artifacts
```

Cross-cutting controls:

```text
Safe path resolution
File ownership
Network restriction
Resource limits
Structured execution results
Artifact versioning
Traceability
```

---

# 27. Misconceptions

## “หลาย Agents ทำให้ Code มีคุณภาพสูงขึ้นเสมอ”

ไม่จริง

Agents เพิ่ม Specialization แต่เพิ่ม Interface Drift, Context Transfer และ Failure Points

## “Design.md ทำให้ทุกคนทำงานตรงกัน”

ไม่เสมอ

Markdown เป็นคำแนะนำ ถ้าไม่มี Contract Validation Agents อาจตีความต่างกัน

## “Shared Sandbox คือ Shared Memory”

ไม่ใช่

Sandbox เป็น Mutable Filesystem ส่วน Memory เป็นการจัดเก็บและค้นคืนบริบท

## “Unit Tests ผ่านแปลว่าระบบเสร็จ”

ไม่จริง

อาจยังไม่มี Integration, UI, Security และ Requirement Coverage Tests

## “MCP ทำให้ใช้ API ถูกต้อง”

ไม่รับประกัน

MCP ให้ Documentation Tools แต่ Agent ยังต้องตีความและ Implement ให้ถูก

## “Test Engineer เป็นอิสระจากผู้สร้าง Code”

ไม่ทั้งหมด

Test Engineerสามารถแก้ Backend และเขียน Tests เอง จึงยังมีโอกาสปรับทั้งโจทย์และคำตอบให้ผ่านพร้อมกัน

---

# 28. Checklist

### Crew มี Agents กี่ตัว

```text
4 ตัว:
Engineering Lead
Backend Engineer
Frontend Engineer
Test Engineer
```

### Process คืออะไร

```text
Process.sequential
```

### Engineering Lead เขียน Code หรือไม่

ไม่ เขียน Detailed Design และ Function Signatures

### Agent ใดใช้ Context7 MCP

Engineering Lead และ Frontend Engineer

### Agent ใดใช้ Sandbox Tools

Backend, Frontend และ Test Engineer

### Active Sandbox ถูกเตรียมอย่างไร

ถูกลบและสร้างใหม่เป็น UV Python 3.13 Project พร้อม Gradio

### Frontend Validation เรียก `.launch()` หรือไม่

ไม่ เพราะจะ Block Execution

### Test Engineer ใช้ Framework ใด

Built-in `unittest`

### มี Final Integration Task หรือไม่

ยังไม่มี

### มี Structured Design หรือ Structured Final Result หรือไม่

ยังไม่มี เป็น Markdown และ Files เป็นหลัก

---

# แก่นของ Lab 5

```text
Engineering Lead
→ สร้าง Contract

Backend Engineer
→ สร้าง Core Logic

Frontend Engineer
→ สร้าง User Interface

Test Engineer
→ ตรวจ Behavior

Shared Sandbox
→ เก็บ Source Code ร่วมกัน

Task Context
→ ส่งความหมายและผลลัพธ์

MCP
→ ดึง Documentation ล่าสุด

Docker
→ ให้ Execution Feedback
```

บทเรียนสำคัญที่สุดคือ:

> **การแบ่งงานให้หลาย Coding Agents ไม่ได้แก้ปัญหาความสอดคล้องโดยอัตโนมัติ แต่ย้ายปัญหาจาก “Agent เขียนโค้ดผิด” ไปเป็น “Agents เข้าใจ Contract ไม่ตรงกัน” ดังนั้น Interface Design, Shared Artifacts และ Integration Gates จึงสำคัญกว่าจำนวน Agents**

และ:

> **Unit Tests ควรเป็นเพียงหนึ่ง Gate ในระบบ ไม่ใช่ Gate สุดท้าย เพราะ Agent ที่แก้ Backend หลัง Frontend เสร็จสามารถทำให้ Tests ผ่านพร้อมกับทำให้ Application พังได้ ระบบจึงต้องตรวจ Backend, Frontend และ Integration ซ้ำหลังการเปลี่ยนแปลงครั้งสุดท้าย**

[1]: https://github.com/ed-donner/agents/tree/main/3_crewai/reference/engineering_team "agents/3_crewai/reference/engineering_team at main · ed-donner/agents · GitHub"
[2]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/engineering_team/pyproject.toml "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/engineering_team/src/engineering_team/config/tasks.yaml "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/engineering_team/src/engineering_team/config/agents.yaml "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/engineering_team/src/engineering_team/crew.py "raw.githubusercontent.com"
[6]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/engineering_team/sandbox_gpt/design.md "raw.githubusercontent.com"
[7]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/engineering_team/src/engineering_team/main.py "raw.githubusercontent.com"
[8]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/engineering_team/src/engineering_team/tools/sandbox_tools.py "raw.githubusercontent.com"
[9]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/engineering_team/src/engineering_team/patch.py "raw.githubusercontent.com"
[10]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/engineering_team/sandbox_gpt/test_summary.md "raw.githubusercontent.com"
