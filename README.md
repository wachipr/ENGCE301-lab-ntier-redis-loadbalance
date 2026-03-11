# engce301-lab-ntier-redis-loadbalance

## บันทึกผลการทดลอง

### สถานการณ์ที่ 1
**ปัญหาที่พบ:** เมื่อสั่ง `docker stop` เพื่อจำลองสถานการณ์ Server ล่ม พบว่าหน้าเว็บเกิด Error `502 Bad Gateway` หรือ `504 Gateway Timeout` บางครั้งต้องกด Refresh หลายรอบ หรือต้อง Restart nginx ถึงจะกลับมาใช้งานได้ สาเหตุเพราะ nginx เรียกหา App Instance เดิมที่ล่มไปแล้ว และไม่มี Redirect ไปยัง App Instance ตัวอื่น

**การแก้ไข:** ทำการแก้ไข Config nginx `default.conf` ดังนี้
* ใช้คำสั่ง `resolve` เพื่อให้ nginx อัปเดตรายชื่อ IP จาก DNS ของ Docker เรื่อยๆ (Service Discovery)
* เพิ่ม `proxy_next_upstream` เพื่อให้ระบบทำ **Failover** (เมื่อยิงไปเจอตัวที่ล่ม ให้สลับไปยิงตัวอื่นที่ยังติดอยู่อยู่ทันทีแบบอัตโนมัติ)

**ผลลัพธ์:** เมื่อสั่งปิด App บางตัว ระบบยังคงสามารถตอบสนองได้ต่อเนื่องผ่าน Instance ตัวอื่นที่เหลืออยู่ โดยไม่เกิดอาการล่มยาวให้ฝั่ง Frontend เห็น

### สถานการณ์ที่ 2
เป็นการทดสอบเพื่อดูว่า การทำ Load Balancing แบบ Round-Robin โดยขยายจำนวน App Instance ช่วยเพิ่ม Throughput และลด Latency ได้จริงหรือไม่ 

ทดสอบด้วยคำสั่ง `wrk -t4 -c50 -d20s http://localhost/api/tasks`

| Case | #App Instances | Concurrency (c) | Duration | Avg Latency | Requests/sec | สรุป |
|:---:|:---:|:---:|:---:|:---:|:---:|---|
| **A** | 1 | 50 | 20s | `25.68 ms` | `2984.25` | baseline |
| **B** | 3 | 50 | 20s | `20.15 ms` | `3226.00` | after scale-out |

**สรุปผลการทดลอง**
1. **Avg Latency** ลดลงอย่างชัดเจน จาก 25.68 ms เหลือเพียง 20.15 ms บ่งบอกว่าประสิทธิภาพในการตอบสนองแต่ละ Request ไวขึ้น
2. **Requests/sec** ค่า Throughput เพิ่มขึ้น จากประมาณ 2,984 Req/s เป็น 3,226 Req/s รองรับโหลดได้มากขึ้น
3. การใช้ N-Tier Architecture ควบคู่กับ Redis Caching และ nginx Load Balancing ช่วยให้ระบบมีการ Scale และ Highly Available

---

## สรุปความรู้ที่ได้จากการเรียนสัปดาห์นี้

ในการออกแบบ Software ที่ดีควรประกอบด้วย 3 ส่วนหลักคือ **Frontend, Backend และ Database** โดยก่อนลงมือพัฒนา เราควรเลือกสถาปัตยกรรม (Architecture) ให้เหมาะสมกับเป้าหมายของระบบ เช่น เน้น Performance, เน้นความทนทานต่อการล่ม (High Availability) หรือเน้น Security 

### N-Tier ยังไง? แล้วเมื่อไหร่ควรเพิ่ม Tier?
* **Database ถูก Query ซ้ำ ๆ:** เพิ่ม Cache Tier (เช่น Redis) เพื่อลดภาระ DB และเพิ่มความเร็ว
* **App Server รับ Load ไม่ไหว:** เพิ่ม Load Balancer Tier (เช่น nginx) และ Scale App ให้กระจายโหลด
* **เมื่อไหร่ที่ไม่ควรเพิ่ม:** ระบบยังมีขนาดเล็ก ผู้ใช้ไม่มาก การเพิ่ม Tier อาจทำให้ระบบซับซ้อนเกินความจำเป็น (Over-engineering)

---