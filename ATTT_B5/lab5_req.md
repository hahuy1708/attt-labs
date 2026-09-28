# Lab 5 — Context-Based Access Control (CBAC) Firewall

Môn học: An Toàn Thông Tin (ATTT). Tài liệu tham khảo: *CCNA Security 04 — Firewall* (slide CBAC, tr. 55–71). Topology/IP lấy lại từ **Lab 3 — ACL**.

## 1. Mục tiêu đề bài

Triển khai một **tường lửa nâng cao (CBAC)** trên router **East**:

- **East** cho phép các máy chủ trong mạng nội bộ (LAN3) truy cập tài nguyên bên ngoài (LAN1, phía sau West).
- **East** chặn các máy chủ bên ngoài chủ động khởi tạo kết nối vào tài nguyên nội bộ (LAN3).
- Sau khi cấu hình xong, xác minh chức năng tường lửa từ cả hai phía (nội bộ và bên ngoài).

## 2. Topology & bảng địa chỉ IP (Lab 3 — ACL)

| Router  | Interface | IP Address       | Kết nối tới                |
|---------|-----------|------------------|------------------------------|
| Gateway | F0/0      | 16.19.16.19/24   | Internet (Cloud/NAT)        |
| Gateway | S0/0      | 172.16.3.2/24    | West S0/0                   |
| Gateway | S0/1      | 172.16.4.2/24    | East S0/1                   |
| West    | S0/0      | 172.16.3.1/24    | Gateway S0/0                |
| West    | F0/0      | 10.10.1.1/24     | LAN1 — 10.10.1.0/24         |
| West    | F0/1      | 10.10.2.1/24     | LAN2 — 10.10.2.0/24         |
| East    | S0/1      | 172.16.4.1/24    | Gateway S0/1                |
| East    | F0/0      | 10.10.3.1/24     | LAN3 — 10.10.3.0/24         |
| East    | F0/1      | 10.10.4.1/24     | LAN4 — 10.10.4.0/24         |

### Ánh xạ vai trò giữa đề CBAC ↔ topology ACL

| Tên trong đề CBAC | Chính là | IP/Subnet |
|---|---|---|
| Router East | East | S0/1 = 172.16.4.1 (cổng ngoài), F0/0 = 10.10.3.1 (cổng nội bộ) |
| Router Gateway | Gateway | S0/1 = 172.16.4.2 (phía East), S0/0 = 172.16.3.2 (phía West) |
| LAN3 Server | Server trong LAN3 (10.10.3.0/24) — **mạng nội bộ cần bảo vệ**, đã có sẵn server IIS từ Lab 3 | vd. 10.10.3.10 |
| LAN1 Server | Server trong LAN1 (10.10.1.0/24, phía sau West) — **đại diện "tài nguyên bên ngoài"** mà LAN3 được phép truy cập ra | vd. 10.10.1.10 |

## 3. ⚠️ LƯU Ý TRƯỚC khi bắt đầu

- Lab 3 (ACL): *"LAN1 và LAN4 có thể dùng cổng loopback (không cần server thật)"* — nghĩa là ở Lab 3, LAN1 **chưa hề có server thật**, chỉ là loopback giả lập mạng con.

- CBAC (Lab 5):
   - *"Từ LAN3 Server, mở trình duyệt web đến LAN1 Server để hiển thị trang web"* (Nhiệm vụ 1, Bước 1)
   - Test lại HTTP thành công LAN3 → LAN1 sau khi bật CBAC (Nhiệm vụ 2)

→ **LAN1 bắt buộc phải có một server thật chạy dịch vụ Web (IIS)** cho Lab 5, không thể chỉ dùng loopback như Lab 3 nữa.

**Việc cần làm:** Trước khi cấu hình CBAC, dựng thêm 1 VM/server có IIS trong LAN2 hoặc gắn thẳng vào LAN1 (10.10.1.0/24), gán IP tĩnh (vd. `10.10.1.10/24`, gateway `10.10.1.1`), bật dịch vụ Web trên đó — tương tự cách đã làm với server LAN2/LAN3 ở Lab 3.

> LAN2 (đã có server IIS từ Lab 3) **không được nhắc tới trong đề CBAC** — không cần dùng tới trong lab này, cứ để nguyên trạng thái từ Lab 3.

## 4. Lý thuyết nền: CBAC là gì

CBAC (Context-Based Access Control) là tính năng tường lửa có trạng thái (stateful) trên Cisco IOS Firewall, cung cấp 4 chức năng chính:

- **Traffic Filtering** — lọc gói TCP/UDP dựa trên thông tin phiên ở tầng ứng dụng, không chỉ dựa vào IP/port tĩnh như ACL thường.
- **Traffic Inspection** — theo dõi trạng thái thiết lập kết nối TCP, số thứ tự (sequence number), truy vấn DNS, các loại thông điệp ICMP phổ biến, và các ứng dụng dùng nhiều kênh (FTP, multimedia).
- **Intrusion Detection** — phát hiện bất thường trong luồng traffic đã được theo dõi.
- **Generation of Audits and Alerts** — sinh log cảnh báo và nhật ký kiểm toán truy cập.

### Cơ chế hoạt động (ví dụ Telnet đi ra ngoài)

1. ACL đầu vào trên cổng nội bộ (F0/0 của East) cho phép gói Telnet đi ra.
2. IOS đối chiếu loại gói với inspection rule để quyết định có theo dõi (track) phiên này không.
3. Thêm thông tin trạng thái (state table) để theo dõi phiên.
4. Thêm một **entry động (dynamic ACL entry)** vào ACL đầu vào của cổng ngoài (S0/1 của East) — chỉ cho phép đúng gói phản hồi của phiên đó quay lại.
5. Khi phiên kết thúc, router tự xóa cả state entry lẫn dynamic ACL entry.

→ Đây chính là lý do CBAC "nâng cao" hơn ACL tĩnh: **ACL động chỉ mở đúng lúc có phiên hợp lệ đi ra, tự đóng lại khi phiên kết thúc**, thay vì phải mở cố định 1 dải port như ACL thường ở Lab 3.

### Quy trình 4 bước cấu hình CBAC

| Bước | Nội dung | Áp dụng cho lab này |
|---|---|---|
| 1 | Chọn interface áp CBAC | East, mô hình 2 interface: F0/0 (trong) và S0/1 (ngoài) |
| 2 | Cấu hình ACL mở rộng tại interface | ACL deny toàn bộ traffic từ ngoài, áp inbound trên S0/1 |
| 3 | Định nghĩa inspection rule | `ip inspect name` cho ICMP, Telnet, HTTP |
| 4 | Áp inspection rule vào interface | Áp outbound trên S0/1 (chiều traffic hợp lệ đi ra) |

### Cú pháp lệnh nền tảng

```
Router(config)# ip inspect name <inspection_name> <protocol> [alert {on|off}] [audit-trail {on|off}] [timeout seconds]
```

## 5. Nhiệm vụ 1 — Chặn lưu lượng từ bên ngoài

### Bước 1: Xác minh kết nối mạng cơ bản (trước khi bật firewall)

- [ ] Từ **LAN3 Server** (vd. 10.10.3.10), `ping` đến **LAN1 Server** (vd. 10.10.1.10) → phải thành công.
- [ ] Từ **LAN3 Server**, `telnet` đến **Router Gateway** (172.16.3.2 hoặc 172.16.4.2, tùy cổng gõ `line vty`) → phải thành công, sau đó thoát phiên Telnet.
- [ ] Từ **LAN3 Server**, mở trình duyệt truy cập **LAN1 Server** → trang web phải hiển thị được, sau đó đóng trình duyệt.
- [ ] Từ **LAN1 Server**, `ping` đến **LAN3 Server** → phải thành công.

> Mục đích bước này: xác nhận **trước khi có CBAC**, traffic 2 chiều đều thông suốt (đúng như đã kiểm chứng ở Bước 4, Lab 3) — để sau này so sánh, chứng minh chính CBAC là nguyên nhân gây thay đổi hành vi, không phải lỗi định tuyến.

### Bước 2: Cấu hình ACL mở rộng trên East — chặn toàn bộ traffic từ bên ngoài

Tạo 1 extended ACL trên **East**, deny toàn bộ traffic có nguồn từ mạng ngoài (172.16.4.0/24, 172.16.3.0/24, 10.10.1.0/24, 10.10.2.0/24) hướng vào LAN3 (10.10.3.0/24), permit các traffic cần thiết khác (routing RIPv2 nếu cần).

### Bước 3: Áp ACL vào cổng Serial

Áp ACL vừa tạo theo chiều **inbound** trên **S0/1 của East** (cổng nối ra phía Gateway).

### Bước 4: Xác nhận traffic bị chặn

- [ ] Từ **LAN3 Server**, `ping` đến **LAN1 Server** → phản hồi ICMP echo phải **bị chặn** (khác hẳn kết quả ở Bước 1).

## 6. Nhiệm vụ 2 — Tạo quy tắc kiểm tra CBAC

### Bước 1: Tạo inspection rule cho ICMP, Telnet, HTTP

Dùng đúng cú pháp `ip inspect name` (mục 4) — tạo 1 rule bao gồm cả 3 protocol: `icmp`, `telnet`, `http`.

### Bước 2: Bật ghi log có dấu thời gian + audit-trail

Dùng tham số `audit-trail on` trong `ip inspect name`, kết hợp bật timestamp trong log hệ thống (`service timestamps log datetime`) để có bản ghi đầy đủ thời gian truy cập qua tường lửa, gồm cả các nỗ lực truy cập trái phép.

### Bước 3: Áp inspection rule vào chiều đi ra (outbound) trên cổng Serial

Áp trên **S0/1 của East**, chiều **out**.

> Lưu ý phân biệt chiều: ACL (Nhiệm vụ 1) áp **inbound** trên S0/1 để chặn mặc định; inspection rule (Nhiệm vụ 2) áp **outbound** trên cùng S0/1 để theo dõi traffic hợp lệ đi ra và tự mở lỗ hổng tạm thời cho traffic phản hồi.

### Kiểm tra sau khi áp inspection rule

- [ ] Từ **LAN3 Server** → **LAN1 Server**: `ping`, `Telnet`, `HTTP` — **tất cả đều phải thành công**.
  - Riêng lưu ý: bản thân **LAN1 Server sẽ từ chối phiên Telnet** (do cấu hình dịch vụ Telnet trên Server 2003, không liên quan CBAC) — đây là hành vi mong đợi, không phải lỗi.
- [ ] Từ **LAN1 Server** → **LAN3 Server**: `ping`, `Telnet` — **tất cả đều phải bị chặn** (vì traffic khởi tạo từ bên ngoài, CBAC không mở dynamic ACL cho chiều này).

## 7. Nhiệm vụ 3 — Xác minh chức năng tường lửa

### Bước 1: Kiểm tra phiên Telnet qua `show ip inspect sessions`

1. Từ **LAN3 Server**, mở phiên Telnet đến **Router Gateway** (phải thành công).
2. Trong lúc phiên đang mở, chạy trên **East**:
   ```
   East# show ip inspect sessions
   ```
3. Ghi lại và trả lời:
   - Địa chỉ IP nguồn và số cổng (source IP:port) là gì?
   - Địa chỉ IP đích và số cổng (destination IP:port) là gì?
4. Thoát phiên Telnet.

### Bước 2: Kiểm tra phiên HTTP qua `show ip inspect sessions`

1. Từ **LAN3 Server**, mở trình duyệt truy cập trang web **LAN1 Server** bằng địa chỉ IP (phải thành công).
2. Trong lúc phiên HTTP đang mở, chạy trên **East**:
   ```
   East# show ip inspect sessions
   ```
3. Ghi lại và trả lời tương tự: source IP:port, destination IP:port.
4. Đóng trình duyệt trên LAN3 Server.

### Bước 3: Kiểm tra tổng quan interface

```
East# show ip inspect interfaces
```

Lệnh này hiển thị các interface đang áp CBAC và các phiên hiện đang được theo dõi.

## 8. Nhiệm vụ 4 — Xem lại cấu hình CBAC

### Bước 1: Xem toàn bộ cấu hình inspection

```
East# show ip inspect config
```

### Bước 2: Debug chi tiết sự kiện CBAC xử lý

```
East# debug ip inspect detailed
```

> Lưu ý an toàn khi dùng `debug`: lệnh này có thể sinh log rất nhiều nếu traffic lớn — nhớ `undebug all` (hoặc `no debug ip inspect detailed`) sau khi quan sát xong để tránh treo CPU router trong môi trường lab kéo dài.

## 9. Bảng tổng hợp lệnh chính dùng trong lab

| Lệnh | Mục đích |
|---|---|
| `access-list <n> deny/permit ...` | Tạo ACL mở rộng chặn traffic ngoài (Nhiệm vụ 1) |
| `interface Serial1/1` → `ip access-group <n> in` | Áp ACL inbound lên S0/1 của East |
| `ip inspect name <name> icmp` <br> `ip inspect name <name> telnet` <br> `ip inspect name <name> http` | Định nghĩa inspection rule cho từng giao thức |
| `ip inspect name <name> ... audit-trail on` | Bật ghi log kiểm toán CBAC |
| `interface Serial1/1` → `ip inspect <name> out` | Áp inspection rule outbound trên S0/1 |
| `show ip inspect sessions` | Xem phiên đang được CBAC theo dõi (source/destination IP:port) |
| `show ip inspect interfaces` | Xem interface nào đang áp CBAC |
| `show ip inspect config` | Xem toàn bộ cấu hình CBAC hiện tại |
| `debug ip inspect detailed` | Xem chi tiết sự kiện CBAC xử lý theo thời gian thực |

## 10. Checklist chuẩn bị trước khi cấu hình

- [ ] Topology + IP đã kế thừa nguyên vẹn từ Lab 3 (mục 2) — không cần dựng lại.
- [ ] **Thêm server IIS thật vào LAN1** (chưa có ở Lab 3, xem mục 3) — bắt buộc cho phần test HTTP của lab này.
- [ ] Xác nhận RIPv2 vẫn đang chạy đúng, định tuyến full-mesh giữa các LAN còn thông (kiểm tra lại `show ip route` trên East/Gateway/West trước khi bắt đầu Nhiệm vụ 1 — đề bài giả định "đã định tuyến thành công").
- [ ] ACL từ Lab 3 (permit FTP, deny khác giữa LAN2↔LAN3) **vẫn giữ nguyên, không xóa** — CBAC lab này là một lớp cấu hình bổ sung trên East, tách biệt khỏi ACL đã áp cho LAN2↔LAN3 ở Lab 3.