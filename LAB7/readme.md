# 🛡️ LAB 7: Web Proxy, Content Filtering & Bypass Control Policies on pfSense & Ubuntu

[![pfSense](https://img.shields.io/badge/pfSense-2.7.2--RELEASE-blue.svg)](https://www.pfsense.org/)
[![Platform](https://img.shields.io/badge/Platform-VMware%20Workstation-orange.svg)](https://www.vmware.com/)
[![OS](https://img.shields.io/badge/Server%20OS-Ubuntu%20Server%2024.04%20LTS-E95420.svg)](https://ubuntu.com/)

Hệ thống tài liệu, kịch bản triển khai và báo cáo cấu hình dịch vụ **Web Proxy (Squid)** trên Ubuntu Server kết hợp tường lửa **pfSense CE** để thiết lập chính sách lọc nội dung web, ngăn chặn hành vi vượt qua Proxy (Proxy Bypass Control), và quản lý truy cập mạng nâng cao trong môi trường ảo hóa.

---

## 📌 1. Thông Tin Sinh Viên & Môi Trường Thực Hiện

- **Họ và tên:** Trần Hồ Quang Vinh
- **Mã sinh viên:** 1150080082
- **Nền tảng ảo hóa:** VMware Workstation Pro
- **Phiên bản Firewall:** pfSense CE 2.7.2-RELEASE (amd64)

---

## 🌐 2. Mô Hình Phân Vùng Mạng (Topology)

Hệ thống được thiết kế với 4 phân vùng mạng tách biệt nhằm đảm bảo tính an toàn, kiểm soát luồng dữ liệu web và quản lý tập trung:

| Phân vùng | Interface pfSense | Chế độ mạng (VMware) | Dải mạng / IP Gateway | Vai trò & Chức năng |
| :--- | :--- | :--- | :--- | :--- |
| **WAN** | `em0` | NAT (`VMnet8`) | IP DHCP (`192.168.250.x/24`) | Kết nối mạng bên ngoài / Internet |
| **MGMT** | `em1` | Host-Only (`VMnet1`) | `192.168.56.254/24` | Phân vùng quản trị WebGUI cách ly |
| **LAN** | `em2` | Custom (`VMnet2`) | `10.6.0.1/24` | Mạng nội bộ chứa Web Proxy Server & Clients |
| **DMZ** | `em3` | Custom (`VMnet4`) | `172.16.6.1/24` | Phân vùng cách ly chứa Web Server đa IP |

---

## 🖥️ 3. Danh Sách Máy Ảo Trong Hệ Thống

1. **`pfSense-LAB7` (Firewall Appliance)**
   - **OS:** FreeBSD 14.0-CURRENT (amd64)
   - **Cấu hình:** 2 GB RAM | 20 GB Disk | 4 Network Adapters (`em0`, `em1`, `em2`, `em3`)
2. **`WEB-PROXY` (Forward Proxy Server)**
   - **OS:** Ubuntu Server 24.04 LTS
   - **Network:** `10.6.0.30/24` | Gateway: `10.6.0.1` | DNS: `10.6.0.10`
   - **Services:** Squid Proxy Server (Listening Port `8080`), Netplan Static IP Configuration
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

### 🐧 A. Cấu Hình Mạng Static & Dịch Vụ Proxy Trên Ubuntu Server
- **Cấu hình Netplan (`50-cloud_init.yaml`):**
  - Khắc phục các lỗi cú pháp chuẩn YAML (thụt lề 2 khoảng trắng `Space`, xử lý khoảng trắng sau dấu `-`, định dạng danh sách DNS `[10.6.0.10]`).
  - Thiết lập phân quyền file an toàn `chmod 600 /etc/netplan/50-cloud_init.yaml` để xóa cảnh báo bảo mật.
  - Áp dụng cấu hình `sudo netplan apply` và xác nhận IP `10.6.0.30/24` trên card `ens33`.
- **Triển khai & Cấu hình Squid Proxy:**
  - Cài đặt Squid Proxy Service listening trên Port `8080`.
  - Khai báo các dải Access Control Lists (ACLs) cho phép mạng LAN (`10.6.0.0/24`) thực hiện HTTP/HTTPS forwarding.

### 🏷️ B. Khai Báo Aliases & Rules Chặn Bypass Trên pfSense
- **Aliases:**
  - `PROXY_SERVER`: `10.6.0.30`
  - `POLICY_CLIENTS`: `10.6.0.21`, `10.6.0.22`
  - `HTTP_PORTS`: `80`, `443`
  - `PROXY_PORT`: `8080`
- **Chính Sách Tường Lửa Chống Vượt Proxy (Anti-Bypass Firewall Rules):**
  - **Rule 1 (Pass Proxy Traffic):** Cho phép `POLICY_CLIENTS` truy cập tới `PROXY_SERVER` trên `PROXY_PORT` (`8080`).
  - **Rule 2 (Block Direct Web Access):** Chặn toàn bộ kết nối trực tiếp (Block) từ `POLICY_CLIENTS` đi ra ngoài Internet / DMZ thông qua `HTTP_PORTS` (`80`, `443`).
  - Bắt buộc mọi lưu lượng duyệt web của máy trạm phải đi qua `WEB-PROXY` (`10.6.0.30:8080`).

### 📋 C. Kịch Bản Kiểm Thử & Xác Minh Luồng Mạng
1. **Kiểm thử truy cập trực tiếp (Direct Connection):** Tắt cấu hình Proxy trên Windows Client -> Thử truy cập Web -> Tường lửa pfSense kích hoạt Rule Block, ngắt kết nối trực tiếp.
2. **Kiểm thử truy cập qua Proxy (Proxy Connection):** Bật Manual Proxy (`10.6.0.30:8080`) trên Windows Client -> Duyệt Web thành công qua Squid Proxy.
3. **Phân tích Log & Reset States:**
   - Kiểm tra **Status > System Logs > Firewall** để xác nhận các gói tin bị Block khi cố tình lách luật.
   - Thực hiện xóa bảng trạng thái (*Reset States*) tại **Diagnostics > States** khi thay đổi luật tường lửa.

---

## 📂 5. Cấu Trúc Thư Mục Project

```text
.
├── docs/
│   └── 1150080082-Lab7_TranHoQuangVinh.docx    # Báo cáo chi tiết kèm ảnh minh chứng & câu hỏi lý thuyết
├── config/
│   ├── pfsense-lab7-config.xml                # Tệp cấu hình Backup từ pfSense
│   └── netplan/
│       └── 50-cloud_init.yaml                 # Tệp cấu hình mạng chuẩn YAML cho Ubuntu Server
└── README.md                                  # Tài liệu tổng quan dự án (GitHub standard)
```

---

## 📝 6. Hướng Dẫn Khôi Phục Cấu Hình (Restore)

1. Đăng nhập WebGUI pfSense qua giao diện MGMT: `http://192.168.56.254`.
2. Truy cập **Diagnostics > Backup & Restore**.
3. Tại mục **Restore configuration**, chọn file `config/pfsense-lab7-config.xml`.
4. Bấm **Restore Configuration** và chờ hệ thống Reboot.
5. Trên máy Ubuntu Server, copy tệp `config/netplan/50-cloud_init.yaml` vào `/etc/netplan/`, thực hiện `sudo chmod 600 /etc/netplan/50-cloud_init.yaml` và chạy `sudo netplan apply`.