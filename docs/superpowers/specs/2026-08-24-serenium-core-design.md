# Serenium Core — Design Spec

วันที่: 2026-08-24 · สถานะ: รออนุมัติจากผู้ใช้ · ภาษา: prose ไทย, identifier/path/code อังกฤษ

---

## 1. สรุปย่อ

Serenium Core คือ **AI OS** ที่รันเป็น desktop app บนเครื่องผู้ใช้ หน้าจอหลักคือทรงกลม
(*World Network*) ที่ประกอบขึ้นจากข้อมูลจริงในระบบ ล้อมด้วยสนามอนุภาค (*Network Vectors*)
ล้อมด้วยวงแหวนของ Session และ Artifact ล่าสุด และมี widget ที่ผู้ใช้จัดวางเองได้อยู่รอบนอก

กดเข้าไปในทรงกลม → กล้องบินทะลุเปลือก → เข้าสู่ **Knowledge Graph** ของทั้งระบบ

เป้าหมายระยะยาวคือรองรับ AI หลายเจ้า ไม่ใช่แค่ Claude — v1 รองรับ Claude อย่างเดียว
แต่โครงสร้างต้องไม่ปิดประตูนั้น

---

## 2. ขอบเขต

### อยู่ใน v1

- Electron desktop app (Windows)
- `core` อ่านข้อมูลจาก `~/.claude` และ Obsidian vault ทั้งหมด
- World Network + Network Vectors + วงแหวน Session/Artifact
- Knowledge Graph 7 ชนิดโหนด พร้อมระบบคลัสเตอร์
- ระบบ widget แบบ iOS (เพิ่ม/ลบ/ย้าย/ปรับขนาด/หลายหน้า)
- widget 6 ตัว: Live Agents · Burn & Projection · Model/Effort · Skill deck · Usage calendar · Routine
- daily usage rollup (ทำไปแล้ว ต้องย้ายเข้า core)

### ไม่อยู่ใน v1 — ระบุไว้เพื่อกันการบานปลาย

- widget marketplace / การติดตั้ง widget จากภายนอก
- AI เจ้าอื่นนอกจาก Claude
- auto-routing ระหว่างโมเดล
- Email widget
- การแก้ไขเอกสารในตัวแอป (อ่านอย่างเดียว ยกเว้น `settings.json`, `schedules.json`, `layout.json`)
- ซิงก์ข้ามเครื่อง / cloud

### สิ่งที่ถูกแทนที่

`~/.claude/agent-os/app/dashboard.html` (cockpit เดิม) **ถูกแทนที่ด้วย Serenium Core**
พาเนลเดิมหกตัวไม่ได้ย้ายมาเป็น widget ตรงๆ ทั้งหมด — Rubrics, Rubric deck, Portfolio และ Vault
**ไม่อยู่ใน v1** เพราะเนื้อหาของมันกลายเป็นโหนดในกราฟแทนแล้ว

`~/.claude/agent-os/` ส่วนที่เหลือ (rubrics, routines, knowledge, commands, rubric-grader)
**ยังใช้งานต่อตามเดิม** และกลายเป็นแหล่งข้อมูลของ Serenium ผ่าน `agentos.mjs`

---

## 3. สถาปัตยกรรม

แยกสองส่วนเด็ดขาด **`core` ห้าม import อะไรจาก `shell`**

| | หน้าที่ | ทดสอบยังไง |
|---|---|---|
| `core` | Node ล้วน อ่านไฟล์ แปลง คำนวณ เฝ้าดู เขียนกลับ | unit test ธรรมดา ไม่ต้องเปิดหน้าต่าง |
| `shell` | Electron วาดภาพ รับ input | smoke test เท่านั้น |

เหตุผลหลัก: ตรรกะ "อ่านสถานะ agent หลายตัวแล้วตัดสินใจส่งงานไปไหน" คือหัวใจของ AI OS
มันต้องไม่อยู่ในโค้ดวาดภาพ ถ้าวันหนึ่ง Electron ไม่ใช่คำตอบ `core` ต้องรอด

### ที่อยู่

| | path | เหตุผล |
|---|---|---|
| โค้ด | `%USERPROFILE%\Serenium\` | โปรเจกต์จริง มี git ของตัวเอง |
| ข้อมูล | `%USERPROFILE%\.claude\serenium\` | อยู่กับต้นทาง ไม่ย้ายตามโปรเจกต์ |

```
Serenium/
├── core/
│   ├── sources/
│   │   ├── transcripts.mjs    projects/**/*.jsonl
│   │   ├── live.mjs           sessions/<pid>.json
│   │   ├── settings.mjs       settings.json
│   │   ├── skills.mjs         .claude.json → skillUsage
│   │   ├── artifacts.mjs      Artifact toolUseResult + frame-link
│   │   ├── vault.mjs          %USERPROFILE%\Obsidian\**
│   │   ├── agentos.mjs        agent-os/{rubrics,routines,knowledge}
│   │   └── limits.mjs         .claude.json → cachedUsageUtilization
│   ├── wikilinks.mjs          แปลง [[…]] เป็นเส้นเชื่อม
│   ├── graph.mjs              ประกอบ node + edge + cluster
│   ├── rollup.mjs             snapshot รายวัน
│   ├── schedules.mjs          schedules.json ของเราเอง
│   ├── pricing.mjs            ตารางราคาต่อ token
│   ├── watch.mjs              เฝ้าไฟล์ → patch
│   ├── model.mjs              ประกอบ World Model
│   └── cli.mjs                serenium scan | rollup | graph
├── shell/
│   ├── main.mjs               เป็นเจ้าของ core, เปิด IPC
│   ├── preload.mjs            contextBridge
│   └── renderer/
│       ├── scene/             three.js: worldnet, vectors, graph, camera
│       ├── ring/              วงแหวน DOM
│       ├── widgets/           widget 6 ตัว + gallery
│       └── grid/              gridstack + pages + keep-out disc
└── docs/superpowers/specs/
```

---

## 4. World Model — สัญญาระหว่าง core กับ shell

`core` คายก้อนเดียว `shell` กินก้อนนี้อย่างเดียว **shell ไม่แตะไฟล์เอง**

```js
{
  meta:     { generatedAt, sourceCounts, degraded: [{source, error}] },
  graph:    {
    nodes:    [{ id, type, label, area, folder, cluster, at, touched }],
    edges:    [{ source, target, rel }],
    clusters: [{ id, label, area, nodeCount, collapsed }]
  },
  ring:     [{ id, kind: 'session'|'artifact', label, at }],
  live:     [{ pid, sessionId, name, status, waitingFor, cwd, since }],
  usage:    { days: [...], today, projection },
  limits:   { fiveHour, sevenDay, fetchedAt, stale },
  config:   { configured: {model, effort}, actual: {models, efforts} },
  skills:   [{ id, usageCount, lastUsedAt }],
  routines: [{ id, cadence, lastRun, nextRun, status }]
}
```

### กฎที่ห้ามละเมิด

1. **ก้อนเดียว ไม่ใช่หลาย endpoint** — กราฟต้อง join ข้ามแหล่งอยู่แล้ว
   (artifact→session→file) แยก endpoint จะต้อง join สองรอบ

2. **`id` ของโหนดคงที่ตลอดกาล** — ใช้ `sessionId`, artifact URL, path เทียบ vault root
   **ห้ามใช้ index หรือเลขรัน** เพราะตำแหน่งในกราฟถูก cache ตาม `id` ถ้า id เปลี่ยน
   ตำแหน่งจะสลับทุกครั้งที่เปิด และผู้ใช้จะจำตำแหน่งอะไรไม่ได้เลย

3. **แต่ละแหล่งพังแยกกัน** — source ที่ throw ต้องกลายเป็น `meta.degraded` ไม่ใช่ทำแอปล่ม
   widget ที่ `needs` แหล่งนั้นจะขึ้นสถานะว่างที่ซื่อสัตย์ **ห้ามแหล่งเดียวพังแล้วจอขาว**

4. **ค่าใช้จ่ายเก็บเป็น ledger ต่อท้ายอย่างเดียว** — แถวละ
   `(date, provider, model, tokensIn, tokensOut, priceAtTime, cost)`
   **ห้ามเก็บยอดรวมแล้วคำนวณใหม่จากราคาปัจจุบัน** — นั่นคือบั๊กที่ทำให้ตัวนับค่าใช้จ่าย
   ของ Roo Code พังตอนสลับโมเดล และมันพังเงียบตอนราคาเปลี่ยน

---

## 5. แหล่งข้อมูล

ทุกตัวเลขด้านล่าง **วัดจากเครื่องจริงเมื่อ 2026-08-23/24** ไม่ใช่ประมาณ

| source | อ่านจาก | ได้อะไร | สภาพจริง |
|---|---|---|---|
| `transcripts` | `~/.claude/projects/**/*.jsonl` | session, token, model, effort, skill attribution, tool use, file paths | 29 ไฟล์ 62MB 14,287 records |
| `live` | `~/.claude/sessions/<pid>.json` | agent ที่รันอยู่ตอนนี้ + `status` + `waitingFor` | 4 entries (transient) |
| `settings` | `~/.claude/settings.json` | model/effort ที่ตั้งไว้ | `opus` / `xhigh` |
| `skills` | `~/.claude.json → skillUsage` | usageCount + lastUsedAt | 16 entries |
| `artifacts` | transcripts (`toolUseResult`, `frame-link`) | title, url, version, session | 22 publish + 44 frame-link |
| `vault` | `%USERPROFILE%\Obsidian\**` | เอกสาร .md ทั้งหมด | **2,787 ไฟล์** |
| `agentos` | `~/.claude/agent-os/` | rubric 4 · routine 4 · decision 5 | มีอยู่แล้ว |
| `limits` | `~/.claude.json → cachedUsageUtilization` | 5h / 7d utilization | snapshot จุดเดียว |

### กับดักที่ยืนยันแล้วจากการสำรวจ

- **ห้ามอ่าน session list จาก `~/.claude/sessions/`** — มันคือทะเบียน PID ที่มี 4 รายการ
  ของจริงอยู่ที่ `projects/*.jsonl` มี **27** ต่างกัน 7 เท่า ต่อผิดจะดูเหมือนข้อมูลหายเกลี้ยง
- **`history.jsonl` ไม่มี Obsidian vault เลย** — 82 records จาก 5 project และไม่มี MyProject
  **ห้ามใช้เป็นแหล่งนับกิจกรรม**
- **`telemetry/` ฝัง account/org UUID** — ห้ามอ่าน ห้ามแสดง ห้ามส่งออก
- **`settings.json` เก็บ alias (`opus`) แต่ transcript เก็บ id จริง (`claude-opus-5`)**
  ต้องมี alias map ไม่งั้นจะจับคู่ไม่ได้
- ที่ตั้งไว้กับที่ใช้จริงไม่ตรงกัน: ตั้ง `opus`/`xhigh` แต่มี `claude-sonnet-5` 767 ข้อความ
  และ `effort=max` 437 ครั้ง — widget Model/Effort ต้องแสดง **ทั้งสองค่า**

---

## 6. Knowledge Graph

### ชนิดโหนดและจำนวนจริง

| ชนิด | จำนวน | ที่มา |
|---|---|---|
| Session | 27 | transcripts |
| Artifact | 22 | transcripts |
| Skill | 63 | `.agents/skills` |
| Rubric | 4 | agent-os |
| Routine | 4 | agent-os |
| Decision | 5 | agent-os |
| Document | **2,787** | vault |
| **รวม** | **~2,912** | |

### เส้นเชื่อม

| rel | ที่มา | จำนวนโดยประมาณ |
|---|---|---|
| `links` | wiki-link `[[…]]` ระหว่างเอกสาร | **9,052** |
| `touched` | `tool_use.input.file_path` + `file-history-delta.trackingPath` | ~800 |
| `produced` | artifact → session | 22 |
| `used` | skill → session (`attributionSkill`) | ~400 |
| `enforces` | skill → rubric | 4 |
| `in` | node → cluster | 2,912 |
| **รวม** | | **~10,500** |

### คลัสเตอร์ — สภาพเริ่มต้น

**8 คลัสเตอร์** ไม่ใช่ 5 — External Knowledge แตกเป็นห้องสมุดละก้อน ไม่ยุบรวมเป็นก้อนเดียว

| คลัสเตอร์ | โหนด | เริ่มต้น |
|---|---|---|
| Claude (session, artifact, skill, rubric, routine, decision) | 125 | คลี่ |
| MyProject | 18 | คลี่ |
| Cybersecurity Knowledge Graph | 26 | คลี่ |
| Hermes Knowledge Base | 84 | คลี่ |
| Reference Library A | 388 | ยุบ |
| Cybersecurity Handbook | 535 | ยุบ |
| AI Security Second Brain | 560 | ยุบ |
| Reference Library B | 1,175 | ยุบ |

เปิดมาเห็น ~253 โหนดจริง + 4 ก้อนที่กดแตกได้

**เหตุผล:** 2,912 โหนดพร้อมกันจะเป็นก้อนขนยุ่งที่อ่านไม่ออก และห้องสมุดภายนอกทั้งสี่
รวมกันคือ 91% ของทั้งหมด แต่เป็นความรู้ที่นำเข้ามา ไม่ใช่งานที่ผู้ใช้คิดเอง ถ้าไม่ยุบ
งานของผู้ใช้เอง (128 เอกสาร + 125 ของ Claude) จะจมหายไปใต้ของนำเข้า

**ทำไมสี่ก้อนไม่ใช่ก้อนเดียว:** ก้อนเดียวชื่อ "External Knowledge" บอกได้แค่ว่ามีของ
นำเข้าอยู่เยอะ สี่ก้อนบอกว่า**เรื่องอะไรบ้างและหนักไปทางไหน** ซึ่งเป็นข้อมูลที่ดูแล้ว
ใช้ต่อได้จริง แถมยังให้แต่ละห้องสมุดมีสีของตัวเองในฉาก

> **Reference Library A/B ใช้ชื่อกลางในเอกสารนี้** ชื่อโฟลเดอร์จริงบนดิสก์อ่านเหมือน
> ชื่อบุคคล และ repo นี้เป็น public — จำนวนไฟล์เป็นของจริง ชื่อไม่ใช่ โค้ดที่อ่าน vault
> จริงจะเห็นชื่อจริงจากดิสก์ตามปกติ ไม่มีการ map ชื่อฝังไว้ในโค้ด

### ลำดับการสร้างกลไกกันขนยุ่ง

**B (คลัสเตอร์) → C (ตัวกรอง) → A (LOD)**

คลัสเตอร์มาก่อนเพราะ **ถ้าไม่มีคลัสเตอร์ ก็ไม่มีหน้าจอเริ่มต้น** ตัวกรองกับ LOD เป็นของที่
เพิ่มทีหลังได้จริง

### ไฟล์ที่ไม่เชื่อมกับใคร

**1,374 ไฟล์ (49%) ไม่มี wiki-link เลย** → จับกลุ่มตามโฟลเดอร์ให้มีที่ยืน
ไม่ซ่อน ไม่ปล่อยลอยเป็นฝุ่น

### ตัวแปลง wiki-link — งานที่ต้องทำจริงจัง

Obsidian แปลง `[[ชื่อ]]` โดยค้นจาก **ชื่อไฟล์ ไม่ใช่ path** ที่ 2,787 ไฟล์จะมีชื่อซ้ำแน่นอน

ต้องรองรับ: `[[Name]]` · `[[Name|alias]]` · `[[Name#heading]]` · `[[folder/Name]]` ·
ลิงก์ที่ชี้ไปไฟล์ที่ไม่มีอยู่

- สร้างดัชนี `filename → [paths]` ก่อน
- ชื่อซ้ำ → ใช้กฎ "ใกล้ที่สุดก่อน" (โฟลเดอร์เดียวกัน → พาเรนต์ → ทั้ง vault)
- **ลิงก์ที่หาปลายทางไม่เจอต้องบันทึกเป็น `broken` ไม่ใช่ทิ้งเงียบ** — ผู้ใช้อาจอยากเห็น

---

## 7. ฉาก 3 มิติ

```
พื้นหลัง grid dot        CSS radial-gradient หลัง canvas
└─ scene (three.js)      ฉากเดียว — เงื่อนไขที่ทำให้บินทะลุได้จริง
   ├─ WorldNet           ทรงกลม + halo
   ├─ VectorField        อนุภาคไหลรอบทรงกลม
   └─ Graph              three-forcegraph Object3D (ซ่อนจนกดเข้าไป)
└─ วงแหวน Session/Artifact   DOM
└─ Widget layer              DOM ชั้นบนสุด
```

### WorldNet

- **จุดสว่าง 2,912 จุด — หนึ่งจุดต่อหนึ่งโหนดจริง** ตำแหน่งจาก hash ของ `id` (คงที่ตลอดกาล)
- **ไม่ใช้ texture เลย** — ตาม GitHub globe
- ทั้งหมดผ่าน `InstancedMesh` = **1 draw call**
- **halo** = ทรงกลมด้านหลังขยาย 1.15× shader ไล่สี ไว้บังขอบหยัก
- สีจุด = ชนิดโหนด · ความสว่าง = ความสดใหม่ · **เต้นเป็นจังหวะเมื่อมี agent ทำงานจริง**

> หมายเหตุ: เดิมออกแบบให้มีจุดพื้นหลัง 4,000 จุดเสริมความหนาแน่น **ตัดทิ้งแล้ว**
> เพราะโหนดจริง 2,912 จุดหนาแน่นพอด้วยตัวเอง และซื่อสัตย์กว่า

### Network Vectors

อนุภาควิ่งตามส่วนโค้งวงกลมใหญ่ที่รัศมี 1.25–1.6 มีหางจาง สีอิงชนิดโหนด
**ไหลเร็วขึ้นเมื่อมี session ทำงานอยู่** — ของตกแต่งที่บอกสถานะไปในตัว

### วงแหวน — DOM ไม่ใช่ 3 มิติ

เหตุผล: ชิ้นด้านหลังทรงกลมจะไม่โดนบัง · hover/คลิกได้ฟรี · ใช้คีย์บอร์ดได้ ·
ไม่กินงบ FPS · และภาพอ้างอิง (RUBRIC) ก็ทำแบบนี้

- จำนวนชิ้นคำนวณจากเส้นรอบวงให้พอดีจอ
- **Session = วงกลม · Artifact = สี่เหลี่ยม**
- hover → ขึ้นชื่อ · คลิก → บินเข้ากราฟแล้วโฟกัสโหนดนั้น

### Search (Ctrl+F)

**ตัวที่ตรงหลุดออกจากผิวทรงกลม ลอยเข้าหากล้อง กลายเป็นการ์ดมีชื่อ**

เลือกวิธีนี้เพราะเป็นความจริงตามตัวอักษร — ทรงกลมมีจุดครบทุกโหนดอยู่แล้ว
วงแหวนคือของล่าสุด การค้นหาคือการดึงออกมาจากทรงกลม ทั้งสามอย่างจึงเป็นเรื่องเดียวกัน

### การบินทะลุ

1. กด → วงแหวนกับ widget จางหาย
2. กล้องพุ่งไปข้างหน้า จุดบนเปลือกแหวกออกด้านข้างแล้วจางตอนกล้องผ่าน
3. ข้ามเปลือก → กราฟปรากฏ **โดยจัดวางเสร็จแล้ว**
4. ถอยกลับ = ย้อนทุกขั้น

**ตำแหน่งกราฟต้อง cache ตาม `id`** — รันฟิสิกส์ครั้งเดียว เก็บผล เรียกคืนตอนเข้า
ถ้าไม่ทำ ผู้ใช้จะเห็นกราฟดิ้นเข้าที่ทุกครั้ง ซึ่งช้าและทำลายความทรงจำเชิงตำแหน่ง

### สองกลไกบังคับ

- **ลดคุณภาพอัตโนมัติ** — เฝ้า FPS ถ้าตกต่ำกว่า ~55 ติดกัน 50 เฟรม ลดเป็นขั้น:
  pixel density 2.0→1.5 → ลดอนุภาค → ลดจุด (กลไกเดียวกับ GitHub globe)
- **หยุดคำนวณเมื่อนิ่ง** — ฟิสิกส์ดับเองเมื่อเข้าที่ · หยุด render เมื่อหน้าต่างถูกซ่อน

---

## 8. ระบบ Widget

ใช้ **gridstack** (0 dependency, MIT, 87KB) สำหรับกลไก drag/resize/ชนกัน
ส่วนหน้าและเขตห้ามวางทำเองทับข้างบน

> ไม่ใช้ muuri เพราะปรับขนาดไม่ได้ · ไม่ใช้ react-grid-layout เพราะต้องใช้ React
> และ compaction เป็น O(n²)
> **ต้องใช้ `renderCB` ไม่ใช่ `content`** — `content` เก็บ HTML ดิบและเป็นช่องโหว่ XSS

### ขนาด

ช่องสี่เหลี่ยมจัตุรัส ~110px ระยะห่าง 16px คอลัมน์ปรับตามความกว้างหน้าต่าง
(ตัวเลขนี้ยังไม่ล็อก — ปรับตอนเห็นของจริง)

| ขนาด | ช่อง | ใช้กับ |
|---|---|---|
| S | 2×2 | Model/Effort |
| M | 4×2 | Live Agents · Burn |
| L | 4×4 | Skill deck · Routine |
| XL | 6×4 | Usage calendar |

กดเพิ่มได้ขนาดสำเร็จรูป **แต่ลากขอบขยายได้ตามช่องกริด** ภายใน `min`/`max` ที่ widget ประกาศ

### เขตห้ามวาง

จานวงกลมกลางจอ รัศมี = รัศมีทรงกลม + ระยะขอบ

- วางอัตโนมัติ → ข้ามช่องที่จุดกึ่งกลางตกในจาน (ไปเกาะขอบเสมอ)
- ลากเอง → ทับได้ (layer สูงกว่า)
- เมื่อทับ → widget ได้พื้นหลังทึบขึ้นอัตโนมัติ ไม่งั้นอ่านไม่ออกบนอนุภาคที่วิ่งอยู่

### หน้า

เริ่มหน้าเดียว กด + เพิ่มเอง · เปลี่ยนหน้า → **ทรงกลมหมุนรอบแกน Y** พร้อม widget เลื่อน ·
ทรงกลมอยู่ทุกหน้าเป็นหมุดยึด · จุดบอกหน้าแบบ iPad

### สัญญาของ widget

```js
{ type:'live-agents', name:'Live Agents', icon,
  sizes:['S','M'], min:{w:2,h:2}, max:{w:6,h:4},
  needs:['live'],        // อ่านส่วนไหนของ World Model
  refresh:'live',        // live | slow
  render(el, model, size) }
```

`needs` ทำให้เปลือกวาดสถานะว่างให้เองเมื่อแหล่งนั้น degraded — widget ไม่ต้องเขียนซ้ำ

`refresh` มีสองค่าเท่านั้น:
- **`live`** — วาดใหม่ทันทีที่ได้ patch ของแหล่งที่มันต้องการ (ใช้กับ Live Agents, Model/Effort)
- **`slow`** — วาดใหม่เมื่อได้ patch หรืออย่างช้าทุก **60 วินาที** แล้วแต่อะไรมาก่อน

**ทุก widget ต้องแสดงเวลาที่ข้อมูลของตัวเองอัปเดตล่าสุด** — dashboard ที่โชว์ข้อมูลเก่าเงียบๆ
แย่กว่าไม่โชว์เลย (Dashy ดึงข้อมูลแค่ตอนโหลดหน้าถ้าไม่ตั้ง `updateInterval` และเป็นข้อร้องเรียนจริง)

### widget ใน v1

| widget | ขนาด | refresh | เปิดเป็นค่าเริ่มต้น |
|---|---|---|---|
| Live Agents | M | live | ✅ |
| Burn & Projection | M | slow | ✅ |
| Model / Effort | S | live | ✅ |
| Skill deck | L | slow | — |
| Usage calendar | XL | slow | — |
| Routine | L | slow | — |

**สร้าง 6 เปิด 3** — ผู้ใช้ปรับเองได้ว่าจะเปิดเท่าไร
เหตุผล: งานวิจัยยืนยันปัญหา "กำแพงตัวเลือก" (Open WebUI มี 250+ โมเดลจนผู้ใช้บ่นหนัก)

### รูปแบบเก็บ layout

`~/.claude/serenium/layout.json`

```json
{ "version": 1,
  "pages": [{ "id": "p1", "widgets": [
    { "id": "w1", "type": "live-agents", "x": 0, "y": 0, "w": 4, "h": 2, "settings": {} }
  ]}]}
```

กฎจาก JSON Canvas:
1. **ฟิลด์ที่ไม่รู้จัก → เก็บไว้ เพิกเฉย ห้ามเป็น error**
2. **widget type ที่ไม่รู้จัก → แสดงไทล์บอกว่าหายไปอะไร ห้ามทิ้งเงียบ**
   (ทิ้งเงียบ = การจัดวางของผู้ใช้หายถาวรโดยไม่มีใครรู้)

---

## 9. การไหลของข้อมูล

### IPC

`contextIsolation: true` · `nodeIntegration: false` · `sandbox: true`
**renderer ไม่มี `fs` ไม่มี `require`**

```js
window.serenium = {
  getModel(), onPatch(cb), action(name, args), layout: { get(), save() }
}
```

### patch ไม่ใช่ก้อนเต็ม

โหลดแรก → ก้อนเต็ม · หลังจากนั้น → patch เจาะจงส่วน
**patch ของ `live` ห้ามแตะ `graph`** ไม่งั้นตำแหน่งที่ cache ไว้จะหาย

### การเฝ้าไฟล์ — กับดักใหญ่สุดของส่วนนี้

`~/.claude/projects/` มี 62MB และ **ถูกเขียนต่อท้ายหลายบรรทัดต่อวินาทีขณะ session ทำงาน**
เฝ้าแบบอ่านใหม่ทั้งไฟล์ = แยกไฟล์ 15MB ซ้ำๆ ตลอดเวลา

**ต้องจำ byte offset ต่อไฟล์ อ่านเฉพาะส่วนที่เพิ่ม** และรวบเหตุการณ์ถี่ๆ เป็น patch เดียวทุก ~250ms

| เฝ้า | ความถี่ | วิธี |
|---|---|---|
| `sessions/*.json` | บ่อย | เฝ้าตรงๆ |
| `projects/**/*.jsonl` | ถี่มาก 62MB | **อ่านต่อท้ายจาก offset** |
| `settings.json`, `.claude.json` | นานๆ ครั้ง | เฝ้าตรงๆ |
| vault `**/*.md` | นานๆ ครั้ง | เฝ้า ยกเว้น `node_modules`, `.obsidian`, `.git`, `dist` |

### ลำดับตอนเปิดแอป

1. หน้าต่างเปิด → วาด grid dot + ทรงกลมเปล่า **ทันที**
2. อ่าน cache จากดิสก์ → จุดขึ้นเต็มทรงกลม
3. สแกนจริงเบื้องหลัง → ส่ง patch
4. เริ่มเฝ้าไฟล์

**ห้ามให้เห็นจอเปล่าระหว่างแยก 62MB**

### การเขียนกลับ

`action()` เป็นทางเดียวที่เขียนได้ · renderer ไม่เขียนไฟล์เอง ·
การแก้ `settings.json` คือการแก้ config จริงของ Claude Code **ต้องเห็นชัด ไม่เปลี่ยนเงียบ**

---

## 10. Rollup และการรักษาข้อมูล

**ปัญหา:** Claude Code มี retention sweep ลบ `projects/*.jsonl` เก่าทิ้ง
เดิม `cleanupPeriodDays` ไม่ได้ตั้ง → ใช้ค่า default 30 วัน → **ข้อมูลจะหายถาวร**

**แก้แล้ว:** ตั้ง `cleanupPeriodDays: 90` ไม่ได้แทนที่การเก็บ snapshot แต่ขยายหน้าต่าง
ความปลอดภัยเป็นสามเท่าให้ทุกทางเลือกของ trigger

ข้อมูลที่มีตอนนี้: **10 วัน (2026-08-15 → 08-24)** ไม่มีอะไรเก่ากว่านั้นในเครื่อง

**สถานะ: ทำแล้ว** — `rollup.mjs` เก็บ 10 วัน + 30 session ไว้ที่
`~/.claude/serenium/usage/` ต้องย้ายเข้า `core/`

### แบ่งวันตามเวลาท้องถิ่น

**ตัดสินแล้ว: เวลาท้องถิ่น** (`Asia/Bangkok`, UTC+7) ไม่ใช่ UTC — 01:00 ของวันพฤหัสคือ
วันพฤหัส ไม่ใช่วันพุธที่ UTC จะจัดให้ ทุกไฟล์ประทับ `tz` และ `tzOffsetMinutes` ไว้
**ห้ามอ่านไฟล์รายวันโดยไม่ดูสองค่านี้** — ไม่งั้นประวัติที่คร่อมการย้าย timezone
จะอ่านออกมาเป็นรอยขาดแบบเงียบๆ

ย้ายข้อมูลเดิมครบทั้ง 10 วันแล้ว ทำได้เพราะต้นฉบับยังอยู่ครบ ตรวจแล้วทุกวันในอดีต
ตรงกับต้นฉบับ**ทุกข้อความ** ผลที่เปลี่ยน: **08-21 จาก 1,095 → 2,124 ข้อความ**
งานหลังเที่ยงคืนเกือบพันข้อความเคยถูกนับเป็นของวันก่อนหน้า

> หน้าต่างการย้ายนี้ปิดตัวเอง — ถ้าปล่อยจน sweep กินต้นฉบับไป จะย้ายวันเก่าไม่ได้อีกเลย

### คุณสมบัติที่ห้ามเสีย

- **วันที่ผ่านไปแล้วต้องไม่ถูกคำนวณใหม่** — ต้นทางอาจถูกลบไปแล้ว การคำนวณใหม่จะทำให้เป็นศูนย์
  (ทดสอบแล้ว: รันซ้ำได้ `9 kept, 1 written`)
- วันปัจจุบันเขียนทับได้เพราะยังสะสมอยู่

### รันเมื่อไหร่

| ช่วง | คนสั่ง |
|---|---|
| ตอนนี้ → จบเฟส 2 | **SessionStart hook** — `rollup-hook.mjs` |
| หลังเฟส 2 | **Serenium Core เอง** — autostart ตอน login + ย่อลง tray แล้วลบ hook ทิ้ง |

**กฎที่ทำให้เปลี่ยนคนสั่งได้โดยไม่ต้องแตะตรรกะ:** `rollup.mjs` ต้องรันเดี่ยวได้เสมอผ่าน
`serenium rollup` แอปเป็นแค่คน**สั่ง** ไม่ใช่คน**บรรจุ** แอป spawn มันเป็น process แยก —
แอปพังหรือ auto-update ล้ม ก็แค่ trigger หาย ตัวเก็บข้อมูลไม่เจ็บ

**ทำไมไม่ใช้ Task Scheduler:** ครอบคลุมเท่าแอปที่ autostart แต่**พังแบบเงียบ**
จะรู้ตัวอีกทีตอนข้อมูลหายไปแล้ว แอปมีหน้าจอบอกได้ →
**หน้าแรกต้องแสดง `index.json.updatedAt` และเตือนเมื่อค้างเกิน 2 วัน**

**สองกฎที่ hook ห้ามละเมิด:**

1. **ห้ามเขียน stdout** — stdout ของ SessionStart ถูกยัดเข้า context ของโมเดล
   hook ที่พูดมากจะปนเปื้อนทุก session แบบเงียบๆ
2. **ห้าม fail** — hook พังจะไปโผล่เป็น error ใน session ที่ไม่เกี่ยวอะไรด้วยเลย
   กลืน exception ทั้งหมด เขียนลง `usage/rollup.log` แทน แล้ว exit 0 เสมอ

หน่วงด้วย `index.json.updatedAt`: ถ้าเก็บของวันนี้แล้วก็ออกเลย ถ้ายังก็ spawn
แบบ detached ไม่ถ่วงการเปิด session ที่สำคัญคือ index จะขยับเฉพาะตอนรันสำเร็จ
**รันพลาดจึงถูกลองใหม่ ไม่ใช่ถูกข้าม** และมี lock กัน session ที่เปิดพร้อมกัน
ไม่ให้ยิงสแกน 80MB พร้อมกันหลายตัว

**ต้นทุนจริงที่วัดได้:** ทางลัด (เก็บวันนี้แล้ว) เสีย ~0.28 วิ ซึ่งเกือบทั้งหมดคือเวลา
start ตัว node เอง ไม่ใช่งานของ hook — ทุก session จ่ายเท่านี้ ส่วนวันที่ต้องเก็บจริง
จ่ายเท่าเดิมเพราะ rollup (2.6 วิ) แยกไปเป็นอีก process ตัวเลข 2.6 วินี้โตตามขนาด
transcript ที่สะสม จึงเป็นเหตุผลที่มันต้องไม่อยู่ในเส้นทางเปิด session

---

## 11. ความปลอดภัยและความเป็นส่วนตัว

- **ห้ามอ่าน:** `.credentials.json`, `daemon/control.key`, `daemon/pipe.key`, `sessions/*.key`
- **ห้ามแสดงหรือส่งออก:** `telemetry/**` (ฝัง account/org UUID + base64 process metadata)
- **fixture ในเทสต์ต้องลบเนื้อหาออก** — เก็บแค่โครง key **ห้าม commit prompt หรือบทสนทนาจริง**
- แอปไม่ส่งอะไรออกนอกเครื่อง — ไม่มี telemetry ไม่มี network call ใน v1

---

## 12. การตัดสินใจที่ยังค้าง

บันทึกไว้ตามจริง ไม่เดาแทน

| # | เรื่อง | ทางเลือก |
|---|---|---|
| 1 | ขนาดช่องกริด 110px + gap 16px | รอดูของจริง |
| 2 | กลไก marketplace ในอนาคต | JS module (แบบ HACS) หรือ declarative card (แบบ Windows) |
| 3 | รายละเอียด multi-AI routing | เลื่อนไปหลัง v1 |

**ปิดไปแล้ว** (2026-08-24)

- ~~แบ่งวันตาม UTC หรือเวลาท้องถิ่น~~ → **เวลาท้องถิ่น** ย้ายข้อมูลเดิมแล้ว ดูข้อ 10
- ~~rollup รันเองยังไงเมื่อไม่ได้เปิดแอป~~ → **hook ก่อน แล้วให้แอปรับช่วงหลังเฟส 2** ดูข้อ 10

---

## 13. การแบ่งเฟส

spec นี้ใหญ่เกินกว่าจะเป็นแผน implement เดียว แบ่งเป็น 4 เฟส **แต่ละเฟสจบแล้วต้องมีของที่รันได้จริง**

### เฟส 1 — `core` เปล่าๆ ไม่มี UI

sources ทั้ง 8 · `wikilinks.mjs` · `graph.mjs` · `model.mjs` · ย้าย `rollup.mjs` เข้ามา · `cli.mjs`

**จบเฟสได้อะไร:** `serenium scan` พิมพ์ World Model ออกมาทาง stdout ครบทุกส่วน
พร้อม golden test **พิสูจน์ได้ว่าข้อมูลถูกต้องก่อนที่จะเสียเวลาวาดอะไรเลย**

### เฟส 2 — เปลือกกับฉาก

Electron + IPC + `watch.mjs` · พื้นหลัง grid dot · WorldNet + halo · Network Vectors ·
วงแหวน DOM · Ctrl+F · **autostart ตอน login + ย่อลง tray** · **รับช่วง trigger ของ rollup
ต่อจาก hook** พร้อมตัวบอกวันที่เก็บล่าสุดบนหน้าแรก

**จบเฟสได้อะไร:** หน้าจอหลักที่สวยและมีชีวิต แต่ยังกดเข้าไปข้างในไม่ได้

> autostart + tray ไม่ใช่ของตกแต่ง — มันคือสิ่งที่ทำให้แอปครอบคลุมเท่า Task Scheduler
> ได้ ถ้าไม่มี "แอปเปิดอยู่" จะแปลว่า "ผู้ใช้กดเปิดเอง" และการเก็บข้อมูลจะกลับไปพึ่งดวง
> พอเฟสนี้จบ ให้ลบ hook ใน `settings.json` กับ `rollup-hook.mjs` ทิ้ง

### เฟส 3 — กราฟและการบินทะลุ

`three-forcegraph` ในฉากเดียวกัน · คลัสเตอร์ + คลี่/ยุบ · cache ตำแหน่ง · animation บินทะลุ ·
ลดคุณภาพอัตโนมัติ

**จบเฟสได้อะไร:** วงจรครบ — จากทรงกลม เข้ากราฟ กลับออกมา

### เฟส 4 — widget

gridstack + เขตห้ามวาง + หน้า + `layout.json` + gallery · widget 6 ตัว

**จบเฟสได้อะไร:** v1 สมบูรณ์

> เฟส 1 เป็นเฟสที่ลดความเสี่ยงมากที่สุด เพราะถ้าข้อมูลผิด (เช่นตัวแปลง wiki-link พลาด
> หรือ session list ต่อผิดที่) จะรู้ตั้งแต่ยังไม่ลงทุนกับงานภาพเลย

---

## 14. การทดสอบ

- **fixture ลบเนื้อหา** เก็บแค่โครง key จาก transcript จริง
- **golden test** — fixture tree → World Model ก้อนเดิมเป๊ะ จับ regression ตอนแก้ parser
- **เทสต์ rollup ไม่คำนวณวันเก่าใหม่** — ถ้าพังจะพังเงียบและกู้ไม่ได้
- **เทสต์อ่านต่อท้าย** — เขียนต่อท้าย fixture แล้วต้องอ่านเฉพาะส่วนใหม่
- **เทสต์ตัวแปลง wiki-link** — ชื่อซ้ำ, alias, heading, path เต็ม, ลิงก์เสีย
- **งบเวลา** — สร้าง model จาก fixture ใหญ่ต้องเสร็จในเวลาที่กำหนด
- **shell** — smoke test ว่าเปิดหน้าต่างแล้ววาดได้เท่านั้น (เทสต์ UI ลึกแพงและเปราะ)

---

## 15. ที่มาของการตัดสินใจสำคัญ

| ตัดสินใจ | เพราะ |
|---|---|
| สร้างเอง ไม่ติดตั้ง RUBRIC | วิธีติดตั้งคือ prompt ที่รันในตัว agent = รันคำสั่งจากแหล่งที่ไม่น่าเชื่อถือด้วยสิทธิ์เข้าถึงไฟล์ |
| Electron ไม่ใช่ Tauri | ไม่ต้องลง Rust toolchain ใหม่ · Chromium ที่ฝังมาทำให้ผล WebGL คาดเดาได้ · แลกกับขนาดไฟล์ |
| แกนแยกจากเปลือก | ปลายทางคือ multi-AI — ตรรกะเลือกเส้นทางต้องไม่อยู่ในโค้ดวาดภาพ |
| `three-forcegraph` ไม่ใช่ `3d-force-graph` | ตัวหลังสร้าง renderer/scene ของตัวเอง ทำให้กล้องบินทะลุไม่ได้จริง ตัวแรกเป็น `Object3D` ที่ยัดเข้าฉากเราได้ |
| วงแหวนเป็น DOM | เข้าถึงด้วยคีย์บอร์ดได้ · ไม่โดนทรงกลมบัง · ไม่กินงบ FPS |
| คลัสเตอร์มาก่อนตัวกรอง | 2,912 โหนดทำให้ก้อนขนยุ่งเป็นสิ่งที่เกิดแน่ ไม่ใช่ความเสี่ยง — ถ้าไม่มีคลัสเตอร์ก็ไม่มีหน้าจอเริ่มต้น |
| ค่าใช้จ่ายเป็น ledger | ตัวนับของ Roo Code พังตอนสลับโมเดล (issue #7755) |
| เปิด 3 จาก 6 widget | "กำแพงตัวเลือก" เป็นปัญหาที่มีหลักฐานจริงในเครื่องมือแนวนี้ |

### สัญญาณเตือนจากตลาดที่รับมาแล้ว

- Google Antigravity ทำ Agent Manager ดูหลาย agent พร้อมกัน แล้ว **ถอดทิ้งใน v2**
- OpenAI Agent Builder **ปิด 30 พ.ย. 2026**
- AgentGPT **archive แล้ว**

→ canvas/graph เหมาะกับการ "เขียน" และ "อ่านย้อนหลัง" **ไม่เหมาะกับการเฝ้าดูตอนทำงาน**
ซึ่งตรงกับการออกแบบนี้พอดี: World Network เป็นของสวยงามและเป็นทางเข้า ส่วนงานเฝ้าดูจริง
อยู่ที่ widget แบบแผง — ซึ่งเป็นรูปแบบที่รอดในตลาด
