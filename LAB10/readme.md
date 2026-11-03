# 🛡️ LAB 10: Remote DNS Cache Poisoning Attack on pfSense & Ubuntu

[![pfSense](https://img.shields.io/badge/pfSense-2.7.2--RELEASE-blue.svg)](https://www.pfsense.org/)
[![Platform](https://img.shields.io/badge/Platform-VMware%20Workstation-orange.svg)](https://www.vmware.com/)
[![OS](https://img.shields.io/badge/Server%20OS-Ubuntu%20Server%2024.04%20LTS-E95420.svg)](https://ubuntu.com/)

Hệ thống tài liệu, kịch bản triển khai và báo cáo thực hành tấn công đầu độc bộ nhớ đệm máy chủ phân giải tên miền từ xa (**Remote DNS Cache Poisoning Attack** - Kỹ thuật Kaminsky/Race Condition), bao gồm việc thiết lập mô hình mạng cô lập phân vùng tấn công `ATTACKNET`, kỹ thuật tạo subdomain ngẫu nhiên tránh Cache Effect, và dự đoán đồng thời Transaction ID cùng UDP Source Port trong môi trường mạng ảo hóa cô lập qua pfSense.

---

## 📌 1. Thông Tin Sinh Viên & Môi Trường Thực Hiện

- **Họ và tên:** Trần Hồ Quang Vinh
- **Mã sinh viên:** 1150080082
- **Nền tảng ảo hóa:** VMware Workstation Pro
- **Phiên bản Firewall:** pfSense CE 2.7.2-RELEASE (amd64)

---

## 🌐 2. Mô Hình Phân Vùng Mạng (Topology)

Hệ thống được thiết kế mở rộng từ Lab 9, tách biệt hoàn toàn máy tấn công ra phân vùng riêng `ATTACKNET` (`10.10.0.0/24`) để loại bỏ khả năng nghe lén gói tin (Sniffing/Promiscuous mode = Deny), tái hiện chính xác đòn tấn công từ xa trên môi trường Internet thực tế:

| Phân vùng | Interface pfSense | Chế độ mạng (VMware) | Dải mạng / IP Gateway | Vai trò & Chức năng |
| :--- | :--- | :--- | :--- | :--- |
| **WAN** | `em0` | NAT (`VMnet8`) | IP DHCP (`192.168.250.x/24`) | Kết nối mạng bên ngoài / Internet |
| **MGMT** | `em1` | Host-Only (`VMnet1`) | `192.168.56.254/24` | Phân vùng quản trị WebGUI cách ly |
| **LAN** | `em2` | Custom (`VMnet2`) | `10.6.0.1/24` | Mạng nội bộ kế thừa từ các LAB trước |
| **DNS-LAB** | `em4` (OPT3) | Custom (`VMnet3`) | `10.9.0.1/24` | Mạng ảo chứa Nạn nhân (VICTIM) và Resolver (DNS-SERVER) |
| **ATTACKNET**| `em5` (OPT4) | Custom (`VMnet5`) | `10.10.0.1/24` | Phân vùng cô lập dành riêng cho Máy tấn công (ATTACKER) |
| **DMZ** | `em3` | Custom (`VMnet4`) | `172.16.6.1/24` | Phân vùng chứa Authoritative DNS giả lập |

---

## 🖥️ 3. Danh Sách Máy Ảo Trong Hệ Thống

1. **`pfSense-LAB10` (Firewall Appliance)**
   - **OS:** FreeBSD 14.0-CURRENT (amd64)
   - **Cấu hình:** 2 GB RAM | 20 GB Disk | 6 Network Adapters (`em0` đến `em5`)
2. **`DNS-SERVER` (Recursive / Caching DNS Resolver)**
   - **OS:** Ubuntu Server 24.04 LTS / Debian
   - **Network:** `10.9.0.10/24` | Gateway: `10.9.0.1`
   - **Services:** BIND9 (`named`), `dnsutils`, `python3-dnspython`
3. **`VICTIM` (DNS Client)**
   - **OS:** Windows Server 2025 / Ubuntu Desktop
   - **Network:** `10.9.0.100/24` | Subnet Mask: `255.255.255.0` | Gateway: `10.9.0.1` | DNS Server: `10.9.0.10`
4. **`ATTACKER` (Kali Linux / Attacker Node)**
   - **OS:** Kali Linux / Ubuntu
   - **Network:** `10.10.0.200/24` | Gateway: `10.10.0.1` | Promiscuous Mode: `Deny` (Không sniff trực tiếp)
   - **Services:** Python3, Scapy, DNS Spoofing / Race Condition Exploit Scripts
5. **`AUTH-DNS` (Authoritative DNS Server cho domain `.test`)**
   - **Network:** `172.16.6.20/24`
   - **Services:** `auth_dns_delayed.py` (Mô phỏng phản hồi DNS có chủ đích với độ trễ 1.5s để tạo Race Window)

---

## ⚙️ 4. Nội Dung Cấu Hình & Kết Quả Kiểm Thử

### 🐧 A. Cấu Hình pfSense Firewall & Mạng ATTACKNET / DNS-LAB
- **Gán Interface OPT4:** Đặt tên mô tả `ATTACKNET`, cấu hình IP tĩnh `10.10.0.1/24`.
- **Chính sách tường lửa cô lập (Firewall Rules trên WebGUI pfSense):**
  - **Tab DNSLAB Rule 1:** Cho phép `10.9.0.100` (VICTIM) gửi truy vấn DNS (Port 53 TCP/UDP) tới `10.9.0.10` (DNS-SERVER).
  - **Tab DNSLAB Rule 2:** Cho phép `10.9.0.10` (DNS-SERVER) gửi truy vấn đệ quy (Port 53 UDP) tới `172.16.6.20` (AUTH-DNS).
  - **Tab ATTACKNET Rule 1:** Cho phép `10.10.0.200` (ATTACKER) kích hoạt truy vấn đệ quy hoặc gửi các gói tin tấn công giả mạo tới `10.9.0.10` (DNS-SERVER).
  - **Reset States:** Thực hiện **Diagnostics > States > Reset States** sau khi Apply thay đổi.

### 🛠️ B. Cấu Hình BIND9 Trên Ubuntu Server (`DNS-SERVER`)
- Cài đặt và cấu hình dịch vụ BIND9 phân giải đệ quy cho domain thử nghiệm `example.lab10.test`.
- Cấu hình cơ chế nới lỏng tạm thời kiểm tra cổng nguồn (Source Port) để phục vụ việc thử nghiệm mô phỏng tấn công từ xa.
- Kiểm tra trạng thái hoạt động của BIND9 qua `sudo named-checkconf` và `sudo ss -lunp | grep :53`.

### 🎯 C. Kịch Bản Thực Hiện Tấn Công Remote DNS Cache Poisoning
1. **Khởi tạo chuỗi Subdomain ngẫu nhiên (Bypass Cache Effect):**
   - Attacker liên tục gửi các yêu cầu phân giải tên miền dạng `randomXXXX.example.lab10.test` tới `DNS-SERVER` nhằm ép Resolver luôn phải thực hiện truy vấn đệ quy sang `AUTH-DNS` (`172.16.6.20`), tạo ra cửa sổ thời gian tấn công (Race Window).
2. **Thực thi tấn công giả mạo đệ quy (Race Condition Spoofing):**
   - Đồng thời gửi hàng loạt gói tin phản hồi DNS giả mạo (Forged Responses) từ dải `ATTACKNET` (`10.10.0.200`) về `DNS-SERVER