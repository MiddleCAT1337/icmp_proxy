# ICMP PING

โปรแกรมคอนโซลบน Windows ที่ใช้เช็คว่า IP และ port ปลายทางตอบสนองหรือไม่ โดยวัดเวลาเป็นมิลลิวินาที แบบคล้าย ping แต่ไม่ใช้ ICMP จริง ใช้การเปิด TCP connect หรือส่ง UDP แทน การเชื่อมไปยังปลายทางทำผ่าน SOCKS5 proxy จากไฟล์ verified_socks5.txt ไม่ได้เชื่อมตรงจากเครื่องผู้ใช้

โปรแกรมเขียนด้วยภาษา C มาตรฐาน C11 ลิงก์กับ Winsock2 คอมไพล์เป็น exe แบบ static ด้วย MinGW gcc

## โครงสร้างไฟล์

- src/main.c จุดเริ่มโปรแกรม จัดลำดับการทำงานทั้งหมด
- src/net.c และ net.h งานเครือข่าย resolve probe ตรวจ protocol และสถิติ
- src/proxy.c และ proxy.h โหลดรายการ proxy SOCKS5 สแกนลบตัวเสีย และ tunnel
- src/ui.c และ ui.h แสดงผลบน console รับ input และจัดการ Ctrl+C
- verified_socks5.txt รายการ proxy หนึ่งบรรทัดต่อหนึ่งตัว รูปแบบ host:port

## ภาพรวมการทำงาน

ลำดับหลักใน main.c มีดังนี้

1. ตั้งค่า console และแสดง banner
2. อ่าน IP และ port ปลายทางจากผู้ใช้
3. เริ่ม Winsock
4. โหลด proxy จาก verified_socks5.txt ที่อยู่โฟลเดอร์เดียวกับไฟล์ exe
5. หา proxy ตัวแรกที่ตอบ SOCKS5 ได้ ถ้าไม่มีเลยจบโปรแกรม
6. เปิด thread สแกน proxy ในพื้นหลัง
7. resolve ชื่อหรือ IP ปลายทางเป็น sockaddr_in IPv4
8. ตรวจว่าปลายทางน่าจะใช้ TCP หรือ UDP
9. วนลูป probe ทุก 100 ms จนกว่าผู้ใช้กด Ctrl+C
10. แสดงสรุปสถิติ ปิด thread proxy คืนทรัพยากร

ค่า timeout ที่กำหนดใน main

- DETECT_TIMEOUT_MS 800 ใช้ตอนตรวจ protocol
- PING_TIMEOUT_MS 200 ใช้ตอน probe ในลูปหลัก
- PING_INTERVAL_MS 100 หน่วงระหว่างแต่ละครั้ง
- DETECT_MAX_PROXY_ATTEMPTS 10 จำกัดจำนวน proxy ที่ลองตอน detect เท่านั้น ลูป ping ลองได้ตามจำนวนที่เหลือใน list

## โมดูล net

### การวัดเวลา

ใช้ QueryPerformanceCounter แปลงเป็นหน่วย ms เก็บความถี่ไว้ตอน net_init

### net_resolve

เรียก getaddrinfo แบบ IPv4 แล้วตั้ง port ตามที่ผู้ใช้ใส่

### Probe ตรง (ไม่ผ่าน proxy)

มีไว้เป็นพื้นฐานและใช้เมื่อไม่มี proxy list

TCP: สร้าง socket แบบ non-blocking แล้ว connect รอ select จน timeout หรือสำเร็จ อ่าน SO_ERROR ถ้า connect สำเร็จถือว่า ok และ latency คือเวลาตั้งแต่เริ่ม connect

UDP: connect socket ไปปลายทาง ส่ง 1 byte รอ recv หรือ timeout ถ้าได้รับ packet ถือว่า ok ถ้าไม่มี response แต่ไม่มี error แบบ refused หรือ unreachable บางกรณีถือว่า ok อยู่ดี (พฤติกรรมคล้าย UDP ping ที่ไม่มี echo กลับ)

### Probe ผ่าน proxy

TCP: เรียก socks5_tcp_connect ถ้า tunnel สำเร็จ latency คือเวลาทั้งหมดของขั้นตอน แล้วปิด socket ไม่ได้ส่งข้อมูล application ต่อ

UDP: เปิด socks5_udp_open ส่ง 1 byte ผ่าน framing SOCKS5 UDP รอรับกลับภายใน timeout ถ้า associate หรือ send ล้มเหลวตั้ง proxy_failed

### net_should_failover

ถือว่าควรเปลี่ยน proxy เมื่อ proxy_failed หรือ error เป็น WSAETIMEDOUT

### net_probe_with_failover

ดึง snapshot ของ proxy ปัจจุบัน ลอง probe ถ้าสำเร็จเรียก proxy_promote_current_to_front เพื่อเลื่อน proxy ที่ใช้ได้ไป index 0 ใน memory

ถ้าไม่สำเร็จและควร failover

- ถ้า proxy_failed ลบ proxy ปัจจุบันออกจาก list และเขียนกลับไฟล์ ตั้ง proxy_removed ใน result
- ถ้า timeout อย่างเดียว แค่ proxy_advance ไปตัวถัดไป ไม่ลบจากไฟล์

ลองซ้ำจนถึง max_proxy_attempts หรือ list ว่าง

### net_detect_proto

ลอง TCP และ UDP ผ่าน proxy แบบ failover เหมือนลูป ping แต่ใช้ timeout ของ detect

จากผล tcp_ok และ udp_ok เลือก protocol

- TCP ได้ UDP ไม่ได้ ใช้ TCP
- UDP ได้ TCP ไม่ได้ ใช้ UDP
- ได้ทั้งคู่ เลือก TCP
- ถ้า UDP ผ่าน proxy ไม่ได้เลย (udp_via_proxy เป็น 0) บังคับ TCP
- ถ้าทั้งคู่ไม่ชัด ใช้ heuristic จากหมายเลข port เช่น 53 443 80

main จะเปลี่ยนกลับเป็น TCP ถ้า detect เป็น UDP แต่ proxy รองรับ UDP ไม่ได้

### stats

นับ sent received min max sum ของ latency เฉพาะ probe ที่ ok

## โมดูล proxy

### ไฟล์ verified_socks5.txt

proxy_resolve_default_path หา path จาก GetModuleFileNameA แล้วต่อชื่อไฟล์

proxy_load อ่านบรรทัด แยก host:port ข้ามบรรทัดว่างและ comment ที่ขึ้นต้นด้วย #

proxy_save เขียน list ปัจจุบันทับไฟล์ทั้งไฟล์

### SOCKS5

รองรับ no authentication เท่านั้น

ขั้นตอน TCP tunnel

1. TCP connect ไปที่ proxy
2. ส่ง greeting เลือก method 0x00
3. ส่ง request CONNECT ไปยัง IPv4 และ port ปลายทาง
4. อ่าน reply ถ้า success ได้ socket ที่ tunnel ไปปลายทางแล้ว

UDP ใช้ CMD UDP ASSOCIATE บน TCP control connection ได้ relay address แล้วใช้ UDP socket ส่ง packet ที่ห่อ header SOCKS5 (RSV FRAG ATYP ที่อยู่ปลายทาง port แล้วตามด้วย data)

### การตรวจว่า proxy ยังมีชีวิต

socks5_proxy_alive ทำแค่ connect และ handshake method ไม่ CONNECT ไปปลายทาง ใช้ timeout สั้น (PROXY_PURGE_TIMEOUT_MS 600 ms)

### proxy_acquire_first_alive

เริ่มจาก index ปัจจุบัน ถ้าไม่ alive ลบออกจาก list และไฟล์ทันที วนจนเจอตัวแรกที่ alive หรือ list ว่าง

ทำให้เริ่ม ping ได้เร็วโดยไม่รอเช็คทั้งไฟล์ก่อน

### Background scanner

proxy_scanner_start สร้าง thread ที่วนไปเรื่อยๆ

- ข้าม index ที่เป็น proxy ที่กำลังใช้งานอยู่ (active index)
- เช็ค proxy ที่ scan_index ด้วย socks5_proxy_alive
- ถ้าตาย ลบออกและตั้ง scan_notify_pending ให้ main แสดงข้อความ
- ถ้าใช้ได้ เลื่อน scan_index ไปตัวถัดไป
- หน่วง PROXY_SCAN_INTERVAL_MS 80 ms ต่อรอบ

ใช้ CRITICAL_SECTION ป้องกัน race ระหว่าง main thread และ scanner

proxy_scanner_stop เรียกตอน proxy_free

### การลบและ index

เมื่อลบ entry ที่ index ใด จะ memmove array และปรับ index กับ scan_index ถ้าจำเป็น

proxy_remove_current ลบตัวที่ index ปัจจุบัน

proxy_promote_current_to_front ย้าย entry ที่ index ไปตำแหน่ง 0 ใน memory ไม่เขียนไฟล์ใหม่เพื่อเรียงลำดับ

## โมดูล ui

ui_init ตั้ง console title UTF-8 เปิด virtual terminal processing สำหรับสี ANSI

ui_read_target อ่าน IP และ port จาก stdin แล้วแสดงกล่อง TARGET CONFIG

แสดง PROXY CONFIG หลังได้ proxy ตัวแรก

ui_detect_begin สร้าง thread หมุนข้อความ Analyzing จน ui_detect_end แสดงผล detect

ลูป probe แสดงตาราง มีคอลัมน์ลำดับ target protocol latency status proxy ที่ใช้

ถ้า scanner หรือ failover ลบ proxy แสดง Removed dead proxy

ถ้าเปลี่ยน proxy เพราะ timeout แสดง Switched proxy

ui_install_ctrl_handler ตั้ง g_running เป็น 0 เมื่อ Ctrl+C หรือ Ctrl+Break

ui_print_summary แสดง loss และ min avg max latency

## สิ่งที่โปรแกรมไม่ได้ทำ

- ไม่ใช้ ICMP echo request จริง
- ไม่รองรับ SOCKS5 username password
- ไม่รองรับ IPv6 ใน SOCKS request (ปลายทาง resolve เป็น IPv4)
- TCP probe ผ่าน proxy สำเร็จแปลว่า tunnel เปิดได้ ไม่ได้อ่าน HTTP หรือ TLS
- การลบ proxy จากไฟล์ถาวร ถ้า target ล่มแต่ proxy ดี อาจยังอยู่ใน list แต่ถ้า handshake proxy ไม่ผ่านจะถูกลบ

## การเชื่อมโมดูล

main เป็นตัวกลาง ส่ง proxy_list_t เข้า probe_config_t ให้ net เรียกฟังก์ชันใน proxy โดยตรง ui รับ struct จาก net และ proxy เพื่อแสดงผลเท่านั้น ไม่มีการส่ง packet ใน ui
