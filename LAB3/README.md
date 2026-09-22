
# Báo cáo thực hành Lab 3

- **Họ và tên:** Trần Hồ Quang Vinh
- **MSSV:** 1150080082
- **Bài thực hành:** Lab 3 – Nhận diện và ứng phó các mối đe dọa đến an toàn thông tin

---

## 1. Môi trường thực hành
- **Hệ điều hành:** Windows 10/11 Pro trên VMware Workstation (Chế độ mạng: Host-only).
- **Công cụ sử dụng:**
  - Python 3.12.7
  - Wireshark / TShark 4.6.8
  - Sysmon 15.22
  - Autoruns 14.3
  - Process Explorer 17.14

---

## 2. Cách dựng môi trường
1. Tạo thư mục làm việc: `C:\LAB3\Downloads`, `C:\LAB3\Tools`, `C:\LAB3\Evidence`.
2. Cài đặt Python (tích chọn `Add to PATH`), Wireshark (Npcap) và giải nén bộ công cụ Sysinternals vào thư mục `Tools`.
3. Kích hoạt dịch vụ `Sysmon64`.
4. Chuyển card mạng máy ảo về `Host-only` để cách ly an toàn.
5. Xuất thông tin baseline ban đầu (OS, Defender, Firewall, Process, Network) vào thư mục `Evidence`.

---

## 3. Tiến độ và kết quả các tình huống
- **Baseline:** Thu thập trạng thái hệ thống trước khi thực hành — **PASS**
- **Tình huống 1 (TH1):** Lập Risk Register và phân loại nguồn đe dọa — **PASS**
- **Tình huống 2 (TH2):** Kiểm chứng phát hiện mã độc bằng chuỗi EICAR trên Windows Defender — **PASS**
- **Tình huống 3 (TH3):** Bật Audit Logon, tạo user `lab3user`, sinh sự kiện Event 4624/4625/4648 và đổi mật khẩu — **PASS**
- **Tình huống 4 (TH4):** Kiểm tra cơ chế persistence qua Registry và Scheduled Task — **PASS**
- **Tình huống 5 (TH5):** Bắt gói tin dịch vụ HTTP nội bộ bằng Wireshark — **PASS**

---

## 4. Lỗi gặp phải và cách khắc phục
- **Chạy nhầm cmd thay vì PowerShell:** Khi dùng lệnh `Get-ChildItem` bị báo lỗi không nhận lệnh; khắc phục bằng cách gõ `powershell` để chuyển shell.
- **Lỗi lệnh Auditpol:** Lỗi cú pháp `0x00000057` do tham số GUID không bọc dấu ngoặc kép; khắc phục bằng cách đặt tên subcategory dạng chuỗi: `auditpol /set /subcategory:"Logon" ...`.
- **Tải nhầm gói Python MSIX:** Tải nhầm tệp `python-manager.msix` không có CLI; khắc phục bằng cách tải lại bộ cài Windows installer `python-3.12.7-amd64.exe`.
