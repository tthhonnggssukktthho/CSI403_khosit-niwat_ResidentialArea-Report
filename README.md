# 🏢 โฆษิตนิวาส (Khosit Niwat)
### ระบบแจ้งเตือนอพาร์ตเมนต์อัตโนมัติผ่าน LINE

> แจ้งค่าน้ำ ค่าไฟ ประกาศ และเหตุฉุกเฉินแบบอัตโนมัติ — ส่งตรงถึงผู้เช่าแต่ละห้องผ่าน LINE OA

---

## 📋 สารบัญ

- [ภาพรวมระบบ](#ภาพรวมระบบ)
- [สถาปัตยกรรมระบบ](#สถาปัตยกรรมระบบ)
- [n8n Workflow](#n8n-workflow)
- [โครงสร้าง Repository](#โครงสร้าง-repository)
- [การติดตั้ง](#การติดตั้ง)
- [การกำหนดค่า](#การกำหนดค่า)
- [วิธีใช้งาน](#วิธีใช้งาน)

---

## ภาพรวมระบบ

**โฆษิตนิวาส** คือระบบอัตโนมัติที่ช่วยให้เจ้าของและผู้ดูแลอพาร์ตเมนต์ไม่ต้องเสียเวลาแจ้งค่าใช้จ่ายและข่าวสารทีละห้องอีกต่อไป ระบบดึงข้อมูลจาก Google Sheets แล้วส่งข้อความแจ้งเตือนผ่าน LINE OA ตรงถึงผู้เช่าแต่ละรายโดยอัตโนมัติ

### ปัญหาที่แก้ไข

| ปัญหาเดิม | ทางออกของระบบ |
|---|---|
| พิมพ์ทักผู้เช่าทีละห้อง 1–3 ชั่วโมง/เดือน | ส่งข้อความอัตโนมัติพร้อมกันทุกห้อง |
| ปริ้นกระดาษติดบอร์ด ผู้เช่าอาจไม่เห็น | ส่งตรงถึง LINE ส่วนตัว |
| คำนวณค่าน้ำ ค่าไฟด้วยตัวเอง | คำนวณอัตโนมัติจากข้อมูลใน Google Sheets |
| ประกาศฉุกเฉินไม่ทั่วถึง | แจ้งเตือนทันทีผ่าน Webhook |

---

## ⚙️ System Architecture & Workflow

ระบบนี้ทำงานแบบ Automation 100% ผ่าน n8n โดยรองรับทั้งการแจ้งหนี้, ประกาศฉุกเฉิน และการรับแจ้งปัญหาจากผู้เช่า:

```mermaid
flowchart TD
    A([Start]) --> B{Trigger}
    
    B -->|Schedule| C[Fetch Billing Data]
    B -->|Webhook| D[Create Emergency Msg]
    B -->|Chat| E[Receive Tenant Issue]
    
    %% Billing Flow
    C --> F[Google Sheets API\nRoom + LINE User ID]
    F --> G[Calculate Costs]
    G --> H[Generate Billing Msg]
    
    %% Emergency Flow
    D --> H2[Generate Emergency Message]
    
    %% Chat Flow
    E --> I[Forward to Admin\nLINE]
    
    %% Merge to loop
    H --> J[Loop Each Room]
    H2 --> J
    
    %% Send LINE
    J --> K[LINE Messaging API\nSend via LINE OA]
    
    %% Result Check
    K --> L{Success}
    
    L -->|Yes| M[Log Result]
    L -->|No| N[Retry or Error]
    
    %% Notify Admin
    M --> O[Notify Admin]
    N --> O
    I -.-> O
    
    O --> P([End])
