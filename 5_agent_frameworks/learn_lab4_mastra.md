# Week 5 — Lab 4: Mastra ด้วย TypeScript

ตำแหน่ง Lab:

```text
5_agent_frameworks/
└── 4_mastra/
    ├── lab.md
    ├── step1.ts
    ├── step2.ts
    ├── step3.ts
    ├── step4.ts
    ├── step5.ts
    ├── worker.ts
    ├── board.ts
    ├── tools.ts
    ├── workspace/
    ├── src/mastra/index.ts
    ├── scripts/dev.ts
    ├── SWAP_AI.md
    └── package.json
```

Lab 4 เป็นวันที่ Framework เปลี่ยนจาก Python ไปเป็น **TypeScript** แต่ยังใช้ Contract เดิมของ Week 5:

```text
อ่าน Goal จาก SQLite board
→ สร้าง Steps
→ อ่าน notes.txt ผ่าน MCP
→ แปลเป็นภาษาสเปน
→ เขียน spanish.txt
→ ปิด Steps
→ ปิด Goal
```

จุดประสงค์คือพิสูจน์ว่า Agent Pattern ไม่ได้ผูกกับ Python เพราะ Model–Tool Loop, MCP และ External State ยังมีโครงสร้างเดิม แม้เปลี่ยนภาษาและ Framework เป็น Mastra.

---

## Learning Objectives

หลังจบ Lab นี้ควรอธิบายได้ว่า:

1. Mastra สร้าง Agent ด้วย `id`, `name`, `instructions` และ `model` อย่างไร
2. Model routing string เช่น `openai/gpt-5.4-mini` ทำงานอย่างไร
3. `agent.generate()` เริ่ม Model–Tool Loop อย่างไร
4. Mastra Tools สร้างด้วย `createTool()` อย่างไร
5. Zod ทำหน้าที่เป็น Runtime Input Schema อย่างไร
6. `tools` ของ Mastra ต่างจาก Python Frameworks อย่างไร
7. TypeScript board ใช้ `node:sqlite` อย่างไร
8. `MCPClient` เชื่อม Filesystem MCP Server อย่างไร
9. `listTools()` และ `disconnect()` มีหน้าที่อะไร
10. Function Tools และ MCP Tools ถูกรวมให้ Agent อย่างไร
11. `maxSteps` เป็น Execution Budget อย่างไร
12. `onStepFinish` ช่วย Observability อย่างไร
13. Terminal Worker กับ Studio Worker ต่างกันอย่างไร
14. Mastra Studio ช่วยพัฒนาและ Debug อย่างไร
15. Model Portability ผ่าน Vercel AI SDK ทำงานอย่างไร
16. Zod Validation ต่างจาก Business Validation อย่างไร
17. ความเสี่ยงจาก Board, Filesystem และ Shared-worker Mode มีอะไรบ้าง

---

# 1. Setup

Lab นี้ไม่ได้ใช้ Jupyter Notebook แต่ใช้ TypeScript Programs ห้าตัว:

```text
step1.ts
step2.ts
step3.ts
step4.ts
step5.ts
```

แต่ละไฟล์ตรงกับ Five-step Pattern ของ Week 5:

```text
1. Create the agent
2. Run it
3. Add tools
4. Add MCP
5. Put it in a loop with a goal
```

Course ระบุให้ใช้ **Node 24 LTS** เพราะ `board.ts` ใช้ SQLite ที่มากับ Node ผ่าน `node:sqlite` และความสามารถนี้ใช้งานโดยไม่ต้องเปิด Flag ตั้งแต่ Node 23.4 ขึ้นไป.

ตรวจ Node:

```powershell
node --version
npm --version
npx --version
```

เข้าโฟลเดอร์ Lab:

```powershell
cd 5_agent_frameworks/4_mastra
```

ติดตั้ง Packages:

```powershell
npm install
```

Environment Variable ที่ต้องมีใน `.env` ของ Repository Root:

```env
OPENAI_API_KEY=...
```

Scripts ที่กำหนดใน `package.json`:

```json
{
  "step1": "node --disable-warning=ExperimentalWarning --import tsx step1.ts",
  "step2": "node --disable-warning=ExperimentalWarning --import tsx step2.ts",
  "step3": "node --disable-warning=ExperimentalWarning --import tsx step3.ts",
  "step4": "node --disable-warning=ExperimentalWarning --import tsx step4.ts",
  "step5": "node --disable-warning=ExperimentalWarning --import tsx step5.ts",
  "worker": "node --disable-warning=ExperimentalWarning --import tsx worker.ts",
  "dev": "node --import tsx scripts/dev.ts"
}
```

Dependencies หลัก ได้แก่ `@mastra/core`, `@mastra/mcp`, `@ai-sdk/openai`, Zod, TypeScript และ `tsx`.

---

# 2. Mastra Mental Model

```text
Mastra Agent
= Identity
+ Instructions
+ Model
+ Tools
+ Model-driven execution loop
```

Conceptual flow:

```text
User assignment
    ↓
Mastra Agent
    ↓
Model decides next action
    ├── Call tool
    ├── Read tool result
    ├── Call another tool
    └── Return final answer
```

Mastra ใช้ Vercel AI SDK เป็น Model Layer จึงสามารถระบุ Model ด้วย Routing String:

```text
provider/model
```

ตัวอย่าง:

```text
openai/gpt-5.4-mini
```

ส่วนแรกเลือก Provider และส่วนหลังเลือก Model

---

# 3. Step 1 — Create the Agent

รัน:

```powershell
npm run step1
```

Code:

```typescript
import "./env.ts";
import { Agent } from "@mastra/core/agent";

const agent = new Agent({
  id: "assistant",
  name: "Assistant",
  instructions:
    "You are a concise, friendly assistant. " +
    "Reply in a single short sentence.",
  model: "openai/gpt-5.4-mini",
});

console.log(`Created agent: ${agent.name}`);
```

องค์ประกอบ:

```text
id
→ Stable programmatic identifier

name
→ Human-readable agent name

instructions
→ System-level behavioral guidance

model
→ Provider/model routing information
```

ขั้นนี้สร้าง Object เท่านั้น ยังไม่มี Model Call หรือ Agent Loop เกิดขึ้น.

---

# 4. `id` กับ `name`

Mastra แยกสองค่า:

```typescript
id: "assistant",
name: "Assistant",
```

Mental model:

```text
id
= Primary key ของ Agent

name
= Display name ของ Agent
```

`id` ควรมีเสถียรภาพ เพราะอาจถูกใช้ใน Registry, Studio, API หรือ Tracing

`name` สามารถเป็นข้อความที่อ่านง่ายกว่า

---

# 5. Step 2 — Run the Agent

รัน:

```powershell
npm run step2
```

Code:

```typescript
const reply = await agent.generate(
  "Say hello in Spanish.",
);

console.log(reply.text);
```

Final Text อยู่ใน:

```typescript
reply.text
```

ตอนนี้ Agent ยังไม่มี Tools:

```text
Input
→ Model
→ Text
```

จึงยังเป็น LLM Call ที่ห่อด้วย Agent API มากกว่างาน Agentic แบบหลายขั้น.

---

# 6. `generate()` คืออะไร

เมื่อไม่มี Tools:

```text
generate()
→ One model interaction
→ Final text
```

เมื่อมี Tools:

```text
generate()
→ Model call
→ Tool request
→ Tool execution
→ Tool result
→ Next model call
→ Final text
```

ดังนั้น Method เดียวกันสามารถเริ่ม Agent Loop ได้เมื่อ Agent มี Action Surface

---

# 7. ทำไมต้องมี `process.exit(0)`

Step Scripts ลงท้ายด้วย:

```typescript
process.exit(0);
```

Comment ใน Course ระบุว่า Mastra ยังคงเปิด Model Connection Pool หลังงานเสร็จ จึงบังคับให้ Terminal Process ปิดหลังทำงานเสร็จ.

นี่เหมาะกับ Script แบบ One-shot:

```text
Start
→ Do one task
→ Exit
```

แต่ไม่ควรใช้ Pattern นี้ใน Long-running Server เพราะจะหยุด Process ทั้งหมดทันที

---

# 8. SQLite Board ใน TypeScript

`board.ts` เป็นคู่แฝดของ `board.py` จาก Days 1–3

ใช้:

```typescript
import { DatabaseSync } from "node:sqlite";
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

ชนิดของ Todo:

```text
Goal
→ parent_id = null

Step
→ parent_id = goal.id
```

สถานะ:

```text
pending
→ in_progress
→ done
```

Board เปิด WAL Mode และ `busy_timeout=5000` เพื่อช่วยลดปัญหาการเข้าถึง SQLite พร้อมกัน.

---

# 9. Board Functions

```text
resetBoard()
→ ลบ Board เดิมและสร้าง Table ใหม่

addGoal()
→ เพิ่ม Goal

addStep()
→ เพิ่ม Step ใต้ Goal

listTodos()
→ อ่าน Todos ทั้งหมด

claimTodo()
→ เปลี่ยน Status เป็น in_progress

completeTodo()
→ เปลี่ยน Status เป็น done

showBoard()
→ แสดง Board สำหรับมนุษย์
```

Mastra Agent ไม่เข้าถึง Functions เหล่านี้โดยตรงทั้งหมด แต่เข้าผ่าน Tools ที่กำหนดใน `tools.ts`

---

# 10. `BOARD_PATH`

Board Path ถูกกำหนดด้วย:

```typescript
export const BOARD_PATH =
  process.env.BOARD_PATH ??
  join(
    dirname(fileURLToPath(import.meta.url)),
    "board.sqlite",
  );
```

Standalone Mode:

```text
ไม่มี BOARD_PATH
→ ใช้ board.sqlite ใน Mastra folder
```

Shared-board Mode:

```text
Orchestrator ตั้ง BOARD_PATH
→ Mastra Worker เปิด Shared Board เดียวกับ Workers อื่น
```

เนื่องจากค่าอ่านตอน Module ถูก Import จึงต้องตั้ง Environment Variable ก่อนโหลด `board.ts`

---

# 11. Node `DatabaseSync`

Board ใช้ Synchronous SQLite API:

```typescript
const db = new DatabaseSync(path);
```

ข้อดี:

```text
Code เรียบง่าย
ไม่ต้องจัด Async database lifecycle
เหมาะกับ Demo และงานขนาดเล็ก
```

ข้อจำกัด:

```text
Database operation block Event Loop
ไม่เหมาะกับ High-throughput server
ทุก Board Function เปิดและปิด Connection ใหม่
```

ใน Lab งานเล็กมาก จึงยังเหมาะสม แต่ Production Server ควรพิจารณา Connection Strategy และ Async Workload ให้รอบคอบ

---

# 12. Step 3 — Add Tools

รัน:

```powershell
npm run step3
```

Mastra Tool ไม่ใช่ Plain Function โดยตรง แต่สร้างผ่าน:

```typescript
createTool()
```

และใช้ Zod กำหนด Input Schema

Imports:

```typescript
import { createTool } from "@mastra/core/tools";
import { z } from "zod";
```

---

# 13. `show_todos`

```typescript
export const showTodos = createTool({
  id: "show_todos",

  description:
    "List every todo on the board. " +
    "A goal has parent_id null; " +
    "a step points to its goal.",

  inputSchema: z.object({}),

  execute: async () => ({
    todos: listTodos(),
  }),
});
```

องค์ประกอบ:

```text
id
→ Tool identifier ที่ Model เห็น

description
→ บอกว่า Tool ทำอะไร

inputSchema
→ Runtime-valid input contract

execute
→ Implementation ที่ Application รัน
```

---

# 14. `plan_steps`

```typescript
export const planSteps = createTool({
  id: "plan_steps",

  description:
    "Break a goal into an ordered checklist.",

  inputSchema: z.object({
    goalId: z.number(),
    steps: z.array(z.string()),
  }),

  execute: async ({ goalId, steps }) => ({
    goalId,
    stepIds: steps.map(
      (step) => addStep(goalId, step),
    ),
  }),
});
```

Zod ตรวจว่า:

```text
goalId
→ ต้องเป็น number

steps
→ ต้องเป็น array of strings
```

ถ้า Model สร้าง Arguments ผิด Shape Tool จะไม่ควรได้รับ Input ที่ไม่ผ่าน Schema

---

# 15. `complete_task`

```typescript
export const completeTask = createTool({
  id: "complete_task",

  description:
    "Mark a todo done and record a result.",

  inputSchema: z.object({
    taskId: z.number(),
    result: z.string(),
  }),

  execute: async ({ taskId, result }) => {
    completeTodo(taskId, result);

    return {
      taskId,
      status: "done",
    };
  },
});
```

Mastra Tool Schema ชัดเจนกว่า Plain Function เพราะแยก:

```text
Tool metadata
Input validation
Implementation
```

ออกจากกันโดยตรง.

---

# 16. Zod Validation ไม่เท่ากับ Business Validation

Zod ตรวจได้ว่า:

```text
taskId เป็น number
result เป็น string
steps เป็น array
```

แต่ไม่ได้ตรวจว่า:

```text
taskId มีอยู่จริงหรือไม่
Task เป็นของ Worker นี้หรือไม่
Goal ยังมี Steps ค้างหรือไม่
spanish.txt มีอยู่จริงหรือไม่
Result ถูกต้องหรือไม่
```

ดังนั้น:

```text
Schema-valid call
≠ Valid business operation
```

Business Invariants ยังต้องอยู่ใน Tool Implementation หรือ Application Validator

---

# 17. Add Tools to Agent

```typescript
const boardAgent = new Agent({
  id: "board-agent",
  name: "Board Agent",
  instructions:
    "You help manage a shared todo board.",
  model: "openai/gpt-5.4-mini",

  tools: {
    showTodos,
    completeTask,
  },
});
```

สังเกตว่า Mastra ใช้ **Tool Object Map**:

```typescript
tools: {
  showTodos,
  completeTask,
}
```

ไม่ใช่ List แบบ Python Frameworks หลายตัว

เมื่อถาม:

```typescript
await boardAgent.generate(
  "What is on the board right now?",
);
```

Agent สามารถเลือกเรียก `show_todos` ก่อนตอบได้เอง.

---

# 18. Tool Variable Name กับ Tool ID

ใน Code:

```typescript
showTodos
```

เป็นชื่อ Variable ใน TypeScript

แต่:

```typescript
id: "show_todos"
```

เป็น Tool Identity ที่ Model และ Trace ใช้

Mental model:

```text
showTodos
= Developer-facing variable

show_todos
= Agent-facing tool ID
```

การแยกนี้ช่วยให้ TypeScript Code ใช้ `camelCase` ขณะที่ Tool Protocol ใช้ชื่อที่ต้องการได้

---

# 19. Step 4 — Add MCP

รัน:

```powershell
npm run step4
```

Mastra ใช้:

```typescript
import { MCPClient } from "@mastra/mcp";
```

สร้าง Filesystem Client ผ่าน Function:

```typescript
export function makeFilesystem(
  dir = WORKSPACE,
): MCPClient {
  return new MCPClient({
    servers: {
      filesystem: {
        command: "npx",
        args: [
          "-y",
          "@modelcontextprotocol/server-filesystem",
          dir,
        ],
        stderr: "ignore",
        cwd: dir,
      },
    },
  });
}
```

Architecture:

```text
Mastra Agent
    ↓
MCPClient
    ↓ stdio
Node Filesystem MCP Server
    ↓
Workspace
```

---

# 20. MCP Options

## `cwd`

```typescript
cwd: dir
```

ทำให้ Relative Paths เช่น:

```text
notes.txt
spanish.txt
```

ถูก Resolve ภายใน Workspace

## `stderr: "ignore"`

```typescript
stderr: "ignore"
```

ซ่อน Startup Banner และช่วยหลีกเลี่ยงปัญหาที่ Child Process เขียนไปยัง `stderr` ในบาง Environment

ข้อดี:

```text
Configuration อยู่ใน Public Options
ไม่ต้อง Subclass
ไม่ต้อง Monkeypatch
```

นี่สะอาดกว่า Workarounds ที่ใช้กับบาง Python Frameworks ใน Lab 2–3

ข้อเสียคือ Error Logs จาก MCP Server อาจถูกซ่อนด้วย

---

# 21. Discover MCP Tools

```typescript
const filesystem = makeFilesystem();

const tools =
  await filesystem.listTools();
```

`listTools()` ทำงานเชิงแนวคิดดังนี้:

```text
Start/connect MCP server
→ Request tool catalog
→ Convert MCP schemas
→ Return Mastra-compatible tools
```

จากนั้นให้ Agent:

```typescript
const fileAgent = new Agent({
  id: "file-agent",
  name: "File Agent",
  instructions:
    "Use your file tools.",
  model: "openai/gpt-5.4-mini",
  tools: await filesystem.listTools(),
});
```

---

# 22. MCP Cleanup

หลัง Agent Run:

```typescript
await filesystem.disconnect();
```

Lifecycle:

```text
Create MCPClient
→ listTools()
→ Agent uses tools
→ disconnect()
```

ถ้าไม่ Disconnect อาจเกิด:

```text
Child process ยังทำงาน
Node process ไม่ยอมจบ
Resources ไม่ถูกคืน
Subsequent runs ชนกัน
```

Mastra ไม่ได้ใช้ `async with` แบบ Python แต่ Developer ต้องเรียก `disconnect()` อย่างชัดเจน

---

# 23. Function Tools กับ MCP Tools

Board Tools:

```typescript
const boardTools = {
  showTodos,
  planSteps,
  completeTask,
};
```

Filesystem Tools:

```typescript
await filesystem.listTools()
```

รวมกันด้วย Object Spread:

```typescript
tools: {
  ...boardTools,
  ...(await filesystem.listTools()),
}
```

จากมุมมอง Agent:

```text
Board tools
และ
Filesystem MCP tools
```

อยู่ใน Capability Surface เดียวกัน

---

# 24. Step 5 — Goal-driven Worker

รัน:

```powershell
npm run step5
```

Worker:

```typescript
const worker = new Agent({
  id: "worker",
  name: "Worker",
  instructions: INSTRUCTIONS,
  model: "openai/gpt-5.4-mini",

  tools: {
    ...boardTools,
    ...(await filesystem.listTools()),
  },
});
```

Agent Run:

```typescript
await worker.generate(
  "Please work the pending goal on the board.",
  {
    maxSteps: 25,
  },
);
```

---

# 25. Expected Agent Loop

```text
show_todos
    ↓
Find in-progress goal
    ↓
plan_steps
    ↓
Read notes.txt
    ↓
Translate content
    ↓
Write spanish.txt
    ↓
complete_task(step)
    ↓
Repeat remaining steps
    ↓
complete_task(goal)
```

สรุป:

```text
Read
→ Plan
→ Act
→ Check off
→ Repeat
```

Framework เป็นผู้จัด Model–Tool Loop ภายใน `generate()`

---

# 26. `maxSteps: 25`

```typescript
{
  maxSteps: 25
}
```

ทำหน้าที่เป็น Execution Budget:

```text
จำกัดจำนวน Model/Tool Steps
ป้องกัน Agent วนไม่จบ
ควบคุม Cost และ Latency บางส่วน
```

แต่:

```text
Agent stopped within 25 steps
≠ Task succeeded
```

หาก Agent ใช้ครบ 25 Steps โดยยังทำไม่เสร็จ Code ปัจจุบันยังต้องตรวจ Board และ Artifact เอง

---

# 27. Seed Lifecycle

ก่อนรัน Step 5:

```typescript
mkdirSync(
  WORKSPACE,
  { recursive: true },
);

rmSync(
  join(WORKSPACE, "spanish.txt"),
  { force: true },
);

resetBoard();

const goalId = addGoal(GOAL);

claimTodo(goalId);
```

Application เป็นผู้:

```text
สร้าง Workspace
ลบ Output เก่า
Reset Board
Seed Goal
Claim Goal
```

Agent เป็นผู้:

```text
อ่าน Goal
สร้าง Plan
ใช้ File Tools
ปิด Steps
ปิด Goal
```

หลักยังเหมือน Labs ก่อนหน้า:

```text
Application
= Lifecycle authority

Agent
= Flexible execution
```

---

# 28. Terminal Worker

`worker.ts` คือ Step 5 เวอร์ชันสำหรับใช้งานผ่าน Terminal และ Day 5 Orchestrator

Standalone:

```powershell
npm run worker
```

มันจะ:

```text
Reset Board
→ Add Goal
→ Claim Goal
→ Start MCP
→ Run Agent
→ Print tool calls
→ Display Board
→ Display spanish.txt
```

---

# 29. `onStepFinish`

```typescript
await worker.generate(message, {
  maxSteps: 25,

  onStepFinish: (step) => {
    for (
      const call of step.toolCalls ?? []
    ) {
      console.log(
        `called ${call.payload.toolName}` +
        `(${JSON.stringify(
          call.payload.args,
        )})`,
      );
    }
  },
});
```

ช่วยแสดง:

```text
Tool name
Tool arguments
ลำดับ Tool Calls
จำนวน Steps
```

นี่คือ Runtime Observability ระดับเบื้องต้น

แต่:

```text
Tool was called
≠ Tool achieved correct outcome
```

ยังต้องตรวจ Tool Results, Board และ Files

---

# 30. Standalone Mode กับ Shared-board Mode

`worker.ts` อ่าน Command-line Arguments:

```typescript
const args =
  process.argv.slice(2);

const TASK_ID =
  args.length >= 2
    ? Number(args[0])
    : null;
```

## Standalone

```text
TASK_ID = null
→ Worker สร้าง Goal เอง
```

## Shared-board

```text
TASK_ID มีค่า
→ Worker Claim เฉพาะ Task ที่ได้รับ
```

Message:

```text
Work only task #<id>
Create its steps
Mark that task done
Then stop
```

---

# 31. จุดสำคัญของ `BOARD_PATH`

ใน Shared-board Mode ตัว Worker ใช้:

```typescript
const WORK_DIR =
  TASK_ID === null
    ? WORKSPACE
    : dirname(BOARD_PATH);
```

`BOARD_PATH` มาจาก Environment Variable ที่ `board.ts` อ่านตอน Import

ดังนั้น Manual Run ต้องมีลักษณะ:

```powershell
$env:BOARD_PATH = "C:\path\shared\board.sqlite"

npx tsx worker.ts `
  3 `
  "C:\path\shared\board.sqlite"
```

การส่ง `boardPath` เป็น Argument อย่างเดียวไม่เพียงพอ เพราะ Code ปัจจุบันไม่ได้อ่าน Argument ตัวที่สองไปตั้ง `BOARD_PATH` เอง แต่พึ่ง Orchestrator เป็นผู้ตั้ง Environment Variable ก่อนเริ่ม Process.

---

# 32. Model Selection ผ่าน `WORKER_MODEL`

```typescript
model:
  "openai/" +
  (
    process.env.WORKER_MODEL ??
    "gpt-5.4-mini"
  ),
```

ช่วยให้ Day 5 Orchestrator เปลี่ยน Model ต่อ Worker ได้โดยไม่แก้ Code

ตัวอย่าง:

```powershell
$env:WORKER_MODEL = "gpt-5.4-mini"
npm run worker
```

แต่ Provider ยังคงถูก Prefix เป็น:

```text
openai/
```

หากต้องการ Provider หรือ Endpoint อื่น ต้องใช้ Model Instance ตาม `SWAP_AI.md`

---

# 33. Mastra Studio

เริ่ม:

```powershell
npm run dev
```

เปิด:

```text
http://localhost:4111
```

Studio ช่วยดู:

```text
Agent conversation
Model steps
Tool calls
Tool results
Execution traces
Settings
Model selection
```

Lab ยก Studio เป็นจุดเด่นด้าน Developer Experience ของ Mastra.

---

# 34. Studio Registration

Studio โหลด:

```text
src/mastra/index.ts
```

Code:

```typescript
export const worker =
  new Agent({
    id: "worker",
    name: "Worker",
    instructions: "...",
    model: "openai/gpt-5.4-mini",
    tools: boardTools,
  });

export const mastra =
  new Mastra({
    agents: {
      worker,
    },
  });
```

`Mastra` Object ทำหน้าที่เป็น Registry ที่ Studio ใช้ค้นพบ Agents.

---

# 35. Terminal Worker กับ Studio Worker ไม่เหมือนกันทั้งหมด

นี่เป็นรายละเอียดสำคัญใน Code ปัจจุบัน

## Terminal Worker

มี:

```text
Board tools
Filesystem MCP tools
notes.txt
spanish.txt
```

## Studio Worker

มีเพียง:

```text
Board tools
```

ไม่มี Filesystem MCP

เหตุผลที่ระบุใน Source คือ Studio Bundle รันจาก `.mastra/output` และไม่พบ `npx` ใน PATH จึงเกิด `spawn npx ENOENT`

Studio Version จึงใส่ข้อความต้นฉบับไว้ใน Goal โดยตรง แล้วให้ Agent แปลและบันทึกผลผ่าน `complete_task` แทนการอ่านและเขียนไฟล์.

ดังนั้นอย่าสรุปว่า Studio Run ทดสอบ Filesystem MCP Pipeline เดียวกับ Terminal Run

---

# 36. Studio Reset Behavior

เมื่อ `src/mastra/index.ts` ถูกโหลด มันเรียก:

```typescript
resetBoard();

claimTodo(
  addGoal(...)
);
```

ผลคือ:

```text
Studio startup
หรือ module reload
→ Board ถูก Reset
→ Goal ใหม่ถูก Seed
```

เหมาะกับ Demo ที่ต้องการสถานะเริ่มต้นคงที่

ไม่เหมาะกับ Durable Application เพราะ Development Reload อาจลบ State เดิม

---

# 37. Model Portability

Mastra สามารถใช้ Provider Routing String:

```typescript
model:
  "openai/gpt-5.4-mini"
```

หรือสร้าง OpenAI-compatible Provider เอง:

```typescript
import {
  createOpenAI,
} from "@ai-sdk/openai";

const provider = createOpenAI({
  baseURL:
    process.env.OPENAI_BASE_URL,

  apiKey:
    process.env.OPENAI_API_KEY,
});

const worker = new Agent({
  id: "worker",
  name: "Worker",
  instructions: INSTRUCTIONS,

  model:
    provider("gpt-5.4-mini"),

  tools: {
    ...boardTools,
    ...(await filesystem.listTools()),
  },
});
```

Board, Tools และ MCP ไม่ต้องเปลี่ยน.

---

# 38. Mastra และ Vercel AI SDK

Routing String ช่วยลด Model-specific Setup:

```text
openai/gpt-5.4-mini
provider/model
```

ข้อดี:

```text
เปลี่ยน Model ง่าย
Provider abstraction ชัด
ใช้ Ecosystem ของ Vercel AI SDK
```

ข้อควรระวัง:

```text
Provider capabilities ไม่เท่ากัน
Tool-calling behavior ต่างกัน
Structured output support ต่างกัน
Model routing ไม่รับประกันผลลัพธ์เหมือนเดิม
```

Model Portability ทาง API ไม่เท่ากับ Behavioral Portability

---

# 39. Native A2A

Lab ระบุว่า Agents ที่ Register ใน Mastra สามารถถูก Expose พร้อม Agent Card และ JSON-RPC Endpoint ผ่าน Native A2A Support ได้

Lab นี้ไม่ได้สร้าง A2A Integration เพิ่มเติม จึงควรถือเป็น Capability Preview ไม่ใช่ส่วนที่ถูกทดสอบใน Worker Pipeline.

---

# 40. Shared Weakness — `complete_task`

```typescript
execute: async ({
  taskId,
  result,
}) => {
  completeTodo(
    taskId,
    result,
  );

  return {
    taskId,
    status: "done",
  };
}
```

ไม่ได้ตรวจว่า:

```text
Task ID มีอยู่จริง
Task เป็นของ Worker นี้
Goal มี Steps ค้างหรือไม่
Output File มีอยู่หรือไม่
File ไม่ว่างหรือไม่
Translation ถูกต้องหรือไม่
```

Zod ตรวจ Argument Shape แต่ไม่ตรวจ Business Invariants

---

# 41. Shared Weakness — `claimTodo`

```typescript
db.prepare(
  "UPDATE todos " +
  "SET status = 'in_progress' " +
  "WHERE id = ?",
).run(taskId);
```

ถ้า Workers สองตัว Claim Task เดียวกัน ทั้งสองอาจทำงานซ้ำ

Safer Query:

```sql
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
  AND status = 'pending'
```

จากนั้นตรวจจำนวน Rows ที่ถูกแก้:

```text
1 row
→ Claim สำเร็จ

0 rows
→ Task ถูก Claim แล้ว
```

WAL ช่วย Database Concurrency แต่ไม่สร้าง Business Ownership

---

# 42. Shared Weakness — Filesystem

Filesystem MCP ถูกจำกัดไว้ใน Workspace

ช่วยลด:

```text
Access นอกพื้นที่งาน
Accidental path exposure
```

แต่ Agent ยังสามารถ:

```text
เขียนทับ notes.txt
ลบ spanish.txt
สร้างไฟล์จำนวนมาก
อ่านและทำตาม Prompt Injection ในไฟล์
```

ดังนั้น:

```text
Workspace root
≠ Full security sandbox
```

---

# 43. Shared Weakness — MCP Version

MCP Server เริ่มผ่าน:

```text
npx -y @modelcontextprotocol/server-filesystem
```

ไม่มี Version ระบุใน Command

ผลคือ Installation ครั้งใหม่อาจได้ Server Version ใหม่ที่มี:

```text
Tool names เปลี่ยน
Schemas เปลี่ยน
Behavior เปลี่ยน
Bug หรือ Breaking Change ใหม่
```

ระบบที่ต้องการ Reproducibility ควร Pin Version ที่ผ่านการทดสอบ

---

# 44. Shared Weakness — Hidden Errors

```typescript
stderr: "ignore"
```

ทำให้ Development Output สะอาด แต่ยังอาจซ่อน:

```text
Server startup error
Permission error
Invalid path
Package installation error
Protocol warning
```

Production ควรส่ง MCP Logs ไปยัง Structured Logging แทนการทิ้งทั้งหมด

---

# 45. `maxSteps` กับ Definition of Done

`maxSteps: 25` ช่วยหยุด Loop แต่ไม่ได้ตรวจ:

```text
ทุก Step เป็น done
Goal เป็น done
spanish.txt มีอยู่
Output ถูกต้อง
```

Safer Flow:

```text
Agent run
    ↓
Board validator
    ↓
Artifact validator
    ↓
Goal completion gate
```

ตัวอย่าง:

```typescript
function validateResult(): void {
  const todos = listTodos();

  const unfinished =
    todos.filter(
      (todo) =>
        todo.status !== "done",
    );

  if (unfinished.length > 0) {
    throw new Error(
      "Unfinished todos remain",
    );
  }

  const output =
    join(
      WORKSPACE,
      "spanish.txt",
    );

  if (!existsSync(output)) {
    throw new Error(
      "spanish.txt is missing",
    );
  }

  if (
    !readFileSync(
      output,
      "utf-8",
    ).trim()
  ) {
    throw new Error(
      "spanish.txt is empty",
    );
  }
}
```

---

# 46. Mastra เทียบกับ Framework ก่อนหน้า

| Framework   | Tool declaration       | Run              | MCP lifecycle                  |
| ----------- | ---------------------- | ---------------- | ------------------------------ |
| Google ADK  | Typed Python functions | Runner           | `McpToolset`                   |
| Strands     | `@tool`                | `invoke_async()` | `MCPClient`                    |
| Pydantic AI | Plain functions        | `run()`          | `async with agent`             |
| MAF         | Plain functions        | `run()`          | `async with MCPStdioTool`      |
| Agno        | Plain functions        | `arun()`         | `async with MCPTools`          |
| Mastra      | `createTool()` + Zod   | `generate()`     | `listTools()` + `disconnect()` |

Core Loop ยังเหมือนเดิม:

```text
Model
→ Tool request
→ Tool execution
→ Observation
→ Next model decision
→ Completion
```

---

# 47. จุดเด่นของ Mastra

```text
TypeScript-native
Zod tool schemas
Vercel AI SDK model routing
Clean MCP configuration
Mastra Studio
Web-development ecosystem
Native A2A direction
```

เหมาะเมื่อ:

```text
ทีมหลักใช้ TypeScript
สร้าง AI features ใน Web application
ต้องการ Runtime validation ด้วย Zod
ต้องการ Local Agent Studio
ต้องการใช้ Vercel AI SDK ecosystem
```

---

# 48. จุดที่ต้องระวัง

```text
Framework และ APIs เปลี่ยนเร็ว
Node version มีผล
MCP child-process lifecycle ต้องปิดเอง
Studio Worker ไม่เหมือน Terminal Worker
Studio import reset Board
Zod ไม่บังคับ Business Invariants
SQLite Sync API block Event Loop
npx MCP package ไม่ถูก Pin
Connection pool ทำให้ One-shot scripts ต้อง exit
```

---

# 49. Exercises

## Exercise 1 — เปลี่ยน Goal

แก้ใน `step5.ts`:

```typescript
const GOAL =
  "Write a short haiku about Madrid " +
  "into madrid.txt.";
```

ตรวจว่า Agent:

```text
สร้าง Steps เหมาะสมหรือไม่
เลือก File Tools ถูกหรือไม่
สร้าง madrid.txt จริงหรือไม่
ปิด Goal หลังสร้างไฟล์หรือไม่
```

---

## Exercise 2 — Goal Completion Gate

สร้าง Tool แยก:

```typescript
completeGoal
```

ให้ตรวจ:

```text
ทุก Step เป็น done
Required File มีอยู่
File ไม่ว่าง
```

ก่อนอนุญาตให้ Goal เป็น Done

---

## Exercise 3 — Atomic Claim

แก้ `claimTodo()` ให้ Claim เฉพาะ Task ที่เป็น `pending`

แล้วรัน Workers สองตัวกับ Task ID เดียวกัน

---

## Exercise 4 — MCP Error Logging

แทน:

```typescript
stderr: "ignore"
```

ให้ส่งไปยัง Log File หรือ Parent Process

จากนั้นทำให้ MCP Server Start ไม่สำเร็จและตรวจ Error

---

## Exercise 5 — Terminal vs Studio

รัน:

```powershell
npm run worker
```

และ:

```powershell
npm run dev
```

เปรียบเทียบ:

```text
Tool set
MCP availability
Board content
Artifact creation
Trace visibility
```

---

# 50. Checklist

### Lab 4 ใช้ภาษาอะไร

```text
TypeScript
```

### Framework คืออะไร

```text
Mastra
```

### Agent สร้างอย่างไร

```typescript
new Agent({
  id,
  name,
  instructions,
  model,
  tools,
})
```

### Agent รันอย่างไร

```typescript
await agent.generate(...)
```

### Final text อยู่ที่ไหน

```typescript
reply.text
```

### Tool สร้างอย่างไร

```typescript
createTool({
  id,
  description,
  inputSchema,
  execute,
})
```

### Input validation ใช้อะไร

```text
Zod
```

### SQLite ใช้อะไร

```text
node:sqlite
```

### MCP Client คืออะไร

```typescript
MCPClient
```

### ดึง MCP Tools อย่างไร

```typescript
await filesystem.listTools()
```

### ปิด MCP อย่างไร

```typescript
await filesystem.disconnect()
```

### จำกัด Agent Loop อย่างไร

```typescript
maxSteps: 25
```

### Studio เริ่มอย่างไร

```powershell
npm run dev
```

### Studio อยู่ที่ไหน

```text
http://localhost:4111
```

---

# แก่นของ Week 5 Lab 4

```text
Mastra Agent
= TypeScript model–tool loop

createTool + Zod
= Typed and runtime-validated tools

node:sqlite
= Shared board state

MCPClient
= External filesystem capabilities

Mastra Studio
= Development and tracing UI

Vercel AI SDK
= Model-provider abstraction
```

บทเรียนสำคัญที่สุดคือ:

> **การข้ามจาก Python ไป TypeScript ไม่ได้เปลี่ยนโครงสร้างพื้นฐานของ Agent ระบบยังคงเป็น Model ที่อ่าน Goal เลือก Tool รับ Observation แล้วทำซ้ำจนคิดว่างานเสร็จ**

Mastra แสดงจุดแข็งของ TypeScript ได้ชัดผ่าน:

> **Zod Schemas, Tool Objects, Vercel AI SDK Model Routing และ Studio ที่เชื่อม Agent Development เข้ากับ Web Ecosystem โดยตรง**

แต่ข้อสรุปที่ยังเหมือนทุก Lab คือ:

> **Type Safety และ Runtime Schema Validation ช่วยป้องกันข้อมูลผิดรูปแบบ แต่ไม่สามารถยืนยันว่า Task ถูก Claim อย่างถูกต้อง, Artifact ถูกสร้างจริง หรือ Goal ควรถูกปิดได้ เรื่องเหล่านี้ยังต้องบังคับด้วย Application Code และ Deterministic Validators**
