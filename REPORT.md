# รายงานสรุปผลการปฏิบัติการ: Data Integration Pipeline
## TechTrove E-Commerce: จากข้อมูลดิบหลายระบบสู่ข้อมูลพร้อมวิเคราะห์
**รายวิชา:** Data Warehousing / Data Engineering 
**รหัสนิสิต:** 67160230 
**บทบาท:** Data Engineer 

---

## 1. บทสรุปผู้บริหาร (Executive Summary)
TechTrove E-Commerce ประสบปัญหาข้อมูลกระจัดกระจายอยู่ในหลายระบบ (Transaction, CRM, Procurement และ Payment Gateway) ซึ่งมีทั้งปัญหา **Schema Drift**, **ข้อมูลซ้ำซ้อน (Duplicates)**, **ค่าผิดปกติ (Anomalies)**, **รูปแบบข้อความไม่เป็นมาตรฐาน (Data Inconsistency)** และ **รหัสที่ไม่มีใน Master Data (Referential Integrity Violations)**

ในปฏิบัติการนี้ ได้พัฒนา **Data Integration Pipeline** ด้วย Python และ Pandas เพื่อสกัด, ตรวจสอบคุณภาพ, ปรับมาตรฐาน, เชื่อมโยง และคำนวณยอดขายสุทธิ ($\text{Net Sales}$) เข้าสู่โมเดล **Star Schema** ซึ่งประกอบด้วย `dim_customer`, `dim_product` และ `fact_sales` พร้อมทั้งจัดทำ `data_quality_report` ที่สามารถตรวจสอบย้อนกลับได้ (Audit Trail) ครบถ้วน 100%

### ตัวเลขผลลัพธ์หลัก (Key Performance Highlights):
- **จำนวนคำสั่งซื้อดิบทั้งหมด:** 752 แถว (ม.ค. 361 แถว + ก.พ. 391 แถว)
- **จำนวนธุรกรรมยอดขายที่ถูกต้องและชำระเงินสำเร็จ (PAID Fact Sales):** **660 รายการ**
- **ยอดขายสุทธิรวมทั้งสิ้น (Total Net Sales):** **10,224,044.09 บาท**
- **จำนวนสินค้าที่จำหน่ายได้รวม:** 1,337 ชิ้น
- **จังหวัดที่สร้างยอดขายสูงสุด:** **กรุงเทพมหานคร** (2,612,955.88 ฿ คิดเป็น 25.56%)
- **หมวดหมู่สินค้าที่สร้างยอดขายสูงสุด:** **Smartphone** (3,092,117.34 ฿ คิดเป็น 30.24%)

---

## 2. แหล่งข้อมูลดิบและปัญหาคุณภาพข้อมูลที่พบ (Raw Data Profiling)

| แหล่งข้อมูล | รูปแบบไฟล์ | จำนวนเรคอร์ด | ประเด็นคุณภาพข้อมูลที่พบ (Data Quality Issues) |
| :--- | :---: | :---: | :--- |
| `orders_2026_01.csv` | CSV | 361 | คำสั่งซื้อ ม.ค. มี Quantity ติดลบ 1 แถว, Unit Price เป็น Null 1 แถว, Order ID ซ้ำ 1 แถว |
| `orders_2026_02.csv` | CSV | 391 | คำสั่งซื้อ ก.พ. พบ **Schema Drift** (ชื่อคอลัมน์ต่างกัน, ส่วนลดเป็น String `5%`, วันที่ DD/MM/YYYY), Quantity ติดลบ 1 แถว, Unit Price เป็น Null 1 แถว, Order ID ซ้ำ 1 แถว |
| `customers_crm.csv` | CSV | 163 | ข้อมูลลูกค้าซ้ำ 3 แถว (`C0012`, `C0045`, `C0088`), อีเมลมีตัวพิมพ์ใหญ่/ช่องว่าง 9 แถว และ Missing Email 5 แถว, ชื่อจังหวัดไม่เป็นมาตรฐาน 14 รูปแบบ |
| `product_master.xlsx` | Excel | 40 | ข้อมูล Master สินค้า 40 รายการ ถูกต้องสมบูรณ์ |
| `payments.json` | JSON | 752 | เป็น Nested JSON มี Payment ID ซ้ำ 1 แถว (`PAY000101`), สถานะ FAILED 47 รายการ, REFUNDED 18 รายการ, และ Orphan Payment 1 รายการ (`ORD999999`) |

---

## 3. ขั้นตอนการทำงานของ Data Integration Pipeline

```
[ orders_2026_01.csv ] [ orders_2026_02.csv ]
 │ │ (Schema Alignment: rename, parse %, dayfirst dates)
 └───────────┬───────────────┘
 ▼
 [ pd.concat (752 rows) ]
 │
 ▼
 [ Deduplication (750 rows) ] ── (Dropped 2 duplicate order_ids)
 │
 ▼
 [ Business Rules (746 rows) ] ── (Dropped quantity <= 0 and null unit_price)
 │
 ├──────────────────────────── [ dim_customer (160 rows) ]
 ▼ (Clean email, map 6 standard provinces)
 [ Customer Match (724 rows) ] ── (Dropped 22 rows with missing customer_ids)
 │
 ├──────────────────────────── [ dim_product (40 rows) ]
 ▼
 [ Product Match (722 rows) ] ── (Dropped 2 rows with invalid product_id 'P999')
 │
 ├──────────────────────────── [ payments_clean (751 rows) ]
 ▼ (Flatten nested JSON, deduplicate)
 [ Payment Filter (660 rows) ] ── (Filtered out 44 FAILED and 18 REFUNDED)
 │
 ▼
 [ fact_sales.csv ] ── (Calculated Net Sales = qty * price * (1 - discount))
 │
 ┌─────────────┴─────────────┐
 ▼ ▼
[ summary_by_province.csv ] [ summary_by_category.csv ]
```

---

## 4. โครงสร้างตารางผลลัพธ์ (Star Schema & Deliverables)

### 4.1 ตารางมิติลูกค้า: `output/dim_customer.csv` (160 แถว)
- `customer_id` (PK): รหัสลูกค้า Unique 100%
- `full_name`: ชื่อ-นามสกุลลูกค้า
- `email`: อีเมลที่ปรับเป็น Lowercase และ Trim ช่องว่าง
- `province`: ชื่อจังหวัดทางการ 6 จังหวัด (`กรุงเทพมหานคร`, `ขอนแก่น`, `ชลบุรี`, `เชียงใหม่`, `ภูเก็ต`, `ระยอง`)
- `signup_date`: วันที่สมัครสมาชิก

### 4.2 ตารางมิติสินค้า: `output/dim_product.csv` (40 แถว)
- `product_id` (PK): รหัสสินค้า Unique 100%
- `product_name`: ชื่อสินค้า
- `category`: หมวดหมู่สินค้า 4 หมวด (`Accessory`, `Notebook`, `Smart Home`, `Smartphone`)
- `standard_price`: ราคามาตรฐาน
- `active_flag`: สถานะการจำหน่าย (`Y`)

### 4.3 ตารางข้อเท็จจริงยอดขาย: `output/fact_sales.csv` (660 แถว)
- `order_id` (PK): รหัสคำสั่งซื้อ
- `order_date`: วันที่และเวลาสั่งซื้อ (Standard Datetime)
- `customer_id` (FK): เชื่อมโยงไปยัง `dim_customer`
- `product_id` (FK): เชื่อมโยงไปยัง `dim_product`
- `quantity`: จำนวนสินค้าที่สั่งซื้อ ($> 0$)
- `unit_price`: ราคาต่อหน่วย ($> 0$)
- `discount`: อัตราส่วนลด ($0.0 - 1.0$)
- `channel`: ช่องทางการขาย (`Marketplace`, `Mobile App`, `Web`)
- `payment_id`: รหัสการชำระเงิน
- `payment_method`: วิธีชำระเงิน (`Bank Transfer`, `Credit Card`, `E-Wallet`, `QR Code`)
- `payment_status`: สถานะชำระเงิน (`PAID`)
- `paid_at`: วันที่และเวลาชำระเงิน
- `net_sales`: ยอดขายสุทธิที่คำนวณจาก $\text{quantity} \times \text{unit\_price} \times (1 - \text{discount})$

---

## 5. คำตอบคำถามวิเคราะห์เชิงธุรกิจ 6 ข้อ (Business Analysis Answers)

### ข้อ 1: หลังรวมไฟล์ orders มีจำนวนแถวเท่าใด และเหลือกี่แถวหลังลบ duplicate?
- **รวมไฟล์ Orders ทั้ง 2 เดือน:** มีจำนวนแถวรวมทั้งสิ้น **752 แถว** (มกราคม 361 แถว + กุมภาพันธ์ 391 แถว)
- **จำนวนแถวหลังลบ Duplicate:** เหลือข้อมูล **750 แถว**
- **รายละเอียด:** ตรวจพบ `order_id` ซ้ำกัน 2 รายการ ได้แก่ `ORD000056` (ในไฟล์เดือน ม.ค.) และ `ORD000416` (ในไฟล์เดือน ก.พ.) โดย Pipeline ทำการเก็บแถวล่าสุดตามลำดับที่ปรากฏ (`keep='last'`) ตามกติกาทางธุรกิจ

---

### ข้อ 2: มีแถวที่ customer_id หรือ product_id ไม่พบใน Master Data อย่างละกี่แถว?
- **Customer ID ไม่พบใน Master Data:** มีจำนวน **22 แถวคำสั่งซื้อ**
 - เกิดจากรหัสลูกค้า 5 รายที่ไม่มีใน CRM ได้แก่ `C0161`, `C0162`, `C0163`, `C0164`, `C0165`
- **Product ID ไม่พบใน Master Data:** มีจำนวน **2 แถวคำสั่งซื้อ**
 - เกิดจากรหัสสินค้าที่ไม่มีใน Procurement Master คือ `P999`

---

### ข้อ 3: มียอดขายที่ใช้ได้จริงกี่ธุรกรรม และยอดขายสุทธิรวมเท่าใด?
- **จำนวนธุรกรรมยอดขายที่ใช้ได้จริง (PAID Transactions):** **660 ธุรกรรม**
 - จาก 722 คำสั่งซื้อที่เชื่อมโยง Master ครบถ้วน พบสถานะ `PAID` 660 รายการ, `FAILED` 44 รายการ และ `REFUNDED` 18 รายการ
- **ยอดขายสุทธิรวมทั้งสิ้น (Total Net Sales):** **10,224,044.09 บาท**

---

### ข้อ 4: จังหวัดใดมียอดขายสุทธิสูงสุด?
- **อันดับ 1: กรุงเทพมหานคร** มียอดขายสุทธิ **2,612,955.88 บาท** (25.56% ของยอดขายรวม, จำนวน 154 ออเดอร์, รวม 323 ชิ้น)

**ตารางสรุปยอดขายแยกตาม 6 จังหวัด (`summary_by_province.csv`):**

| อันดับ | จังหวัด (`province`) | จำนวนออเดอร์ (`total_orders`) | จำนวนชิ้น (`total_quantity`) | ยอดขายสุทธิ (`total_net_sales`) | สัดส่วนยอดขาย |
| :---: | :--- | :---: | :---: | :---: | :---: |
| 1 | **กรุงเทพมหานคร** | 154 | 323 | **2,612,955.88 ฿** | 25.56% |
| 2 | **ขอนแก่น** | 110 | 225 | **2,031,943.40 ฿** | 19.87% |
| 3 | **ระยอง** | 120 | 248 | **1,523,168.61 ฿** | 14.90% |
| 4 | **เชียงใหม่** | 104 | 206 | **1,477,338.01 ฿** | 14.45% |
| 5 | **ภูเก็ต** | 86 | 164 | **1,427,388.73 ฿** | 13.96% |
| 6 | **ชลบุรี** | 86 | 171 | **1,151,249.46 ฿** | 11.26% |
| **รวม** | **ทั้งหมด 6 จังหวัด** | **660** | **1,337** | **10,224,044.09 ฿** | **100.00%** |

---

### ข้อ 5: หมวดสินค้าใดมียอดขายสุทธิสูงสุด?
- **อันดับ 1: Smartphone** มียอดขายสุทธิ **3,092,117.34 บาท** (30.24% ของยอดขายรวม, จำนวน 178 ออเดอร์, รวม 384 ชิ้น)

**ตารางสรุปยอดขายแยกตาม 4 หมวดสินค้า (`summary_by_category.csv`):**

| อันดับ | หมวดสินค้า (`category`) | จำนวนออเดอร์ (`total_orders`) | จำนวนชิ้น (`total_quantity`) | ยอดขายสุทธิ (`total_net_sales`) | สัดส่วนยอดขาย |
| :---: | :--- | :---: | :---: | :---: | :---: |
| 1 | **Smartphone** | 178 | 384 | **3,092,117.34 ฿** | 30.24% |
| 2 | **Accessory** | 180 | 338 | **2,710,582.77 ฿** | 26.51% |
| 3 | **Notebook** | 161 | 324 | **2,221,495.49 ฿** | 21.73% |
| 4 | **Smart Home** | 141 | 291 | **2,199,848.49 ฿** | 21.52% |
| **รวม** | **ทั้งหมด 4 หมวด** | **660** | **1,337** | **10,224,044.09 ฿** | **100.00%** |

---

### ข้อ 6: หากสลับลำดับ merge ก่อน cleaning ผลลัพธ์หรือความเชื่อมั่นของข้อมูลเปลี่ยนอย่างไร?
การ Clean ข้อมูลก่อน Merge ถือเป็นหัวใจสำคัญของกระบวนการ Data Integration หากสลับลำดับโดย Merge ก่อน Cleaning จะส่งผลกระทบอย่างร้ายแรง 4 ประการ:
1. **เกิดปัญหา Cartesian Explosion (Fan-out Duplication):** 
 หากใน Master Data มีคีย์ซ้ำ (เช่น `C0012`, `C0045`, `C0088` ใน CRM) การ Merge จะจับคู่แถวแบบ One-to-Many ทำให้เรคอร์ดคำสั่งซื้อขยายตัวทวีคูณ ส่งผลให้ตัวเลขยอดขายบวมเกินจริง (Double Counting)
2. **ข้อมูลหลุดการเชื่อมโยง (Unmatched Key Fragmentation):** 
 หากไม่แปลงชื่อจังหวัดให้เป็นมาตรฐานก่อน (`'Chonburi'` vs `'ชลบุรี'` หรือ `'Bangkok'` vs `'กรุงเทพมหานคร'`) เมื่อนำไป Group By หรือวิเคราะห์ Dimension ยอดขายจะแตกกระจายเป็นกลุ่มย่อยที่ไม่ถูกต้อง
3. **การคำนวณยอดขายผิดพลาด (Revenue Miscalculation):** 
 หากไม่กรองข้อมูลที่มี `quantity <= 0` (เช่น -1), `unit_price` เป็น Null หรือรายการชำระเงินที่ `FAILED` / `REFUNDED` ออกก่อน ระบบจะนำตัวเลขเหล่านี้ไปรวมในยอดขายสุทธิ ทำให้รายงานการเงินของบริษัทผิดพลาด
4. **สูญเสีย Audit Trail & Traceability:** 
 การทำ Data Quality Tracking ทีละขั้นตอนทำให้ทราบชัดเจนว่าแถวใดถูกคัดออกด้วยเหตุผลทางธุรกิจใด หาก Merge รวดเดียว จะไม่สามารถชี้ชัดสาเหตุของความผิดพลาดในแต่ละมิติได้

---

## 6. Challenge Bonus (+2 คะแนน): Automated Validation & Funnel Chart

### 6.1 ฟังก์ชัน Automated Assertions (`validate_data`)
Pipeline มีฟังก์ชัน `validate_data(fact_sales, dim_customer, dim_product)` ซึ่งผ่านการตรวจสอบ 100%:
- **Uniqueness Check:** `order_id`, `customer_id`, `product_id` เป็น Primary Key ที่ไม่ซ้ำซ้อน 100%
- **Referential Integrity Check:** `fact_sales.customer_id` และ `fact_sales.product_id` ทั้งหมดมีอยู่ใน Dimension Tables
- **Business Bounds Check:** `quantity > 0`, `unit_price > 0`, `0 <= discount <= 1`, `payment_status == 'PAID'` และ `net_sales > 0`

### 6.2 Data Quality Funnel (อัตราการรอดของข้อมูลในแต่ละด่าน)

| ด่านที่ (Pipeline Stage) | จำนวนเรคอร์ด (Rows) | % เมื่อเทียบกับ Raw Data | แถวที่ตัดออก / เหตุผล |
| :--- | :---: | :---: | :--- |
| **1. Raw Orders (Combined)** | 752 | 100.0% | รวมคำสั่งซื้อเดือน ม.ค. (361) และ ก.พ. (391) |
| **2. Deduplicated Orders** | 750 | 99.7% | ตัด Duplicate Order ID ออก 2 แถว (`ORD000056`, `ORD000416`) |
| **3. Valid Business Rules** | 746 | 99.2% | ตัด Quantity $\le 0$ ออก 2 แถว, ตัด Null Unit Price ออก 2 แถว |
| **4. Matched Master Data** | 722 | 96.0% | ตัด Orphan Customer ออก 22 แถว, ตัด Orphan Product ออก 2 แถว |
| **5. Valid Paid Sales (Fact)** | **660** | **87.8%** | ตัด Non-PAID Payments ออก 62 แถว (44 FAILED, 18 REFUNDED) |

---

## 7. สรุปเปรียบเทียบคุณภาพข้อมูล ก่อนและหลังทำ Data Integration

| มิติการประเมิน (DQ Dimension) | สภาพข้อมูลก่อนทำ Integration (Raw Data) | ผลลัพธ์หลังทำ Integration (Clean & Fact) | การปรับปรุงที่เกิดขึ้น |
| :--- | :--- | :--- | :--- |
| **Schema Consistency** | ก.พ. ใช้ชื่อคอลัมน์ต่างกัน, ส่วนลดเป็น %, วันที่ DD/MM/YYYY | รวมเป็น Schema เดียวกัน Datetime และ Float มาตรฐาน | ข้อมูลรวมกันได้อย่างสมบูรณ์ 100% |
| **Uniqueness (ความไม่ซ้ำซ้อน)** | พบ Order ซ้ำ 2 แถว, Customer ซ้ำ 3 แถว, Payment ซ้ำ 1 แถว | คีย์หลักทุกตาราง Unique 100% (Asserted) | ขจัดปัญหาข้อมูลบวมและ Double Counting |
| **Domain Validity (ความถูกต้อง)** | พบ Quantity ติดลบ 2 แถว, Unit Price เป็น Null 2 แถว | ทุกแถว Quantity > 0, Price > 0, Discount อยู่ในช่วง [0,1] | ป้องกันการคำนวณยอดขายผิดพลาด |
| **Referential Integrity** | พบ Orphan Customer 22 แถว, Orphan Product 2 แถว | Fact Table ทุกแถวอ้างอิง Dimension Master ได้ 100% | รายงานวิเคราะห์ตามมิติถูกต้องสมบูรณ์ |
| **Text Standardization** | ชื่อจังหวัดปนเป 14 รูปแบบ, อีเมลมีช่องว่างและตัวพิมพ์ใหญ่ | ชื่อจังหวัดเหลือ 6 จังหวัดทางการ, อีเมล lowercase/trimmed | รายงานสรุปตามภูมิภาคแม่นยำ |
| **Business Readiness** | รวมรายการ FAILED 44 รายการ และ REFUNDED 18 รายการ | กรองเหลือเฉพาะ PAID Sales รวม 660 รายการ | รายงานยอดขายสุทธิ 10.22M ฿ พร้อมใช้งานจริง |

