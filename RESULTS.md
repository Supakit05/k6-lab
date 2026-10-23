# K6 lab results

## Part 0

### Step 0.1: First test
- `http_req_duration`: 261.54 ms
- HTTP requests: 1; Iterations: 1; Checks: 100%
- ผล: ผ่าน

### Step 0.2: More users for longer
- Virtual users: 5; Duration: 20 วินาที
- HTTP requests: 80; Req/s: 3.879317; p(95): 255.85 ms; Checks: 100%
- เหตุผลที่ Req/s ต่ำกว่า 5: ผู้ใช้จำลองแต่ละคนต้องรอคำตอบจาก API และ `sleep(1)` ก่อนเริ่มรอบใหม่ จึงไม่ได้ส่งคำขอหนึ่งครั้งต่อวินาทีตลอดเวลา

### Step 0.6: Thresholds
- Normal run: p(95) = 376.82 ms; เกณฑ์ `p(95)<1500`; Exit code = 0 (ผ่าน)
- Strict run: p(95) = 375.44 ms; เกณฑ์ `p(95)<300`; Exit code = 99 (ไม่ผ่านตามที่ตั้งใจ)
- สรุป: เมื่อกำหนดเกณฑ์เข้มงวดกว่าเวลาที่วัดได้ K6 แจ้งว่า Threshold ไม่ผ่าน

### Step 0.7: Requirement to thresholds
- Original requirements: 3 VUs, 30 วินาที; `p(95)<1000`, `p(99)<1500`, `http_req_failed` ต่ำกว่า 1%
- Original run: p(95) = **367.59 ms**; p(99) = **372.36 ms**; `http_req_failed` = 0%; Checks = 100%; ทุก Threshold ผ่าน
- Modified run: เปลี่ยนเกณฑ์ p(95) จาก 1000 เป็น 200 ms; p(95) = **365.21 ms**; p(99) = **377.47 ms**; `http_req_failed` = 0%; Exit code = **99**
- สรุป: รอบแก้ไขไม่ผ่านเฉพาะเกณฑ์ p(95) ตามที่ตั้งใจ ส่วนเกณฑ์ p(99) และอัตราคำขอล้มเหลวยังผ่าน

### Step 0.8: Bug hunt
- ก่อนแก้: 75 HTTP requests; `no pepperoni` ไม่ผ่าน 7 ครั้ง; `no knife needed` ไม่ผ่าน 26 ครั้ง
- สาเหตุ: API ใช้ชื่อแบบแยกตัวพิมพ์เล็ก/ใหญ่ (`Pepperoni`, `Knife`) แต่คำขอเดิมใช้ `pepperoni`, `knife`
- วิธีแก้: เปลี่ยนเป็น `excludedIngredients: ['Pepperoni']` และ `excludedTools: ['Knife']`
- หลังแก้: 75 HTTP requests; Checks 225/225 (100%); `http_req_failed` = 0%
- สรุป: การทดสอบหลายรอบช่วยพบว่าบางคำตอบมีวัตถุดิบหรืออุปกรณ์ที่ผู้ใช้ไม่ต้องการ เมื่อแก้ชื่อให้ตรงกับ API แล้ว Checks ผ่านทั้งหมด

## Part A results

| Step | Test type | Users | Req/s | p(95) | Max | Failed | Checks |
| ---- | --------- | ----: | ----: | ----- | --- | ------ | ------ |
| A1 | Smoke | 1 | 0.488138 | 368.76 ms | 375.61 ms | 0% | 100% |
| A2 | Average load | 3 | 1.327971 | 353.13 ms | 697.55 ms | 0% | 100% |
| A3 | Stress | 5 | 2.225601 | 358.68 ms | 379.86 ms | 0% | 100% |
| A4 | Spike | 5 | 1.048462 | 354.65 ms | 376.71 ms | 0% | 100% |
| A5 | Soak | 2 | 0.976410 | 377.36 ms | 4.87 s | 0% | 100% |


## Part B

### 1. Fill in the results table for Steps A1 to A5.
ตาราง `Part A results` ด้านบนบันทึกผลครบ A1–A5 โดยทุกการทดสอบมี Checks ผ่าน 100% และ `http_req_failed` เท่ากับ 0%

### 2. In Step 0.2, why was the request rate below 5 requests per second with 5 virtual users?
เพราะผู้ใช้จำลองแต่ละคนต้องรอ API ตอบกลับและพัก 1 วินาที (`sleep(1)`) ก่อนส่งคำขอใหม่ แม้มี 5 VUs แต่จึงส่งคำขอได้เฉลี่ยประมาณ 3.88 ครั้ง/วินาที ไม่ใช่ 5 ครั้ง/วินาที

### 3. Did the response time grow from the average-load test to the stress test? Explain your answer with the operational envelope.
เพิ่มขึ้นเล็กน้อย โดย p(95) จาก A2 = 353.13 ms เป็น A3 = 358.68 ms หรือเพิ่ม 5.55 ms ขณะที่ Checks ยังผ่าน 100% และไม่มี HTTP requests ล้มเหลว แสดงว่าในช่วงโหลดที่ทดลอง (สูงสุด 3–5 VUs) ระบบยังทำงานได้ตามเกณฑ์ที่กำหนด แต่ยังบอกขีดจำกัดสูงสุดของระบบ (operational envelope) ทั้งหมดไม่ได้

### 4. In the spike test, how long did the response time take to return to normal after the spike?
จากกราฟ A4 เวลาตอบสนองแกว่งขึ้นลงเล็กน้อยในช่วงเพิ่มผู้ใช้จาก 1 เป็น 5 คน และหลังลดกลับเป็น 1 คนก็ยังอยู่ในระดับใกล้เคียงเดิม ไม่พบการช้าลงชัดเจนที่ต้องใช้เวลาฟื้นตัว จึงระบุจำนวนวินาทีที่แน่นอนไม่ได้

### 5. In the soak test, did the response time or the error rate get worse over time? Name one kind of defect that a soak test of several hours could reveal.
การทดสอบ A5 ใช้เวลา 11 นาที พบช่วงเวลาตอบสนองสูงขึ้นชั่วคราวราว 19:07–19:08 และ 19:11–19:12 (สูงสุด 4.87 วินาที) แต่ช่วงท้ายกลับใกล้เคียงช่วงต้น และ `http_req_failed` ยังเป็น 0% จึงไม่พบแนวโน้มแย่ลงต่อเนื่อง หากทดสอบหลายชั่วโมงอาจพบ **memory leak** หรือการใช้หน่วยความจำเพิ่มขึ้นโดยไม่คืนหน่วยความจำ

### 6. Why do the thresholds in this lab use p(95) rather than the average?
เพราะค่าเฉลี่ยอาจซ่อนคำขอที่ตอบสนองช้า ค่า p(95) แสดงเวลาที่ประมาณ 95% ของคำขอใช้ไม่เกินค่านั้น จึงเหมาะกับการตรวจสอบว่าผู้ใช้ส่วนใหญ่ได้รับการตอบสนองเร็วตามเกณฑ์หรือไม่

### 7. Your results come from a shared public site reached over the internet. Give two reasons why the same tests against a system in a lab could give different numbers.
1. อินเทอร์เน็ตภายนอกมีความหน่วงและความแออัดของเครือข่ายเปลี่ยนแปลงได้ ต่างจากเครือข่ายห้อง Lab ที่ควบคุมได้มากกว่า
2. เว็บไซต์ QuickPizza เป็นระบบสาธารณะที่มีผู้ใช้อื่นร่วมใช้งาน ปริมาณโหลดของเซิร์ฟเวอร์จึงไม่แน่นอน ต่างจากระบบทดสอบใน Lab

### 8. Why must you never run a stress, spike or breakpoint test with hundreds of users against a site that you do not own?
เพราะอาจทำให้เซิร์ฟเวอร์ทำงานหนักจนเว็บช้าหรือหยุดให้บริการ กระทบผู้ใช้จริง และเป็นการทดสอบโดยไม่ได้รับอนุญาต ควรทดสอบเฉพาะระบบที่มีสิทธิ์และอยู่ภายใต้ขอบเขตที่เจ้าของระบบอนุมัติ

### 9. Write one vague performance requirement for an app you use every day. Rewrite it as a testable requirement, and show the K6 thresholds that check it.
**Application:** แอปสั่งอาหาร

**Vague requirement:** "แอปต้องตอบสนองเร็ว"

**Testable requirement:** เมื่อมีผู้ใช้พร้อมกัน 3 คน เป็นเวลา 30 วินาที คำขออย่างน้อย 95% ต้องตอบสนองภายใน 1 วินาที, 99% ภายใน 1.5 วินาที และอัตราคำขอล้มเหลวต้องต่ำกว่า 1%

**K6 thresholds:**

```javascript
export const options = {
  vus: 3,
  duration: '30s',
  thresholds: {
    http_req_duration: ['p(95)<1000', 'p(99)<1500'],
    http_req_failed: ['rate<0.01'],
  },
};
```
