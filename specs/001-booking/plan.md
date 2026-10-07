# แผนงานฟีเจอร์: จองคิวตรวจสุขภาพ (Booking)

## 1. สรุปแนวทาง
- ฟีเจอร์นี้ให้ผู้รับบริการที่ยืนยันตัวตนแล้วเลือกแพ็กเกจ วัน และช่วงเวลาตรวจสุขภาพ แล้วได้รับหมายเลขคิว เพื่อให้การจองเกิดขึ้นภายใน 3 นาที โดยยึดหลักป้องกันการจองซ้ำและคงสภาพความสมดุลของโควตาแต่ละช่วงเวลา
- ผู้ใช้หลักคือผู้รับบริการที่ผ่านการยืนยันตัวตนแล้ว และระบบที่ทำหน้าที่จัดการการจอง การคำนวณช่วงเวลาว่าง การออกหมายเลขคิว และการส่งข้อความยืนยันแบบ asynchronous
- แนวทางหลักคือแยกภารกิจเป็น 3 ลำดับ: ดึงข้อมูลช่วงเวลาและสิทธิ์การจอง, ตรวจเงื่อนไขก่อนยืนยัน, และยืนยันการจองพร้อมบันทึกข้อมูลและคิวส่งข้อความซ้ำ
- ระบบจะใช้ข้อมูล HN จาก HIS เพื่ออ้างอิงผู้รับบริการและไม่เก็บเลขบัตรประชาชนในตารางการจองตาม IF-HIS-01
- การออกแบบจะเน้นความปลอดภัย (TLS, audit log, รหัสผู้เข้าถึง) และความสามารถตอบสนอง p95 <= 2 วินาที สำหรับผู้ใช้พร้อมกัน 200 คน ตาม NFR-PERF-01 และ NFR-SEC-01

## 2. เทคโนโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| MySQL | CON-TECH-01 | ใช้เก็บข้อมูลการจอง ข้อมูลช่วงเวลาและ audit log ตามข้อกำหนดฝ่าย IT |
| React (Vite) สำหรับหน้าบ้าน | ทีมเลือกเอง ไม่ได้มาจาก spec | สำหรับแสดงวัน/ช่วงเวลาว่าง การยืนยัน และแสดงหมายเลขคิว |
| Python FastAPI สำหรับ API | ทีมเลือกเอง ไม่ได้มาจาก spec | สำหรับตรวจเงื่อนไขจอง บันทึกข้อมูล และจัดการส่งข้อความยืนยันแบบ asynchronous |
| TLS 1.2+ สำหรับข้อมูลรับส่ง | NFR-SEC-01 | ครอบคลุมทุกการรับส่งข้อมูลที่เกี่ยวกับการจอง |
| ระบบแจ้งเตือน SMS/LINE แบบ asynchronous | IF-NOT-01 | การส่งข้อความยืนยันจะไม่ทำให้กระบวนการจองหยุดชะงัก |
| Audit log แบบบันทึกผู้เข้าถึง เวลา และรหัสผู้รับบริการ | DOM-PDPA-01 | ใช้ทุกครั้งที่เข้าถึงข้อมูลการจองหรือข้อมูลผู้รับบริการ |

## 3. โมเดลข้อมูล

| Entity | ฟิลด์หลัก | รองรับ FR/Constraint |
|---|---|---|
| PatientProfile | patient_id, HN, verified_at, status | รองรับ IF-IDP-01, IF-HIS-01, FR-BKG-02, FR-BKG-04 |
| ServicePackage | package_id, name, slot_duration, status | รองรับ FR-BKG-01, FR-BKG-06 |
| BookingSlot | slot_id, service_date, start_time, end_time, capacity, remaining_seats, package_id | รองรับ FR-BKG-01, FR-BKG-03, FR-BKG-04, FR-BKG-06 |
| Booking | booking_id, patient_id, HN, package_id, service_date, slot_id, queue_number, status, created_at, confirmed_at | รองรับ FR-BKG-02, FR-BKG-04, FR-BKG-05, AC-BKG-01, AC-BKG-02, AC-BKG-04 |
| BookingQueueNumber | queue_no_id, service_date, sequence_no, reset_policy | รองรับ FR-BKG-04, AC-BKG-01 | 
| NotificationRequest | notification_id, booking_id, channel, payload, status, created_at, retry_at, attempts | รองรับ IF-NOT-01, FR-BKG-05, NFR-REL-02, AC-BKG-04 |
| AuditLog | log_id, actor_user, accessed_at, patient_hn, action, resource | รองรับ DOM-PDPA-01, AC-BKG-06 |

หมายเหตุ:
- ตารางการจองจะเก็บ HN เท่านั้น และไม่มีการเก็บเลขบัตรประชาชนตาม IF-HIS-01
- จองคิว 1 รายการต่อผู้รับบริการต่อวัน จะถูกควบคุมที่ระดับ Booking โดยตรวจสอบ patient_id + service_date และ status ไม่ใช่การใช้งานครั้งก่อนหน้า
- ข้อมูลจำนวนที่นั่งคงเหลือจะถูกลดลงหลังยืนยันสำเร็จ และกู้กลับได้ก็ต่อเมื่อมีการยกเลิกหรือรีเซ็ตตามนโยบายที่ UC-02 / UC-09 กำหนด แต่เป็น Out of scope ในเฟสนี้

## 4. API / หน้าจอ

| ชื่อ | รายละเอียด | รองรับ |
|---|---|---|
| GET /booking/slots | ดึงช่วงเวลาว่างภายใน 30 วันข้างหน้า พร้อมจำนวนที่นั่งคงเหลือ และลิงก์ค่ากรองตามแพ็กเกจ | FR-BKG-01, FR-BKG-06 |
| POST /booking/availability/check | ตรวจสอบว่าช่วงเวลาที่เลือกอาจเต็มระหว่างยืนยัน และคืนค่า alternative slots | FR-BKG-03 |
| POST /booking/validate | ตรวจสอบว่าผู้รับบริการมีคิวที่ยังไม่ได้ใช้ในวันเดียวกันหรือไม่ | FR-BKG-02 |
| POST /booking/confirm | บันทึกการจอง ลดจำนวนที่นั่งออกหมายเลขคิว และสร้างคำขอแจ้งเตือน | FR-BKG-04, AC-BKG-01 |
| POST /booking/notification/retry | ส่งซ้ำข้อความในคิวเมื่อส่งไม่สำเร็จ | FR-BKG-05, NFR-REL-02 |
| GET /booking/{bookingId} | แสดงหมายเลขคิวและสถานะปัจจุบันหลังยืนยันสำเร็จ | FR-BKG-04, FR-BKG-05 |
| หน้าจอเลือกแพ็กเกจ/วัน/ช่วงเวลา | ผู้ใช้เลือกแพ็กเกจ จากนั้นดูช่วงเวลาว่างและจำนวนที่นั่งคงเหลือ | FR-BKG-01, FR-BKG-06 |
| หน้าจอยืนยัน | แสดงสรุปและปุ่มยืนยัน พร้อมข้อความกรณี slot เต็มหรือมีคิวเดิม | FR-BKG-02, FR-BKG-03, FR-BKG-04 |
| หน้าจอผลลัพธ์ | แสดงหมายเลขคิวและสถานะการส่งข้อความยืนยัน | FR-BKG-04, FR-BKG-05 |

## 5. ตารางตรวจ Constraints

| Constraint ID | ถูกนำไปใช้ที่ไหนใน plan | สถานะ |
|---|---|---|
| CON-TECH-01 | MySQL ใช้เป็นฐานข้อมูลหลักสำหรับ Booking, Slot, Notification, AuditLog | ใช้แล้ว |
| DOM-PDPA-01 | AuditLog model และการบันทึกทุกครั้งที่เข้าถึงข้อมูลผู้รับบริการ | ใช้แล้ว |
| IF-IDP-01 | ขั้นตอน validate / confirm ดำเนินต่อก็ต่อเมื่อ verified_at ถูกยืนยันจากระบบยืนยันตัวตน | ใช้แล้ว |
| IF-HIS-01 | PatientProfile ใช้ HN แทนเลขบัตรประชาชน และไม่เก็บเลขบัตรประชาชนใน Booking | ใช้แล้ว |
| IF-NOT-01 | NotificationRequest ใช้ช่องทาง SMS/LINE แบบ asynchronous และให้การจองไม่รอผลส่ง | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | ทดสอบอย่างไร |
|---|---|---|
| AC-BKG-01 | test_AC_BKG_01_successful_booking_reduces_capacity | Given slot 09.00 มีที่นั่งว่าง 1 ที่ เมื่อยืนยันการจอง แล้วต้องบันทึก Booking, แสดง queue number, และ remaining_seats = 0 |
| AC-BKG-02 | test_AC_BKG_02_duplicate_booking_same_day_blocked | Given มีคิวที่ยังไม่ได้ใช้ในวันเดียวกัน เมื่อพยายามจองอีกรายการในวันเดียวกัน จะต้องปฏิเสธ และแสดง queue number เดิม |
| AC-BKG-03 | test_AC_BKG_03_slot_full_suggests_alternatives | Given slot ที่เลือกเต็มก่อนยืนยัน เมื่อกดยืนยัน จะต้องแสดงข้อความ “ช่วงเวลาเต็ม” และเสนอ 3 ตัวเลือกใกล้เคียง โดยไม่บันทึกการจองซ้อน |
| AC-BKG-04 | test_AC_BKG_04_notification_failure_keeps_booking | Given ระบบแจ้งเตือนไม่ตอบสนอง เมื่อยืนยันการจอง จะต้องเก็บ Booking, แสดงหมายเลขคิว, และมี NotificationRequest ในคิวซ้ำภายใน 5 นาที |
| AC-BKG-05 | test_AC_BKG_05_slot_lookup_performance | Given 200 ผู้ใช้พร้อมกัน เมื่อเรียก GET /booking/slots ต้องได้ p95 <= 2 วินาที |
| AC-BKG-06 | test_AC_BKG_06_audit_log_written | Given มีการเปิดดูข้อมูลการจอง เมื่อเข้าถึงเสร็จสิ้น จะต้องมี AuditLog ที่ระบุ actor_user, accessed_at, patient_hn |

## 7. ลำดับงาน

1. กำหนดโครงสร้างข้อมูลและ schema MySQL สำหรับ Booking, Slot, NotificationRequest, AuditLog และ PatientProfile ตาม FR-BKG-01, FR-BKG-04, DOM-PDPA-01
2. สร้าง API ดึงช่วงเวลาว่างและคำนวณจำนวนที่นั่งคงเหลือจากแพ็กเกจ วันที่ และช่วงเวลา ตาม FR-BKG-01 และ FR-BKG-06
3. สร้างเงื่อนไขตรวจสอบคิวซ้ำในวันเดียวกันและผลลัพธ์แสดงหมายเลขคิวเดิม ตาม FR-BKG-02 และ AC-BKG-02
4. สร้าง logic สำหรับกรณี slot เต็มระหว่างยืนยัน และเสนอ 3 ตัวเลือกใกล้เคียงรวมวันถัดไป ตาม FR-BKG-03 และ ASM-03
5. สร้าง flow ยืนยันการจอง: บันทึก Booking, ปรับ remaining_seats, ออก queue number, และสร้าง NotificationRequest ตาม FR-BKG-04 และ AC-BKG-01
6. สร้างส่วนจัดการส่งข้อความยืนยันแบบ asynchronous และการส่งซ้ำภายใน 5 นาที ตาม IF-NOT-01, FR-BKG-05 และ NFR-REL-02
7. สร้าง audit log และตรวจสอบสิทธิ์เข้าถึงข้อมูลผู้รับบริการตาม DOM-PDPA-01 และ IF-IDP-01 พร้อมทดสอบ AC-BKG-06
8. ทดสอบประสิทธิภาพและความถูกต้องครบทุก AC รวม NFR-PERF-01, TLS, และกรณี exception ที่คาดหวัง

## 8. สิ่งที่ยังไม่ทำ

- Q-02: หมายเลขคิวรีเซ็ตรายวัน หรือนับต่อเนื่อง? -> รอคำตอบจากเจ้าหน้าที่เวชระเบียน
- ส่วนที่เกี่ยวข้องกับ Q-02 จะยังไม่สร้างจนกว่าจะได้คำตอบจากเจ้าหน้าที่เวชระเบียน เช่น นโยบาย reset queue number และการคำนวณหมายเลขคิวในระบบการจอง

## 9. ข้อสรุปสำหรับทีม
- แผนนี้ยังคงยึด spec.md เป็นความจริงหนึ่งเดียว และไม่เพิ่มความต้องการใด ๆ นอกเหนือจาก FR/Constraint ที่ระบุไว้
- ความคิดที่ต้องระวังมากที่สุดคือการคำนวณหมายเลขคิว เพราะตอนนี้ยังมี Open Question ค้างอยู่ และต้องรอคำตอบก่อน finalizing logic อย่างเต็มรูปแบบ
- ส่วนที่สามารถเริ่มทำได้ทันทีคือ schema การจอง, ดึงช่วงเวลาว่าง, ป้องกันคิวซ้ำ, flow confirm, audit log, และ retry queue
