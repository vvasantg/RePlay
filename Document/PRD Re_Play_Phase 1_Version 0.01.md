# **Product Requirements Document: Re:Play** 

**Project:** Re:Play  
**Phase 1 (MVP)**  
**Update:** 2025‑09‑10  
**Version:** 0.01  
**Scope:** Mobile‑only (iOS/Android), แอปเดียวทุกบทบาท (role‑based UI)

---

## **1\. Introduction / Overview**

เอกสารนี้กำหนดความต้องการของ Re:Play แอปมือถือสำหรับคนรักเพลงที่อยาก “กลับไปฟังใหม่ เพื่อกลับไปรู้สึกอีกครั้ง” – *Replay the Music. Relive the Moment.* ระบบรวม **อัลบั้มจาก YouTube/YouTube Music** และคัดสรรตามช่วงอายุของผู้ใช้ (คำนวณจากปีเกิด \+ ช่วงอายุที่เลือก)

**โฟกัสของ MVP:**

* ค้นหา (Unified Search)  
* ฟีดอัลบั้มตามช่วงอายุ (Age-based Album Discovery)  
* เล่นเพลงในแอปตามนโยบาย YouTube (Visible-only Playback \+ ปุ่ม “Open in YouTube”)  
* Member: Save/Unsave Album  
* Admin: CRUD ศิลปินและ Official Albums

**ข้อปฏิบัติตามนโยบาย:** ฝังตัวเล่นของ YouTube (IFrame Player) วิดีโอต้องมองเห็นระหว่างเล่น และ **หยุดเล่นเมื่อแอปถูกย่อ/จอล็อก** พร้อมปุ่ม **“Open in YouTube”** เป็นทางเลือก ผู้ให้บริการไม่โฮสต์ไฟล์เพลง/วิดีโอเอง

---

## **2\. Goals / Objectives**

**เป้าหมายหลัก (MVP):** เวลาฟังรวม (Total Replay Minutes) ≥ 30,000 นาที/สัปดาห์ (เฉพาะการฟังอัลบั้ม) ภายใน 4 สัปดาห์หลังเปิดตัว (ประเทศไทย, Asia/Bangkok)

**เป้าหมายรอง:**

* UX เรียบง่ายและเข้าถึงง่าย  
* Member → Save Album ได้ทันที  
* Discovery ผ่าน Feed ตามช่วงอายุ \+ ฟิลเตอร์แนวเพลง  
* Admin → curate Official Albums และจัดการศิลปิน

---

## **3\. Target Audience / User Personas**

**Visitor**: สำรวจบางหน้า กระตุ้นให้สมัคร/ล็อกอิน

**Member**: ฟังอัลบั้มย้อนหลัง, ใช้ฟิลเตอร์, Save Album

**Admin**: สร้าง/แก้ไข/ลบศิลปิน, จัดการ Official Albums

Note: **ทุกบทบาทใช้แอปเดียวกัน** โดย UI/เมนูแสดงตามสิทธิ์จากเซิร์ฟเวอร์

---

## **4\. Functional Requirements**

### **FR1: User Management**

* **FR1.1** ระบบรองรับสมัคร/เข้าสู่ระบบด้วย Google  
* **FR1.2** ผู้ใช้สามารถแก้ไขข้อมูลโปรไฟล์พื้นฐานได้  
  * Profile picture  
  * Displayname  
  * First name  
  * Last name  
  * Date of birth  
  * Gender  
* **FR1.3** ระบบต้องแสดง Privacy Policy และ Terms of Service (ภาษาไทย) และบันทึกการยินยอมตาม PDPA  
* **FR1.4** ผู้ใช้สามารถลบบัญชีตนเองได้ (Immediate Deletion)

### **FR2: Catalog & Metadata (YouTube‑first)**

* **FR2.1** ปีที่เปิดตัวอ้างอิงจาก **YouTube/YouTube Music** เป็นหลัก หากไม่ครบ/ไม่ชัดเจน ให้ fallback เป็น **MusicBrainz/Discogs** หรือ **บันทึกด้วยมือ**   
* **FR2.2** แยก **Master** และ **Remaster** เป็น **คนละอัลบั้ม** โดยปีอิงตามเวอร์ชันนั้น  
* **FR2.3** วิดีโอ **Full‑album เดี่ยว** สามารถนับเป็นอัลบั้มได้ และสามารถเก็บ tracklist ภายในระบบเพื่อความสะดวก  
* **FR2.4** แอดมินสามารถ **สร้าง / แก้ไข / ลบ** โปรไฟล์ศิลปินจริงได้ โดยมีฟิลด์:  
  * Artist type (Singer/Band)  
  * Profile picture  
  * Displayname   
  * Debut year  
  * First name (Singer)  
  * Last name (Singer)  
  * Member of the band (Band)  
  * About   
  * Genres

### **FR3: Age-based Album Discovery**

* **FR3.1** สูตรคำนวณช่วงปี: ปีเกิด \+ \[อายุต่ำสุด, อายุสูงสุด\]  
* **FR3.2** หากคอนเทนต์ไม่พอ ให้ขยายช่วงปีแบบสมมาตรครั้งละ **±1 ปี** สูงสุด **±5 ปี** และแสดงป้าย **“Expanded \+N”** ให้ผู้ใช้เห็น  
* **FR3.3** ฟิลเตอร์: **แนวเพลง (Pop/Rock/Indie/Hip‑hop/R\&B/EDM)**  
* **FR3.4** แสดงผลรายการ **Albums/Artists** ที่ตรงกับช่วงปีสุดท้ายและฟิลเตอร์

### **FR4: Unified Search**

* **FR4.1** ค้นหาแบบ **Unified Search (Artist/Album)** ในช่องเดียว พร้อม badge ประเภท  
* **FR4.2** แสดงปีเปิดตัวและป้าย **master/remaster** และรองรับฟิลเตอร์ที่เกี่ยวข้อง

### **FR5: Playback (Policy Compliance)**

* **FR5.1** ใช้ **YouTube IFrame Player** ใน WebView และ **วิดีโอต้องมองเห็นขณะเล่น**  
* **FR5.2** เมื่อแอปถูกย่อหรือจอล็อก ให้ **หยุดเล่น** (ห้ามเล่นพื้นหลัง)  
* **FR5.3** มีปุ่ม **“Open in YouTube”** เป็นทางเลือก  
* **FR5.4 ห้าม** บล็อก UI/โฆษณา/ลิงก์มาตรฐานของตัวเล่น

### **FR6: Albums & Save**

* **FR6.1** Member: สามารถ Save/Unsave อัลบั้มที่สนใจไว้ใน My Library ได้  
* **FR6.2** Admin: สามารถ **สร้าง / แก้ไข / ลบ** อัลบั้มทางการ (Official Album) ได้ไม่จำกัด และไม่แสดงชื่อผู้สร้างต่อสาธารณะ  โดยมีฟิลด์  
  * Youtube URL  
  * Cover picture   
  * Album name   
  * About album   
  * Select release years   
  * Item songs   
  * Genres   
  * Master/Remaster badge

---

## **5\. User Stories / Use Cases**

1. ### **Registration & Profile**

* **UC-01** ในฐานะผู้ใช้ใหม่ ฉันต้องการสมัครด้วย **Google** เพื่อเข้าใช้งานได้ง่าย  
* **UC-02** ในฐานะผู้ใช้ ฉันต้องการกรอกโปรไฟล์พื้นฐานและให้ **ความยินยอม PDPA**  
* **UC-03** ในฐานะผู้ใช้ ฉันต้องการแก้ไขโปรไฟล์/รูปภาพภายหลังได้  
* **UC-04**: ในฐานะผู้ใช้ ฉันต้องการลบบัญชีได้ด้วยตัวเอง (Immediate Deletion)

2. ### **Age‑based Discovery & Filters**

* **UC-05** ฉันต้องการเลือก **ช่วงอายุ** (เช่น 20–25) เพื่อดูเพลงในปีที่เกี่ยวข้อง  
* **UC-06** เมื่อคอนเทนต์น้อย ฉันต้องการให้ระบบ **ขยายช่วงปีอัตโนมัติสูงสุด ±5 ปี** และแสดง **Expanded \+N**  
* **UC-07** ฉันต้องการใช้ฟิลเตอร์ **แนวเพลง** เพื่อจำกัดผลลัพธ์ในแท็บ **Albums**

3. ### **Search & Playback**

* **UC-08** ในฐานะผู้ใช้ ฉันต้องการพิมพ์คำค้นเพียงครั้งเดียว และเห็นผลลัพธ์รวมทุกประเภท **(Artist, Album)** พร้อม badge ที่ถูกต้อง เพื่อเข้าถึงคอนเทนต์ได้อย่างรวดเร็ว  
* **UC-09** ฉันต้องการเล่นเพลง **ในแอปโดยเห็นวิดีโอ** และคาดหวังว่าเมื่อย่อแอป/ล็อกจอ ระบบจะ **หยุดเล่น**  
* **UC-10** ฉันต้องการปุ่ม **“Open in YouTube”** เมื่ออยากเปลี่ยนไปใช้งานที่นั่น

4. ### **Albums & Save**

* **UC-11** ในฐานะ Member ฉันต้องการ Save/Unsave อัลบั้มที่สนใจไว้ใน My Library ได้  
* **UC-12** ในฐานะ Admin ฉันต้องการสร้างอัลบั้มทางการ (Official Album)  
* **UC-13** ในฐานะ Admin ฉันต้องการแก้ไขข้อมูลอัลบั้มทางการ  
* **UC-14** ในฐานะ Admin ฉันต้องการลบอัลบั้มทางการ  
* **UC-15** ในฐานะ Admin ฉันต้องการสร้างโปรไฟล์ศิลปินจริงได้ (Artist Profile)  
* **UC-16** ในฐานะ Admin ฉันต้องการแก้ไขโปรไฟล์ศิลปิน  
* **UC-17** ในฐานะ Admin ฉันต้องการลบโปรไฟล์ศิลปิน

---

## **6\. Design Considerations / Mockups**

* **ความเรียบง่ายมาก่อน:** โครงหน้าชัดเจน ปุ่มเด่น ข้อความอ่านง่าย  
* **การเข้าถึง (Accessibility):** ฟอนต์ใหญ่/คอนทราสต์สูง ไอคอนคู่กับข้อความ รองรับ screen reader  
* **Mobile‑only & แอปเดียว:** เรนเดอร์ตามสิทธิ์บทบาท (ไม่ต้องลงแอปใหม่เมื่อสิทธิ์เปลี่ยน)  
* **การเล่นตามนโยบาย:** วิดีโอต้องมองเห็นเสมอ ขณะย่อ/ล็อกจอให้หยุดเล่น ไม่บล็อก UI/โฆษณามาตรฐาน มีปุ่ม **“Open in YouTube”**  
* **ต้นแบบ:** ใช้ไวร์เฟรม low‑fi และตัวอย่าง UI บน React preview เพื่อทดสอบก่อนทำดีไซน์ระดับสูง  
