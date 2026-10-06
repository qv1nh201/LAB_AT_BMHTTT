# LAB 5: Triển khai và Cấu hình Tường lửa pfSense

Hệ thống tài liệu, kịch bản triển khai và báo cáo cấu hình tường lửa pfSense CE trong môi trường ảo hóa phục vụ kiểm thử an toàn mạng[cite: 1].

---

## 1. Thông tin sinh viên & Môi trường thực hiện

* **Họ và tên:** Trần Hồ Quang Vinh
* **Mã sinh viên:** 1150080082
* **Nền tảng ảo hóa:** VMware Workstation[cite: 2]
* **Phiên bản Firewall:** pfSense CE 2.7.2-RELEASE (amd64)[cite: 1]

---

## 2. Mô hình phân vùng mạng (Topology)

Hệ thống được thiết kế với 3 phân vùng mạng tách biệt[cite: 1]:

| Phân vùng | Interface trên pfSense | Chế độ mạng (VMware) | Dải mạng / IP Gateway | Vai trò |
| :--- | :--- | :--- | :--- | :--- |
| **WAN** | `em0`[cite: 1, 11] | NAT[cite: 6, 21] | Nhận DHCP tự động[cite: 1, 11] | Kết nối mạng ngoài / Internet[cite: 1] |
| **LAN** | `em1`[cite: 1, 11] | Host-only (`VMnet1`)[cite: 1, 6] | `10.0.0.1/8`[cite: 1, 11] | Mạng nội bộ cho Domain Controller và máy trạm[cite: 1] |
| **DMZ** | `em2`[cite: 1, 11] | Custom (`VMnet2`)[cite: 1, 6] | `172.16.0.1/16`[cite: 1, 11] | Phân vùng cách ly đặt Web Server (IIS)[cite: 1] |

---

## 3. Danh sách máy ảo trong hệ thống

1. **pfSense Firewall:**
   * OS: FreeBSD 64-bit[cite: 1, 3]
   * RAM: 2 GB | Ổ cứng: 20 GB[cite: 1, 5, 21]
   * Network Adapters: 3 NIC (`em0` - WAN, `em1` - LAN, `em2` - DMZ)[cite: 1, 6, 11]
2. **Domain Controller (LAN Host 1):**
   * OS: Windows Server 2025[cite: 1, 2]
   * IP tĩnh: `10.0.0.2/8` | Gateway: `10.0.0.1` | DNS: `10.0.0.2`[cite: 1]
3. **DMZ-Web (Vùng cách ly):**
   * OS: Windows Server (chạy Web Server IIS)[cite: 1]
   * IP tĩnh: `172.16.0.2/16` | Gateway: `172.16.0.1` | DNS: `8.8.8.8`[cite: 1]
4. **LAN-Test (LAN Host 2 - Tùy chọn):**
   * OS: Windows 11 x64 / Ubuntu Server[cite: 1, 2]
   * IP tĩnh: `10.0.0.3/8` | Gateway: `10.0.0.1`[cite: 1]

---

## 4. Các nội dung cấu hình & Kết quả kiểm thử

### Tình huống 1: Khởi tạo Interface và Cấu hình ban đầu
* Gán thành công 3 giao diện `em0` (WAN), `em1` (LAN), `em2` (DMZ) trực tiếp từ Console[cite: 1, 11].
* Thiết lập IP LAN `10.0.0.1/8` và DMZ `172.16.0.1/16`, tắt DHCP Server nội bộ[cite: 1, 11].
* Đăng nhập thành công WebConfigurator từ máy Windows Server qua địa chỉ `http://10.0.0.1`[cite: 1, 11].
* Bỏ kích hoạt `Block RFC1918 Private Networks` và `Block bogon networks` trên giao diện WAN để hỗ trợ môi trường Lab ảo hóa[cite: 1].

### Tình huống 2: Kiểm soát truy cập mạng LAN ra Internet
* Vô hiệu hóa luật mặc định `Default allow LAN to any rule` (IPv4 & IPv6) và thực hiện xóa bảng trạng thái (*Reset States*)[cite: 1, 16].
* Thiết lập luật kiểm soát chủ động: `Pass | Interface: LAN | Source: LAN subnets | Destination: Any`[cite: 1, 17].
* **Kết quả:** Kiểm thử từ máy chủ LAN (`10.0.0.2`) thực hiện `ping 8.8.8.8` thành công với độ trễ thấp và tỷ lệ mất gói 0%[cite: 1, 23].

### Tình huống 3: Cô lập phân vùng DMZ khỏi mạng LAN
* Thiết lập chính sách kiểm soát trên tab DMZ[cite: 1]:
  * Luật 1 (Chặn về LAN): `Block | Interface: DMZ | Source: DMZ net | Destination: LAN net`[cite: 1].
  * Luật 2 (Ra Internet): `Pass | Interface: DMZ | Source: DMZ net | Destination: Any`[cite: 1].
* **Kết quả:** Máy chủ DMZ-Web (`172.16.0.2`) không thể ping hay truy cập dữ liệu sang mạng LAN (`10.0.0.2`), nhưng vẫn truy cập được các dịch vụ Internet công cộng[cite: 1].

### Tình huống 4: Cấu hình Port Forwarding (NAT) dịch vụ Web DMZ
* Cấu hình Port Forward trên giao diện WAN[cite: 1]:
  * `Interface: WAN | Protocol: TCP | Destination Port: 8080 | Redirect Target IP: 172.16.0.2 | Redirect Target Port: 80`[cite: 1].
  * Kèm theo tùy chọn tự động sinh luật tường lửa liên kết (`Add associated filter rule`)[cite: 1].
* **Kết quả:** Máy từ phân vùng ngoài có thể truy cập trang Web IIS của DMZ thông qua địa chỉ `http://<IP-WAN-pfSense>:8080`[cite: 1].

---


```text
├── docs/
│   └── Lab3_Bao_Cao_pfSense.docx    # Báo cáo chi tiết có dán kèm hình ảnh minh chứng[cite: 1]
├── scripts/
│   └── VERIFY_SHA256.ps1            # Kịch bản kiểm tra mã băm ISO pfSense[cite: 1]
└── README.md                        # Tài liệu tổng quan bài thực hành
