# คู่มือสำหรับ AI: แปลงไฟล์ข้อสอบดิบ → เนื้อหาใน content-hub

อ่าน [GUIDE.md](GUIDE.md) ก่อนเสมอ — ไฟล์นี้ไม่ทวนกฎ schema ซ้ำ แค่บอก**ลำดับงาน**ที่ใช้จริง

---

## 0. ข้อห้าม (ผิดแล้วพัง แก้ยาก)

- **ห้ามแก้ `id` ที่เผยแพร่ไปแล้ว** — แอปอ้างอิงด้วย id ถ้าเปลี่ยนคือข้อมูลผู้ใช้หาย
- **ห้ามแก้ `.enc` หรือ sha256 ใน `manifest.json` ด้วยมือ** — ต้องให้ตัว encryptor สร้างเท่านั้น
- **ห้าม commit ไฟล์ `.json`** — `.gitignore` กันไว้แล้ว (`*.json`) push แค่ `.enc` + `manifest.json`
- **ห้ามรัน `QuizContentEncryptor.exe` เอง** — AI รันไม่ได้ ต้องให้ผู้ใช้กดเอง (ดูขั้นตอน 6)

---

## 1. อ่านไฟล์ต้นฉบับ แล้วแมปเข้าบทเรียน

ไฟล์ดิบอยู่ที่ `_source/book/quiz/<ชื่อชุด>/`

หาบทเรียนที่ตรงจาก `x/*.json` (books → chapters):

```bash
python -c "import json,glob; [print(b['id'], '|', c['id'], c['title']) for f in glob.glob('x/*.json') for b in json.load(open(f,encoding='utf-8'))['books'] for c in b['chapters']]"
```

| กรณี | `category` | `refBookId` | `bookId`/`chapterId` ในคำถาม |
|---|---|---|---|
| ตรงกับบทเรียน | `"book"` | id ของหนังสือ | ใส่ให้ตรงบท |
| ไม่มีบทเรียนตรง | `"custom"` | `null` | `null` ทั้งคู่ |

ค่าอื่นนอกจาก `book`/`custom` → ชุดข้อสอบจะ**ไม่ขึ้นในแอป**

---

## 2. เขียนไฟล์ชุดข้อสอบลง scratchpad ก่อน (ห้ามเขียนลง repo ตรงๆ)

ที่ scratchpad: `<scratchpad>/sets/<ชื่อ>.json` ไฟล์ละ 1 ชุดสอบใหญ่ (เช่น 1 เดือน)

**ใช้ Write tool เท่านั้น** — heredoc ใน bash พังกับข้อความไทยยาว (`unexpected EOF`)

รูปแบบ:

```json
{
  "questionSets": [
    { "id": "y69m1-set-01", "name": "...ชุดที่ 1", "category": "custom", "refBookId": null }
  ],
  "questions": [
    {
      "id": "q-y69m1-001", "questionSetId": "y69m1-set-01",
      "bookId": null, "chapterId": null,
      "textMarkdown": "...", "imagePath": null,
      "explanationMarkdown": "...", "difficulty": "easy",
      "choices": [
        { "textMarkdown": "...", "imagePath": null, "isCorrect": true },
        { "textMarkdown": "...", "imagePath": null, "isCorrect": false },
        { "textMarkdown": "...", "imagePath": null, "isCorrect": false },
        { "textMarkdown": "...", "imagePath": null, "isCorrect": false }
      ]
    }
  ]
}
```

กติกาการตั้งชื่อ id ที่ใช้อยู่:

| แบบ | ตัวอย่าง |
|---|---|
| ชุดประจำเดือน (ชุดเดิม) | `mo1-set-01` / `q-mo1-001` |
| ชุดประจำเดือน ปี 2569 | `y69m1-set-01` / `q-y69m1-001` |
| ข้อสอบท้ายบท | `rv-cit-ch4-s3` |

**ตัดชุดละ 10 ข้อ** (ข้อสอบ 100 ข้อ = 10 ชุด) — ถ้าจำนวนต่างจากนี้ให้ถามผู้ใช้ก่อน

### เรื่องเนื้อหา

- คณิตใช้ `$...$` และใน JSON ต้อง escape backslash เป็นสองตัว: `\\frac`, `\\times`
- `explanationMarkdown` ห้ามว่าง — เขียนเหตุผลสั้นๆ **ตัวหนาที่คำตอบ**
- แอปสุ่มลำดับ**ข้อ** แต่ไม่สุ่ม**ตัวเลือก** → อย่าวางคำตอบถูกไว้ข้อ ก. ทุกข้อ กระจายให้ทั่ว
- ข้อที่อ้างรูปในต้นฉบับแต่ไม่มีไฟล์รูป → **เขียนใหม่ให้อ่านเข้าใจได้เองโดยไม่ต้องดูรูป** คงคำตอบเดิมไว้

### เฉลยต้นฉบับผิดบ่อย — ต้องตรวจเอง

ตรวจคำนวณทุกข้อด้วยตัวเอง (แปลงฐานเลข, 2's complement, subnet, ASCII) อย่าเชื่อเฉลย

เจอผิด → **ใช้คำตอบที่ถูก** แล้ว**จดไว้รายงานผู้ใช้ตอนจบ** (ข้อที่เท่าไหร่ เฉลยว่าอะไร ที่ถูกคืออะไร เพราะอะไร)

---

## 3. สคริปต์ merge + validate

ก๊อป `merge3.py` จาก scratchpad มาแก้ชื่อไฟล์/ปลายทาง แล้วรัน:

```bash
export PYTHONIOENCODING=utf-8; python "<scratchpad>/merge3.py"
```

(ไม่ตั้ง `PYTHONIOENCODING` แล้วภาษาไทยใน stdout จะเป็นตัวประหลาด)

สคริปต์ทำ 3 อย่าง:
1. รวมไฟล์ใน `sets/` → `e/eNN.json` (**ใช้เลขใหม่เสมอ** อย่าแก้ไฟล์เดิม ผู้ใช้จะได้โหลดเฉพาะของใหม่)
2. ต่อ requirement เข้า `p/pNN.json`
3. validate ทั้ง repo

### validate ต้องเช็คอะไรบ้าง

- id ซ้ำ (ทั้ง set และ question) ข้าม**ทุกไฟล์**ใน `e/`
- `category` เป็น `book`/`custom` เท่านั้น
- `category: book` → `refBookId` ต้องมีจริงใน `x/`
- `category: custom` → `refBookId` ต้องเป็น `null`
- `chapterId` / `bookId` ที่ไม่ใช่ null ต้องมีจริง
- แต่ละข้อมี `isCorrect: true` **พอดี 1 อัน**, มี 4 ตัวเลือก, `explanationMarkdown` ไม่ว่าง
- ทุกตัวเลือกมี `textMarkdown` หรือ `imagePath` อย่างน้อยหนึ่ง
- ข้อความตัวเลือกห้ามซ้ำกันเองในข้อเดียวกัน
- `p/*.json` → ทุก `refId` มีจริง และ `orderIndex` เรียง 1..N ไม่ขาด

> **false positive ที่รู้แล้ว:** `q-graph-choice-image-001` ใน `e/e1.json` ใช้ตัวเลือกเป็นรูปล้วน (`textMarkdown: null` ทั้ง 4) — เช็ค duplicate จะเตือน ให้ข้าม (กรองเฉพาะตัวที่มี textMarkdown)

ต้องได้ `ERRORS: none` ถึงจะไปต่อ

---

## 4. เพิ่มเข้า learning path (`p/`)

ถ้าชุดข้อสอบต้องอยู่ในเส้นทางของตำแหน่ง:

- ต่อท้าย `requirements` ด้วย `{"requiredType": "questionset", "refId": "<set id>", "orderIndex": <ต่อจากเดิม>}`
- `orderIndex` ต้องต่อเนื่องจากตัวสุดท้ายเดิม ห้ามข้าม ห้ามซ้ำ
- ลำดับปกติ: บทเรียน (`requiredType: "chapter"`) ทั้งหมดก่อน แล้วค่อยชุดข้อสอบ
- เกณฑ์ผ่านแต่ละขั้นคือ 60%
- อัปเดต `description` ให้ตรงกับของที่เพิ่มเข้าไปด้วย

ตัวอย่างสถานะปัจจุบัน: `p/p5.json` = `pos-computer-technical-officer` 128 requirements (18 บทเรียน + 70 ชุดเดิม + 40 ชุดปี 2569)

---

## 5. ตรวจว่าไฟล์ `.json` ยังถูก ignore อยู่

```bash
git check-ignore -v e/e17.json p/p5.json
```

ต้องมี output ชี้ไปที่ `.gitignore:2:*.json` ถ้าไม่มี = หลุด อย่าเพิ่ง commit

---

## 6. ส่งต่อให้ผู้ใช้ (AI ทำเองไม่ได้)

บอกผู้ใช้ให้:

1. รัน `KnowledgeQuizApp\tools\QuizContentEncryptor.exe`
2. เลือก **"สร้างเนื้อหา + manifest"** ชี้ไปที่ `D:\claude.ai\content-hub`
3. commit + push (จะได้เฉพาะ `.enc` + `manifest.json`)

> `.enc` เก่าที่มีอยู่จะยังเป็นเวอร์ชันก่อนหน้าจนกว่าจะรันขั้นตอนนี้ — ไฟล์ที่แก้แล้วเช่น `p/p5.json` ต้องเข้ารหัสใหม่ด้วย ไม่ใช่แค่ไฟล์ที่สร้างใหม่

---

## 7. รายงานปิดงาน

บอกให้ครบ:
- ไฟล์ใหม่/ไฟล์ที่แก้ พร้อมจำนวนชุดและจำนวนข้อ แยกตามชุดสอบ
- ผลการ validate
- **รายการเฉลยต้นฉบับที่แก้** ทุกข้อ พร้อมเหตุผล
- ขั้นตอนที่ 6 ที่ผู้ใช้ต้องทำต่อ
