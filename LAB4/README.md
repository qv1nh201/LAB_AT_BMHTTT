# Báo cáo thực hành Lab 4

- **Họ và tên:** Trần Hồ Quang Vinh
- **MSSV:** 1150080082
- **Bài thực hành:** Lab 4 – Khảo sát và đánh giá bề mặt mạng bằng Nmap[cite: 1]

---

## 1. Môi trường thực hành
- **Nền tảng ảo hóa:** VMware Workstation[cite: 2].
- **Chế độ mạng:** Host-Only (`VMnet1` - Subnet `192.168.56.0/24`)[cite: 1, 9].
- **Các thiết bị:**
  - **Máy thật (Host):** Windows 11 (`192.168.56.1`)[cite: 1, 9].
  - **Máy quét:** ubuntu Linux (`192.168.56.129`)[cite: 3, 9].
  - **Máy đích:** Metasploitable 2 (`192.168.142.129`)[cite: 1, 9].
- **Công cụ:** Nmap 7.99[cite: 9], xsltproc[cite: 1].

---

## 2. Cách dựng môi trường
1. Import 2 máy ảo đóng gói sẵn (Kali Linux và Metasploitable 2) vào VMware[cite: 2, 3, 4].
2. Chỉnh dải mạng `VMnet1` về `192.168.56.0/24`, bật DHCP[cite: 1].
3. Gán card mạng của cả 2 VM sang `VMnet1 (Host-only)`[cite: 1].
4. Tạo snapshot `Before-LAB4` cho cả hai máy[cite: 1].
5. Khởi động hệ thống, kiểm tra IP qua `ip -br addr` / `ifconfig` và xác nhận kết nối bằng `ping -c 4` thông tuyến 100%[cite: 1, 8, 9].

---

## 3. Tiến độ và kết quả
- **Host Discovery (`-sn`):** Quét dải `192.168.56.0/24`, phát hiện 4 host UP — **PASS**[cite: 1, 9]
- **TCP Port Scan (`-sT`, `-sS`):** Phát hiện chính xác 23 cổng open, so sánh tốc độ và quyền hạn giữa 2 kỹ thuật — **PASS**[cite: 1, 9]
- **Kỹ thuật quét nâng cao (`-sF`, `-sX`, `-sN`, `-sA`):** Ghi nhận trạng thái `open|filtered` với FIN/Xmas/NULL và `unfiltered` với ACK — **PASS**[cite: 1, 9]
- **UDP Scan (`-sU`):** Quét 20 cổng UDP phổ biến, ghi nhận các dịch vụ như DNS (53), NetBIOS (137) — **PASS**[cite: 1, 9]
- **Service & OS Detection (`-sV`, `-O`):** Xác định phiên bản dịch vụ (vsftpd 2.3.4, Apache 2.2.8, Samba 3.X) và nhân Linux 2.6.X — **PASS**[cite: 1, 9]
- **NSE Script:** Chạy `smb-os-discovery` nhận diện thông tin Samba; xác nhận không bị ảnh hưởng bởi lỗ hổng `smb-vuln-ms17-010` — **PASS**[cite: 1, 9]
- **Xuất báo cáo:** Xuất thành công tệp `ket_qua.txt`, `ket_qua.xml` và chuyển sang `bao_cao.html` bằng xsltproc — **PASS**[cite: 1]
- **Hardening:** Tắt dịch vụ `vsftpd`, quét kiểm tra cổng 21 chuyển từ `open` sang `closed` — **PASS**[cite: 1]

---

## 4. Lỗi gặp phải và cách khắc phục
- **Sai dải IP ban đầu:** Máy nhận dải NAT `192.168.44.0/24`[cite: 7, 8]; khắc phục bằng cách cấu hình lại VMnet1 sang `192.168.56.0/24` và restart dịch vụ mạng[cite: 1, 9].
- **Gõ nhầm lệnh tại màn hình login:** Gõ nhầm lệnh vào mục username của Metasploitable[cite: 6]; khắc phục bằng cách nhấn Enter để đăng nhập lại với `msfadmin` / `msfadmin`[cite: 6].
- **Dán nhầm nội dung vào terminal:** Bị lỗi `command not found`; khắc phục bằng cách dùng `Ctrl + C` để hủy và tự gõ lại lệnh chuẩn[cite: 1].