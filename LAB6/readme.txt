# 🛡️ LAB 6: Active Directory Authentication & Access Control Policies on pfSense

[![pfSense](https://img.shields.io/badge/pfSense-2.7.2--RELEASE-blue.svg)](https://www.pfsense.org/)
[![Platform](https://img.shields.io/badge/Platform-VMware%20Workstation-orange.svg)](https://www.vmware.com/)
[![OS](https://img.shields.io/badge/Server%20OS-Windows%20Server%202025-0078D6.svg)](https://www.microsoft.com/)

Hệ thống tài liệu, kịch bản triển khai và báo cáo cấu hình tường lửa **pfSense CE** tích hợp xác thực Active Directory qua **LDAP**, **Captive Portal**, và thiết lập **Access Control Policies (Firewall Rules / Aliases / Schedules)** trong môi trường ảo hóa phục vụ kiểm thử an toàn mạng.

---

## 📌 1. Thông Tin Sinh Viên & Môi Trường Thực Hiện

- **Họ và tên:** Trần Hồ Quang Vinh
- **Mã sinh viên:** 1150080082
- **Nền tảng ảo hóa:** VMware Workstation Pro
- **Phiên bản Firewall:** pfSense CE 2.7.2-RELEASE (amd64)

---

## 🌐 2. Mô Hình Phân Vùng Mạng (Topology)

Hệ thống được thiết kế với 4 phân vùng mạng tách biệt nhằm đảm bảo tính an toàn và quản lý tập trung:

| Phân vùng | Interface pfSense | Chế độ mạng (VMware) | Dải mạng / IP Gateway | Vai trò & Chức năng |
| :--- | :--- | :--- | :--- | :--- |
| **WAN** | `em0` | NAT (`VMnet8`) | IP DHCP (`192.168.250.x/24`) | Kết nối mạng bên ngoài / Internet |
| **MGMT** | `em1` | Host-Only (`VMnet1`) | `192.168.56.254/24` | Phân vùng quản trị WebGUI cách ly |
| **LAN** | `em2` | Custom (`VMnet2`) | `10.6.0.1/24` | Mạng nội bộ cho Domain Controller & Clients |
| **DMZ** | `em3` | Custom (`VMnet4`) | `172.16.6.1/24` | Phân vùng cách ly chứa Web Server đa IP |

---

## 🖥️ 3. Danh Sách Máy Ảo Trong Hệ Thống

1. **`pfSense-LAB6` (Firewall Appliance)**
   - **OS:** FreeBSD 14.0-CURRENT (amd64)
   - **Cấu hình:** 2 GB RAM | 20 GB Disk | 4 Network Adapters (`em0`, `em1`, `em2`, `em3`)
2. **`DC01` (Domain Controller)**
   - **OS:** Windows Server 2025
   - **Network:** `10.6.0.10/24` | Gateway: `10.6.0.1` | DNS: `10.6.0.10`
   - **Roles:** Active Directory Domain Services (`lab6.test`), DNS Server, OU `LAB6-Users` (`sv_a`, `sv_b`, `svc_pfsense_ldap`)
3. **`Client-A` (Trạm kiểm thử 1)**
   - **OS:** Windows 11 x64
   - **Network:** `10.6.0.21/24` | Gateway: `10.6.0.1` | DNS: `10.6.0.10`
4. **`Client-B` (Trạm kiểm thử 2)**
   - **OS:** Windows 11 x64
   - **Network:** `10.6.0.22/24` | Gateway: `10.6.0.1` | DNS: `10.6.0.10`
5. **`DMZ-WEB` (Multi-IP Web Server)**
   - **OS:** Kali Linux / Ubuntu Server (Nginx)
   - **Network:**
     - `172.16.6.10/24` (GENERAL SITE)
     - `172.16.6.20/24` (JOB SITE)
     - `172.16.6.30/24` (SOCIAL SITE)
     - `172.16.6.40/24` (RESTRICTED SITE)

---

## ⚙️ 4. Nội Dung Cấu Hình & Kết Quả Kiểm Thử

### 🔑 A. Tích hợp Active Directory (LDAP) & Captive Portal
- **LDAP Authentication Server (`LAB6-AD`):**
  - Khai báo Server `10.6.0.10:389`, Base DN `DC=lab6,DC=test`, Bind DN `CN=Administrator,CN=Users,DC=lab6,DC=test` (hoặc `svc_pfsense_ldap`).
  - Kiểm thử xác thực thành công tài khoản miền `sv_a` và `sv_b` tại **Diagnostics > Authentication**.
- **Captive Portal Zone (`LAB6_USERS`):**
  - Kích hoạt Captive Portal trên giao diện **LAN**, buộc người dùng xác thực thông qua backend `LAB6-AD` trước khi ra Internet.

### 🏷️ B. Khai báo Aliases & Schedules
- **Aliases:**
  - `POLICY_CLIENTS`: `10.6.0.21`, `10.6.0.22`
  - `JOB_SITE`: `172.16.6.20` | `SOCIAL_SITE`: `172.16.6.30` | `RESTRICTED_SITE`: `172.16.6.40`
  - `WEB_PORTS`: `80`, `443`
- **Schedules:**
  - `LUNCH_WINDOW`: 11:30 - 13:00 (Thứ 2 - Thứ 6)
  - `WORK_HOURS`: 08:00 - 17:00 (Thứ 2 - Thứ 6)

### 📋 C. Kịch Bản Kiểm Soát Truy Cập (Access Policies)
1. **Kịch bản 1:** Block `Client-A` truy cập `JOB_SITE` (`172.16.6.20`), vẫn cho phép truy cập `GENERAL_SITE` (`172.16.6.10`).
2. **Kịch bản 2:** Block toàn bộ `POLICY_CLIENTS` truy cập `RESTRICTED_SITE` (`172.16.6.40`).
3. **Kịch bản 3:** Cho phép `Client-B` truy cập `SOCIAL_SITE` trong giờ nghỉ trưa (`LUNCH_WINDOW`) và chặn ngoài giờ bằng Rule Block bổ trợ bên dưới.
4. **Kịch bản 4:** Giới hạn dịch vụ web (`WEB_PORTS`) trong giờ hành chính (`WORK_HOURS`), chặn các giao thức khác như ICMP ping.
5. **Kịch bản 5:** Cô lập hoàn toàn DMZ khỏi mạng LAN (chặn mọi kết nối khởi tạo từ `DMZ net` tới `LAN net`).
6. **Kịch bản 6:** Phân biệt cơ chế **Block** (Quiet Drop / Timeout) và **Reject** (Gửi TCP RST / ICMP Unreachable lập tức).
7. **Kịch bản 7:** Phân tích Firewall Log và thực hiện xóa bảng trạng thái (*Reset States*) tại **Diagnostics > States**.

---

## 📂 5. Cấu Trúc Thư Mục Project

```text
.
├── docs/
│   └── 1150080082-Lab6_TranHoQuangVinh.docx    # Báo cáo chi tiết kèm ảnh minh chứng & câu hỏi lý thuyết
├── config/
│   └── pfsense-lab6-config.xml                # Tệp cấu hình Backup từ pfSense
└── README.md                                  # Tài liệu tổng quan dự án (GitHub standard)
```

---

## 📝 6. Hướng Dẫn Khôi Phục Cấu Hình (Restore)

1. Đăng nhập WebGUI pfSense qua giao diện MGMT: `http://192.168.56.254`.
2. Truy cập **Diagnostics > Backup & Restore**.
3. Tại mục **Restore configuration**, chọn file `config/pfsense-lab6-config.xml`.
4. Bấm **Restore Configuration** và chờ hệ thống Reboot.