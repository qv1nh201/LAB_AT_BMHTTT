# 🛡️ LAB 8: Local DNS Attack - DNS Spoofing & Cache Poisoning on pfSense & Ubuntu

[![pfSense](https://img.shields.io/badge/pfSense-2.7.2--RELEASE-blue.svg)](https://www.pfsense.org/)
[![Platform](https://img.shields.io/badge/Platform-VMware%20Workstation-orange.svg)](https://www.vmware.com/)
[![OS](https://img.shields.io/badge/Server%20OS-Ubuntu%20Server%2024.04%20LTS-E95420.svg)](https://ubuntu.com/)

Hệ thống tài liệu, kịch bản triển khai và báo cáo thực hành tấn công hệ thống phân giải tên miền cục bộ (**Local DNS Attack**), bao gồm các kỹ thuật thay đổi file Hosts, giả mạo phản hồi DNS trực tiếp (**Direct DNS Spoofing**) và đầu độc bộ nhớ đệm máy chủ phân giải (**DNS Cache Poisoning**) trong môi trường mạng ảo hóa cô lập qua pfSense.

---

## 📌 1. Thông Tin Sinh Viên & Môi Trường Thực Hiện

- **Họ và tên:** Trần Hồ Quang Vinh
- **Mã sinh viên:** 1150080082
- **Nền tảng ảo hóa:** VMware Workstation Pro
- **Phiên bản Firewall:** pfSense CE 2.7.2-RELEASE (amd64)

---

## 🌐 2. Mô Hình Phân Vùng Mạng (Topology)

Hệ thống được thiết kế mở rộng từ Lab trước, bổ sung phân vùng `DNS-LAB` cô lập để thực hiện các thử nghiệm tấn công DNS an toàn:

| Phân vùng | Interface pfSense | Chế độ mạng (VMware) | Dải mạng / IP Gateway | Vai trò & Chức năng |
| :--- | :--- | :--- | :--- | :--- |
| **WAN** | `em0` | NAT (`VMnet8`) | IP DHCP (`192.168.250.x/24`) | Kết nối mạng bên ngoài / Internet |
| **MGMT** | `em1` | Host-Only (`VMnet1`) | `192.168.56.254/24` | Phân vùng quản trị WebGUI cách ly |
| **LAN** | `em2` | Custom (`VMnet2`) | `10.6.0.1/24` | Mạng nội bộ kế thừa từ LAB 7 |
| **DNS-LAB** | `em4` (OPT3) | Custom (`VMnet5`) | `10.9.0.1/24` | Mạng ảo cô lập dành riêng cho Lab tấn công DNS[cite: 1] |
| **DMZ** | `em3` | Custom (`VMnet4`) | `172.16.6.1/24` | Phân vùng chứa Authoritative DNS & Web Server giả lập[cite: 1] |

---

## 🖥️ 3. Danh Sách Máy Ảo Trong Hệ Thống

1. **`pfSense-LAB8` (Firewall Appliance)**
   - **OS:** FreeBSD 14.0-CURRENT (amd64)
   - **Cấu hình:** 2 GB RAM | 20 GB Disk | 5 Network Adapters (`em0`, `em1`, `em2`, `em3`, `em4`)[cite: 1]
2. **`DNS-SERVER` (Recursive / Caching DNS Resolver)**
   - **OS:** Ubuntu Server 24.04 LTS[cite: 1]
   - **Network:** `10.9.0.10/24` | Gateway: `10.9.0.1`[cite: 1]
   - **Services:** BIND9 (`named`), `dnsutils`[cite: 1]
3. **`VICTIM` (DNS Client)**
   - **OS:** Ubuntu Desktop / Server 24.04 LTS[cite: 1]
   - **Network:** `10.9.0.100/24` | DNS Server: `10.9.0.10`[cite: 1]
4. **`ATTACKER` (Kali Linux / Ubuntu Testing Node)**
   - **OS:** Kali Linux / Ubuntu[cite: 1]
   - **Network:** `10.9.0.200/24` | Promiscuous Mode: `Allow VMs`[cite: 1]
5. **`APP-TEST / AUTH-DNS` (Simulated Authoritative DNS)**
   - **Network:** `172.16.6.20/24`[cite: 1]
   - **Services:** `auth_dns_delayed.py` (Mô phỏng phản hồi DNS có chủ đích với độ trễ 1.5s)[cite: 1]
6. **`DMZ-WEB` (Legitimate Web Server)**
   - **Network:** `172.16.6.10/24`[cite: 1]

---

## ⚙️ 4. Nội Dung Cấu Hình & Kết Quả Kiểm Thử

### 🐧 A. Cấu Hình pfSense Firewall & Mạng DNS-LAB
- **Gán Interface OPT3:** Đặt tên mô tả `DNSLAB`, cấu hình IP tĩnh `10.9.0.1/24` (Tuyệt đối không gán Upstream Gateway để tránh lệch định tuyến)[cite: 1].
- **Chính sách tường lửa cô lập (Firewall Rules trên tab DNSLAB)[cite: 1]:**
  - **Rule 1 (Pass DNS):** Cho phép `10.9.0.10` truy cập tới `172.16.6.20` qua cổng UDP/TCP 53[cite: 1].
  - **Rule 2 (Pass Web):** Cho phép toàn dải `10.9.0.0/24` truy cập tới `172.16.6.10` qua cổng TCP 80[cite: 1].
  - **Rule 3 & 4 (Block LAN & MGMT):** Chặn hoàn toàn lưu lượng từ `10.9.0.0/24` đi sang mạng `10.6.0.0/24` (LAN) và `192.168.56.0/24` (MGMT), có bật tính năng ghi log (`Log packets`)[cite: 1].
  - **Rule 5 (Block Default):** Chặn mọi kết nối còn lại để cô lập hoàn toàn mạng tấn công[cite: 1].

### 🛠️ B. Cấu Hình BIND9 Trên Ubuntu Server (`DNS-SERVER`)
- Cài đặt gói `bind9` và `dnsutils`[cite: 1].
- Tùy chỉnh `named.conf.options` và `named.conf.local`: thiết lập phân giải thuận/nghịch cho domain `lab9.test`, cấu hình forwarder trỏ về Authoritative DNS giả lập (`172.16.6.20`), tắt cơ chế cookie (`send-cookie no`) đối với server mô phỏng để phục vụ mục đích kiểm thử on-path[cite: 1].
- Kiểm tra tính hợp lệ bằng lệnh `named-checkconf` và `named-checkzone`[cite: 1].

### 🎯 C. Kịch Bản Thực Hiện 3 Nhiệm Vụ Tấn Công (Tasks)
1. **TASK 1 - HOSTS-file Redirection[cite: 1]:** 
   - Sửa file `/etc/hosts` trên máy `VICTIM` trỏ tên miền `www.lab9.test` về IP của kẻ tấn công (`10.9.0.200`)[cite: 1].
   - Xác thực hành vi chuyển hướng cục bộ qua lệnh `curl`, trong khi truy vấn trực tiếp qua `dig @10.9.0.10` vẫn trả về kết quả gốc[cite: 1].
2. **TASK 2 - Direct DNS Spoofing[cite: 1]:**
   - Sử dụng script Scapy (`dns_spoof_lab9.py` với `--mode victim`) trên máy `ATTACKER` để lắng nghe truy vấn từ `VICTIM` và lập tức gửi gói tin phản hồi giả mạo (Forged Response) với Transaction ID khớp nhau[cite: 1].
   - Kết quả: Nạn nhân truy cập vào trang web giả lập trên máy tấn công[cite: 1].
3. **TASK 3 - DNS Cache Poisoning[cite: 1]:**
   - Chạy script ở chế độ `--mode resolver` để nhắm mục tiêu vào máy chủ phân giải `DNS-SERVER` khi máy chủ này thực hiện recursive lookup sang Authoritative DNS (`172.16.6.20`)[cite: 1].
   - Kiểm tra kết quả đầu độc bằng lệnh `rndc dumpdb -cache` trên `DNS-SERVER`, xác nhận bản ghi giả mạo (`10.9.0.200`) đã bị lưu vào bộ nhớ cache trong khoảng thời gian TTL[cite: 1].

---

## 📂 5. Cấu Trúc Thư Mục Project

```text
.
├── docs/
│   └── 1150080082-Lab8_TranHoQuangVinh.docx    # Báo cáo chi tiết kèm ảnh minh chứng cấu hình & câu hỏi phân tích
├── config/
│   ├── pfsense-lab8-config.xml                # Tệp cấu hình Backup từ pfSense (đã tích hợp DNS-LAB & Rules)
│   ├── bind9/
│   │   ├── named.conf.options                 # File cấu hình tùy chọn BIND9
│   │   └── named.conf.local                   # File cấu hình khai báo Zone thuận/nghịch
│   └── scripts/
│       └── dns_spoof_lab9.py                  # Kịch bản Python Scapy hỗ trợ giả mạo DNS
└── README.md                                  # Tài liệu tổng quan dự án (GitHub standard)