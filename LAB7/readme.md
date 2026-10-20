# HƯỚNG DẪN BÀI THỰC HÀNH LAB 7: CẤU HÌNH PROXY SERVER VÀ TƯỜNG LỬA PFSENSE

---

## 1. SƠ ĐỒ VÀ THÔNG SỐ MẠNG (TOPOLOGY)

| Thiết bị | Card mạng / Interface | Địa chỉ IP / Subnet | Default Gateway | Chức năng |
| :--- | :--- | :--- | :--- | :--- |
| **pfSense** | WAN (`em0`) | Nhận IP động (DHCP) | - | Tường lửa chính |
| **pfSense** | LAN (`em1`) | `10.6.0.1/24` | - | Gateway nội bộ |
| **Ubuntu Server** | `ens33` | `10.6.0.30/24` | `10.6.0.1` | Proxy Server (Squid) |
| **Windows Client A**| `Ethernet` | `10.6.0.21/24` | `10.6.0.1` | Máy trạm kiểm thử |
| **Windows Client B**| `Ethernet` | `10.6.0.22/24` | `10.6.0.1` | Máy trạm kiểm thử |

* **DNS Server:** `10.6.0.10`
* **Cấu hình Virtual Network trên VMware:** Tất cả các card LAN của pfSense, Ubuntu Server và Windows Client đều phải trỏ chung về một **VMnet** nội bộ (Custom VMnet).

---

## 2. BƯỚC 1: CẤU HÌNH MẠNG TRÊN UBUNTU SERVER (NETPLAN)

### 2.1 Cú pháp file cấu hình chuẩn
Mở file cấu hình bằng lệnh:
```bash
sudo nano /etc/netplan/50-cloud_init.yaml
```

Điền nội dung theo đúng cấu trúc YAML (thụt lề 2 khoảng trắng `Space`, **tuyệt đối không dùng phím Tab**):

```yaml
network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 10.6.0.30/24
      routes:
        - to: default
          via: 10.6.0.1
      nameservers:
        addresses: [10.6.0.10]
```

### 2.2 Sửa lỗi phân quyền và áp dụng cấu hình
```bash
# Phân quyền bảo mật file (khắc phục cảnh báo "Permissions are too open")
sudo chmod 600 /etc/netplan/50-cloud_init.yaml

# Áp dụng cấu hình mạng
sudo netplan apply

# Kiểm tra địa chỉ IP đã gán thành công
ip a
```
*Minh chứng:* Dòng `inet 10.6.0.30/24` xuất hiện dưới card `ens33`.

---

## 3. BƯỚC 2: CẤU HÌNH VÀ PHỤC HỒI PFSENSE CONSOLE

### 3.1 Gán lại cổng mạng (Assign Interfaces)
Tại màn hình Console của pfSense:
1. Nhập lựa chọn **`1`** (`Assign Interfaces`).
2. `Should VLANs be set up now?` -> Nhập **`n`**.
3. `WAN interface name` -> Nhập **`em0`**.
4. `LAN interface name` -> Nhập **`em1`**.
5. `OPT1 interface name` -> Nhập **`em2`**.
6. `OPT2 interface name` -> Nhập **`em3`**.
7. Nhấn **Enter** bỏ qua các tùy chọn tiếp theo và nhấn **`y`** để lưu.

### 3.2 Đặt địa chỉ IP tĩnh cho cổng LAN
1. Nhập lựa chọn **`2`** (`Set interface(s) IP address`).
2. Chọn số **`2`** (`LAN`).
3. `Enter the new LAN IPv4 address` -> Nhập **`10.6.0.1`**.
4. `Enter the new LAN IPv4 subnet bit count` -> Nhập **`24`**.
5. `Upstream gateway address` -> Nhấn **Enter** (để trống).
6. `LAN IPv6 address` -> Nhấn **Enter** (để trống).
7. `Enable DHCP server on LAN?` -> Nhập **`n`**.
8. `Revert to HTTP protocol?` -> Nhập **`y`**.

*Minh chứng:* Console xuất hiện đường dẫn `http://10.6.0.1/`.

---

## 4. BƯỚC 3: CẤU HÌNH VÀ KIỂM TRA TRÊN WINDOWS CLIENT

### 4.1 Cấu hình IP tĩnh trong máy ảo
1. Bên trong **máy ảo Windows Client**, nhấn `Windows + R`, gõ `ncpa.cpl`.
2. Nhấp chuột phải vào `Ethernet` -> **Properties** -> nhấp đúp **TCP/IPv4**.
3. Chọn **Use the following IP address**:
   * **IP address:** `10.6.0.21` (hoặc `10.6.0.22`)
   * **Subnet mask:** `255.255.255.0`
   * **Default gateway:** `10.6.0.1`
   * **Preferred DNS:** `10.6.0.10`

### 4.2 Kiểm tra kết nối mạng
Mở **Command Prompt (CMD)** trong máy ảo Client và chạy lệnh:
```cmd
ping 10.6.0.1
ping 10.6.0.30
```
*Yêu cầu:* Phải nhận phản hồi `Reply from ...` không bị timeout.

### 4.3 Đăng nhập WebGUI pfSense
1. Mở trình duyệt Web trên máy ảo Client.
2. Truy cập: `http://10.6.0.1`
3. Tài khoản mặc định:
   * **Username:** `admin`
   * **Password:** `pfsense`

---

## 5. BƯỚC 4: TẠO ALIAS VÀ FIREWALL RULES TRÊN PFSENSE (NỘI DUNG LAB 7)

### 5.1 Tạo Alias
1. Truy cập **Firewall** > **Aliases** > **IP**.
2. Thêm Alias cho danh sách máy Client:
   * **Name:** `CLIENT_LIST`
   * **Type:** Hosts
   * **IPs:** `10.6.0.21`, `10.6.0.22`
3. Thêm Alias cho Proxy Server:
   * **Name:** `PROXY_SERVER`
   * **Type:** Host
   * **IP:** `10.6.0.30`

### 5.2 Tạo Luật Tường Lửa (Firewall Rules)
1. Truy cập **Firewall** > **Rules** > **LAN**.
2. **Rule 1 (Cho phép HTTP/HTTPS tới Proxy):**
   * **Action:** Pass
   * **Interface:** LAN
   * **Protocol:** TCP
   * **Source:** `CLIENT_LIST`
   * **Destination:** `PROXY_SERVER`, Port: `8080` (hoặc port của Squid Proxy)
3. **Rule 2 (Chặn truy cập Web trực tiếp ra ngoài):**
   * **Action:** Block
   * **Interface:** LAN
   * **Protocol:** TCP
   * **Source:** `CLIENT_LIST`
   * **Destination:** Any, Port: `80`, `443`

---

## 6. NHỮNG LỖI THƯỜNG GẶP VÀ CÁCH XỬ LÝ (TROUBLESHOOTING)

| Hiện tượng lỗi | Nguyên nhân | Cách khắc phục |
| :--- | :--- | :--- |
| `expected sequence - 10.6.0.30/24` | Thiếu khoảng trắng sau dấu gạch ngang trong Netplan. | Sửa thành `- 10.6.0.30/24` (có dấu cách). |
| `expected mapping (check indentation)` | Thụt lề sai giữa `nameservers:` và `addresses:`. | Dùng phím Space đẩy `addresses:` lùi vào trong `nameservers:`. |
| `inconsistent indentation` | Dùng phím Tab hoặc số lượng dấu cách không đều. | Xóa hết khoảng trắng đầu dòng và thụt lề bằng phím Space (mỗi cấp 2 space). |
| `Permissions for ... are too open` | Cảnh báo phân quyền file Netplan chưa an toàn. | Chạy lệnh `sudo chmod 600 /etc/netplan/50-cloud_init.yaml`. |
| `This IP address conflicts with another interface` | IP `10.6.0.1` đang dán ở cổng WAN nên không đặt được cho LAN. | Chuyển cổng WAN sang nhận DHCP trước, sau đó mới đặt IP `10.6.0.1` cho LAN. |
| `Site can't be reached` khi vào `10.6.0.1` | 1. Máy Client còn bật Proxy.<br>2. Card mạng trên VMware chọn sai VMnet.<br>3. Thao tác nhầm trên máy thật. | 1. Tắt Manual Proxy trong Windows.<br>2. Kiểm tra Network Adapter của máy ảo cùng VMnet.<br>3. Thao tác cài IP tĩnh đúng bên trong máy ảo Client. |