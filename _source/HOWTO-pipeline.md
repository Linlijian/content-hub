# HOWTO: จาก PDF ต้นฉบับ → บทเรียน + ข้อสอบ + เว็บ HTML โต้ตอบ (ทำจบในรอบเดียว)

> ไฟล์นี้เป็น **พรอมพ์สำหรับ Claude Code** รอบถัดไป อ่านให้จบก่อนลงมือ แล้วทำตามลำดับ
> repo: `D:\claude.ai\content-hub` (แอป KnowledgeQuizApp) · คู่มือหลัก: `GUIDE.md` · คู่มือซ่อม md: `_source/fix-markitdown.md`

---

## 0. พรอมพ์ตัวอย่างที่ผู้ใช้จะพิมพ์

```
อ่าน _source/HOWTO-pipeline.md แล้วทำทั้งหมดกับ PDF ใน <โฟลเดอร์ PDF>
เล่ม 09 = "<ชื่อหนังสือ>"  บทที่ต้องการ: 1, 3, 6
หมวด (category): "นักวิชาการคอมพิวเตอร์ศาลยุติธรรม"
```

ถ้าผู้ใช้ไม่ได้บอกค่าใดข้างบน ให้ใช้ค่าเริ่มต้นในข้อ 1.3 แล้วบอกในรายงานตอนจบ **อย่าหยุดถาม**
ยกเว้นเรื่องที่ทำเองไม่ได้จริงๆ (ดูข้อ 8)

---

## 1. ภาพรวม pipeline

```
PDF ต้นฉบับ (แบ่งไฟล์รายช่วงหน้า เช่น 07_pages_11-40.pdf)
  │ ขั้น A  batch_markitdown.py                      → 07_pages_11-40.md       (ดิบ)
  │ ขั้น B  fix_markitdown.py                        → 07_pages_11-40.clean.md (ซ่อมอักขระ/หัวข้อ)
  │ ขั้น C  Claude อ่าน clean.md → เขียน SUMMARY    → quiz_build/ch/<mod>.py  (สรุปรูปแบบมาตรฐาน)
  │ ขั้น D  Claude เขียนข้อสอบ Q + build.py          → x/xN.json, e/eN.json    (บท + ข้อสอบ)
  │ ขั้น E  Claude เขียน html_build/chapters/<mod>.py + gen.py → _source/html/<book>/<ID>.html
  │ ขั้น F  build.py ใส่ htmlFileName ให้บทที่มี HTML
  └ ขั้น G  ผู้ใช้ เข้ารหัส + manifest + push ด้วย QuizContentEncryptor.exe (Claude ทำไม่ได้)
```

### 1.1 โครงสร้างโฟลเดอร์ที่เกี่ยวข้อง

```
content-hub/
├─ GUIDE.md                       ← กติกา JSON / การเข้ารหัส (อ่านข้อ 3–8 ก่อนเริ่ม)
├─ x/xN.json  e/eN.json  p/pN.json ← ต้นฉบับ JSON (gitignored) ขึ้น git เฉพาะ .enc
├─ book/*.html.enc                ← ผู้ใช้เข้ารหัสเอง
└─ _source/
   ├─ batch_markitdown.py(.pyw GUI) fix_markitdown.py  fix-markitdown.md
   ├─ book/*.clean.md             ← ผลขั้น B
   ├─ quiz_build/build.py         ← สร้าง x/e JSON + ตรวจความถูกต้อง
   ├─ quiz_build/ch/<mod>.py      ← 1 ไฟล์ต่อบท: SUMMARY + Q (แหล่งความจริงของเนื้อหา)
   ├─ html_build/gen.py           ← engine สร้างหน้าเว็บ (CSS/JS/helper) — ไม่ต้องแก้
   ├─ html_build/chapters/<mod>.py← 1 ไฟล์ต่อบท: SECTIONS/ภาพ/การ์ด/คำศัพท์/QUIZ
   └─ html/<book-folder>/<ID>.html← ผลลัพธ์ HTML (gitignored)
```

### 1.2 สิ่งที่ทำไปแล้ว (ใช้เป็นตัวอย่างได้ ห้ามชน id)

| เล่ม | x/e | book id | prefix | mod | บท (ID) | โฟลเดอร์ HTML |
|---|---|---|---|---|---|---|
| 07 | x5/e5 | `computer-it-basics` | `cit` | cit1,2,4,5,7,8 | cit-ch1…ch8 | `html/computer-it` |
| 08 | x6/e6 | `computer-info-systems` | `cis` | cis3,5,8 | cis-ch3,5,8 | `html/computer-info-systems` |
| เครือข่าย | x1–x4 | (ของเดิม) | `net` | – | net-ch1…7 | `html/networking` |

ตัวอย่างที่ดีที่สุดให้เปิดดูก่อนเขียน: `quiz_build/ch/cis8.py` และ `html_build/chapters/cis8.py`, `cit8.py`

### 1.3 ค่าเริ่มต้นเมื่อผู้ใช้ไม่ระบุ

| ค่า | ค่าเริ่มต้น |
|---|---|
| เลขไฟล์ x/e | เลขว่างถัดไป (ดู `ls x e`) |
| book id / prefix | ภาษาอังกฤษตัวเล็ก-ขีดกลาง ตั้งจากชื่อเล่ม, prefix 3 ตัวอักษรที่ยังไม่ถูกใช้ |
| category | `นักวิชาการคอมพิวเตอร์ศาลยุติธรรม` (ต้องสะกดตรงกับเล่มอื่นเป๊ะ) |
| author | `Knowledge Hub` |
| บทที่ทำ | ทุกบทที่มีใน clean.md ที่ผู้ใช้ชี้ |
| จำนวนข้อสอบ | ~1 ข้อต่อ 1–2 bullet ของสรุป (บทละ 35–90 ข้อ) |
| HTML | ทำทุกบทที่ผู้ใช้ขอ (ถ้าไม่ได้พูดถึง HTML = ทำด้วย) |

---

## 2. ขั้น A — PDF → .md (markitdown)

```bash
cd /d/claude.ai/content-hub/_source
pip install "markitdown[all]"         # ครั้งแรกเท่านั้น
python batch_markitdown.py "<โฟลเดอร์ PDF>" --dry-run          # ดูรายการก่อน
python batch_markitdown.py "<โฟลเดอร์ PDF>" -o book              # ได้ book/<ชื่อ>.md
```

- ถ้าไฟล์ .md สั้นผิดปกติ = PDF สแกน ต้อง OCR ก่อน (`ocrmypdf --language tha in.pdf out.pdf`) — ถ้าไม่มีเครื่องมือ ให้แจ้งผู้ใช้
- ถ้า `.clean.md` มีอยู่แล้วใน `_source/book/` ข้ามขั้น A–B ได้เลย

## 3. ขั้น B — ซ่อม .md → .clean.md

```bash
python fix_markitdown.py book/07_pages_11-40.md                      # ตรวจอย่างเดียว ดูรายงานก่อน
python fix_markitdown.py book/07_pages_11-40.md -o book/07_pages_11-40.clean.md --profile book --lines
```

- หนังสือจัดหน้าหลายคอลัมน์ → ใช้ `--lines` (คงบรรทัดเดิม เติมแค่ `#` และรายการ) ถ้าไม่ผ่านการตรวจ ลองตัด `--lines` ออก
- สคริปต์ **ไม่เขียนไฟล์ถ้าตรวจความครบถ้วนไม่ผ่าน** — ห้ามบังคับเขียน
- ตรวจด้วยตา: `grep -n "^#" book/X.clean.md` ต้องเห็น `# บทที่ N ...` ครบทุกบท และค้น `" า"` ต้องไม่เหลือ

### ข้อบกพร่องที่ยังเหลือแม้ผ่านขั้น B (Claude ต้องแก้ในใจตอนอ่าน ไม่ต้องแก้ไฟล์)

| อาการ | ตัวอย่างใน clean.md | อ่านเป็น |
|---|---|---|
| สระ/วรรณยุกต์เลื่อน | `เกยี่ วกับ`, `ที้เพื้น` | เกี่ยวกับ, ที่พื้น |
| หัวข้อ `###` ตัดกลางคำ | `### 3.3 ผู` | ผู้ใช้ … (ดูบรรทัดถัดไป) |
| คำบรรยายภาพปนเนื้อหา | `ภาพจาก wikipedia` | ตัดทิ้ง |
| ตาราง/แผนภาพหาย | ข้อความเรียงมั่ว | เดาจากบริบท ถ้าสำคัญมากให้บอกผู้ใช้ว่าควรเทียบ PDF |

**ห้ามแก้ `.clean.md` โดยตรงหลังเริ่มขั้น C** — ต้นฉบับความจริงของเนื้อหาคือ `quiz_build/ch/<mod>.py`

---

## 4. ขั้น C — เขียนสรุป (format .md) → `quiz_build/ch/<mod>.py`

1 บท = 1 ไฟล์ ชื่อ `<prefix><เลขบท>.py` เช่น `cit2.py`

```python
ID = "cit-ch2"                                   # ห้ามซ้ำทั้ง repo, ห้ามเปลี่ยนหลังเผยแพร่
TITLE = "บทที่ 2 ไฟล์และนามสกุลไฟล์ (File and File Extension)"   # ตรงกับหัวบทใน PDF
QSET = "cit-qset-2"
QSET_NAME = "แบบทดสอบ บทที่ 2 ไฟล์และนามสกุลไฟล์"
QPREFIX = "q-cit2-"                              # id ข้อ = q-cit2-001, 002, ...

SUMMARY = """หัวข้อที่ 1
- ประเด็น 1 (1 บรรทัด = 1 ข้อเท็จจริง)
- ประเด็น 2

หัวข้อที่ 2
- ...
"""

Q = [
 ("โจทย์ ใช้ **ตัวหนา** ได้",
  "คำอธิบายเฉลย ใช้ **ตัวหนา** ได้",
  "normal",                                       # easy / normal / hard
  ["ตัวเลือก ก", "ตัวเลือก ข", "ตัวเลือก ค", "ตัวเลือก ง"],
  1),                                             # index ข้อถูก 0–3
]
```

### กติกา SUMMARY (สำคัญ — gen.py ใช้ตรวจความครบของ HTML)

- **บรรทัดที่ไม่ขึ้นต้นด้วย `- ` = ชื่อหัวข้อ** ทุกหัวข้อต้องมี section ใน HTML ที่ `title` ตรงกันทุกตัวอักษร
- แอปแสดง SUMMARY เป็น **ข้อความธรรมดา** → ห้าม `**`, `#`, `$`, ตาราง ใช้แค่ขึ้นบรรทัด + `- `
- ครอบคลุม **ทุกหัวข้อในบทของ PDF** ไม่ตัดหัวข้อทิ้ง; หัวข้อย่อยเล็กๆ รวมเป็น bullet ในหัวข้อแม่ได้
- ชื่อหัวข้อ: ไทย + ศัพท์อังกฤษในวงเล็บ เช่น `ภัยคุกคาม (Threat)`; ไม่ต้องใส่เลขข้อ
- ข้อเท็จจริงต้องมาจากเนื้อหาจริง ห้ามแต่งเพิ่ม ตัวเลข/ปี/ชื่อ ต้องตรงต้นฉบับ
- ความยาวรวม ~4,000–9,500 ตัวอักษรต่อบท

### กติกาข้อสอบ Q (build.py บังคับด้วย assert)

- 4 ตัวเลือก **ไม่ซ้ำกัน** ข้อถูก 1 ข้อ, difficulty ∈ easy/normal/hard
- **ตำแหน่งข้อถูกต้องกระจาย**: จำนวนข้อถูกในแต่ละตำแหน่ง (0–3) ต่างกันไม่เกิน 2 — วางวน 0,1,2,3 แล้วสลับบ้าง
- ถามจาก SUMMARY เท่านั้น (ผู้เรียนต้องตอบได้จากสรุป) ครบทุกหัวข้อ
- ตัวลวงต้องสมเหตุสมผล ความยาวใกล้เคียงข้อถูก; ข้อ "ไม่ใช่/ผิด" ให้ทำ **ตัวหนา**
- เฉลยอธิบายเหตุผลสั้นๆ ไม่ใช่แค่ทวนคำตอบ
- สูตรใช้ `$...$` และใน Python string ต้องเขียน `\\frac` (ถ้าไม่มีสูตรไม่ต้องสนใจ)

## 5. ขั้น D — สร้าง x/e JSON ด้วย `quiz_build/build.py`

เล่มใหม่: เพิ่ม entry ใน `BOOKS`

```python
{"x": "x/x7.json", "e": "e/e7.json", "id": "<book-id>",
 "title": "<ชื่อเล่ม>",
 "html": "<โฟลเดอร์ใน _source/html>",
 "chapters": ["abc1", "abc3"]},
```

- `category` ตอนนี้ฝังตายตัวใน build.py (`นักวิชาการคอมพิวเตอร์ศาลยุติธรรม`) — ถ้าเล่มใหม่ต่างหมวด ให้ย้ายไปเป็นคีย์ใน BOOKS
- รัน: `cd _source/quiz_build && PYTHONIOENCODING=utf-8 python build.py` → ต้องจบด้วย `OK`
  - พิมพ์จำนวนข้อ/ตำแหน่งข้อถูก/ความยาวสรุปต่อบท
  - ตรวจ id ไม่ชนกับ x/e JSON อื่นทั้ง repo
- `htmlFileName` ของบทใส่ให้อัตโนมัติ = `book/<ID>.html.enc` **เฉพาะเมื่อมี** `_source/html/<html>/<ID>.html` → รัน build.py อีกรอบหลังขั้น E
- **ห้ามแก้ `.enc` หรือ `manifest.json`** และห้ามเขียน x/e JSON ด้วยมือ ให้แก้ที่ ch/*.py แล้วรันใหม่
- ถ้าเป็นเล่มใหม่ เสนอ (ไม่ต้องทำเองถ้าไม่ได้สั่ง) ให้สร้าง `p/pN.json` เส้นทางตามตำแหน่ง

---

## 6. ขั้น E — หน้าเว็บ HTML โต้ตอบรายบท

### 6.1 ข้อกำหนดจากผู้ใช้ (ต้องครบทุกข้อ)

- ไฟล์ `.html` ไฟล์เดียวต่อบท CSS/JS inline, SVG + CSS animation + JS เท่านั้น (ไม่มี GIF/รูปภายนอก)
- **สื่อด้วยภาพเป็นหลัก** ข้อความแต่ละหัวข้อ 2–4 ประโยค (`P(...)`)
- **ทุกหัวข้อในสรุปต้องมี section + ภาพ/แอนิเมชันอย่างน้อย 1 ชิ้น**
- กระบวนการหลายขั้น → `stepper` (ปุ่ม ▶ เล่น / ⏸ หยุด / ⏭ ทีละขั้น / ↺ เริ่มใหม่ + คำบรรยายเปลี่ยนตามขั้น)
- สีเดียวกัน = สิ่งเดียวกันทั้งหน้า + legend (COLORS ของหน้า + `leg=` ของแต่ละภาพ)
- เริ่มเล่นเมื่อเลื่อนถึง (IntersectionObserver), รองรับ prefers-reduced-motion, mobile + dark/light
- ส่วนหัว mind map, nav ติดบน + แถบความคืบหน้า, การ์ดพลิก "ประเด็นสำคัญ", glossary ค้นหาได้, แบบทดสอบ 5–10 ข้อเฉลยทันที
- ภาษาไทย คงศัพท์เทคนิคอังกฤษ; ฟอนต์ Sarabun จาก Google Fonts + fallback ระบบ
  (ขัด GUIDE ข้อ "ห้ามฟอนต์ออนไลน์" — ผู้ใช้ขอเอง ออฟไลน์จะใช้ฟอนต์เครื่องแทน ให้แจ้งในรายงาน)
- ถ้าสรุปมีส่วน "💡 ไอเดียภาพประกอบ" ให้ทำตามนั้น ถ้าไม่มี ออกแบบเองและแจ้งผู้ใช้

ทั้งหมดนี้ **gen.py จัดให้แล้ว** — งานของ Claude คือเขียนไฟล์บทเท่านั้น

### 6.2 โครงไฟล์ `html_build/chapters/<mod>.py`

```python
from gen import *

QMOD = "cis8"                     # ชื่อไฟล์ใน quiz_build/ch (ดึง TITLE, SUMMARY, Q)
BOOK = "<ชื่อเล่ม>"
SHORT = "<ชื่อสั้นกลาง mind map ≤ 14 ตัวอักษร>"
LEAD = "<1 ประโยคบอกว่าบทนี้มีอะไร>"
COLORS = [(1, "ความหมายของสี 1"), (2, "..."), ...]   # สี 1–8 ความหมายคงที่ทั้งหน้า

sec1 = P("2–4 ประโยค ใช้ <b>ตัวหนา</b> ได้") + fig(svg(480, 200, "...", "ชื่อภาพ"), "คำบรรยาย", [(1, "legend")])
...
SECTIONS = [dict(id="intro", nav="ชื่อสั้นในแถบนำทาง", title="<หัวข้อ SUMMARY ตรงตัว>", html=sec1, c=1), ...]
KEYPOINTS = [("หน้าการ์ด", "หลังการ์ด ใช้ <b> <br> ได้"), ...]      # 8–12 ใบ
GLOSSARY = [("Term", "ความหมายไทยสั้นๆ"), ...]                     # 30–55 คำ
QUIZ = [5, 14, 20, 23, 28, 34, 40, 43, 48, 61]                     # index ใน Q, 10 ข้อ กระจายทุกหัวข้อ
```

- ถ้า 1 section ครอบหลายหัวข้อ ใส่ `covers="<หัวข้อที่ 2>"` เพิ่มได้ 1 หัวข้อ (แต่ควรทำ 1:1)
- ส่งออกไปที่ `OUT[q.ID[:3]]` ใน gen.py — **prefix ใหม่ต้องเพิ่ม mapping ใน `OUT`** เช่น `"abc": os.path.join(SRC, "html", "<folder>")`

### 6.3 Helper ใน gen.py (ใช้ให้มากที่สุด ไม่ต้องเขียนใหม่)

| helper | ใช้เมื่อ |
|---|---|
| `T(x, y, s, size, anchor, cls, weight)` | ข้อความ SVG, `\n` = หลายบรรทัด |
| `box(x, y, w, h, label, c, size, rx, attrs, sub, emoji)` | กล่องมีสี (label กลาง, sub บรรทัดสอง, emoji ซ้าย) |
| `svg(w, h, body, label)` | ห่อ SVG — **ใช้กว้าง 480** เสมอ (ย่ออัตโนมัติบนมือถือ) |
| `fig(svg, cap, leg)` | ภาพ/แอนิเมชันวนซ้ำ |
| `stepper(svg, caps, ms, leg)` | กระบวนการหลายขั้น (caps = คำบรรยายทีละขั้น) |
| `flow(steps, caps, c, cols, ...)` | ขั้นตอนแบบงูเลื้อยพร้อมจุดวิ่ง steps=`[(emoji, label, sub, c)]` — ป้าย ≤ ~16 ตัวอักษร |
| `layers(items, caps, side, title, leg)` | ชั้นซ้อนเน้นทีละชั้น เช่น OSI, ระดับการป้องกัน |
| `bars(items, unit, ...)` | กราฟแท่งแนวนอนยืดออก |
| `icons([(emoji, name, desc, c)])` | กริดการ์ดไอคอน (ไม่ใช่ SVG) |
| `vs(left, right, lt, rt, lc, rc)` | เปรียบเทียบ 2 ฝั่ง |
| `P(*sents)` | ย่อหน้า |

คลาสสี: `f1–8` fill เข้ม, `p0–8` fill อ่อน, `s0–8` เส้น, `t1–8` ข้อความสี, `mut` ข้อความจาง, `inv` ข้อความขาว, `line` เส้นเทา, `nofill`, `track`
แอนิเมชันวนซ้ำ: `pulse`, `spin`, `blink`, `dash` (เส้นประวิ่ง), `grow` (ยืดแนวนอน), `<animateMotion>` (จุดวิ่งตามเส้น)

### 6.4 data-attribute ของ stepper (ขั้นเริ่มที่ 1)

| attribute | ผล |
|---|---|
| `data-s="1 3"` | เด่นเฉพาะขั้นที่ระบุ ขั้นอื่นจาง |
| `data-only="2"` | แสดงเฉพาะขั้นที่ระบุ ขั้นอื่นซ่อน (ใช้สลับฉากทั้งฉาก) |
| `data-upto="3"` | จางจนถึงขั้น 3 แล้วเด่นค้าง (ขั้นตอนสะสม) |
| `data-from="2" data-to="4"` | ปรากฏเฉพาะช่วงขั้น (ใช้ตัวเดียวก็ได้) |
| `data-hl="3"` | ขอบเรืองในขั้นที่ระบุ |
| `data-pos="1:x,y;3:x,y"` | เลื่อน (translate) ไปตำแหน่งตามขั้น — วาดชิ้นนั้นที่จุด (0,0) |
| `data-txt="a|b|c"` | เปลี่ยนข้อความตามขั้น |

ลูกศร: ประกาศ marker เองต่อ SVG (ดู `AR` ใน `chapters/cis8.py`) id ต้องไม่ซ้ำในหน้า เช่น `ar5`, `ar13`

### 6.5 กับดักที่เคยเจอ (ห้ามพลาดซ้ำ)

1. `class="pulse|spin"` บน `<g transform="translate(...)">` → CSS transform ทับ ตำแหน่งหลุด **ให้ซ้อน `<g>` ด้านใน** แล้วใส่ class ที่ตัวใน
2. `<animateMotion begin="0.8s">` → จุดค้างที่มุม (0,0) ก่อนเริ่ม **ใช้ begin ติดลบ** `begin="-0.8s"`
3. `data-pos` วัตถุต้องวาดที่ origin แล้วให้ translate พาไป; ตำแหน่ง marker/จุดวิ่งให้ **คำนวณจากสูตรเดียวกับที่วางกล่อง** อย่าพิมพ์พิกัดเอง
4. ข้อความล้นกล่อง: ประมาณความกว้าง ≈ จำนวนตัวอักษร × ขนาดฟอนต์ × 0.55 — ป้ายยาวให้ย่อหรือขึ้นบรรทัด `\n`
5. ป้ายของเส้น (label) มักทับกล่องปลายทาง → เลื่อนด้วย dx/dy ออกนอกกล่อง
6. เนื้อหาเกินความสูง viewBox (เช่น 5 กล่องซ้อน) → ตรวจ y สุดท้าย + h ≤ ความสูง svg
7. `grow` คือ scaleX ใช้กับแท่งแนวนอนเท่านั้น แท่งแนวตั้งใช้แบบไม่มีแอนิเมชันหรือ `pop`
8. Emoji ใหม่ๆ (เช่น 🪱) เป็นกล่องสี่เหลี่ยมบน Windows → ใช้ emoji พื้นฐาน
9. f-string ที่มี `\` ในส่วน `{}` พังบน Python < 3.12 → แยกเป็นตัวแปรก่อน
10. ข้อความใน `data-txt` ที่มี `"` ให้เขียน `&quot;`
11. **แก้ไฟล์ที่มีภาษาไทยผ่าน Bash heredoc → `python -` จาก stdin อาจ match ไม่เจอ** ให้เขียนสคริปต์ลงไฟล์ใน scratchpad แล้วรัน `python -X utf8 script.py` หรือใช้ Edit tool
12. รัน Python ที่พิมพ์ภาษาไทยบน Windows ต้องมี `PYTHONIOENCODING=utf-8`

### 6.6 สร้าง + ตรวจอัตโนมัติ

```bash
cd /d/claude.ai/content-hub/_source/html_build
PYTHONIOENCODING=utf-8 python gen.py cis8          # หรือไม่ใส่ชื่อ = ทุกบท
```

ต้องเห็น `<path>  NN KB  N หัวข้อ` ต่อบท และ **ไม่มีบรรทัด `!! ... ไม่มีหัวข้อ`** (exit code 0)
gen.py ตรวจว่าทุกหัวข้อ SUMMARY มี section และ QUIZ มี 5–10 ข้อ

### 6.7 ตรวจในเบราว์เซอร์ (บังคับ ก่อนส่งงาน)

1. เปิด server (background): `cd _source/html && python -m http.server 8765` (อย่าใช้ `file://` — ภาพไม่ขยับ)
2. built-in browser: `resize_window preset=mobile` แล้ว navigate `http://localhost:8765/<folder>/<ID>.html`
3. ฉีด helper แล้วดูทีละภาพ (screenshot scale 0.5):

```js
document.documentElement.style.scrollBehavior='auto';
window.F=[...document.querySelectorAll('.viz')];
window.show=async(i,step)=>{const f=F[i];f.scrollIntoView({block:'center'});
  if(f._st){f._st.started=true;f._st.pause();f._st.reset();
    const n=step||JSON.parse(f.dataset.caps).length;for(let k=1;k<n;k++)f._st.next()}
  await new Promise(r=>setTimeout(r,1400));return i};
F.map((f,i)=>i+':'+(f.closest('section')?.id)+(f._st?'*':'')).join(' ')   // * = stepper
```

   - `await show(i)` = ขั้นสุดท้าย, `await show(i,k)` = ขั้น k — **stepper ที่ใช้ `data-only` ต้องดูทุกขั้น**
4. `read_console_messages onlyErrors` ต้องว่าง
5. ลองกดแบบทดสอบ 1 ข้อ ดูว่าคะแนนเปลี่ยน, นับ `.qz` `.flip` `.gl .it`
6. เจอทับซ้อน/ล้น → แก้ chapters/*.py → `gen.py` → reload → ดูซ้ำ
7. จบแล้ว: `resize_window preset=desktop` และหยุด http.server (TaskStop)

---

## 7. ขั้น F — ผูก HTML เข้ากับบท

```bash
cd /d/claude.ai/content-hub/_source/quiz_build && PYTHONIOENCODING=utf-8 python build.py
```

ตรวจผล:

```bash
cd /d/claude.ai/content-hub && python -X utf8 -c "
import json
for f in ['x/x5.json','x/x6.json']:
    d=json.load(open(f,encoding='utf-8'))
    for b in d['books']: print(b['category'], [(c['id'],c.get('htmlFileName')) for c in b['chapters']])"
```

⚠️ ถ้าแก้ HTML ของบทที่ **เคยเผยแพร่แล้ว** ต้องเปลี่ยนชื่อไฟล์ (เช่น `cit-ch2-v2.html`) และ path ใน JSON — แอปไม่โหลดไฟล์ชื่อเดิมซ้ำ (GUIDE ข้อ 8)
ถ้ายังไม่เคยเผยแพร่ ใช้ชื่อเดิมได้

---

## 8. ขั้น G — สิ่งที่ Claude ทำไม่ได้ (บอกผู้ใช้ในรายงาน)

`QuizContentEncryptor.exe` เป็นโปรแกรม GUI ที่ต้องใช้รหัสผ่าน ผู้ใช้ต้องทำเอง:

1. เปิดโปรแกรม → รหัสผ่าน → **+ เลือกไฟล์** เลือก HTML ทุกไฟล์ใน `_source/html/<folder>/` → ปลายทาง `D:\claude.ai\content-hub\book` → **เข้ารหัส (+ตรวจย้อนกลับ)** → ได้ `book/<ID>.html.enc`
2. **สร้างเนื้อหา + manifest (x/e/p)** เลือก `D:\claude.ai\content-hub` ใส่ changeLog
3. **commit + push**

อย่า commit/push เอง เว้นแต่ผู้ใช้สั่ง และห้ามรัน/แก้ `.enc` `manifest.json`

---

## 9. Checklist ก่อนรายงาน

- [ ] ทุกบทที่ขอมี `quiz_build/ch/<mod>.py` และ `html_build/chapters/<mod>.py`
- [ ] `build.py` จบด้วย `OK`, `gen.py` exit 0 ไม่มี `!!`
- [ ] x/e JSON: category ตรงกันทุกเล่ม, `htmlFileName` ครบเฉพาะบทที่มี HTML
- [ ] ทุก stepper ดูครบทุกขั้นในจอมือถือ ไม่มีข้อความทับ/ล้น, console ไม่มี error
- [ ] viewport กลับ desktop, http.server หยุดแล้ว
- [ ] `git status` ไม่มี `.json` (ยกเว้น manifest) `.html` `.pdf` โผล่ (`.gitignore` กันไว้)

## 10. รูปแบบรายงานตอนจบ (ภาษาไทย สั้น)

1. ผลลัพธ์: จำนวนบท/ข้อสอบ/หน้า HTML + ลิงก์ไฟล์
2. ผลการตรวจ (ผ่านอะไรบ้าง)
3. ขั้นที่ผู้ใช้ต้องทำเอง (ข้อ 8)
4. ข้อสังเกต: Google Fonts vs GUIDE, บทที่ไม่มี HTML, หัวข้อที่ออกแบบภาพเอง, ส่วนที่ต้นฉบับเสีย (ตาราง/ภาพ) ที่ควรเทียบ PDF
5. ข้อเสนอถัดไป (เช่น `p/pN.json`) — เสนอ ไม่ทำเอง
