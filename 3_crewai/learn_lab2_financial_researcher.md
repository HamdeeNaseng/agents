# Week 3 — Lab 2: CrewAI Financial Researcher

โปรเจกต์:

```text
3_crewai/reference/financial_researcher/
```

Lab นี้พัฒนาจาก Debate Crew ซึ่งใช้ความรู้ของโมเดลเป็นหลัก ไปสู่ Crew ที่มี **External Search Tool** และส่งข้อมูลจาก Research Age([CrewAI Documentation][1])รเจกต์ปัจจุบันใช้ CrewAI `1.14.4` และ Python `>=3.10,<3.14`. ([GitHub][2])Learning Objectives

เมื่อจบ Lab นี้ คุณควรอธิบายได้ว่า:

1. Researcher Agent และ Analyst Agent แบ่งความรับผิดชอบกันอย่างไร
2. `SerperDevTool` เพิ่มความสามารถอะไรให้ Agent
3. Tool ถูกกำหนดใน Python แต่ Role ถูกกำหนดใน YAML อย่างไร
4. `{company}` และ `{current_date}` ถูกส่งจาก Runtime อย่างไร
5. Task `context` ส่งผลจากงานหนึ่งไปยังอีกงานอย่างไร
6. Sequential Process ทำให้ Research เกิดก่อน Analysis อย่างไร
7. Tool Access แตกต่างจาก Knowledge Source อย่างไร
8. ทำไมข้อมูลจาก Web Search ยังไม่เท่ากับข้อมูลทางการเงินที่ได้รับการยืนยัน
9. ทำไม Analyst ไม่ควรค้นเว็บเองใน Architecture นี้
10. จุดอ่อนด้าน Citations, Source Quality และ Financial Safety อยู่ตรงไหน

---

# 1. ภาพรวม Workflow

```text
Company Name
     ↓
Researcher Agent
     ↓
SerperDevTool
     ↓
Current company information
     ↓
Research Task Output
     ↓
Analyst Agent
     ↓
Analysis Task
     ↓
output/report.md
```

ระบบมี Agents สองตัว:

```text
Researcher
→ ค้นหาและรวบรวมข้อมูล

Analyst
→ วิเคราะห์ จัดโครงสร้าง และเขียนรายงาน
```

และมี Tasks สองตัว:

```text
research_task
→ สร้างเอกสารวิจัย

analysis_task
→ ใช้ผลจาก research_task เขียนรายงาน
```

Crew ใช้ `Process.sequential` พร้อมเปิด `verbose` และ `tracing` จึงทำงานเรียงจาก Research ไป Analysis และสามารถสังเกต Execution ได้. ([GitHub][3]). ความแตกต่างจาก Debate Lab

## Debate Lab

```text
Motion
→ Pro Argument
→ Con Argument
→ Judge
```

ข้อมูลส่วนใหญ่มาจาก Model Knowledge

## Financial Researcher Lab

```text
Company
→ External Web Research
→ Financial/Market Analysis
→ Report
```

จุดใหม่ที่สำคัญคือ:

```text
Agent
+ External Tool
+ Explicit Task Context
```

Researcher ไม่ได้ถูกจำกัดอยู่กับความรู้ภายในโมเดล แต่สามารถร้องขอการค้นอินเทอร์เน็ตผ่าน Tool ได้

---

# 3. Project Structure

```text
financial_researcher/
├── knowledge/
├── output/
├── src/
│   └── financial_researcher/
│       ├── config/
│       │   ├── agents.yaml
│       │   └── tasks.yaml
│       ├── crew.py
│       └── main.py
├── pyproject.toml
├── uv.lock
└── README.md
```

Repository มีทั้ง `knowledge/` และ `output/` แต่จาก `crew.py` ปัจจุบันยังไม่ได้ผูก Knowledge Source เข้า Agent หรือ Crew ดังนั้น Lab นี้ใช้ **Web Tool** ไม่ใช่ CrewAI Knowledge Retrieval. นี่เป็นข้อสรุปจากโครงสร้างและโค้ดปัจจุบัน. ([GitHub][2]). `agents.yaml`

## Researcher Agent

Researcher ถูกกำหนดให้เป็น Senior Financial Researcher สำหรับบริษัทที่รับผ่าน `{company}` โดยมีเป้าหมายค้นข้อมูลบริษัท ข่าว และศักยภาพที่เป็นปัจจุบัน ณ `{current_date}`. ([GitHub][4])odel:

```text
Role
= นักวิจัยทางการเงิน

Goal
= หาข้อมูลล่าสุดและมีประโยชน์

Backstory
= มีประสบการณ์ค้นและกลั่นข้อมูลบริษัท

Tool
= ค้นอินเทอร์เน็ต
```

## Analyst Agent

Analyst มีบทบาทเป็น Market Analyst และ Report Writer เน้นวิเคราะห์บริษัท ค้นหา Pattern และเปลี่ยนข้อมูลวิจัยให้เป็นรายงานที่อ่านง่าย. ([GitHub][4])odel:

```text
Researcher
= เก็บวัตถุดิบ

Analyst
= ตรวจ จัดหมวด และตีความวัตถุดิบ
```

Agents ทั้งสองใช้ `openai/gpt-5.4-mini` ใน Configuration ปัจจุบัน. ([GitHub][4]). ทำไมต้องแยก Researcher กับ Analyst

ระบบสามารถใช้ Agent ตัวเดียวค้นและเขียนรายงานได้ แต่การแยก Agent ช่วยให้ความรับผิดชอบชัดเจนขึ้น

```text
Researcher
โฟกัส Coverage และ Recency

Analyst
โฟกัส Interpretation และ Communication
```

ข้อดี:

* Prompt ของแต่ละ Agent แคบลง
* Tool Permission แยกตามหน้าที่
* Debug ได้ว่าปัญหาเกิดจาก Research หรือ Analysis
* เปลี่ยน Model ของแต่ละบทบาทได้
* เพิ่ม Validation ระหว่างสอง Stage ได้

ข้อเสีย:

* มี Model Calls เพิ่ม
* ข้อมูลอาจสูญหายระหว่างส่งต่อ
* Analyst อาจตีความ Research ผิด
* Latency และ Cost เพิ่มขึ้น

---

# 6. Principle of Least Privilege

ใน `crew.py` มีเพียง Researcher ที่ได้รับ:

```python
tools=[SerperDevTool()]
```

ส่วน Analyst ไม่มี Search Tool. ([GitHub][5])Design ที่ดีในเชิงสิทธิ์:

```text
Researcher
→ ค้นเว็บได้

Analyst
→ วิเคราะห์เฉพาะ Context ที่ได้รับ
```

ถ้า Analyst มี Search Tool ด้วย อาจ:

* ค้นข้อมูลใหม่โดยไม่อยู่ใน Research Plan
* ใช้ Source ที่ Researcher ไม่ได้ตรวจ
* สร้างข้อสรุปจากข้อมูลคนละชุด
* ทำให้ตรวจย้อนหลังยากขึ้น

อย่างไรก็ตาม ข้อจำกัดคือเมื่อ Research ขาดข้อมูล Analyst ไม่สามารถค้นเพิ่มเติมเองได้

---

# 7. `SerperDevTool`

CrewAI อธิบายว่า `SerperDevTool` ใช้ Serper API เพื่อค้นอินเทอร์เน็ตและคืนผลลัพธ์ที่เกี่ยวข้อง โดยต้องติดตั้ง `crewai[tools]` และกำหนด Environment Variable ชื่อ `SERPER_API_KEY`. ([CrewAI Documentation][1])ent ที่ต้องมีอย่างน้อย:

```env
OPENAI_API_KEY=...
SERPER_API_KEY=...
```

`OPENAI_API_KEY` ใช้สำหรับ Agents ส่วน `SERPER_API_KEY` ใช้สำหรับ Search Tool ตาม Configuration ปัจจุบัน. ([GitHub][4])Tool ทำงานอย่างไร

```text
Research Task
     ↓
Researcher ตัดสินใจว่าต้องค้นอะไร
     ↓
เรียก SerperDevTool
     ↓
Serper API
     ↓
Search Results
     ↓
Researcher อ่านและสรุป
```

สิ่งสำคัญคือ Tool คืน **Search Results** ไม่ได้คืน “ข้อเท็จจริงที่ผ่านการตรวจสอบแล้ว”

```text
Search Result
≠
Verified Financial Fact
```

Search Results อาจมาจาก:

* เว็บไซต์บริษัท
* ข่าว
* Blog
* Social Media
* Aggregator
* Marketing Content
* เว็บไซต์ที่ข้อมูลเก่า

Researcher ยังต้องประเมินคุณภาพของ Source

---

# 8. Serper Tool Parameters

Tool รองรับการตั้งค่า เช่น:

```text
country
location
locale
n_results
search_url
```

ค่าเริ่มต้นของจำนวนผลลัพธ์ในเอกสารปัจจุบันคือ 10 และสามารถเปลี่ยน Endpoint ไปใช้ Search ประเภทอื่น เช่น Scholar ได้. ([CrewAI Documentation][1])inancial Research อาจปรับตาม Use Case เช่น:

```python
SerperDevTool(
    country="us",
    locale="en",
    n_results=10
)
```

แต่การเพิ่มจำนวน Results ไม่ได้แปลว่าคุณภาพสูงขึ้นเสมอไป

---

# 9. `research_task`

Research Task สั่งให้ค้นข้อมูลบริษัทในห้ามิติ:

```text
1. สถานะและสุขภาพของบริษัทปัจจุบัน
2. ผลการดำเนินงานในอดีต
3. ความท้าทายและโอกาสสำคัญ
4. ข่าวและเหตุการณ์ล่าสุด
5. แนวโน้มและพัฒนาการในอนาคต
```

Task ย้ำว่าข้อมูลต้องเป็นปัจจุบัน ณ `{current_date}` และต้องจัดเป็นส่วนที่ชัดเจน พร้อมข้อเท็จจริง ตัวเลข และตัวอย่างเมื่อเหมาะสม. ([GitHub][3])odel:

```text
Company
→ Company health
→ History
→ Risks/opportunities
→ Recent events
→ Outlook
```

นี่เป็น Research Checklist ที่ช่วยลดโอกาสมองบริษัทเพียงมิติเดียว

---

# 10. `analysis_task`

Analysis Task รับ Research Findings แล้วสร้างรายงานที่ประกอบด้วย:

```text
Executive Summary
Key research findings
Trend and pattern analysis
Market outlook
Conclusion
```

Task ระบุด้วยว่า Market Outlook ไม่ควรถูกใช้เพื่อตัดสินใจซื้อขายหลักทรัพย์. ([GitHub][3])ตสำคัญ:

> การใส่ Disclaimer ใน Prompt ช่วยให้รายงานเตือนผู้อ่าน แต่ไม่ได้ทำให้การวิเคราะห์ถูกต้องหรือปลอดภัยโดยอัตโนมัติ

ระบบยังต้องควบคุมเรื่อง Source, Accuracy และความไม่แน่นอน

---

# 11. Task `context`

ใน `tasks.yaml`:

```yaml
context:
  - research_task
```

หมายความว่า Output จาก `research_task` ถูกใช้เป็น Context ของ `analysis_task`. ([GitHub][3])ะบุว่า `context` เป็นรายการ Tasks ที่ Output จะถูกส่งเป็น Context ให้ Task ปัจจุบัน และ Task ที่อ้างถึงต้องเป็น Task ก่อนหน้า. ([CrewAI Documentation][6])``text
research_task.output
↓
analysis_task.context
↓
Analyst Prompt

````

---

## ทำไม `context` สำคัญ

หากไม่มีการส่งต่อข้อมูล:

```text
Researcher ทำงาน
Analyst ไม่เห็นผล
````

Analyst อาจต้องเดาจาก Model Knowledge เอง

เมื่อกำหนด Context:

```text
Analyst
= Original task instructions
+ Research Task Output
```

นี่คือ Agent Collaboration ที่เกิดผ่าน **Task Artifact Transfer** ไม่ใช่การที่ Agents สนทนากันโดยตรง

---

# 12. Sequential Process กับ Explicit Context

CrewAI ระบุว่า Sequential Process รัน Tasks ทีละตัว และใน Sequential Crew Output ของ Task ก่อนหน้าสามารถถูกส่งต่อไปยัง Task ถัดไปได้ ส่วน `context` ช่วยระบุอย่างชัดเจนว่าต้องใช้ Output ของ Task ใด. ([CrewAI Documentation][6])ี้:

```text
Process.sequential
กำหนดลำดับ

context: [research_task]
กำหนด Dependency ของข้อมูล
```

สองอย่างนี้ตอบคำถามคนละข้อ:

```text
Process
= Task ไหนทำก่อน

Context
= Task ปัจจุบันต้องเห็น Output ใด
```

---

# 13. `crew.py`

`crew.py` สร้าง:

```text
Researcher Agent
→ มี SerperDevTool

Analyst Agent
→ ไม่มี Tool

Research Task
Analysis Task

Crew
→ Process.sequential
→ verbose=True
→ tracing=True
```

และใช้ Decorators แบบเดียวกับ Debate Lab:

```text
@CrewBase
@agent
@task
@crew
```

([GitHub][5])`researcher()`

Conceptual Code:

```python
@agent
def researcher(self):
    return Agent(
        config=researcher_config,
        tools=[SerperDevTool()]
    )
```

ส่วนสำคัญคือ Agent Configuration มาจาก YAML แต่ Tool Object ถูกกำหนดใน Python

Mental Model:

```text
YAML
→ Identity และ Prompt

Python
→ Capability และ Runtime Integration
```

---

## `analyst()`

Analyst ใช้ Configuration จาก YAML แต่ไม่มี External Tool

```text
Analyst Input
= Analysis Task
+ Research Context
```

บทบาทของ Analyst จึงคล้าย Pure Transformation:

```text
Research Document
→ Analytical Report
```

---

## `analysis_task()`

ในโค้ดปัจจุบัน `output_file='output/report.md'` ถูกระบุใน `crew.py` และใน `tasks.yaml` ก็มี Output File เส้นทางเดียวกันอยู่แล้ว. ([GitHub][3])Configuration ซ้ำ แม้ค่าจะตรงกันจึงยังทำงานได้ แต่ในระบบจริงควรกำหนด Source of Truth เพียงจุดเดียว เพื่อลดโอกาสที่ YAML และ Python จะขัดแย้งกันภายหลัง

---

# 14. `main.py`

เมื่อรัน Crew ระบบถาม:

```text
Enter the company name for research:
```

จากนั้นสร้าง Inputs:

```python
{
    "company": company,
    "current_date": current_date
}
```

โดย `current_date` มาจากวันที่ปัจจุบันของระบบตอนรัน. ([GitHub][7])``text
User enters company
↓
main.run()
↓
inputs:
company
current_date
↓
kickoff(inputs=inputs)
↓
YAML interpolation

````

---

# 15. Runtime Placeholders

YAML ใช้:

```text
{company}
{current_date}
````

Runtime ต้องส่ง Keys ตรงกัน:

```python
{
    "company": "Microsoft",
    "current_date": "2026-07-24"
}
```

หลัง Interpolation:

```text
Research latest information about Microsoft
up to date as of 2026-07-24
```

การส่ง `current_date` เข้า Prompt เป็นแนวปฏิบัติที่ดีกว่าการหวังว่า Model รู้วันที่ปัจจุบันเอง

แต่ยังไม่รับประกันว่า Search Result ทุกแหล่งเป็นข้อมูลล่าสุด

---

# 16. จุดผิดปกติใน `train()` และ `test()`

`run()` ใช้ Keys ถูกต้อง:

```text
company
current_date
```

แต่ helper `train()` และ `test()` ใน `main.py` ปัจจุบันยังใช้:

```text
topic
current_year
```

ซึ่งไม่ตรงกับ Placeholders ใน `agents.yaml` และ `tasks.yaml`. ([GitHub][4])อนเรียน Lab นี้ควรเริ่มจาก:

```powershell
crewai run
```

ก่อน

หากต้องใช้ Training หรือ Testing ให้แก้ Inputs เป็น:

```python
inputs = {
    "company": "Microsoft",
    "current_date": str(datetime.now().date())
}
```

---

# 17. README ยังมี Boilerplate เก่า

README ระบุว่าตัวอย่างเดิมจะสร้างรายงานเกี่ยวกับ LLMs แต่โค้ด `main.py` ปัจจุบันถามชื่อบริษัทและ YAML ถูกออกแบบเพื่อ Company Research ดังนั้นข้อความส่วนนั้นใน README น่าจะเป็นข้อความจาก Template ที่ยังไม่ได้อัปเดต. ([GitHub][7])ชิงวิศวกรรมคือ:

> Documentation อาจล้าหลังกว่า Source Code ดังนั้นควรตรวจ Runtime Code และ Configuration จริงเสมอ

---

# 18. วิธีติดตั้งและรัน

จาก Project Root:

```powershell
cd 3_crewai\reference\financial_researcher
```

สร้าง `.env`:

```env
OPENAI_API_KEY=your_openai_key
SERPER_API_KEY=your_serper_key
```

ติดตั้ง Dependencies:

```powershell
crewai install
```

รัน:

```powershell
crewai run
```

โปรเจกต์รองรับคำสั่งดังกล่าวผ่าน Script Configuration และ README ปัจจุบัน. ([GitHub][2])นเสร็จตรวจ:

```text
output/report.md
```

---

# 19. สิ่งที่ควรสังเกตใน Terminal

เมื่อ `verbose=True` และ `tracing=True` ให้สังเกต:

```text
Research Task เริ่มเมื่อใด
Researcher เรียก Search Tool กี่ครั้ง
ใช้ Search Query อะไร
Serper คืน Source แบบใด
Researcher สรุปอะไรไว้
Analyst ได้ Context อะไร
Analyst เพิ่มข้อสรุปใดจากตนเอง
จำนวน Model Calls
Token Usage
ระยะเวลาของแต่ละ Stage
```

CrewAI Tracing สามารถถูกกำหนดผ่าน Crew Configuration และ Sequential Process จะรันงานต่อกันทีละ Task. ([CrewAI Documentation][8])0. Tool กับ Knowledge ต่างกันอย่างไร

## Tool

```text
Agent ตัดสินใจเรียกเมื่อจำเป็น
→ Execute Action
→ คืน Result
```

ตัวอย่างใน Lab:

```text
SerperDevTool
→ ค้นเว็บ
```

## Knowledge

```text
ข้อมูลถูกเตรียมไว้
→ Index/Store
→ Retrieve ส่วนที่เกี่ยวข้อง
→ เพิ่มเข้า Context
```

CrewAI รองรับ Knowledge Sources เช่น Text, PDF, CSV, Excel และ JSON และสามารถกำหนด Knowledge ระดับ Agent หรือ Crew ได้. ([CrewAI Documentation][9])ีโฟลเดอร์ `knowledge/` แต่ยังไม่ได้ใช้งานใน `crew.py` ปัจจุบัน. ([GitHub][2])1. Current Information ไม่เท่ากับ Verified Information

Task ย้ำให้ Research เป็นข้อมูลล่าสุด แต่ระบบยังไม่มี:

```text
Source Date Validation
Official Filing Verification
Cross-source Comparison
Citation Mapping
Conflict Detection
```

Search Agent อาจพบข่าวล่าสุดจริง แต่ข่าวนั้นอาจ:

* เป็นข่าวลือ
* อ้างแหล่งนิรนาม
* ตีความตัวเลขผิด
* เกิดก่อนงบการเงินล่าสุด
* ถูกเผยแพร่โดยผู้มีผลประโยชน์

ดังนั้น:

```text
Recent
≠
Reliable

Searchable
≠
Verified
```

---

# 22. Financial Data Sources ที่ควรให้ความสำคัญ

สำหรับระบบจริง ควรจัดลำดับ Source เช่น:

```text
1. Regulatory filings
2. Investor relations documents
3. Audited financial statements
4. Official company announcements
5. Reputable financial news
6. Analyst commentary
7. Blogs and social posts
```

Serper Search ช่วยค้นหา Source แต่ไม่ได้จัด Policy การให้น้ำหนัก Source ให้ครบถ้วนโดยอัตโนมัติ

---

# 23. Report ยังไม่มี Citation Contract

`research_task` ขอ Facts และ Figures แต่ไม่ได้บังคับ Output ให้มี:

```text
Source URL
Publisher
Publication Date
Access Date
Claim-to-source Mapping
```

ผลลัพธ์อาจอ่านดี แต่ตรวจย้อนกลับยาก

ระบบที่แข็งแรงกว่าควรใช้โครงสร้าง:

```python
class FinancialFinding:
    claim: str
    value: str
    source_url: str
    source_date: str
    confidence_notes: str
```

แล้วส่ง Structured Findings ไปให้ Analyst

---

# 24. Researcher อาจสรุปผิด

Flow ปัจจุบันมี Compression Layer:

```text
Search Results
→ Researcher Summary
→ Analyst Report
```

ความผิดพลาดสามารถสะสมสองชั้น:

```text
Search Result ถูกตีความผิด
        ↓
Researcher สรุปผิด
        ↓
Analyst วิเคราะห์จาก Summary ที่ผิด
```

นี่เรียกว่า Error Propagation

ระบบจริงควรเก็บทั้ง:

```text
Research Summary
+
Raw Source References
```

---

# 25. Analyst อาจสร้างข้อสรุปเกิน Evidence

ถึง Analyst ไม่มี Search Tool แต่ยังเป็น LLM จึงอาจ:

* เพิ่ม Narrative ที่ Research ไม่ได้สนับสนุน
* สร้างเหตุผลเชิงสาเหตุจาก Correlation
* ทำนายอนาคตอย่างมั่นใจเกินไป
* เลือกเฉพาะข้อมูลที่สอดคล้องกับข้อสรุป
* ลดความสำคัญของข้อมูลที่ขัดแย้งกัน

ดังนั้น:

```text
Context-grounded
≠
Guaranteed faithful
```

---

# 26. ข้อจำกัดของ “Future Outlook”

การทำนายอนาคตบริษัทมีความไม่แน่นอนสูง

รายงานควรแยก:

```text
Known Facts
Current Indicators
Assumptions
Scenarios
Uncertainties
```

แทนการเขียนทุกอย่างเป็นข้อสรุปแบบเดียว

ตัวอย่าง:

```text
Base Case
Bull Case
Bear Case
Key triggers
Major uncertainties
```

---

# 27. Report ไม่ใช่คำแนะนำลงทุน

Task ระบุชัดว่า Market Outlook ไม่ควรใช้สำหรับ Trading Decisions. ([GitHub][3])วรปรากฏในรายงานด้วย เช่น:

```text
This report is for informational and educational
purposes and is not investment advice.
```

แต่ Disclaimer ไม่สามารถแก้:

* ข้อมูลผิด
* Source ไม่ดี
* ตัวเลขเก่า
* Conflict of interest
* การวิเคราะห์ที่มี Bias

---

# 28. Pattern ที่กำลังเรียน

## Tool-Enabled Agent

```text
Researcher
+ SerperDevTool
```

## Research–Analysis Separation

```text
Researcher
→ Analyst
```

## Sequential Collaboration

```text
Task 1
→ Task 2
```

## Explicit Task Context

```text
research_task.output
→ analysis_task.context
```

## Artifact Generation

```text
analysis_task
→ output/report.md
```

## Least-Privilege Tool Assignment

```text
Researcher has Search Tool
Analyst has no Search Tool
```

---

# 29. แบบฝึกหัดที่แนะนำ

## Exercise 1 — ทดลองบริษัทเดียวกันสามครั้ง

ตรวจว่า:

```text
Search Queries เหมือนกันหรือไม่
Sources เหมือนกันหรือไม่
Facts และ Figures เปลี่ยนหรือไม่
Outlook เปลี่ยนหรือไม่
```

เป้าหมายคือเห็น Non-determinism และ Search Variability

---

## Exercise 2 — ตรวจ Source

เลือก Claim สำคัญห้าข้อจาก `report.md` แล้วค้นหา Source ต้นทางด้วยตนเอง

บันทึก:

```text
Claim
Source
Published date
Supported / Unsupported
```

---

## Exercise 3 — เพิ่ม Citation Requirement

แก้ `research_task.expected_output` ให้ทุก Fact สำคัญมี:

```text
Source title
URL
Publication date
```

จากนั้นตรวจว่า Model ทำตามสม่ำเสมอหรือไม่

---

## Exercise 4 — เพิ่ม Source Quality Section

ให้ Researcher แบ่ง Source เป็น:

```text
Primary source
Secondary source
Commentary
Unverified source
```

---

## Exercise 5 — เพิ่ม Structured Output

สร้าง Pydantic Model สำหรับ Research Findings เช่น:

```text
Company status
Financial metrics
Recent events
Risks
Opportunities
Sources
Limitations
```

---

## Exercise 6 — เปรียบเทียบ Tool Access

ทดลองให้ Analyst มี `SerperDevTool` ด้วย แล้วดูว่า:

```text
Report ดีขึ้นหรือไม่
Model Calls เพิ่มเท่าไร
Analyst ใช้ Source ใหม่ที่ Researcherไม่พบหรือไม่
Trace ซับซ้อนขึ้นหรือไม่
```

---

# 30. Checklist ก่อนจบ Lab

### Researcher กับ Analyst ต่างกันอย่างไร

```text
Researcher
= Retrieval และ Information Collection

Analyst
= Interpretation และ Report Writing
```

### `SerperDevTool` ทำอะไร

ใช้ Serper API เพื่อค้นข้อมูลอินเทอร์เน็ตที่เกี่ยวข้องกับ Query. ([CrewAI Documentation][1])ใช้ Environment Variable อะไร

```text
OPENAI_API_KEY
SERPER_API_KEY
```

([CrewAI Documentation][1])text: [research_task]` ทำอะไร

ส่ง Output ของ Research Task เป็น Context ให้ Analysis Task. ([CrewAI Documentation][6]) Analyst ไม่มี Search Tool

เพื่อจำกัดหน้าที่ให้วิเคราะห์ Research ที่ได้รับและลด Tool Permission

### `current_date` มาจากไหน

มาจาก `datetime.now().date()` ใน `main.py` ตอนเริ่ม Run. ([GitHub][7])rt อยู่ที่ไหน

```text
output/report.md
```

([GitHub][3])ch Result เป็นหลักฐานที่ยืนยันแล้วหรือไม่

ไม่ เป็นข้อมูลที่ยังต้องประเมิน Source, วันที่ และความสอดคล้อง

### Knowledge Folder ถูกใช้แล้วหรือยัง

ยังไม่ถูกผูกใน `crew.py` ปัจจุบัน. ([GitHub][2])ก่นของ Lab 2

```text
Agent
= ผู้เชี่ยวชาญ

Tool
= ความสามารถภายนอก

Task
= งานที่มอบหมาย

Context
= ผลงานจาก Task ก่อนหน้า

Process
= ลำดับการทำงาน

Output File
= Artifact สุดท้าย
```

บทเรียนสำคัญที่สุดคือ:

> **Agent Collaboration ไม่จำเป็นต้องเกิดจาก Agents สนทนากันโดยตรง ใน CrewAI ผลงานของ Task หนึ่งสามารถกลายเป็น Context ของ Task ถัดไป ทำให้ Researcher และ Analyst ร่วมมือกันผ่าน Artifact ที่ส่งต่ออย่างมีโครงสร้าง**

และต้องจำอีกด้านหนึ่งว่า:

> **การเพิ่ม Web Search ทำให้ข้อมูลใหม่ขึ้น แต่ไม่ได้ทำให้ข้อมูลจริงขึ้นโดยอัตโนมัติ ระบบ Financial Research ที่น่าเชื่อถือต้องรักษา Source Provenance ตรวจวันที่ ตรวจตัวเลข เปรียบเทียบหลายแหล่ง และแยกข้อเท็จจริงออกจากการวิเคราะห์และการคาดการณ์**

[1]: https://docs.crewai.com/en/tools/search-research/serperdevtool "Google Serper Search - CrewAI"
[2]: https://github.com/ed-donner/agents/tree/main/3_crewai/reference/financial_researcher "agents/3_crewai/reference/financial_researcher at main · ed-donner/agents · GitHub"
[3]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/financial_researcher/src/financial_researcher/config/tasks.yaml "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/financial_researcher/src/financial_researcher/config/agents.yaml "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/financial_researcher/src/financial_researcher/crew.py "raw.githubusercontent.com"
[6]: https://docs.crewai.com/en/concepts/tasks "Tasks - CrewAI"
[7]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/financial_researcher/src/financial_researcher/main.py "raw.githubusercontent.com"
[8]: https://docs.crewai.com/en/concepts/crews "Crews - CrewAI"
[9]: https://docs.crewai.com/en/concepts/knowledge "Knowledge - CrewAI"
