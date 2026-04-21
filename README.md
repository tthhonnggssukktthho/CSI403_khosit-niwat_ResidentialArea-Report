# 🏢 โฆษิตนิวาส (Khosit Niwat)
### ระบบแจ้งเตือนอพาร์ตเมนต์อัตโนมัติผ่าน LINE

> แจ้งค่าน้ำ ค่าไฟ ประกาศ และเหตุฉุกเฉินแบบอัตโนมัติ — ส่งตรงถึงผู้เช่าแต่ละห้องผ่าน LINE OA

---

## ภาพรวมระบบ

**โฆษิตนิวาส** คือระบบอัตโนมัติที่ช่วยให้เจ้าของและผู้ดูแลอพาร์ตเมนต์ไม่ต้องเสียเวลาแจ้งค่าใช้จ่ายและข่าวสารทีละห้องอีกต่อไป ระบบดึงข้อมูลจาก Google Sheets แล้วส่งข้อความแจ้งเตือนผ่าน LINE OA ตรงถึงผู้เช่าแต่ละรายโดยอัตโนมัติ

## ปัญหาที่แก้ไข

| ปัญหาเดิม | ทางออกของระบบ |
|---|---|
| พิมพ์ทักผู้เช่าทีละห้อง 1–3 ชั่วโมง/เดือน | ส่งข้อความอัตโนมัติพร้อมกันทุกห้อง |
| ปริ้นกระดาษติดบอร์ด ผู้เช่าอาจไม่เห็น | ส่งตรงถึง LINE ส่วนตัว |
| คำนวณค่าน้ำ ค่าไฟด้วยตัวเอง | คำนวณอัตโนมัติจากข้อมูลใน Google Sheets |
| ประกาศฉุกเฉินไม่ทั่วถึง | แจ้งเตือนทันทีผ่าน Webhook |
| ผู้เช่าต้องโทรถามข้อมูลทั่วไป | AI ตอบคำถามได้ตลอด 24 ชั่วโมง |
| ไม่รู้ว่ามีพัสดุรอรับ | แจ้งจำนวนพัสดุผ่าน LINE ได้ทันที |

---

## ⚙️ System Architecture & Workflow

ระบบนี้ทำงานแบบ Automation 100% ผ่าน n8n โดยรองรับทั้งการแจ้งหนี้, ประกาศฉุกเฉิน และการรับแจ้งปัญหาจากผู้เช่า

---

### ส่วนที่ 1 — แจ้งยอดรายเดือน (Schedule)

**การทำงาน:** ใช้ Schedule Trigger ตั้งเวลาทำงานอัตโนมัติทุกสิ้นเดือน เพื่อดึงข้อมูลเลขมิเตอร์น้ำ-ไฟจาก Google Sheets มาคำนวณผ่าน Code Node (JavaScript)

**ผลลัพธ์:** ส่งข้อความแจ้งยอดรวม (ค่าห้อง + ค่าน้ำ + ค่าไฟ) ไปยัง LINE ของผู้เช่าแต่ละห้องโดยตรงแบบรายบุคคล

```mermaid
flowchart LR
    T1A(["⏰ Schedule Trigger\nสิ้นเดือน 08:00 / 21:00"])
    T1B(["⏰ Schedule Trigger\nทดสอบ ทุก 15 วิ"])
    N1["📊 Google Sheets\nSheet1 — ดึงผู้เช่าทุกห้อง"]
    N2["🧮 Code Node\nคำนวณค่าห้อง + น้ำ + ไฟ + ส่วนกลาง"]
    N3["📤 LINE Push Message\nส่ง Text Message ทุก LINE ID"]

    T1A --> N1
    T1B --> N1
    N1 --> N2
    N2 --> N3
```

---

### ส่วนที่ 2 — ตอบกลับอัตโนมัติ (Webhook)

**การทำงาน:** ทำงานผ่าน Webhook เพื่อรับข้อความจากผู้เช่า แล้วใช้ Switch Node แยกคำขอออกเป็น 3 เส้นทาง

**ผลลัพธ์:** ผู้เช่าสามารถเช็คข้อมูลส่วนตัวหรือสอบถามข้อมูลที่พักได้ตลอด 24 ชั่วโมง โดยระบบจะตอบกลับเป็น Flex Message ทันที

```mermaid
flowchart TD
    W2(["🔔 LINE Webhook\n/line-webhook"])
    P2["📝 Parse LINE Event\nดึง userId / replyToken / userMessage"]
    SW["🔀 Switch Node\n3 เส้นทาง"]
    RES(["✅ Respond to Webhook\n200 OK"])

    W2 --> P2 --> SW

    SW -->|"contains: ค่าใช้จ่าย"| A1
    SW -->|"contains: พัสดุ"| B1
    SW -->|"ข้อความอื่นๆ"| C1

    subgraph PATH1["เส้นทาง 1 — เช็คยอดค่าใช้จ่าย"]
        A1["📊 Google Sheets\nดึงข้อมูลผู้เช่า"]
        A2["🧮 คำนวณค่าใช้จ่าย\nจับคู่ LINE ID"]
        A3["📤 Reply Flex Message\nยอดค่าเช่า / น้ำ / ไฟ"]
        A1 --> A2 --> A3
    end

    subgraph PATH2["เส้นทาง 2 — เช็คพัสดุ"]
        B1["📊 Google Sheets\nดึง Parcel Count"]
        B2["📦 จำนวนพัสดุ\nวัน-เวลารับของ"]
        B3["📤 Reply Flex Message\nพัสดุรอรับ"]
        B1 --> B2 --> B3
    end

    subgraph PATH3["เส้นทาง 3 — AI Chatbot"]
        C1["📊 Google Sheets\nSheet2 ข้อมูลที่พัก"]
        C2["🤖 Groq API\nllama-3.3-70b-versatile"]
        C3["🎨 จัดรูปแบบ Flex Message"]
        C4["📤 Reply Flex Message\nAI ตอบ"]
        C1 --> C2 --> C3 --> C4
    end

    A3 --> RES
    B3 --> RES
    C4 --> RES
```

---

### ส่วนที่ 3 — Admin Broadcast (!ประกาศ-)

**การทำงาน:** ตรวจสอบสิทธิ์ (Whitelist) จาก LINE UID ของผู้ส่ง หากเป็นแอดมินและพิมพ์คำสั่งขึ้นต้นด้วย `!ประกาศ-` ระบบจะดึงรายชื่อ LINE ID ของผู้เช่าทั้งหมด

**ผลลัพธ์:** ทำการ Push Message แจ้งข่าวสารหรือเหตุฉุกเฉินให้ทราบพร้อมกันทุกห้องทันที

```mermaid
flowchart LR
    W3(["🔔 LINE Webhook\n/line-webhook"])
    P3["📝 Parse LINE Event\nดึง userId / message"]
    CHK["🔐 เช็ค Admin UID\nWhitelist 3 คน\nformat: !ประกาศ-"]
    SW3["🔀 Switch"]
    SKIP(["🚫 Respond OK\nไม่ทำอะไร"])
    SH3["📊 Google Sheets\nดึง LINE ID ทุกห้อง"]
    MSG["✍️ เตรียม Flex Message\nประกาศ Header"]
    PUSH["📤 LINE Push\nส่งทุก LINE ID"]

    W3 --> P3 --> CHK --> SW3
    SW3 -->|"ไม่ใช่ admin หรือ format ผิด"| SKIP
    SW3 -->|"admin + !ประกาศ-"| SH3
    SH3 --> MSG --> PUSH
```
