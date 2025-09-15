# **Functional Requirements Document : Re:Play**

เอกสารนี้สรุป “สิ่งที่ระบบต้องทำ” (WHAT) อย่างละเอียด แยกย่อยเป็นฟังก์ชัน, business rules, เงื่อนไข, เคสขอบ และเกณฑ์ยอมรับ เพื่อใช้ส่งต่อ Dev/QA/Design โดยทีม Design จะวาด wireframe อ้างอิงจาก Use Cases ในเอกสารนี้โดยตรง

**Project:** Re:Play  
**Phase 1 (MVP)**  
**Reference:** PRD V0.01 (2025‑09‑10)  
**Version:** 0.01  (2025‑09‑11)

## **1\) Roles & Permissions (WHAT)**

* **Visitor** → สมัคร/เข้าสู่ระบบด้วย Google  
* **Member** → ฟังอัลบั้มย้อนหลัง, ใช้ฟิลเตอร์, Save/Unsave Albums, จัดการโปรไฟล์  
* **Admin** → สร้าง/แก้ไข/ลบ ศิลปิน และอัลบั้มทางการ

## **2\) System Use Cases with Embedded Test Scenarios**

### **UC-01 Sign up with Google** 

**Actors:** Visitor, Google (SSO Provider)

**Description:** ผู้ใช้ใหม่ต้องการสมัครเข้าใช้งานโดยใช้บัญชี Google เพื่อลดความซับซ้อนในการลงทะเบียน ระบบต้องรองรับการยืนยันตัวตนผ่าน Google OAuth และสร้างบัญชีใหม่ให้ผู้ใช้พร้อมบังคับให้ยอมรับ PDPA ก่อนเริ่มใช้งาน

**Precondition:** 

* ผู้ใช้ยังไม่มีบัญชีในระบบ

* แอปตั้งค่า Google OAuth (client\_id, redirect\_uri, scope=email, profile) เรียบร้อยแล้ว

**Trigger:** ผู้ใช้กด "Continue with Google"

**Main Flow:**

1. ผู้ใช้เลือก "Sign up with Google"

2. Redirect ไป Google → ผู้ใช้ยืนยันตัวตน (scope: email, profile)

3. Google ส่ง token กลับ

4. ระบบตรวจสอบ token → ดึงอีเมล/โปรไฟล์

5. หากยังไม่มีบัญชี → สร้างบัญชีใหม่/ผูก provider\_id → สร้างเซสชัน

6. กรอกข้อมูลเพิ่มเติม  
   * Displayname (Required)  
     * ห้ามซ้ำกับ Displayname ของ member  ในระบบ  
   * Date of birth (Required)  
     * เป็น Calendar ให้เลือกวันเดือนปี  
     * Default วันที่ปัจจุบัน  
   * Gender (Required)  
     * Female  
     * Male  
     * Other

7. อ่านและยอมรับ ToS/PDPA → ยอมรับ PDPA → Home 

8. กด “Continue”

9. ระบบแสดง “Signed up\!” และอยู่หน้า  Home 

**Alternative Flows / Exceptions (รวม Test Scenarios):**

* ผู้ใช้กดยกเลิกบนหน้า Google SSO ระบบพากลับไปหน้า Login  
  TC01-1: Cancel on Google → ระบบ redirect ไปหน้า Login และแจ้ง "Sign up canceled."  
  【Toast】

* token จาก Google ไม่ถูกต้องหรือหมดอายุ ระบบปฏิเสธการเข้าสู่ระบบ  
  TC01-2: Token invalid/expired → ระบบแจ้ง "Authentication failed. Please try again."  
  【Toast \+ Action: Retry】

* email จาก Google ซ้ำกับบัญชีที่มีอยู่ ระบบแสดง flow ผูกบัญชี  
  TC01-3: Duplicate email → ใช้เข้าระบบได้ ถือเป็นการ Login โดยใช้อีเมลบัญชีเดิม (ไม่แจ้งเตือน) พาเข้าหน้า Home เลย

* Google ไม่ส่ง email กลับมา (ปิด scope) ระบบบล็อกการสมัครทันที  
  TC01-4: Email scope missing → ระบบแจ้ง "Cannot sign up without email permission."  
  【Dialog: Cancel/Try again】

* Replay token ถูกนำมาใช้ซ้ำ ระบบบล็อกและบันทึก log  
  TC01-5: Replay token → ระบบแจ้ง "Invalid request. Please try again."  
  【Toast】

* ผู้ใช้ไม่กรอกฟิลด์บังคับ (เช่น วันเกิด เพศ) ระบบไม่อนุญาตให้สมัคร  
   TC01-6: Missing required fields → ระบบแจ้ง "Please fill in all required fields."  
   【Inline Validation】

* ผู้ใช้กรอก Displayname ซ้ำ ระบบไม่อนุญาตให้ใช้ชื่อ  
  TC01-7: Duplicate displayname → ระบบแจ้ง "Displayname already taken."  
  【Toast】

**Business Rules:** 

* ต้องใช้ Google OAuth 2.0 ตาม scope: email, profile

* อีเมลจาก Google ต้องไม่ซ้ำกับบัญชีที่มีอยู่แล้ว (unique constraint)

* ผู้ใช้ต้องยอมรับ PDPA ก่อนเข้าใช้งาน

* ทุกการสมัครต้องบันทึก audit log (user\_id, action, timestamp)

**Postconditions:**

* บัญชี Member ถูกสร้างและผูกกับ Google provider\_id

* เซสชันถูกสร้างและผู้ใช้ถูกนำเข้าแอป

* บันทึก consent (PDPA) และ audit log เรียบร้อย

---

### **UC-02 Sign in with Google**

**Actors:** Visitor, Google

**Description:** ผู้ใช้ที่มีบัญชี ผูกกับ Google อยู่แล้ว ต้องการเข้าสู่ระบบโดยใช้ Google OAuth เพื่อความสะดวกและรวดเร็ว ระบบต้องตรวจสอบ token และสร้างเซสชันให้ผู้ใช้เข้าสู่ระบบสำเร็จ

**Preconditions:**

* ผู้ใช้มีบัญชีที่ผูกกับ Google อยู่แล้วในระบบ

* แอปตั้งค่า Google OAuth (client\_id, redirect\_uri, scope=email, profile) เรียบร้อยแล้ว

**Trigger:** ผู้ใช้กด "Continue with Google"

**Main Flow:**

1. ผู้ใช้เลือก "Sign in with Google"

2. Redirect ไป Google → ยืนยันตัวตน

3. Google ส่ง token กลับ

4. ระบบตรวจสอบ token → match provider\_id/email กับบัญชีที่มีอยู่

5. ระบบสร้างเซสชันใหม่ → Login สำเร็จ

**Alternative Flows / Exceptions (รวม Test Scenarios):**

* ผู้ใช้กดยกเลิกบน Google ระบบพากลับไปหน้า Login  
  TC02-1: Cancel → ระบบพากลับไปหน้า Login และแจ้ง "Sign in canceled."  
  【Toast】

* token invalid/expired ระบบไม่อนุญาตให้เข้าสู่ระบบ  
  TC02-2: Token invalid → ระบบแจ้ง "Authentication failed. Please try again."  
  【Toast \+ Action: Retry】

* email พบในระบบแต่ยังไม่ link ระบบบังคับให้ทำการ link  
  TC02-3: Not linked → ระบบแจ้ง "Account not linked. Please verify."  
  【Dialog: Cancel/Verify】

* Replay token ถูกนำมาใช้ซ้ำ ระบบบล็อกและบันทึก log  
  TC02-4: Token reuse → ระบบแจ้ง "Invalid request. Please try again."  
  【Toast】

**Business Rules:**

* ต้องใช้ Google OAuth 2.0 ตาม scope: email, profile

* ระบบต้องตรวจสอบว่าบัญชีถูกผูกกับ Google แล้วก่อนอนุญาตให้เข้าสู่ระบบ

* ทุกการเข้าสู่ระบบต้องบันทึก audit log (user\_id, action, timestamp)

**Postconditions:**

* ผู้ใช้เข้าสู่ระบบสำเร็จและถูกพาไปหน้า Home

* ระบบสร้างเซสชันใหม่สำหรับผู้ใช้

* audit log ถูกบันทึกว่าเข้าสู่ระบบสำเร็จ

---

### **UC-03 Manage Consent** 

**Actors:** Member

**Description:** ผู้ใช้ต้องการจัดการความยินยอมด้านข้อมูลส่วนบุคคล (PDPA) ภายในหน้า “Data & Privacy” โดยสามารถเปิด/ปิดรายการความยินยอมที่รองรับได้ และระบบต้องบันทึกประวัติการเปลี่ยนแปลงทุกครั้ง

**Preconditions:**

* ผู้ใช้ล็อกอินแล้ว

* บัญชีผู้ใช้มีหน้าการตั้งค่า “Data & Privacy” พร้อมรายการ consent ที่กำหนดไว้ 

**Trigger:** ผู้ใช้เปิดหน้า “Data & Privacy” และสลับสถานะ consent (toggle)

**Main Flow:**

1. ผู้ใช้เปิดหน้า “Data & Privacy”

2. ระบบแสดงรายการ consent ที่มีอยู่ พร้อมสถานะปัจจุบัน (on/off)

3. ผู้ใช้สลับสถานะ consent ที่ต้องการ

4. ระบบตรวจสอบความถูกต้องและบันทึกค่า consent ใหม่

5. ระบบอัปเดต UI ให้เห็นสถานะล่าสุดทันที และแสดงข้อความยืนยันสั้น ๆ

**Alternative / Exception (รวม Test Scenarios):**

* Server error ระหว่างบันทึก consent ระบบ rollback toggle และแจ้ง error  
  TC03-1: Server fail → ระบบแจ้ง "Unable to save consent. Please try again."  
  【Toast \+ Action: Retry】

* ผู้ใช้กด toggle ขณะ offline ระบบ rollback และแจ้งว่าต้องเชื่อมต่อใหม่  
  TC03-2: Offline toggle → ระบบแจ้ง "No internet connection. Please try again."  
  【Toast】

* ผู้ใช้สลับ toggle รัว/ถี่เกินไป ระบบหน่วงไม่ให้เรียกซ้ำจนกว่าจะครบช่วงเวลา  
  TC03-3: Too frequent → ระบบแจ้ง "Action too frequent. Please wait."  
  【Toast】

* Session หมดอายุระหว่างบันทึก consent ระบบพาไปหน้า Login และเมื่อกลับมาค่า toggle เดิมยังอยู่  
  TC03-4: Session expired → ระบบแจ้ง "Session expired. Please log in again."  
  【Dialog: Sign In】

**Business Rules:**

* ต้องบันทึกเหตุการณ์การเปลี่ยน consent ทุกครั้ง (ผู้ใช้/เวลา/ค่าใหม่-เดิม) เพื่อรองรับการตรวจสอบย้อนหลัง

* ค่าความยินยอมต้องมีผลทันทีหลังบันทึก (real-time enforcement)

* ค่าเริ่มต้นของ consent เป็นไปตามนโยบาย (เช่น ปิดสำหรับ Marketing จนกว่าจะได้รับการยินยอม)

* การถอนความยินยอมต้องหยุดการประมวลผลที่เกี่ยวข้องตั้งแต่เวลาถอนเป็นต้นไป

**Postconditions:**

* ค่าความยินยอมถูกอัปเดตเป็นสถานะล่าสุดตามที่ผู้ใช้เลือก

* มีบันทึกการเปลี่ยนแปลง (consent change log) ครบถ้วน

---

### **UC-04 Delete Account (Immediate)** 

**Actors:** Member

**Description:** ผู้ใช้ต้องการลบบัญชี ของตนทันทีด้วยตนเอง ระบบต้องบังคับยืนยันตัวตนอีกครั้ง (re-auth ผ่าน Google) ก่อนลบ จากนั้นลบข้อมูลบัญชีตามนโยบายความเป็นส่วนตัว ออกจากระบบทุกอุปกรณ์ และบันทึกเหตุการณ์ลงบันทึกการทำรายการ (audit log)

**Preconditions:**

* ผู้ใช้ล็อกอินอยู่ในระบบ

* ระบบเชื่อมต่อ Google OAuth สำหรับ re-auth ได้ (พร้อม client\_id / redirect\_uri)

**Trigger:**  ผู้ใช้เปิด Settings → Data & Privacy → กด “Delete account”

**Main Flow:**

1. ผู้ใช้เปิดหน้า Data & Privacy → กด “Delete account”

2. ระบบแสดงคำเตือน \+ ขอ re-auth ผ่าน Google  
   * Your account will be deleted immediately and cannot be recovered.  
   * All your data will be removed, including:  
     * Profile and settings  
     * Saved albums  (My Library)  
     * Search and usage history  
   * You will be signed out from all devices.  
   * To use the service again, you will need to create a new account.

3. กด “Confirm” 

4. แสดง Dialog ยืนยันการลบ

5. กด “Delete”

6. ระบบลบข้อมูลบัญชีทั้งหมดทันที

7. ระบบ sign-out ผู้ใช้ออกจากทุก session

8. ระบบ redirect ไปหน้า Onboarding พร้อมข้อความ   
   “Account deleted. Returning to the start page.” กด “OK” → หน้าแรกของแอป

**Alternative / Exception (รวม Test Scenarios):**

* ผู้ใช้ re-auth ไม่ผ่านหรือหมดเวลา ระบบยกเลิกการลบ  
  TC04-1: Re-auth fail/timeout → ระบบแจ้ง "Re-authentication timed out. Please try again."【Toast \+ Action: Retry】

* ผู้ใช้กด “Cancel” ที่ dialog ยืนยัน ระบบไม่ทำการลบ  
  TC04-2: Cancel at dialog → บัญชีคงอยู่ (ไม่มีข้อความแจ้งเตือน)

* ระบบ purge ข้อมูลล้มเหลวบางส่วน ระบบ retry และแจ้งผู้ใช้  
  TC04-3: Purge fail → ระบบแจ้ง "Some data could not be deleted. Please contact support."【Toast】

* เครือข่ายล้มเหลวระหว่างกระบวนการลบ ระบบให้ retry  
  TC04-4: Network fail → ระบบแจ้ง "Network error. Please retry."  
  【Toast \+ Action: Retry】

* Session หมดอายุระหว่างขั้นตอน ระบบ redirect ไปหน้า Login และกลับมาหน้ายืนยันอีกครั้ง  
  TC04-5: Session expired → ระบบแจ้ง "Session expired. Please log in again."  
  【Toast \+ Action: Sign In】

* ผู้ใช้กดปุ่ม Delete ซ้ำ/เรียก API ซ้ำ (idempotent) ระบบต้องไม่สร้างผลซ้ำซ้อน เช่น ลบซ้ำหรือสร้าง log เกิน 1 ครั้ง  
  TC04-6: Double-submit Delete → ระบบแสดง "Already deleted." ผลลัพธ์คงเดิม และบันทึก log เพียงครั้งเดียว【Toast】

**Business Rules:**

* ต้องมีการ **re-auth ที่เพิ่งผ่าน** ก่อนทำลบ (ภายในกรอบเวลาสั้น ๆ เช่น ≤ 5 นาที)

* การลบต้องเป็น **ทันที (Immediate)** ตามนโยบาย: ปิดการเข้าถึง, ทำลาย/ทำให้ไม่ระบุตัวตนข้อมูลอ่อนไหวโดยเร็วที่สุด

* ต้อง **sign-out ทุกอุปกรณ์** และทำให้โทเคน/เซสชันทั้งหมดเป็นโมฆะ

* ต้อง **บันทึก audit log** ของการลบ (โดยไม่เก็บข้อมูลส่วนตัวเกินจำเป็น)

* ข้อความเตือนต้องชัดเจน ไม่กำกวม และให้ผู้ใช้ยืนยันอย่างมีสติ (explicit confirmation)

* การดำเนินการต้อง **idempotent**: หากผู้ใช้กดย้ำ/เรียกซ้ำ ไม่ควรสร้างผลข้างเคียงซ้ำซ้อน

**Postconditions:**

* บัญชีผู้ใช้ถูกลบ/ปิดการใช้งานตามนโยบายแล้ว

* ผู้ใช้ถูกออกจากระบบทุกอุปกรณ์ และไม่สามารถเข้าสู่ระบบด้วยบัญชีเดิมได้

* ระบบบันทึก audit log ของเหตุการณ์ลบสำเร็จเรียบร้อย

---

### **UC-05 Logout**

**Actors:** Member, Admin

**Description:** ผู้ใช้ต้องการออกจากระบบบนอุปกรณ์นี้อย่างปลอดภัย ระบบต้องเพิกถอนเซสชันฝั่งเซิร์ฟเวอร์และล้างข้อมูลรับรอง/แคชบนเครื่อง พร้อมพากลับไปหน้า Login/Onboarding การทำงานต้องเป็น idempotent (กดซ้ำไม่เกิดผลซ้ำซ้อน)

**Preconditions:**

* ผู้ใช้ล็อกอินอยู่ (มีเซสชัน/โทเคนในอุปกรณ์)

**Trigger:** Settings **→ กด Logout**

**Main Flow:**

1. ผู้ใช้เปิดเมนู Settings → เลือก Logout

2. ระบบแสดง dialog ยืนยัน “คุณต้องการออกจากระบบหรือไม่?”

3. ผู้ใช้กด “Logout”

4. ระบบ revoke session บนอุปกรณ์และฝั่ง server

5. ระบบ redirect ผู้ใช้ไปหน้า Login

**Alternative / Exception (รวม Test Scenarios):**

* การเชื่อมต่อเครือข่ายล้มเหลวระหว่าง revoke ระบบ logout เฉพาะ local และแจ้งเตือน  
  TC05-1: Network fail → ระบบแสดง "Logged out locally. Server session will be revoked when online."【Toast】

* Token ไม่พบ/หมดอายุแล้ว ระบบถือว่า logout สำเร็จ  
  TC05-2: Token invalid/expired → ออกจากระบบสำเร็จ พาไปหน้า Login (ไม่มีข้อความแจ้งเตือน)

* ผู้ใช้กด “Cancel” ที่ dialog ยืนยัน ระบบไม่ทำการ logout  
   TC05-3: Cancel confirm → กด “Cancel” ไม่ออกจากระบบ  (ไม่มีข้อความแจ้งเตือน)

* ผู้ใช้กดปุ่ม Logout ซ้ำ/รัว ระบบทำงานครั้งเดียวและแจ้งสถานะ  
  TC05-4: Double submit → ระบบแสดง "Processing logout… Please wait."  
  【Toast】

**Business Rules**

* เพิกถอนโทเคน/เซสชันฝั่งเซิร์ฟเวอร์ให้หมดสำหรับอุปกรณ์นี้ (access/refresh)

* ล้างข้อมูลส่วนตัวบนเครื่องทั้งหมดที่ผูกกับบัญชี (โทเคน, แคชโปรไฟล์, ค่า personalization)

* ห้ามทำการ sign-out บัญชี Google ของระบบปฏิบัติการ (ไม่แตะบัญชี Google ระดับเครื่อง)

* ต้องเป็น **idempotent**: เรียกซ้ำไม่ก่อให้เกิดข้อผิดพลาด/สถานะค้าง

* หลังออกจากระบบ ให้บล็อกทุกเส้นทางที่ต้องการการยืนยันตัวตนจนกว่าจะเข้าสู่ระบบใหม่

**Postconditions**

* เซสชันของผู้ใช้บนอุปกรณ์ถูกยกเลิก และโทเคนฝั่งเซิร์ฟเวอร์ถูกเพิกถอน (หรือเข้าคิวเพิกถอนเมื่อกลับมาออนไลน์)

* ข้อมูลส่วนตัวบนเครื่องถูกล้าง

* ผู้ใช้ถูกพาไปหน้า Login และไม่สามารถเข้าหน้าที่ต้องยืนยันตัวตนได้จนกว่าจะล็อกอินใหม่

---

### **UC-06 Edit Profile (Member)**

**Actors:** Member

**Description:** ผู้ใช้ที่เป็น Member ต้องการแก้ไขข้อมูลโปรไฟล์ของตนเอง เช่น ชื่อ วันเกิด เพศ หรือรูปโปรไฟล์ เพื่อให้ข้อมูลส่วนตัวอัปเดตและแสดงผลถูกต้องในระบบ

**Preconditions:**

* ผู้ใช้ล็อกอินแล้ว

**Trigger:** เข้า Settings → Edit Profile

**Main Flow:**

1. ผู้ใช้กด Edit Profile

2. ปรับข้อมูล   
   * Profile picture (Required)  
     * รองรับไฟล์: JPG, PNG  
     * ขนาดไม่เกิน 5MB  
     * ห้ามมีภาพโป๊เปลือย, ความรุนแรง, หรือสัญลักษณ์ไม่เหมาะสม (ระบบไม่สามารถตรวจสอบได้แต่ให้เป็นเงื่อนไขในอนาคต)  
   * Displayname (Required)  
     * ห้ามซ้ำกับ Displayname ของ member  ในระบบ  
   * First name (Required)  
     * กรอกได้ไม่เกิน 50 ตัวอักษร  
   * Last name (Required)  
     * กรอกได้ไม่เกิน 50 ตัวอักษร  
   * Date of birth (Required)  
     * เป็น Calendar ให้เลือกวันเดือนปี  
     * Default วันที่ปัจจุบัน  
   * Gender (Required)  
     * Female  
     * Male  
     * Other

3. กด “Save” → แจ้ง “Saved\!”

4. ระบบบันทึกข้อมูลใหม่และอัปเดต UI 

**Alternative / Exception (รวม Test Scenarios):**

* ผู้ใช้ไม่กรอกฟิลด์บังคับ ระบบไม่อนุญาตให้บันทึกและแจ้ง error  
  TC06-1: Missing required fields → อ้างอิง UC-01 (TC01-6) 

* ไฟล์รูปโปรไฟล์ไม่ถูกต้อง (ไม่ใช่ JPG/PNG หรือเกิน 5MB) ระบบบล็อก  
  TC06-2: Invalid image → ระบบแสดง "Only JPG/PNG up to 5MB are supported."  
  【Toast】

* ผู้ใช้กรอก Displayname ซ้ำ ระบบไม่อนุญาตให้ใช้ชื่อ  
  TC06-3: Duplicate displayname → อ้างอิง UC-01 (TC01-7) 

* Network fail ระหว่าง Save ระบบแจ้ง error และให้ retry  
  TC06-4: Save fail → ระบบแสดง "Save failed. Please try again."【Toast \+ Action: Retry】

* Session หมดอายุระหว่างกด Save ระบบ redirect ไป Login และคงค่าฟอร์ม  
  TC06-5: Session expired → ระบบแสดง "Session expired. Please log in again."  
   【Toast \+ Action: Sign In】

* ดับเบิลคลิกปุ่ม Save ระบบทำงานครั้งเดียว  
  TC06-6: Double submit → ระบบแสดง "Processing… Please wait."【Toast】

* ออฟไลน์ตอนกด Save ระบบไม่ส่งข้อมูล แต่เก็บค่าฟอร์มไว้  
  TC06-7: Offline save → ระบบแสดง "No internet connection. Please try again."【Toast】

* อัปโหลดรูปโปรไฟล์ล้มเหลว ระบบให้ retry เฉพาะรูป  
  TC06-8: Image upload fail → ระบบแสดง "Failed to upload profile picture. Please try again." 【Toast \+ Action: Retry】

* บันทึก audit log ไม่สำเร็จ ระบบถือว่าการบันทึกล้มเหลว  
  TC06-9: Audit log fail → ระบบแสดง "Failed to save audit log."【Toast】

**Business Rules:**

* Displayname ต้องไม่ว่าง และความยาวไม่เกิน 50 ตัวอักษร

* Date of birth ต้องอยู่ในรูปแบบวันที่ที่ถูกต้อง และต้องเป็นวันในอดีต

* Gender ต้องเลือกจากค่าที่ระบบกำหนดไว้เท่านั้น (เช่น Male/Female/Other)

* Profile picture ต้องเป็นไฟล์ JPG หรือ PNG และมีขนาดไม่เกิน 5MB

* หากฟิลด์ใดไม่ผ่านการตรวจสอบ (validation) ระบบต้องไม่อนุญาตให้บันทึก

* ทุกครั้งที่มีการแก้ไขข้อมูล ระบบต้องบันทึกการเปลี่ยนแปลงลง audit log (ระบุ user, ฟิลด์ที่เปลี่ยน, เวลา)

**Postconditions:**

* ข้อมูลโปรไฟล์ของผู้ใช้ถูกอัปเดตสำเร็จ

* ระบบแสดงค่าที่แก้ไขแล้วใน UI

* มีบันทึก audit log ของการแก้ไขเกิดขึ้น

---

### **UC-07 View Feed by Year Range**

**Actors:** Member

**Description:** ผู้ใช้ต้องการดูฟีดอัลบั้มและศิลปินตามช่วงปีที่เกี่ยวข้องกับอายุของตนเอง ระบบคำนวณช่วงปีจากวันเกิดของผู้ใช้ (DOB) หรือค่าช่วงอายุที่เลือก แล้วดึงผลลัพธ์มาแสดงในฟีด

**Preconditions:**

* ผู้ใช้ล็อกอินแล้ว

* ผู้ใช้มีวันเกิด (DOB) ในโปรไฟล์ หรือเลือก preset ของช่วงอายุ

**Trigger:** ผู้ใช้เข้าสู่หน้า Home (Age-based Feed)

**Main Flow:**

1. โหลดช่วงปีเริ่มต้น

2. ดึงผลลัพธ์ตามฟิลเตอร์

3. การแสดงผลในแต่ละแท็บ  
   * **Albums**    
     * รูปภาพปก album  
     * album name  
     * artist name  
     * จำนวนเพลงใน album  
     * ปุ่ม play  
     * ปีเปิดตัวของอัลบั้ม  
     * แนวเพลง  
     * ป้าย Master/Remaster (ถ้ามี)  
   * **Artists**    
     * รูปภาพโปรไฟล์  
     * artist name  
     * album ทั้งหมด  
     * ปุ่ม next  
     * ปีที่เดบิ้ว  
     * แนวเพลง

4. แสดงผลพร้อมจำนวนรวม การเรียงลำดับข้อมูลในแต่ละแท็บ  
   * **Albums** → เรียงลำดับจากอายุของ user ล่าสุดไปเก่าสุด, default แนวเพลงทั้งหมด  
   * **Artists** → เรียงลำดับจากอายุของ user ล่าสุดไปเก่าสุด, default แนวเพลงทั้งหมด

**Alternative / Exception (รวม Test Scenarios):**

* ไม่มีข้อมูลวันเกิด (DOB) ผู้ใช้ต้องเลือก preset ช่วงปีก่อน  
  TC07-1: No DOB → ระบบแสดง "Please select a year to continue."  
  【Dialog: เลือกปี \+ OK】

* ผลลัพธ์มีน้อยเกินไป ระบบแสดง banner แนะนำไปค้นหา (Search)  
  TC07-2: Few results → ระบบแสดง banner "Not enough results. Try searching for more content." พร้อมปุ่ม "Search" 【Banner \+ Action】

* ป้ายปี ที่แสดงไม่ตรงกับ metadata ระบบแจ้งเตือนว่าข้อมูลไม่ตรง  
  TC07-3: Metadata mismatch → ระบบแสดง "Year label does not match content metadata."【Toast】

**Business Rules:**

* ระบบต้องคำนวณปีจาก DOB \+ ช่วงอายุที่เลือก โดยสูตร: Year of birth \+ \[min\_age, max\_age\]

* ถ้าผลลัพธ์น้อยกว่าเกณฑ์ขั้นต่ำ (\< 10 รายการ) → ระบบขยายช่วงปีแบบสมมาตรทีละ ±1 จนถึงสูงสุด ±5 ปี

* เมื่อมีการขยายช่วงปี ระบบต้องแสดงป้าย Expanded \+N ให้ตรงกับจำนวนปีที่ขยายจริง

* ระบบต้องรองรับการกรอง (Filter) ตามแนวเพลง และค่าที่เลือกต้องถูกใช้ร่วมกับช่วงปี

* การเรียงลำดับต้องเป็นจากปีล่าสุดไปเก่าสุด

* ต้องแสดงเฉพาะอัลบั้ม/ศิลปินที่มีสถานะเป็น “public/official” ไม่รวม private

**Postconditions:**

* ฟีดถูกสร้างและแสดงผลตามปีและฟิลเตอร์ที่กำหนด

* ถ้ามีการขยายช่วงปี ระบบแสดงป้าย Expanded \+N ให้ผู้ใช้เห็น

* ผลลัพธ์สะท้อนข้อมูลล่าสุดของอัลบั้มและศิลปิน

---

### **UC-08 Adjust Year Range \+ Auto-Expand**

**Actors:** Member

**Description:** ผู้ใช้ปรับ “ช่วงปี” ของฟีดเพื่อให้ผลลัพธ์ตรงกับความสนใจมากขึ้น ระบบต้องคำนวณช่วงปีใหม่แบบเรียลไทม์ ใช้ฟิลเตอร์เดิมร่วมกัน และหากผลลัพธ์น้อยให้ขยายช่วงปีแบบสมมาตรทีละ ±1 ปี (สูงสุด ±5) พร้อมแสดงป้าย **Expanded \+N** ให้ผู้ใช้ทราบ

**Preconditions:**

* ผู้ใช้ล็อกอินแล้ว

* มีช่วงปีตั้งต้นจาก DOB หรือจากค่าที่ผู้ใช้เลือกไว้ (UC-07)

**Trigge:** ผู้ใช้เลื่อนสไลเดอร์/กดปุ่มปรับช่วงปีในหน้า Home (Age-based Feed)

**Main Flow:**

1. รับค่าช่วงใหม่

2. ระบบรับค่าช่วงใหม่และดึงผลลัพธ์

3. หากผลลัพธ์ \< เกณฑ์ขั้นต่ำ → ระบบขยายช่วงปีแบบสมมาตรทีละ ±1 ปี จนถึง ±5  
   * **Albums** → เรียงลำดับจากปีที่ค้นหาล่าสุดไปเก่าสุด, default แนวเพลงทั้งหมด  
   * **Artists** → เรียงลำดับจากปีที่ค้นหาล่าสุดไปเก่าสุด, default แนวเพลงทั้งหมด

4. รีเฟรชผลลัพธ์และ badge ตัวเลข แสดงใน **Albums/Artists**

5. ระบบแสดง badge Expanded \+N ให้ตรงกับจำนวนปีที่ขยายจริง

**Alternative / Exception (รวม Test Scenarios):**

* ผู้ใช้ขยายช่วงปีจนถึง ±5 แล้วยังมีผลลัพธ์น้อย ระบบแสดงหน้าว่างพร้อม CTA ไป Search  
  TC08-1: Expanded max but few results → อ้างอิง UC-07 (TC07-2)

* ป้าย Expanded \+N ต้องตรงกับจำนวนปีที่ขยายจริง  
  TC08-2: Badge mismatch → ระบบแสดง "Expanded range badge mismatch."  
  【Toast】

**Business Rules:**

* ระบบต้องคำนวณช่วงปีใหม่จากค่าที่ผู้ใช้ปรับ โดยอิง DOB หรือ preset ที่เลือกไว้

* ระบบต้องขยายปีแบบสมมาตรสูงสุด ±5 ปี

* หากผลลัพธ์น้อยกว่าเกณฑ์ → ระบบต้องแสดง Empty state \+ CTA ไป Search

* Badge Expanded \+N ต้องตรงกับการขยายจริง

**Postconditions:**

* ฟีดถูกอัปเดตตามช่วงปีใหม่และฟิลเตอร์ที่ใช้งาน

* หากมีการขยายปี ระบบแสดงป้าย Expanded \+N ให้ผู้ใช้เห็น

* หากยังไม่เพียงพอแม้จะขยายจนถึง ±5 → ระบบแสดง Empty state

---

### **UC-09 Apply Filters (Genre)**

**Actors:** Member

**Description:** ผู้ใช้เลือก/ปรับฟิลเตอร์แนวเพลง (Genre) เพื่อให้ฟีดแสดงผลลัพธ์ตรงกับความสนใจ ระบบต้องใช้ฟิลเตอร์ทำงานร่วมกับช่วงปีที่คำนวณได้ และบันทึกค่าฟิลเตอร์ล่าสุดไว้

**Preconditions:**

* ระบบคำนวณช่วงปีตั้งต้นแล้ว

* ผู้ใช้เข้าสู่ระบบแล้ว

**Trigger:** ผู้ใช้เลือก/เปลี่ยนค่าฟิลเตอร์ Genre บนหน้า Home (Age-based)

**Main Flow:**

1. ผู้ใช้เปิดหน้า Home (Age-based) และเห็นตัวกรองแนวเพลง**(Pop/Rock/Indie/Hip‑hop/R\&B/EDM)**

2. ผู้ใช้เลือกค่าหนึ่งหรือหลายค่า

3. ระบบใช้ฟิลเตอร์ร่วมกับช่วงปีที่คำนวณได้

4. ระบบรีเฟรชผลลัพธ์ และแสดง badge/ตัวนับสอดคล้องใน **Albums/Artists**

5. ระบบบันทึกค่าฟิลเตอร์ล่าสุด (profile/local pref)

**Alternative / Exception (รวม Test Scenarios):**

* เลือกฟิลเตอร์เข้มเกินไปจนไม่มีผลลัพธ์ ระบบแสดง empty state พร้อม CTA ไป Search  
  TC09-1: Over-filtered → อ้างอิง UC-07 (TC07-2)

* ผู้ใช้เปลี่ยนค่าฟิลเตอร์เร็วเกินไป ระบบ debounce ไม่ให้รีเฟรชถี่เกิน  
  TC09-2: Rapid filter change → ระบบแสดง "Filtering too frequent. Please wait."  
  【Toast】

* ครั้งแรกที่เข้าใช้งาน ฟิลเตอร์ default ต้องทำงานได้  
  TC09-3: First time → ระบบแสดงค่าทั้งหมด (ไม่มีข้อความแจ้งเตือน)

* ผลลัพธ์มีน้อย ระบบขยายปีอัตโนมัติแต่ยังใช้ฟิลเตอร์เดิม  
  TC09-4: Few results → ระบบแสดง "Expanded year range applied."【Toast】

* ผู้ใช้กลับมาใหม่ ระบบต้องจำฟิลเตอร์ล่าสุดที่เลือกไว้  
  TC09-5: Remember filter → filter คงค่าเดิม (ไม่มีข้อความแจ้งเตือน)

**Business Rules:**

* ระบบต้องใช้ฟิลเตอร์ Genre ร่วมกับช่วงปี

* หากผลลัพธ์น้อยกว่าเกณฑ์ → ต้องขยายช่วงปีอัตโนมัติ แต่ยังคงใช้ฟิลเตอร์เดิม

* ฟิลเตอร์ default ต้องถูกกำหนดไว้และทำงานได้ถูกต้องในการใช้งานครั้งแรก

* ระบบต้องบันทึกและเรียกคืนค่าฟิลเตอร์ล่าสุดที่ผู้ใช้เลือกไว้

* ค่าฟิลเตอร์ต้องรองรับการเปลี่ยนภาษา

**Postconditions:**

* ระบบแสดงผลที่ผ่านการกรองแล้ว และตรงตามช่วงปี \+ ฟิลเตอร์

* หากมีการขยายช่วงปี ระบบแสดงป้าย Expanded \+N ให้ผู้ใช้เห็น

* ผลลัพธ์สะท้อนข้อมูลล่าสุดของอัลบั้มและศิลปิน

---

### **UC-10 Unified Search**

**Actors:** Member

**Description:** ผู้ใช้ต้องการค้นหาศิลปินหรืออัลบั้ม ระบบต้องรองรับการค้นหาผ่านช่อง Search และแสดงผลลัพธ์รวมทุกประเภท (Artist / Album) โดยมีรูปแบบการแสดงผลที่ถูกต้องและสามารถกดเข้าไปดูรายละเอียดได้

**Preconditions:**

* ผู้ใช้ใส่คำค้นถูกต้อง

**Trigger:** ผู้ใช้พิมพ์คำค้นในช่องค้นหา

**Postconditions:** ระบบแสดงผลลัพธ์รวมทุกประเภท (**Artist/Album**) พร้อมการแสดงผลที่ถูกต้อง

**Main Flow:**

1. ผู้ใช้พิมพ์คำค้น

2. ระบบ debounce คำค้น (\~300ms) → ดึงผลลัพธ์จาก index

3. ระบบแสดงผลลัพธ์รวม **(Artist / Album )** พร้อมการแสดงผลตามประเภท  
* **Artist**    
  * รูปโปรไฟล์  
  * ชื่อศิลปินเดี่ยว/ศิลปินกลุ่ม  
  * icon verify profile  
* **Album**    
  * รูปปกอัลบั้ม  
  * ชื่ออัลบั้ม  
  * ป้าย Master/Remaster (ถ้ามี)  
  * ประเภท “Album”

4. ผู้ใช้เลือกผลลัพธ์ → เปิดหน้า **Artist profile, Album detail**   
* **Artist profile**   
  * รูปโปรไฟล์  
  * ชื่อนักดนตรี/วงดนตรี  
  * ชื่อจริง นามสกุล (ถ้าเป็นศิลปินเดี่ยว)  
  * สมาชิกวง (ถ้าเป็นศิลปินกลุ่ม)  
  * ปีเดบิวต์  
  * แนวเพลง  
  * tab album ของศิลปินคนนั้น  
* **Album detail**   
  * รูปปก album  
  * ชื่อเจ้าของ album  
  * ชื่อ album  
    ป้าย Master/Remaster (ถ้ามีสำหรับ artist album เท่านั้น)  
  * คำอธิบาย album  
  * ปีเปิดตัว album  
  * จำนวนเพลงทั้งหมดของ album  
  * แนวเพลงของ album

**Alternative / Exception (รวม Test Scenarios):**

* อัลบั้มมีหลายเวอร์ชัน ระบบต้องแสดง Master/Remaster แยกกัน  
  TC10-1: Album versions → ระบบแยกแสดงผลถูกต้อง (ไม่มีข้อความแจ้งเตือน)

* ไม่มีผลลัพธ์ ระบบแสดง empty state ปกติ  
  TC10-2: No results → ระบบแสดง "No search results found."  
  【Banner】

**Business Rules:**

* ระบบต้อง debounce การค้นหาเพื่อลดการเรียกซ้ำ (\~300ms)

* ผลลัพธ์ต้องถูกจัดประเภท (Artist / Album) อย่างถูกต้อง

* ศิลปิน/อัลบั้มที่แสดงต้องอยู่ในสถานะ public/official เท่านั้น

* หากมีหลายเวอร์ชัน ต้องแสดง badge (เช่น Master/Remaster, ปีเดบิวต์, แนวเพลง) เพื่อป้องกันความสับสน

**Postconditions:**

* ระบบแสดงผลการค้นหา Artist/Album ได้ถูกต้อง

* ผู้ใช้สามารถเลือกผลลัพธ์เพื่อไปยังหน้ารายละเอียดที่ถูกต้อง

---

### **UC-11 In-app Playback (Visible-only)**

**Actors:** Member

**Description:** ผู้ใช้แตะเล่นวิดีโอจากผลลัพธ์หรือเพลย์ลิสต์ ระบบต้องเล่นวิดีโอได้เฉพาะเมื่ออยู่ใน viewport ของหน้าจอ และหยุดเล่นทันทีเมื่อผู้ใช้ย่อหน้าจอ, สลับไปแอปอื่น หรือหน้าจอถูกล็อก

**Preconditions:**

* วิดีโอพร้อมเล่นใน region

**Trigger:** ผู้ใช้แตะปุ่ม Play จากผลลัพธ์หรือเพลย์ลิสต์

**Main Flow:**

1. เปิด Player

2. ระบบตรวจสอบ visibility ของ Player อย่างต่อเนื่อง

3. หาก Player อยู่ใน viewport → เล่นวิดีโอ

4. หาก Player หลุดออกจาก viewport (เลื่อนจอ) → หยุดเล่นทันที

5. หาก Player กลับเข้ามาใน viewport → เล่นต่อ (resume)

**Alternative / Exception (รวม Test Scenarios):**

* วิดีโอถูกบล็อกตาม region ระบบแสดง error และมีปุ่ม Open in YouTube  
  TC11-1: Region blocked → ระบบแสดง "This video isn’t available in your region."  
  【Toast】

* ผู้ใช้เลื่อนหน้าจอจน Player หลุดจากสายตา ระบบหยุดเล่นทันที  
  TC11-2: Player out of view → ระบบต้อง หยุดเล่นทันที (ไม่มีข้อความแจ้งเตือน)

* ผู้ใช้เลื่อนกลับมาแล้ว Player อยู่ในสายตาอีกครั้ง ระบบเล่นต่ออัตโนมัติ  
   TC11-3: Player back in view → ระบบเล่นต่ออัตโนมัติ (ไม่มีข้อความแจ้งเตือน)

* ผู้ใช้สลับไปแอปอื่นหรือกดล็อกหน้าจอ ระบบหยุดเล่นทันที  
   TC11-4: App minimized/lock screen → ระบบหยุดเล่นทันที (ไม่มีข้อความแจ้งเตือน)

**Business Rules:**

* วิดีโอในแอปเล่นได้เฉพาะเมื่อ Player มองเห็นใน viewport

* วิดีโอต้องหยุดเล่นทันทีเมื่อ Player หายไปจากสายตา หรือเมื่อผู้ใช้สลับ/ล็อกเครื่อง

* หากวิดีโอถูกบล็อกใน region ต้องมีปุ่ม Open in YouTube ให้ผู้ใช้

**Postconditions:**

* วิดีโอเล่นได้เฉพาะเมื่อ Player อยู่ในหน้าจอ

* วิดีโอหยุดเล่นทันทีเมื่อ Player พ้นหน้าจอ, ย่อ, สลับแอป หรือจอล็อก

---

### **UC-12 Open in YouTube**

**Actors:** Member

**Description:** ผู้ใช้กดปุ่ม “Open in YouTube” เพื่อเปิดวิดีโอจากระบบ Re:Play บน YouTube โดยตรง ระบบต้องสามารถเปิดผ่านแอป YouTube ได้ ถ้าไม่มีแอปต้องเปิดผ่าน browser

**Preconditions:**

* มีปุ่ม Open in YouTube แสดงอยู่

**Trigger:** ผู้ใช้กดปุ่ม “Open in YouTube”

**Main Flow:**

1. กดปุ่ม Open in YouTube

2. ระบบเปิดลิงก์บน YouTube app หรือ browser

**Alternative / Exception (รวม Test Scenarios):**

* อุปกรณ์ไม่มีแอป YouTube ระบบเปิดผ่าน browser แทน  
  TC12-1: No YouTube app → เปิดผ่าน browser (ไม่มีข้อความแจ้งเตือน)

* เบราว์เซอร์บล็อก popup ระบบใช้ user gesture เปิดลิงก์  
   TC12-2: Popup blocked → ระบบแสดง "Popup blocked. Please tap the button again."  
   【Toast】

**Business Rules:**

* ปุ่ม Open in YouTube ต้องเปิดลิงก์วิดีโอด้วยวิธีที่รองรับอุปกรณ์ผู้ใช้เสมอ

* ถ้ามีแอป YouTube ให้เปิดผ่านแอปก่อนเป็นลำดับแรก

* ต้อง fallback ไป browser ได้ หากไม่มีแอป YouTube หรือตัวแอปไม่รองรับ

**Postconditions:**

* วิดีโอถูกเปิดใน YouTube (ผ่านแอปหรือ browser) สำเร็จ

---

### **UC-13 Save/Unsave Album**

**Actors:** Member

**Description:** ผู้ใช้กด Save หรือ Unsave บนอัลบั้ม เพื่อบันทึกหรือเอาออกจาก My Library ระบบต้องอัปเดตสถานะทั้งใน UI และ sync กับ server แบบเรียลไทม์

**Preconditions:**

* ผู้ใช้ล็อกอินแล้ว

**Trigger:** ผู้ใช้แตะปุ่ม Save/Unsave บนหน้า Album Detail

**Main Flow:**

1. ผู้ใช้แตะปุ่ม Save

2. ระบบแสดงสถานะ Saved แบบ optimistic

3. ระบบ sync กับ server

4. เมื่อ sync สำเร็จ → สถานะ Saved ถูกยืนยันและอัลบั้มปรากฏใน My Library

5. หากผู้ใช้กด Unsave → ระบบแสดงสถานะ Unsave และอัลบั้มหายไปจาก My Library

**Alternative / Exception (รวม Test Scenarios):**

* เกิดปัญหาเครือข่ายระหว่างกด Save ระบบ rollback และแจ้ง error  
  TC13-1: Save failed (network) → ระบบแสดง "Save failed. Please try again."  
  【Toast \+ Action: Retry】

* ผู้ใช้กด Save/Unsave ถี่เกินไป ระบบ debounce ป้องกันการกดซ้ำ  
  TC13-2: Too frequent → ระบบแสดง "Action too frequent. Please wait."【Toast】

* เมื่อ Save สำเร็จ อัลบั้มจะปรากฏใน Library / เมื่อ Unsave สำเร็จ อัลบั้มจะถูกลบออก  
  TC13-3: Save/Unsave success → ระบบแสดง "Library updated."【Toast】

**Business Rules:**

* สถานะ Save/Unsave ต้อง sync กับ server เสมอ

* การกด Save/Unsave ต้องรองรับ optimistic UI (สะท้อนผลก่อน sync)

* ระบบต้องป้องกันการกดซ้ำเกินไปด้วย debounce

* หากการ sync ล้มเหลว ต้อง rollback สถานะให้ถูกต้อง

**Postconditions:**

* สถานะอัลบั้มตรงกับการกระทำของผู้ใช้ (Save หรือ Unsave)

* My Library ของผู้ใช้ถูกอัปเดตให้ตรงกับสถานะล่าสุด

---

### **UC-14 Create Artist Profile (Admin)** {#uc-14-create-artist-profile-(admin)}

**Actors:** Admin

**Description:** Admin ต้องการสร้างโปรไฟล์ศิลปินใหม่ในระบบ เพื่อให้สามารถจัดการอัลบั้มและแสดงผลในหน้า public ได้ครบถ้วน

**Preconditions:**

* ต้องตรวจสอบว่าไม่มีศิลปินซ้ำ (ชื่อ \+ ปีเดบิวต์)

**Trigger:** Admin กด “Create artist profile”

**Main Flow:**

1. Settings → User Management → Artists → เลือก “Create artist profile”  
   * หน้า Artists ให้แสดงจำนวนของ Artists ทั้งหมดในระบบ  
   * เงื่อนไขการแสดงตัวเลขแสดงตัวเลขเต็มจำนวนเลย

2. กรอกฟอร์ม ประกอบไปด้วยฟิลด์ดังนี้  
* Step 1 → เลือกประเภทของศิลปินเดี่ยว/ศิลปินกลุ่ม (Singer/Band)  
* Step 2 → Displayname ชื่อศิลปินเดี่ยว/ศิลปินกลุ่ม (Required)  
  * ห้ามซ้ำกับ Displayname ของ artist ประเภทนั้นๆ ในระบบ → แสดง icon verify  
  * กรอกได้ไม่เกิน 30 ตัวอักษร  
* Step 3 →Debut year (Required)  
  * เป็นตัวเลือกปี   
  * ถ้าชื่อ \+ Debut year ของ artist ประเภทนั้นๆ ในระบบ → ไปต่อใน step ต่อไป  
* Step 4 → ฟิลด์ต่างๆดังนี้  
  * Profile picture (Required)  
    * รองรับไฟล์: JPG, PNG  
    * ขนาดไม่เกิน 5MB  
    * ห้ามมีภาพโป๊เปลือย, ความรุนแรง, หรือสัญลักษณ์ไม่เหมาะสม (ระบบไม่สามารถตรวจสอบได้แต่ให้เป็นเงื่อนไขในอนาคต)  
  * Displayname ชื่อศิลปินเดี่ยว/ศิลปินกลุ่ม (Required) \- Disable   
  * First name (Required) \- ถ้าเลือก Singer  
    * กรอกได้ไม่เกิน 50 ตัวอักษร  
  * Last name (Required) \- ถ้าเลือก Singer  
    * กรอกได้ไม่เกิน 50 ตัวอักษร  
  * Member of the band (Required) \- ถ้าเลือก Band  
    * กรอกได้ไม่เกิน 150 ตัวอักษร  
  * Debut year (Required) \- Disable  
  * About (Required)  
    * กรอกได้ไม่เกิน 150 ตัวอักษร  
  * Genres แนวเพลง เลือกได้แค่ 1 ตัวเลือก (Required)  
    * **Pop/Rock/Indie/Hip‑hop/R\&B/EDM**

3. Validate → บันทึก แจ้ง “Saved\!”

**Alternative / Exception (รวม Test Scenarios):**

* ผู้ใช้กรอก Displayname ซ้ำ ของศิลปินภายใต้ประเภทเดียวกัน ระบบไม่อนุญาตให้ใช้ชื่อ  
  TC14-1: Duplicate displayname artist type → อ้างอิง UC-01 (TC01-7) 

* ผู้ใช้ไม่กรอกฟิลด์บังคับ ระบบไม่อนุญาตให้บันทึกและแจ้ง error  
  TC14-2: Missing required fields → อ้างอิง UC-01 (TC01-6) 

* ไฟล์รูปโปรไฟล์ไม่ถูกต้อง (ไม่ใช่ JPG/PNG หรือเกิน 5MB) ระบบบล็อก  
  TC14-3: Invalid image → อ้างอิง UC-06 (TC06-2) 

* Network fail ระหว่าง Save ระบบแจ้ง error และให้ retry  
  TC14-4: Save fail → อ้างอิง UC-06 (TC06-4)

* Session หมดอายุระหว่างกด Save ระบบ redirect ไป Login และคงค่าฟอร์ม  
  TC14-5: Session expired →  อ้างอิง UC-06 (TC06-5)

* ดับเบิลคลิกปุ่ม Save ระบบทำงานครั้งเดียว  
  TC14-6: Double submit → อ้างอิง UC-06 (TC06-6)

* ออฟไลน์ตอนกด Save ระบบไม่ส่งข้อมูล แต่เก็บค่าฟอร์มไว้  
  TC14-7: Offline save → อ้างอิง UC-06 (TC06-7)

* อัปโหลดรูปโปรไฟล์ล้มเหลว ระบบให้ retry เฉพาะรูป  
  TC14-8: Image upload fail → อ้างอิง UC-06 (TC06-8)

* บันทึก audit log ไม่สำเร็จ ระบบถือว่าการบันทึกล้มเหลว  
  TC14-9: Audit log fail → อ้างอิง UC-06 (TC06-9)

* Admin 2 คนสร้างศิลปินซ้ำกัน ระบบอนุญาตเพียง 1 record  
  TC14-10: Concurrency → ระบบแสดง "Artist already exists." 【Toast】

**Business Rules:**

* ต้องตรวจ uniqueness ของ Displayname \+ Debut year

* Profile picture ต้องเป็น JPG/PNG และ ≤ 5MB

* ทุกฟิลด์ Required ต้องผ่าน validation ก่อนบันทึก

* Audit log ต้องถูกบันทึกเสมอ (หาก log fail ถือว่า fail ทั้ง use case)

* ระบบต้องเป็น idempotent (กด Save ซ้ำ/พร้อมกันไม่สร้างซ้ำ)

**Postconditions:**

* Artist profile ถูกสร้างเรียบร้อย

* แสดงผลในหน้า public ได้ตามข้อมูลที่บันทึก

* Audit log ถูกบันทึกครบถ้วน

---

### **UC-15 Edit Artist Profile (Admin)**

**Actors:** Admin

**Description:** Admin ต้องการแก้ไขข้อมูลโปรไฟล์ศิลปินที่สร้างไว้แล้ว เช่น ชื่อ, รูปโปรไฟล์, ปีเดบิวต์, หรือแนวเพลง โดยหลังจากการแก้ไข ข้อมูลต้องสะท้อนในหน้าสาธารณะ และระบบต้องเก็บ Activity log ทุกครั้งเพื่อการตรวจสอบย้อนหลัง

**Preconditions:**

* ศิลปินต้องมีโปรไฟล์อยู่แล้วในระบบ

**Trigger:** Admin กด “Edit profile”

**Main Flow:**

1. Settings → User Management → Artists → เลือก profile ที่ต้องการแก้ไข   
   * หรือ Admin เลือกศิลปิน → เข้าสู่มุมมอง owner artist profile จุด 3 จุด (ปุ่มนี้แสดงเฉพาะ Admin)

2. แก้ไขข้อมูลและเช็คเงื่อนไขอ้างอิง [**UC-14 Create Artist Profile (Admin)**](#uc-14-create-artist-profile-\(admin\))  
   แต่ไม่ต้อง Diable ฟิลด์ใดเลย และไม่ต้องแสดงฟิลด์ Artist type

3. ระบบแสดง Activity log (เรียงจากล่าสุดไปเก่าสุด) เช่น  
   * Create new artist profile  
   * เปลี่ยนแปลงข้อมูล → แสดงชื่อฟิลด์, ค่าเดิม, ค่าใหม่ (เช่น About: "old" \=\> "new")  
   * แสดงข้อมูลของคนที่แก้ไขข้อมูล   
     * profile picture   
     * Displayname  
     * Firstname Lastname  
     * Data   
     * แสดงวันที่และเวลาที่แก้ไขข้อมูล

4. กด “Save” แจ้ง “Saved\!”

5. ระบบบันทึกข้อมูลที่แก้ไขแล้ว และสร้าง Activity log

**Alternative / Exception (รวม Test Scenarios):**

*  อ้างอิงทั้งหมดจาก [**UC-14 Create Artist Profile (Admin)**](#uc-14-create-artist-profile-\(admin\))

**Business Rules:**

* ใช้ validation และ business rules เดียวกับ UC-14 (Create Artist Profile)

* ต้องบันทึก Activity log ทุกครั้งที่มีการแก้ไข (actor, ฟิลด์ที่แก้, ค่าเดิม/ใหม่, timestamp)

* Activity log ต้องสามารถตรวจสอบย้อนหลังได้

**Postconditions:**

* Artist profile ถูกอัปเดตสำเร็จ

* ข้อมูลใหม่สะท้อนในหน้า public

* Activity log ถูกบันทึกครบถ้วน

---

### **UC-16 Delete Artist Profile (Admin)**

**Actors:** Admin

**Description:** Admin ต้องการลบโปรไฟล์ศิลปินออกจากระบบ โดยหลังจากลบแล้ว ศิลปินจะไม่ปรากฏในหน้า public และไม่สามารถค้นหาได้อีก ระบบต้องบันทึก audit log ของการลบ

**Preconditions:**

* ศิลปินต้องมีโปรไฟล์อยู่แล้ว

* Admin มีสิทธิ์จัดการ

**Trigger:** Admin เปิดหน้า Artist Profile → เลือก “Delete profile”

**Main Flow:**

1. Settings → User Management → Artists → เลือก profile ที่ต้องการแก้ไข   
   * หรือเลือก Artist profile ที่ต้องการลบ (หรือเปิดจากหน้า owner artist profile)   
     จากจุด 3 จุด (ปุ่มนี้แสดงเฉพาะ Admin)

2. กด “Delete profile”

3. ระบบถามยืนยัน (confirm dialog)  
   * แสดง profile picture, displayname  
   * Reason (ต้องกรอก)

4. เมื่อยืนยัน → อัปเดตสถานะเป็น deleted แจ้ง “Deleted\!”

5. บันทึก audit log

**Alternative / Exception (รวม Test Scenarios):**

* Admin กดยกเลิกการยืนยัน  
  TC16-1: Cancel confirm → กด “Cancel” ระบบไม่ลบ โปรไฟล์ศิลปินยังคงอยู่ (ไม่แสดงข้อความแจ้งเตือน)

* พยายามลบศิลปินที่ถูก archived แล้ว  
  TC16-2: Already archived → ระบบแสดง "Artist profile already archived. Cannot delete again."【Toast】

* ไม่กรอก Reason (ฟิลด์บังคับ)  
  TC16-3: Reason empty → ระบบแสดง "Please provide a reason."【Inline Validation】

* Network fail ตอน Delete  
  TC16-4: Network fail → ระบบแสดง "Delete failed. Please try again."  
  【Toast \+ Action: Retry】

* Admin 2 คนลบ profile เดียวกันพร้อมกัน  
  TC16-5: Concurrency → ระบบแสดง "Profile already deleted by another admin. Please refresh."【Toast \+ Action: Reload】

* Admin กดปุ่ม Delete ซ้ำ (idempotent)  
  TC16-6: Double-submit Delete → ระบบแสดง "Already deleted." และบันทึก log เพียงครั้งเดียว【Toast】

**Business Rules:**

* การลบต้องเปลี่ยนสถานะเป็น deleted (soft delete) ไม่ใช่ลบ record ทิ้งทันที

* หากศิลปินมี content เชื่อมโยง ต้องเตือนผลกระทบก่อนยืนยันซ้ำ

* Reason เป็นฟิลด์บังคับในการลบ

* ต้องบันทึก audit log ทุกครั้งที่ลบ

* การลบต้องเป็น idempotent

**Postconditions:**

* โปรไฟล์ศิลปินถูกเปลี่ยนสถานะเป็น deleted

* ศิลปินไม่ปรากฏในหน้า public หรือ Search

* Audit log ของการลบถูกบันทึกเรียบร้อย

---

### **UC-17 Create Official Album (Admin)** {#uc-17-create-official-album-(admin)}

**Actors:** Admin

**Description:** Admin ต้องการสร้างอัลบั้มทางการ (Official Album) ผูกกับศิลปินในระบบ โดยอัลบั้มต้องมีข้อมูลครบถ้วน เช่น ปก ชื่อ ปีเปิดตัว จำนวนเพลง แนวเพลง และสถานะ Master/Remaster

**Preconditions:**

* เลือก Artist profile ได้

* Admin มีสิทธิ์จัดการ

**Trigger:** Admin เลือก “Add new album”

**Main Flow:**

1. Admin เลือกศิลปิน → เข้าสู่มุมมอง owner artist profile 

2. Admin เลือก tab Album → เลือก “**Add new album**” → แสดงหน้าเพิ่ม album 

3. กรอกข้อมูล   
* Step 1: → กรอก URL ระบบดึง metadata  
  * URL ของเพลย์ลิสต์ YouTube จากนั้นระบบดึง metadata มาและสามารถแก้ไขได้ (รูปปก, ชื่อ, ปีเปิดตัว) →  กด “Next”  
* Step 2 → กรอกรายละเอียด  
  * Cover picture (Required)  
    * รองรับไฟล์: JPG, PNG  
    * ขนาดไม่เกิน 5MB  
    * ห้ามมีภาพโป๊เปลือย, ความรุนแรง, หรือสัญลักษณ์ไม่เหมาะสม (ระบบไม่สามารถตรวจสอบได้แต่ให้เป็นเงื่อนไขในอนาคต)  
  * Album name (Required)  
    * กรอกได้ไม่เกิน 30 ตัวอักษร (ให้กันไว้ที่หน้าจอไม่ให้กรอกต่อ)  
  * About album (Required)  
    * กรอกได้ไม่เกิน 150 ตัวอักษร (ให้กันไว้ที่หน้าจอไม่ให้กรอกต่อ)  
  * Select release years ปีเปิดตัว (Required)  
    * ตัวเลือกปี เป็น dropdown  
  * Item songs จำนวนเพลงใน album (Required field)  
    * กรอกได้แค่ตัวเลขตั้งแต่ 1-9999 (ให้กันไว้ที่หน้าจอไม่ให้กรอกต่อ)  
  * Genres แนวเพลง (Required)  
    * ตัวเลือกแนวเพลง  
  * Master/Remaster badge เลือกว่าจะแสดงคำว่า Master หรือ Remaster บน album ไหม  
    * Master  
    * Remaster  
    * Hide → Default

4. Admin กด “Save” → ระบบ Validate และสร้างอัลบั้ม แจ้ง “Saved\!”

**Alternative / Exception (รวม Test Scenarios):**

* ผู้ใช้ไม่กรอกฟิลด์บังคับ ระบบไม่อนุญาตให้บันทึกและแจ้ง error  
  TC17-1: Missing required fields → อ้างอิง UC-01 (TC01-6) 

* ไฟล์รูปโปรไฟล์ไม่ถูกต้อง (ไม่ใช่ JPG/PNG หรือเกิน 5MB) ระบบบล็อก  
  TC17-2: Invalid image → อ้างอิง UC-06 (TC06-2) 

* Network fail ระหว่าง Save ระบบแจ้ง error และให้ retry  
  TC17-3: Save fail → อ้างอิง UC-06 (TC06-4)

* Session หมดอายุระหว่างกด Save ระบบ redirect ไป Login และคงค่าฟอร์ม  
  TC17-4: Session expired →  อ้างอิง UC-06 (TC06-5)

* ดับเบิลคลิกปุ่ม Save ระบบทำงานครั้งเดียว  
  TC17-5: Double submit → อ้างอิง UC-06 (TC06-6)

* ออฟไลน์ตอนกด Save ระบบไม่ส่งข้อมูล แต่เก็บค่าฟอร์มไว้  
  TC17-6: Offline save → อ้างอิง UC-06 (TC06-7)

* อัปโหลดรูปโปรไฟล์ล้มเหลว ระบบให้ retry เฉพาะรูป  
  TC17-7: Image upload fail → อ้างอิง UC-06 (TC06-8)

* บันทึก audit log ไม่สำเร็จ ระบบถือว่าการบันทึกล้มเหลว  
  TC17-8: Audit log fail → อ้างอิง UC-06 (TC06-9)

**Business Rules:**

* ต้องตรวจ uniqueness ของ URL

* อัลบั้มต้องมีเพลงอย่างน้อย 1 เพลง

* Album name ≤ 30 ตัวอักษร, About ≤ 150 ตัวอักษร

* Cover picture ต้องเป็น JPG/PNG ≤ 5MB

* Release year ต้องสมเหตุสมผล

* Genres เลือกได้เพียง 1

* ต้องบันทึก audit log ทุกครั้งที่สร้าง

**Postconditions:**

* Official Album ถูกสร้างสำเร็จ

* ผูกกับ Artist profile ที่เลือก

* แสดงผลในหน้า public พร้อมป้าย Master/Remaster ที่ถูกต้อง

---

### **UC-18 Edit Official Album (Admin)**

**Actors:** Admin

**Description:** Admin ต้องการแก้ไขข้อมูลอัลบั้มทางการ (Official Album) ที่มีอยู่แล้ว เช่น ปก ชื่อ ปีเปิดตัว จำนวนเพลง แนวเพลง หรือสถานะ Master/Remaster โดยเมื่อแก้ไขแล้ว ข้อมูลต้องสะท้อนบนหน้า public และบันทึก Activity log ทุกครั้ง

**Preconditions:**

* Official Album ต้องมีอยู่แล้วในระบบ

**Trigger:** Admin เปิด Official Album → เลือก “Edit”

**Main Flow:**

1. Admin เลือกศิลปิน → เข้าสู่ owner artist profile

2. เปิดแท็บ Albums → เลือก Album ที่ต้องการแก้ไข

3. กดจุด 3 จุดที่ Album → เลือก “Edit album” (ปุ่มนี้แสดงเฉพาะ Admin)

4. ระบบแสดงหน้าแก้ไข Album พร้อมข้อมูลเดิม

5. แสดงข้อมูลและเงื่อนไขการแก้ไขอ้างอิง [**UC-17 Create Official Album (Admin)**](#uc-17-create-official-album-\(admin\))

6. แสดง Activity log → เฉพาะในมุมมองแอดมินเท่านั้น (หน้า Edit ไม่ใช่หน้า View album) 

7. ระบบแสดง Activity log (เรียงจากล่าสุดไปเก่าสุด) เช่น  
   * Create new artist profile  
   * เปลี่ยนแปลงข้อมูล → แสดงชื่อฟิลด์, ค่าเดิม, ค่าใหม่ (เช่น About: "old" \=\> "new")  
   * แสดงข้อมูลของคนที่แก้ไขข้อมูล   
     * profile picture   
     * Displayname  
     * Firstname Lastname  
     * Data   
     * แสดงวันที่และเวลาที่แก้ไขข้อมูล

8. Admin กด Save → ระบบ Validate → บันทึกสำเร็จ แจ้ง “Saved\!”

9. ระบบอัปเดต Album และ Activity log

**Alternative / Exception (รวม Test Scenarios):**

* อ้างอิง [**UC-17 Create Official Album (Admin)**](#uc-17-create-official-album-\(admin\)) 

* Admin 2 คนแก้อัลบั้มเดียวกันพร้อมกัน  
  TC18-1: Concurrency → ระบบแสดง "Album already updated by another admin. Please reload."【Toast \+ Action: Reload】

* ไม่บันทึก Activity log ครบถ้วน  
  TC18-2: Log fail → ระบบแสดง "Failed to save activity log."【Toast】

* ผู้ใช้ที่ไม่ใช่ Admin เห็นปุ่ม Edit  
  TC18-3: Unauthorized → ระบบแสดง "You are not authorized to edit this album."  
  【Toast】

**Business Rules:**

* ใช้ validation และ business rules เดียวกับ UC-17

* Activity log ต้องถูกบันทึกครบ (actor, ฟิลด์ที่เปลี่ยน, ค่าเดิม/ใหม่, เวลา)

* ระบบต้องเป็น idempotent (ป้องกัน save ซ้ำ)

* ปุ่ม Edit แสดงเฉพาะ Admin เท่านั้น

**Postconditions:**

* ข้อมูล Official Album ถูกอัปเดตสำเร็จ

* การเปลี่ยนแปลงสะท้อนในหน้า public

* Activity log ถูกบันทึกครบถ้วน

---

### **UC-19 Delete Official Album (Admin)**

**Actors:** Admin

**Description:** Admin ต้องการลบ Official Album ออกจากระบบ โดยเมื่อทำการลบแล้ว อัลบั้มจะถูกซ่อนจากหน้า public และ Search แต่ข้อมูลยังคงเก็บไว้ในระบบด้วยสถานะ removed เพื่อรองรับ audit log และตรวจสอบย้อนหลัง

**Preconditions:**

* Official Album ต้องมีอยู่แล้วในระบบ

**Trigger:** Admin กด “Delete album”

**Main Flow:**

1. Admin เลือกศิลปิน → เข้าสู่ owner artist profile

2. เปิดแท็บ Albums → เลือก Album ที่ต้องการลบ

3. กดจุด 3 จุด → เลือก “Delete album” (ปุ่มนี้แสดงเฉพาะ Admin)

4. ระบบถามยืนยัน (confirm dialog)  
   * แสดงรูปปก, ชื่อ album, จำนวนเพลง, ป้าย Master/Remaster (ถ้ามี)  
   * Reason (ต้องกรอก)

5. กด “Delete” → Delete album done  แจ้ง “Deleted\!”

6. ระบบเปลี่ยนสถานะ Album เป็น removed

7. ระบบบันทึก audit log

**Alternative / Exception (รวม Test Scenarios):**

* Admin กดยกเลิกที่ dialog ยืนยัน  
  TC19-1: Cancel confirm → กด “Cancel” ระบบไม่ลบ album ยังคงอยู่ (ไม่แสดงข้อความแจ้งเตือน)

* ไม่กรอก Reason (ฟิลด์บังคับ)  
  TC19-2: Reason empty →  อ้างอิง UC-16 (TC16-3) 

* Network fail ตอน Delete  
  TC19-3: Network fail → อ้างอิง UC-16 (TC16-4)

* Admin 2 คนลบอัลบั้มเดียวกันพร้อมกัน  
  TC19-4: Concurrency → ระบบแสดง "Album already deleted by another admin. Please refresh." 【Toast \+ Action: Reload】

* Admin กดปุ่ม Delete ซ้ำ (idempotent)  
  TC19-5: Double-submit Delete →อ้างอิง UC-16 (TC16-6)

**Business Rules:**

* การลบต้องเป็น soft delete → เปลี่ยนสถานะเป็น removed เท่านั้น

* Reason เป็นฟิลด์บังคับในการลบ

* ต้องบันทึก audit log ทุกครั้งที่ลบ

* การลบต้องเป็น idempotent

* ปุ่ม Delete แสดงเฉพาะ Admin

**Postconditions:**

* Official Album ถูกเปลี่ยนสถานะเป็น removed

* ไม่ปรากฏใน Search หรือหน้า Artist profile/public

* Audit log การลบถูกบันทึกเรียบร้อย

