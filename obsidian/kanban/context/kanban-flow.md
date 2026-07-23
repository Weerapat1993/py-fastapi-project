# Kanban Ticket Flow

ใช้ไฟล์นี้เป็น workflow กลางเมื่อสั่ง AI ให้ทำ ticket ใน `obsidian/kanban/tasks/`

โปรเจกต์นี้ (`py-fastapi-project`) เป็น full-stack app: FastAPI + SQLModel backend, React + TypeScript (TanStack Router/Query) frontend admin dashboard ไม่ใช่ public-facing SEO site — ดู `obsidian/kanban/context/requirements.md` สำหรับ stack และ convention เต็ม

## Product Spec / Design Doc (ถ้ามี)

โปรเจกต์นี้ไม่มี `prd.md` กลาง ถ้า ticket อ้างถึง design spec หรือ implementation plan เฉพาะ (เช่นใน `docs/superpowers/specs/` หรือ `docs/superpowers/plans/`) ให้อ่านไฟล์นั้นก่อนเริ่มวางแผนหรือ implement เสมอ

ถ้า ticket ขัดกับ spec/plan ที่อ้างถึง ให้เพิ่มหรืออัปเดต `### Open Questions` และถามผู้ใช้ว่าจะอัปเดต spec หรือถือเป็น exception เฉพาะงานนั้น ห้ามเดาเอง

## Board File

Board อยู่ในไฟล์เดียวที่ `obsidian/kanban/work-kanban-board.md` มี frontmatter `kanban-plugin: board` สำหรับให้ Obsidian render เป็น visual board และมี `## COLUMN` sections:

- `## BACKLOG` — ticket ที่ยังไม่เริ่ม
- `## PLAN` — ticket ที่กำลัง planning
- `## TODO` — ticket ที่ plan แล้วและพร้อมเริ่ม
- `## INPROGRESS` — ticket ที่กำลังทำอยู่
- `## REVIEW` — ticket ที่รอ review
- `## DONE` — ticket ที่เสร็จแล้ว
- `## CLOSED` — ticket ที่ยกเลิกหรือไม่ทำต่อ

Ticket รายใบต้องอยู่ใน:

- `obsidian/kanban/tasks/<PREFIX>-XXXX.md`

Ticket ID prefix เลือกได้ตามหมวดงาน (ตัวอย่างที่มีอยู่แล้ว: `BOT-1001`) ตัวอย่างในไฟล์นี้ใช้ `WORK-XXXX` เป็น placeholder ทั่วไป

## Board Entry Format

ใน board file ให้ใช้รูปแบบ list item แบบนี้ภายใต้ `## COLUMN` section ที่ถูกต้อง:

```md
- [WORK-XXXX](tasks/WORK-XXXX.md): Ticket title
```

ห้ามเก็บรายละเอียด ticket เต็มใน board file ให้เก็บเฉพาะ link เท่านั้น

## Ticket File Format

Ticket file ต้องเป็น source of truth ของงานนั้นเสมอ:

```md
## WORK-XXXX: Ticket title

Status: Backlog
Priority: High
Spec: `path/to/spec-or-plan.md` (ถ้ามี)

### Summary
...

### Acceptance Criteria
...

### Validation
...
```

กติกาสำคัญ:

- หนึ่ง ticket ต่อหนึ่งไฟล์เท่านั้น
- ชื่อไฟล์ต้องตรงกับ ticket id เช่น `obsidian/kanban/tasks/WORK-1001.md`
- `Status:` ใน ticket file ต้องตรงกับ board ที่มี link ไปหา ticket นั้น
- ห้าม duplicate link ของ ticket เดียวกันไว้หลาย board พร้อมกัน
- ห้ามย้ายหรือ rename ticket file เมื่อเปลี่ยนสถานะ ให้ย้ายเฉพาะ link ระหว่าง board files

## Lark Sync

ยังไม่มี automation (artisan-equivalent) สำหรับ sync ไป Lark ในโปรเจกต์นี้ `obsidian/kanban/tasks/lark-sync.json` เป็นเพียง local preference placeholder — ถ้า `"enabled": false` หรือไฟล์ไม่มี ให้ข้าม Lark ทั้งหมดและทำ local flow ต่อ ห้ามพยายามรัน command ที่ไม่มีอยู่จริงในโปรเจกต์นี้

`.kanban.json` ที่ root คุมเฉพาะว่า board operation ควรอ่าน/เขียนผ่าน local `.md` files หรือผ่าน workspace-ai API (ดู `.claude/skills/kanban/SKILL.md` สำหรับ provider routing) — ไม่เกี่ยวกับ Lark

## Flow

### 1. Create Ticket

เมื่อสร้าง ticket ใหม่:

1. ใช้ `using-superpowers` เพื่อเลือก skill ที่เหมาะกับ ticket นี้
2. อ่าน spec/design/plan ที่เกี่ยวข้องถ้ามีการอ้างถึง (เช่นใน `docs/`)
3. ใช้ `brainstorming` ถ้า requirement ยังไม่ชัด, มีหลายแนวทาง, หรือต้องแตก scope
4. สร้าง `obsidian/kanban/tasks/WORK-XXXX.md` พร้อม title, `Status: Backlog`, priority, spec/context, summary, acceptance criteria, และ validation
5. เพิ่ม link ของ ticket ใน `obsidian/kanban/work-kanban-board.md (## BACKLOG)`

ห้ามสร้าง ticket เป็น block ยาวใน `obsidian/kanban/work-kanban-board.md (## BACKLOG)`

### 2. Start Ticket

เมื่อผู้ใช้สั่งให้เริ่มทำ ticket:

1. หา link ของ ticket จาก board file ปัจจุบัน โดยปกติคือ `obsidian/kanban/work-kanban-board.md (## PLAN)`, `obsidian/kanban/work-kanban-board.md (## BACKLOG)` หรือ `obsidian/kanban/work-kanban-board.md (## TODO)`
2. อ่าน ticket file จาก `obsidian/kanban/tasks/WORK-XXXX.md`
3. อ่าน spec/design/plan ที่ ticket อ้างถึง (ถ้ามี)
4. อ่าน spec, context, related files และ acceptance criteria ที่ ticket อ้างถึง
5. ถ้า ticket ขัดกับ spec ที่อ้างถึงและผู้ใช้ยังไม่ approve exception ให้เพิ่มหรืออัปเดต `### Open Questions` และหยุดก่อนย้ายเข้า `In Progress`
6. ถ้า ticket มี `### Implementation Plan` และ `Status: Plan` แล้ว ให้เริ่มต่อได้ทันทีโดยไม่เรียก `writing-plans` ซ้ำ
7. ถ้า ticket อยู่ `Status: Backlog` สามารถข้าม `Plan` และย้ายเข้า `In Progress` ได้ทันทีเมื่อครบทุกเงื่อนไขนี้:
   - acceptance criteria ชัดเจน
   - spec/context ที่จำเป็นมีครบ หรือไม่จำเป็นต่อการตัดสินใจ
   - ไม่มี `### Open Questions` หรือ blocking notes ที่ยังไม่ resolved
   - งานเล็ก ความเสี่ยงต่ำ และไม่เข้าเกณฑ์ที่ควรใช้ `writing-plans` ตาม Plan Ticket flow
8. ถ้า backlog ticket ไม่ครบทุกเงื่อนไขสำหรับการข้าม `Plan` ให้ทำ Plan Ticket flow ก่อน และหยุดถ้า plan ทำให้เกิด `Open Questions`
9. ใช้ skill `using-superpowers` เพื่อเลือก skill ที่เหมาะกับงานก่อนเริ่ม implement
10. ย้ายเฉพาะ link ของ ticket ไป `obsidian/kanban/work-kanban-board.md (## INPROGRESS)`
11. เปลี่ยน `Status:` ใน ticket file เป็น `In Progress`
12. ทำงานตาม scope และ acceptance criteria
13. หลัง implement และ validation ให้ report ผลงานและบอกว่า ticket พร้อม review หรือยัง
14. ห้ามย้าย ticket ไป `Review` อัตโนมัติระหว่างคำสั่ง `start` แม้ implementation และ validation จะผ่านแล้ว
15. ต้องรอคำสั่ง `/kanban move-review WORK-XXXX` แบบ explicit ก่อน จึงค่อยเพิ่ม `Review Notes` และย้าย board link ไป `obsidian/kanban/work-kanban-board.md (## REVIEW)` พร้อมเปลี่ยน `Status:` เป็น `Review`

ห้ามเริ่ม implement หาก ticket ไม่มี acceptance criteria หรือ spec/context ที่จำเป็นต่อการตัดสินใจ

### 2A. Plan Ticket

เมื่อผู้ใช้สั่งให้ plan ticket ก่อนเริ่มงาน:

1. หา ticket file จาก `obsidian/kanban/tasks/WORK-XXXX.md` และอ่าน spec/context/AC/validation ที่จำเป็น
2. อ่าน spec/design/plan ที่ ticket อ้างถึง (ถ้ามี)
3. ใช้ skill `using-superpowers` เพื่อเลือก skill ที่เหมาะกับการวางแผน
4. แสดง pre-plan assessment สั้น ๆ: status/board ปัจจุบัน, เป็น frontend/UI ticket ที่ควรใช้ `impeccable` + `html-design-prototypes` หรือไม่, ควรใช้ `brainstorming` หรือไม่, ควรใช้ `writing-plans` หรือไม่, ไฟล์/สถานะที่จะเปลี่ยน (backend `app/`, frontend `src/`, migration, ฯลฯ)
5. ถามผู้ใช้ด้วย `Yes` / `No` ว่าจะ plan ต่อหรือไม่; ถ้า `No` ให้หยุด
6. ถ้า context ยังไม่พอ หรือ ticket ขัดกับ spec ที่อ้างถึงและยังไม่ได้รับ approval ให้เพิ่ม `### Open Questions` แล้วหยุด
7. ถ้าเป็น frontend UI ticket ให้ทำ UI Design pre-plan gate ก่อนเขียน `### Implementation Plan`:
   - ใช้ `/impeccable shape WORK-XXXX` เพื่อ shape UX/UI direction ของ ticket
   - ส่ง direction ที่ได้ให้ `html-design-prototypes` เพื่อสร้าง HTML mockup prototype แบบ self-contained
   - บันทึก prototype ไว้ที่ `.claude/html/<ticket-id>-prototype.html`
   - เพิ่มหรืออัปเดต reference ใน ticket เช่น `Design Prototype: .claude/html/<ticket-id>-prototype.html`
   - ถือว่า prototype เป็น planning artifact เท่านั้น ห้ามแก้ production UI code (`frontend/src/...`) ระหว่าง `/kanban plan`
8. ถ้า `brainstorming` ถูกแนะนำ ให้ใช้เพื่อ clear scope / approach / decision ที่สำคัญ; ถ้าทำให้ยังตอบไม่ได้ ให้หยุดและบันทึก `Open Questions`
9. ถ้า `writing-plans` ถูกแนะนำ ให้ใช้หลัง UI Design gate และ `brainstorming` resolved แล้ว; ถ้าไม่ถูกแนะนำ ให้เขียน plan แบบ concise ได้เลย
10. เพิ่มหรืออัปเดต `### Implementation Plan` โดยใส่ context, planned changes, files likely to change (แยก backend/frontend ให้ชัด), validation commands (`bash backend/scripts/lint.sh`, `bash backend/scripts/test.sh`, `bun run lint`, `bun run build`, `bun run test`, migration commands ฯลฯ), และ risks / decisions
11. ย้ายเฉพาะ link ของ ticket ไป `obsidian/kanban/work-kanban-board.md (## PLAN)` และเปลี่ยน `Status:` เป็น `Plan`
12. ห้าม implement code changes ระหว่างคำสั่ง plan

แนะนำ UI Design pre-plan gate เมื่อ ticket สร้าง ออกแบบใหม่ ขัดเกลา ปรับโครงสร้าง หรือเปลี่ยนแปลงอย่างมีนัยสำคัญกับ frontend screen, layout, component, form, interaction, responsive behavior, หรือ visual hierarchy ใน `frontend/src`

แนะนำ `brainstorming` เมื่อ requirement ยังไม่ชัด, มีหลาย approach, มี `Open Questions`, ต้องแตก scope, หรือมี decision สำคัญที่ไม่ควรเดาระหว่าง implementation

แนะนำ `writing-plans` เมื่อ ticket แตะหลายไฟล์ หลาย layer (backend + frontend), database/Alembic migration, auth/JWT/permission logic, external integration, หรือมี regression risk ชัดเจน; งาน copy-only, board-only, documentation-only, หรือ single-file low-risk ใช้ plan แบบ concise ได้

เมื่อผู้ใช้สั่ง `start`:

1. ถ้ามี `### Implementation Plan` และ `Status: Plan` แล้ว ให้ดำเนิน Start Ticket ต่อ
2. ถ้าอยู่ `Backlog` และครบเงื่อนไขข้าม `Plan` ให้ย้ายเข้า `In Progress` ได้เลย
3. ถ้ายังไม่มี plan หรือ plan ยังมี `Open Questions` ให้กลับมาทำ Plan Ticket ก่อน
4. หลังย้ายเข้า `In Progress` แล้ว ถ้าเป็น frontend UI implementation ticket ให้ใช้ `/impeccable craft <Ticket-ID>` เป็น implementation driver
   - ใช้เฉพาะ ticket ที่สร้าง ออกแบบใหม่ ขัดเกลา ปรับโครงสร้าง หรือเปลี่ยนแปลงอย่างมีนัยสำคัญกับ frontend screen, layout, component, form, interaction, หรือ visual hierarchy
   - ถ้า ticket มี `Design Prototype:` ให้ถือเป็นสัญญาณแรงว่าควรใช้ `impeccable craft`
   - ใช้ ticket, `### Implementation Plan`, และ `Design Prototype:` เป็น approved direction; ห้ามเริ่ม design exploration ใหม่ ยกเว้น reference ยังไม่พอ
   - ต้องเคารพ user gates ของ `impeccable craft`; ห้าม bypass shape, direction, mock, หรือ approval pause เมื่อ gate นั้น apply
   - ถ้าเป็น full-stack ticket ให้ใช้ `impeccable craft` เฉพาะส่วน frontend/UI หลัง backend contract (API schema, response model) ชัดแล้ว
5. ถ้าเป็น backend-only, database/Alembic migration, API, auth, validation, integration, command, business logic ที่ไม่มีผลต่อ UI จริง ให้ใช้ `/kanban start` flow ปกติและห้าม invoke `/impeccable craft <Ticket-ID>`

### 3. Superpowers Gate

ใช้ skill จาก `obra/superpowers` เป็น workflow helper ระหว่างทำ ticket โดยไม่แทนที่ project instructions, tests, หรือ review gates ของ repo นี้

Skill map:

- `using-superpowers`: ขั้นแรกของทุก ticket
- `impeccable`: `/kanban plan` สำหรับ frontend UI ticket ให้ใช้ `/impeccable shape WORK-XXXX` ก่อนสร้าง HTML prototype; `/kanban start` สำหรับ UI/frontend implementation ticket ให้ใช้ `/impeccable craft <Ticket-ID>` เป็น implementation driver
- `html-design-prototypes`: `/kanban plan` สำหรับ frontend UI ticket หลัง `impeccable shape` เพื่อสร้าง mockup HTML prototype และใช้เป็น reference ใน ticket
- `brainstorming`: requirement ยังคลุมเครือ, ต้องเลือก approach, หรือแตกงานจาก spec ใหญ่เกินไป
- `writing-plans`: `/kanban plan` เมื่อ pre-plan assessment แนะนำ
- `systematic-debugging`: เมื่อ test fail, bug ยังไม่ชัด, behavior ไม่ตรง AC, หรือเจอ runtime/build error — ใช้ตั้งแต่เริ่ม debug สำหรับ bug report/regression ที่ผู้ใช้แจ้งว่า "เสีย", "ไม่ทำงาน", "error", "throwing", "failing"
- `doubt-driven-development`: outsider/adversarial review gate สำหรับ plan/diff/code change ที่มี regression risk, security-sensitive logic (auth, JWT, permission), หรือ cross-layer impact สูง
- `documentation-and-adrs`: bugfix ที่ validate แล้วและต้องเก็บ root cause / decision context ก่อน Done/Close

เกณฑ์ขั้นต่ำ:

- Ticket เล็กและชัดเจน: ใช้ `using-superpowers` อย่างน้อยหนึ่งครั้งก่อนเริ่ม
- Ticket ที่ไม่ชัด: ใช้ `brainstorming` ก่อน finalize scope, AC หรือ `### Implementation Plan`
- Ticket เสี่ยงหรือ context ไม่พอ: ผ่าน `/kanban plan` ก่อน `/kanban start`
- Bug report / regression: ใช้ `systematic-debugging` ก่อนเสนอ fix
- Plan/diff/code change เสี่ยงหรือหลาย layer (auth, migration, cross-service): ใช้ `doubt-driven-development`
- Bugfix ที่มี repro/root cause/fix/validation ครบและควรเก็บไว้อ้างอิงในอนาคต: ใช้ `documentation-and-adrs` ก่อน Done

ถ้าใช้ skill เหล่านี้ระหว่าง ticket ให้บันทึกสั้น ๆ ใน `Review Notes` หรือ `Done Notes` ว่าใช้ skill ใดและช่วยตัดสินใจเรื่องอะไร

### 4. Implementation Gate

ระหว่างทำ ticket ต้องปฏิบัติตาม:

- spec/design/plan ที่ ticket อ้างถึง (ถ้ามี)
- `obsidian/kanban/context/requirements.md`
- `obsidian/kanban/context/coding-style.md`
- Existing project conventions (`backend/app/`, `frontend/src/` patterns ที่มีอยู่แล้ว)

ทุก code change ต้องมีการทดสอบที่เกี่ยวข้อง และต้องรันคำสั่งตรวจสอบเท่าที่จำเป็น เช่น:

- Backend: `bash backend/scripts/lint.sh`, `bash backend/scripts/test.sh` (หรือ `uv run pytest backend/tests/ -k testName` ระหว่าง iterate)
- Frontend: `bun run lint`, `bun run build`, `bun run test` (Playwright)
- Schema change: สร้าง Alembic migration และตรวจว่า apply/rollback ได้

หาก test/build/lint fail และสาเหตุยังไม่ชัด ให้ใช้ skill `systematic-debugging` เพื่อแยก root cause ก่อนแก้ไข

### 5. Move To Review

เมื่อ implementation เสร็จและ test/lint/build ผ่าน:

1. เพิ่ม `Review Notes` แบบสั้นใน ticket file
2. ย้าย link จาก `obsidian/kanban/work-kanban-board.md (## INPROGRESS)` ไป `obsidian/kanban/work-kanban-board.md (## REVIEW)`
3. เปลี่ยน `Status:` เป็น `Review`

ตัวอย่าง `Review Notes`:

```md
### Review Notes
- Files: `backend/app/api/routes/items.py`, `frontend/src/routes/_layout/items.tsx`
- Commands: `bash backend/scripts/test.sh`, `bun run lint`, `bun run build`
- Skills: `using-superpowers`, `writing-plans`
```

### 6. Review Gate

หลัง ticket อยู่ใน `obsidian/kanban/work-kanban-board.md (## REVIEW)`:

- ใช้ `doubt-driven-development` ถ้างานมี plan, หลายไฟล์/หลาย layer, auth/permission/migration change, regression risk, หรือผู้ใช้ขอ review โดยตรง
- ใช้ skill `code-review` (effort `high` ขึ้นไป) เมื่อแตะ frontend/backend หลาย layer, auth, validation, data write, API contract, หรือมี security/regression risk สูง
- งานเล็กและ low-risk ใช้ `code-review` แบบ effort `low`/`medium` โดยเช็ก AC, scope, test/build/lint ผ่านครบ และไม่มี obvious runtime/regression

ให้ผลลัพธ์เป็น:

```md
Review type: Lightweight / Full (code-review high)
Verdict: PASS / NEEDS FIX / BLOCKED

Findings:
- ...

Missing tests:
- ...

Commands run:
- ...

Move to done: Yes / No
```

ถ้า `NEEDS FIX`:

1. แก้ issue ทันที
2. รัน test/build/lint ที่เกี่ยวข้องอีกครั้ง
3. ใช้ `systematic-debugging` เมื่อ root cause ยังไม่ชัด
4. ใช้ `doubt-driven-development` ซ้ำเมื่อ fix เปลี่ยน approach หรือแตะหลาย layer
5. Review ซ้ำด้วย gate เดิม หรืออัปเกรด effort เป็น `high`/`ultra` หาก fix ทำให้งานมี full-stack/security impact เพิ่มขึ้น
6. ห้ามย้ายไป done จนกว่า verdict เป็น `PASS`

ถ้า `BLOCKED`:

1. เก็บ link ของ ticket ไว้ใน `obsidian/kanban/work-kanban-board.md (## REVIEW)`
2. เพิ่ม `Blocked Notes` ใน ticket file
3. ระบุสิ่งที่ต้องการจากผู้ใช้หรือ dependency ที่ขาด

### 7. Move To Done

เมื่อ Review Gate ให้ verdict `PASS`:

1. เพิ่ม `Done Notes` แบบสั้นใน ticket file
2. ถ้าเป็น bugfix ที่คุ้มเก็บ root cause ให้ใช้ `documentation-and-adrs` ก่อนย้ายไป Done
3. ย้าย link จาก `obsidian/kanban/work-kanban-board.md (## REVIEW)` ไป `obsidian/kanban/work-kanban-board.md (## DONE)`
4. เปลี่ยน `Status:` เป็น `done`

ตัวอย่าง `Done Notes`:

```md
### Done Notes
- Review Gate: PASS
- Gates passed: lightweight review, tests, lint, build
- Validation: `bash backend/scripts/test.sh`, `bun run lint`, `bun run build`
- Root cause doc: not required
```

### 8. Close Or Archive Ticket

ใช้ `obsidian/kanban/work-kanban-board.md (## CLOSED)` เฉพาะ ticket ที่ยกเลิกหรือไม่ต้องทำต่อ

เมื่อ close ticket:

1. เพิ่ม `Closed Notes` ใน ticket file
2. ถ้าเป็น bugfix ที่แก้เสร็จแล้วแต่ต้องปิดด้วยเหตุผล workflow เช่น replaced by another ticket หรือ merged elsewhere และมี repro, root cause, fix, validation ครบ ให้ใช้ `documentation-and-adrs` ก่อน close เมื่อ root cause ยังควรถูกเก็บไว้
3. ย้ายเฉพาะ link ของ ticket จาก board ปัจจุบันไป `obsidian/kanban/work-kanban-board.md (## CLOSED)`
4. เปลี่ยน `Status:` ใน ticket file เป็น `closed`
5. ระบุเหตุผลที่ปิด เช่น duplicate, obsolete, out of scope, หรือ replaced by ticket อื่น

### 9. Close Sprint

เมื่อผู้ใช้สั่ง `/kanban sprint close` หรือ `/kanban sprint close --name=splint-X`:

1. อ่าน `obsidian/kanban/work-kanban-board.md` และรวบรวม entries ทั้งหมดใน `## DONE` และ `## CLOSED`
2. ถ้าทั้งสอง section ว่างเปล่า ให้ report ว่าไม่มีอะไร archive และหยุด
3. กำหนดชื่อ splint file ใหม่:
   - ถ้ามี `--name=` ให้ใช้ค่านั้น (เช่น `splint-3`)
   - ถ้าไม่มี ให้ scan ไฟล์ `obsidian/kanban/splint-*.md` ที่มีอยู่ หาเลขสูงสุด แล้ว +1
4. สร้าง `obsidian/kanban/splint-N-kanban-board.md` ด้วย format ที่มีเฉพาะ `## DONE` และ `## CLOSED` sections พร้อม links จากข้อ 1
5. ลบ entries ของ tickets ที่ archive แล้วออกจาก `## DONE` และ `## CLOSED` ใน `work-kanban-board.md`
6. ห้ามแตะ sections อื่น (`## BACKLOG`, `## PLAN`, `## TODO`, `## INPROGRESS`, `## REVIEW`) — tickets ที่ยังค้างอยู่ที่เดิมโดยอัตโนมัติ
7. ห้ามย้ายหรือ rename ticket files ใน `obsidian/kanban/tasks/`
8. ห้ามเปลี่ยน `Status:` ใน ticket files ใดๆ ทั้งสิ้น
9. แสดงผลสรุป: ชื่อไฟล์ splint archive ที่สร้าง, tickets ที่ถูก archive, tickets ที่ยังคงอยู่บน active board

Format ของ splint archive file:

```md
---

kanban-plugin: board

---

## DONE

- [WORK-XXXX](tasks/WORK-XXXX.md): Ticket title


## CLOSED

- [WORK-XXXX](tasks/WORK-XXXX.md): Ticket title

%% kanban:settings
{"kanban-plugin":"board","list-collapse":[false,false]}
%%
```

## Operating Rules

- Board files เก็บเฉพาะ heading และ list item link เท่านั้น
- Ticket file เป็น source of truth ของรายละเอียดทั้งหมด
- ย้ายสถานะด้วยการย้าย link ระหว่าง board files และอัปเดต `Status:` ใน ticket file
- ห้าม duplicate ticket link ไว้หลาย board พร้อมกัน
- ห้ามเปลี่ยน path ของ ticket file ระหว่าง flow ปกติ
- ห้ามย้าย ticket ไป done หาก review verdict ไม่ใช่ `PASS` หรือยังขาด test/build/lint ที่จำเป็น
- ใช้ `code-review` effort `high`/`ultra` เฉพาะ ticket ที่มี full-stack/security impact หรือ regression risk สูง; งานเสี่ยงต่ำใช้ effort `low`/`medium`
- ใช้ `documentation-and-adrs` เฉพาะ bugfix ที่มี repro, root cause, fix, validation และ root cause ยังมีคุณค่าต่อ future debugging
- หากมีการแก้ไขหลังเข้า review ให้ปรับ `Review Notes` และรัน verification ใหม่
- ต้องใช้ `using-superpowers` ก่อนเริ่ม ticket และใช้ Superpowers skill อื่นตามความเหมาะสมของ scope
- หาก ticket มี ambiguity หรือ plan risk ให้ย้อนกลับไป `/kanban plan`; หากมี bug report หรือ regression ให้ใช้ `systematic-debugging`
- ถ้ามีไฟล์เปลี่ยนแปลงที่ไม่ได้เกี่ยวกับ ticket ห้าม revert เอง ให้ทำงานเฉพาะ scope ของ ticket
