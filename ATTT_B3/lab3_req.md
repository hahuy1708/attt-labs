# Lab 3 — Tóm tắt đề bài & yêu cầu

## 1. Đề bài

1. Sử dụng GNS3 để dựng cơ sở hạ tầng như sơ đồ mạng đã cho. Giao thức định tuyến sử dụng là **RIPv2**.
2. Triển khai **Access Control List (ACL)** thực hiện các yêu cầu sau:
   - Cho phép **FTP** giữa LAN2 và LAN3.
   - Chặn **các kết nối khác** giữa LAN2 và LAN3.
   - Cho phép **các kết nối còn lại** (không thuộc 2 trường hợp trên).
3. Ghi chú: dịch vụ Web/FTP dùng **IIS trên Server 2003** đặt tại LAN2 và LAN3. LAN1 và LAN4 có thể dùng cổng loopback (không cần server thật).

## 2. Yêu cầu bổ sung

- Gắn thêm **2 server** vào LAN2 và LAN3 để kiểm tra chặn/cho phép 3 dịch vụ tiêu biểu: **Ping (ICMP)**, **Web (HTTP)**, **FTP**.
- Gán IP phù hợp cho mỗi server theo đúng subnet của LAN2 (10.10.2.0/24) và LAN3 (10.10.3.0/24).
- **Kỳ vọng kết quả** sau khi ACL cấu hình đúng — từ PC ở LAN2 thử truy cập server LAN3 (và ngược lại):
  - ❌ Không ping được (ICMP bị chặn).
  - ❌ Không truy cập Web được (HTTP bị chặn).
  - ✅ Truy cập FTP được (FTP được cho phép).

## 3. Sơ đồ mạng & bảng địa chỉ IP

| Router  | Interface | IP Address       | Kết nối tới                |
|---------|-----------|------------------|------------------------------|
| Gateway | F0/0      | 16.19.16.19/24   | Internet (Cloud/NAT)        |
| Gateway | S0/0      | 172.16.3.2/24    | West S0/0                   |
| Gateway | S0/1      | 172.16.4.2/24    | East S0/1                   |
| West    | S0/0      | 172.16.3.1/24    | Gateway S0/0                |
| West    | F0/0      | 10.10.1.1/24 *   | LAN1 — 10.10.1.0/24         |
| West    | F0/1      | 10.10.2.1/24 *   | LAN2 — 10.10.2.0/24         |
| East    | S0/1      | 172.16.4.1/24    | Gateway S0/1                |
| East    | F0/0      | 10.10.3.1/24 *   | LAN3 — 10.10.3.0/24         |
| East    | F0/1      | 10.10.4.1/24 *   | LAN4 — 10.10.4.0/24         |

`*` Sơ đồ gốc chỉ cho địa chỉ mạng của LAN1–4, không cho IP cổng cụ thể — gán `.1` làm gateway cho mỗi LAN.

## 4. Checklist các bước tổng thể

- [ ] Bước 1: Dựng topology trong GNS3 — 3 router (Gateway, West, East) + 4 switch (LAN1–4) + Internet cloud
- [ ] Bước 2: Gán IP cho tất cả interface theo bảng ở mục 3
- [ ] Bước 3: Cấu hình RIPv2 trên Gateway, West, East (`version 2`, `no auto-summary`)
- [ ] Bước 4: Kiểm tra định tuyến — ping full-mesh giữa 4 LAN (`show ip route`, `show ip protocols`)
- [ ] Bước 5: Gắn Server 2003 (IIS) vào LAN2 và LAN3, bật dịch vụ Web (HTTP) + FTP
- [ ] Bước 6: Viết ACL trên router phù hợp: permit FTP LAN2↔LAN3, deny các kết nối khác giữa LAN2↔LAN3, permit mọi kết nối còn lại
- [ ] Bước 7: Áp ACL vào đúng interface (`ip access-group ... in/out`), kiểm tra lại bằng ping/Web/FTP từ PC LAN2 và LAN3