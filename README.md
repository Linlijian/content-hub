# รูปแบบไฟล์เนื้อหา (สำหรับ QuizContentEncryptor.exe)

repo เนื้อหา: https://github.com/Linlijian/content-hub (ไฟล์อยู่ที่ root)

```
content-hub/
  manifest.json      ← ไม่เข้ารหัส แอปอ่านไฟล์นี้ก่อนเสมอ (เครื่องมือเขียนให้อัตโนมัติ)
  x1.enc  x2.enc     ← บทเรียน/หนังสือ
  e1.enc  e2.enc     ← ข้อสอบ
  book/*.pdf.enc     ← ไฟล์ PDF เข้ารหัส
  image/*.png.enc    ← รูปในโจทย์/ตัวเลือก เข้ารหัส
```

ไฟล์ต้นฉบับ (`x1.json`, `e1.json`, …) **เก็บไว้ในเครื่อง ไม่ต้องอัปขึ้น repo** — อัปเฉพาะ `.enc`

---

## 1. ไฟล์บทเรียน เช่น `x1.json`

มีแค่คีย์ `books` ก็พอ

```json
{
  "books": [
    {
      "id": "civil-service-act-2551",
      "title": "พระราชบัญญัติระเบียบข้าราชการพลเรือน พ.ศ. 2551",
      "author": "สำนักงาน ก.พ.",
      "category": "law",
      "pdfFileName": "book/พระราชบัญญัติระเบียบข้าราชการพลเรือน พ.ศ. 2551.pdf.enc",
      "coverImageFileName": null,
      "chapters": [
        {
          "id": "csa-ch1",
          "orderIndex": 1,
          "title": "หมวด 1 คณะกรรมการข้าราชการพลเรือน",
          "summaryMarkdown": "สรุปเนื้อหา รองรับ **ตัวหนา** *ตัวเอียง* และสูตร $E = mc^2$"
        }
      ]
    }
  ]
}
```

| ฟิลด์ | จำเป็น | หมายเหตุ |
|---|---|---|
| `id` | ✅ | ห้ามซ้ำ และ**ห้ามเปลี่ยนทีหลัง** (ใช้ผูกความคืบหน้าของผู้ใช้) |
| `title` / `author` / `category` | ✅ | `category` ใช้จัดกลุ่มในหน้ารายการหนังสือ เช่น `law`, `tech` |
| `pdfFileName` | – | path relative จาก `manifest.json` ใส่ `null` ถ้าไม่มี PDF |
| `coverImageFileName` | – | ปก ถ้ามีแอปจะสร้าง thumbnail ให้เอง |
| `chapters[].orderIndex` | ✅ | ลำดับที่แสดง เริ่มที่ 1 |

## 2. ไฟล์ข้อสอบ เช่น `e1.json`

```json
{
  "questionSets": [
    { "id": "csa-qset-1", "name": "แบบทดสอบ พ.ร.บ. ข้าราชการพลเรือน", "category": "book", "refBookId": "civil-service-act-2551" }
  ],
  "questions": [
    {
      "id": "q-csa-001",
      "questionSetId": "csa-qset-1",
      "bookId": "civil-service-act-2551",
      "chapterId": "csa-ch1",
      "textMarkdown": "ข้อใดคือ...",
      "imagePath": "image/osi-model-diagram.png.enc",
      "explanationMarkdown": "เฉลยเพราะ **มาตรา 8**",
      "difficulty": "normal",
      "choices": [
        { "textMarkdown": "ตัวเลือกที่ถูก", "imagePath": null, "isCorrect": true },
        { "textMarkdown": "ตัวเลือกผิด",   "imagePath": null, "isCorrect": false }
      ]
    }
  ]
}
```

| ฟิลด์ | จำเป็น | หมายเหตุ |
|---|---|---|
| `questionSetId` | ✅ | ต้องตรงกับ `questionSets[].id` |
| `bookId` / `chapterId` | – | ใส่เพื่อให้ "ทบทวนตามบท" ทำงาน ใส่ `null` ได้ |
| `imagePath` | – | ของโจทย์หรือของตัวเลือกก็ได้ ใส่ทั้งรูปและข้อความพร้อมกันได้ |
| `isCorrect` | ✅ | ต้องมี **ข้อละ 1 ตัวที่เป็น true** |
| `difficulty` | – | `easy` / `normal` / `hard` (ค่าเริ่มต้น `normal`) |

**สูตรคณิตศาสตร์:** ใส่ `$...$` ได้ทั้งใน `textMarkdown` และตัวเลือก เช่น `$2^{(32-n)} - 2$`, `$\frac{64}{2}$`

## 3. เส้นทางตำแหน่งงาน (ไม่บังคับ) — คีย์ `positions`

```json
{
  "positions": [
    {
      "id": "pos-court-officer",
      "name": "เจ้าหน้าที่ศาลยุติธรรม",
      "description": "ลำดับที่ต้องอ่าน/สอบให้ผ่าน",
      "requirements": [
        { "requiredType": "chapter",     "refId": "csa-ch1",    "orderIndex": 1 },
        { "requiredType": "questionset", "refId": "csa-qset-1", "orderIndex": 2 }
      ]
    }
  ]
}
```

---

## ขั้นตอนเพิ่มเนื้อหาใหม่

1. แก้/เพิ่มไฟล์ `.json` ในเครื่อง (จะแยกกี่ไฟล์ก็ได้ — ไฟล์ละคีย์เดียวหรือหลายคีย์ก็ได้)
2. รูป/PDF ใหม่: ลากเข้าเครื่องมือ → ปุ่ม **"เข้ารหัส (+ตรวจย้อนกลับ)"** → ได้ `.enc` เอาไปวางใน `book/` หรือ `image/`
3. ปุ่ม **"สร้างเนื้อหา + manifest"** → เลือก `.json` **ทุกไฟล์พร้อมกัน** → ได้ `.enc` ครบ + `manifest.json` ที่ขึ้นเวอร์ชันให้แล้ว
4. ปุ่ม **"commit + push"** → เลือกโฟลเดอร์ `D:\claude.ai\content-hub`

> เครื่องมือเรียงให้ไฟล์ที่มี `books` มาก่อนไฟล์ข้อสอบเสมอ ไม่ต้องกังวลลำดับที่เลือก

## กฎที่ห้ามพลาด

- **`version` ใน manifest ต้องเปลี่ยนทุกครั้ง** ไม่งั้นแอปจะไม่ดาวน์โหลดใหม่ (เครื่องมือขึ้นเลขท้ายให้อัตโนมัติ)
- **รหัสผ่านต้องเป็นตัวเดียวกันทั้งหมด** — ทั้ง `.enc` ของเนื้อหา, PDF, รูป และ `ContentPassword` ใน `App/MauiProgram.cs`
- `id` ที่เคยปล่อยไปแล้วห้ามเปลี่ยน — เปลี่ยนแล้วผู้ใช้จะได้เนื้อหาซ้ำและประวัติการทำข้อสอบไม่ผูกกับของเดิม
- นำเข้าเป็นแบบ upsert (ทับตาม id) — ลบข้อสอบออกจากไฟล์ไม่ได้ทำให้ของเก่าในเครื่องผู้ใช้หายไป
