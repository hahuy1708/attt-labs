# Lab 6 — Zone-Based Policy Firewall (ZPF) trên Router East

Tái sử dụng topology từ **Lab 3** (GNS3, RIPv2, 3 router Gateway/West/East, 4 LAN, Server 2003 IIS ở LAN2/LAN3). Bài này cấu hình **ZPF** — kiến trúc tường lửa mới của Cisco (khác CBAC ở Lab 5, vốn dùng `ip inspect` trực tiếp trên interface) — chỉ trên router **East**.

## 1. Mục tiêu

- Cho phép máy chủ **nội bộ** (LAN3, phía sau East) truy cập tài nguyên **bên ngoài** (LAN1, phía sau West qua Gateway).
- Chặn máy chủ **bên ngoài** chủ động khởi tạo kết nối vào **nội bộ** (LAN3).
- Xác minh hành vi tường lửa từ cả hai phía.

## 2. Tạo project Lab 6 từ Lab 3 (bằng Save Project As)

Dùng tính năng **Save Project As** của GNS3

1. Mở project **Lab 3** trong GNS3 (đảm bảo đã **Stop** toàn bộ node trước khi save, tránh lỗi ghi file khi đang chạy).
2. Vào menu **File → Save Project As...**
3. Đặt tên project mới là `lab6` — GNS3 tự tạo một **thư mục project hoàn toàn riêng** (`lab6/project-files/...`), sao chép lại toàn bộ cấu hình router (startup-config của Gateway/West/East), file `.gns3`, và layout topology y hệt Lab 3.
4. GNS3 tự động mở project mới (`lab6`) sau khi save — kiểm tra lại thanh tiêu đề cửa sổ để chắc chắn đang làm việc trên `lab6`, không phải `lab3`.
5. Start lại toàn bộ node trong `lab6` — xác nhận Gateway, West, East, Server2003R2-1 (LAN2), Server2003R2-2 (LAN3) vẫn mang đúng cấu hình/IP như Lab 3.

> **Lưu ý về 2 server có sẵn (LAN2, LAN3):** `Save Project As` chỉ sao chép phần cấu hình **router** (Dynamips) một cách độc lập hoàn toàn. Với node VM (VMware), bản chất GNS3 tham chiếu tới **cùng một linked-clone VM** đã tạo trong VMware Workstation inventory. Nên kiểm tra lại: nếu bạn sửa gì trên Server2003R2-1/2 trong project `lab6`, thay đổi đó **có thể ảnh hưởng ngược về Lab 3 gốc** (và cả Lab 5, nếu Lab 5 cũng dùng chung VM đó). Với bài này, vì không cần sửa gì trên 2 server LAN2/LAN3 có sẵn (ZPF chỉ tác động tới LAN1↔LAN3, không đụng LAN2), rủi ro này không phát sinh trong thực tế — chỉ cần biết để tránh nhầm lẫn nếu sau này chỉnh sửa gì trên các server dùng chung.

## 3. Ánh xạ vào topology

| Khái niệm trong đề | Chính là | IP/Interface |
|---|---|---|
| Router EAST | East | F0/0 = `10.10.3.1` (LAN3, nội bộ), S0/1 = `172.16.4.1` (phía Gateway, bên ngoài) |
| LAN3 SERVER (nội bộ) | Server trong LAN3, có sẵn từ Lab 3 | `10.10.3.100/24` |
| LAN1 SERVER (bên ngoài) | Server mới tạo riêng cho Lab 6, gắn vào Switch1 | **`10.10.1.100/24`**, gateway `10.10.1.1` |
| Router GATEWAY | Gateway | Đại diện thêm cho "bên ngoài" khi test Telnet/ping |

> IP `10.10.1.100` được chọn **khác với IP `10.10.1.10` dùng ở Lab 5** — mục đích để 2 server LAN1 của 2 lab độc lập hoàn toàn với nhau, dễ phân biệt khi so sánh log/kết quả giữa 2 lab, tránh nhầm lẫn IP khi đối chiếu báo cáo.

⚠️ **F0/1 của East (LAN4) không được gán vào zone nào trong bài này** — không ảnh hưởng gì, traffic qua interface không thuộc zone nào vẫn hoạt động bình thường như trước, không bị ZPF kiểm soát.

## 4. Thêm server IIS mới vào LAN1 (riêng cho Lab 6)

Vì project `lab6` được tạo từ **Lab 3** (chưa từng có server LAN1), bước này bắt buộc làm mới, độc lập với server LAN1 đã dựng ở Lab 5:

1. Thêm 1 node VM Server 2003 mới (dùng template linked-clone sẵn có trong GNS3), đặt tên vd. `Server2003R2-3-Lab6`, nối vào **Switch1** (LAN1).
2. Start node, gán IP tĩnh trong Windows:
   ```
   IP: 10.10.1.100
   Subnet mask: 255.255.255.0
   Gateway: 10.10.1.1
   ```
3. Cài IIS: Add/Remove Windows Components → Application Server → Internet Information Services (IIS) → tick **World Wide Web Service**.
4. Administrative Tools → IIS Manager → xác nhận **Default Web Site** đang **Start**.
5. Test `ping 10.10.1.1` từ server để xác nhận đã thông trước khi qua bước tiếp theo.

## 5. Thêm VTY password cho Gateway (nếu Lab 3 gốc chưa có)

Vì project `lab6` copy nguyên trạng cấu hình Lab 3, nếu Lab 3 gốc chưa từng cấu hình Telnet, bước Telnet ở Nhiệm vụ 6 sẽ báo lỗi `Password required, but none set. Connection to host lost.` — lỗi này đến từ IOS từ chối Telnet khi VTY chưa có password, không liên quan gì tới ZPF.

```
enable
configure terminal
line vty 0 4
password cisco
login
end
write memory
```

## 6. Nhiệm vụ 1 — Xác minh kết nối trước khi cấu hình ZPF

Thực hiện và xác nhận cả 3 đều **thành công** (baseline — vì East trong project `lab6` chưa có firewall nào, nên mặc định phải thông hết):

- [ ] Từ **LAN1 Server** (`10.10.1.100`), `ping` tới **LAN3 Server** (`10.10.3.100`)
- [ ] Từ **LAN3 Server**, `telnet` tới **GATEWAY** (`172.16.3.2` hoặc `172.16.4.2`), nhập password vty, rồi thoát phiên
- [ ] Từ **LAN3 Server**, mở trình duyệt tới IP **LAN1 Server** (`10.10.1.100`) — phải thấy trang web mặc định IIS

## 7. Cấu hình ZPF trên router East

### 7.1. Toàn bộ lệnh (chạy liên tiếp)

```
enable
configure terminal

zone security IN-ZONE
exit

zone security OUT-ZONE
exit

access-list 101 permit ip 10.10.3.0 0.0.0.255 any

class-map type inspect match-all IN-NET-CLASS-MAP
 match access-group 101
exit

policy-map type inspect IN-2-OUT-PMAP
 class type inspect IN-NET-CLASS-MAP
  inspect
 exit
exit

zone-pair security IN-2-OUT-ZPAIR source IN-ZONE destination OUT-ZONE
 service-policy type inspect IN-2-OUT-PMAP
exit

interface FastEthernet0/0
 zone-member security IN-ZONE
exit

interface Serial0/1
 zone-member security OUT-ZONE
exit

end
write memory
```

### 7.2. Giải thích từng khối lệnh

| Lệnh | Ý nghĩa |
|---|---|
| `zone security IN-ZONE` / `OUT-ZONE` | Tạo 2 "vùng" logic — IN-ZONE đại diện mạng nội bộ (LAN3), OUT-ZONE đại diện bên ngoài |
| `access-list 101 permit ip 10.10.3.0 0.0.0.255 any` | Định nghĩa "traffic nào được coi là từ nội bộ đi ra" — mọi traffic nguồn từ LAN3 |
| `class-map type inspect match-all ...` | Gom ACL 101 thành một "lớp traffic" có tên, dùng `match-all` vì chỉ có 1 điều kiện (ACL) cần khớp |
| `policy-map type inspect ...` + `class type inspect ...` + `inspect` | Định nghĩa chính sách: traffic thuộc class này sẽ được **inspect** (theo dõi trạng thái phiên, tự mở lỗ hổng tạm cho traffic phản hồi) |
| `zone-pair security ... source IN-ZONE destination OUT-ZONE` | Khai báo: traffic đi từ IN-ZONE → OUT-ZONE sẽ áp dụng chính sách nào |
| `service-policy type inspect IN-2-OUT-PMAP` | Gắn chính sách vừa tạo vào đúng cặp zone-pair này |
| `zone-member security IN-ZONE` trên F0/0 | Đưa cổng LAN3 vào vùng nội bộ |
| `zone-member security OUT-ZONE` trên S0/1 | Đưa cổng nối Gateway vào vùng bên ngoài |

Khi gõ `inspect` trong policy-map, IOS in ra:
```
%No specific protocol configured in class IN-NET-CLASS-MAP for inspection. All protocols will be inspected.
```
— **bình thường**, không phải lỗi. Vì ACL 101 permit mọi protocol IP, ZPF sẽ inspect tất cả thay vì chỉ riêng HTTP/FTP/Telnet.

> **Lưu ý:** không có zone-pair nào theo chiều ngược lại (OUT-ZONE → IN-ZONE) được tạo — đây chính là lý do traffic từ bên ngoài khởi tạo vào sẽ bị **DROP mặc định** (hành vi mặc định của ZPF: không có zone-pair tường minh cho chiều nào thì chiều đó bị chặn hoàn toàn).

## 8. Nhiệm vụ 6 — Test chiều IN-ZONE → OUT-ZONE (phải THÀNH CÔNG)

- [ ] Từ **LAN3 Server**: `ping 10.10.1.100` (LAN1 Server) → phải thành công
- [ ] Từ **LAN3 Server**: `telnet <IP GATEWAY>` → phải thành công. Trong lúc phiên đang mở, trên **East** chạy:
  ```
  show policy-map type inspect zone-pair sessions
  ```
  Ghi lại **IP:port nguồn** và **IP:port đích**. Sau đó thoát Telnet.
- [ ] Từ **LAN3 Server**: mở trình duyệt tới `http://10.10.1.100` → phải load được trang IIS. Trong lúc phiên HTTP đang mở, chạy lại lệnh trên, ghi lại IP:port nguồn/đích lần này. Đóng trình duyệt sau khi xong.

### Kết quả mẫu đã xác minh thành công (dùng làm bằng chứng khi nộp bài)

```
east# show policy-map type inspect zone-pair sessions
 Zone-pair: IN-2-OUT-ZPAIR

  Service-policy inspect : IN-2-OUT-PMAP

    Class-map: IN-NET-CLASS-MAP (match-all)
      Match: access-group 101
      Inspect
        Established Sessions
         Session 671D778C (10.10.3.100:1053)=>(172.16.4.2:23) telnet SIS_OPEN
          Created 00:00:42, Last heard 00:00:39
          Bytes sent (initiator:responder) [38:70]
         Session 671D7A54 (10.10.3.100:1054)=>(10.10.1.100:80) http SIS_OPEN
          Created 00:00:30, Last heard 00:00:30
          Bytes sent (initiator:responder) [677:429]

    Class-map: class-default (match-any)
      Match: any
      Drop (default action)
        0 packets, 0 bytes
east#
```

**Cách đọc để trả lời câu hỏi "IP:port nguồn/đích là gì":**

| Phiên | Source IP:Port (nguồn — bên khởi tạo) | Destination IP:Port (đích — bên cung cấp dịch vụ) |
|---|---|---|
| Telnet | `10.10.3.100:1053` (LAN3 Server, cổng ephemeral do Windows cấp) | `172.16.4.2:23` (Gateway, cổng Telnet chuẩn) |
| HTTP | `10.10.3.100:1054` (LAN3 Server, cổng ephemeral) | `10.10.1.100:80` (LAN1 Server, cổng HTTP chuẩn) |

Dòng `Class-map: class-default (match-any) ... Drop (default action)` ở cuối xác nhận: **mọi traffic không khớp ACL 101** (tức không có nguồn từ 10.10.3.0/24) sẽ bị **drop mặc định** — đúng đặc tính "implicit deny" của ZPF.

## 9. Nhiệm vụ 7 — Test chiều OUT-ZONE → IN-ZONE (phải THẤT BẠI)

- [ ] Từ **LAN1 Server** (`10.10.1.100`): `ping 10.10.3.100` (LAN3 Server) → phải **thất bại** (Request timed out)
- [ ] Từ **router GATEWAY**: `ping 10.10.3.100` → phải **thất bại** tương tự

Nếu cả 2 test này fail đúng như mong đợi → ZPF đã hoạt động chính xác theo yêu cầu đề bài.

## 10. Lệnh kiểm tra/khắc phục hữu ích

| Lệnh | Mục đích |
|---|---|
| `show zone security` | Xem danh sách zone đã tạo, interface nào thuộc zone nào |
| `show zone-pair security` | Xem zone-pair đã cấu hình, policy-map đang gắn |
| `show policy-map type inspect zone-pair sessions` | Xem các phiên đang được ZPF theo dõi (dùng ở Nhiệm vụ 6) |
| `show access-lists 101` | Xem ACL 101 có đúng số liệu, có hit counter tăng không khi test |
| `show running-config` | Xem lại toàn bộ cấu hình đã áp, đối chiếu với mục 7.1 |

Nếu test Nhiệm vụ 6 bị fail (đáng lẽ phải thành công): kiểm tra lại ACL 101 đúng subnet LAN3 chưa, và `zone-member security` đã gán đúng interface (F0/0 = IN, S0/1 = OUT) chưa — gán ngược 2 interface là lỗi phổ biến nhất khi làm ZPF.

## Phụ lục — So sánh với Lab 5 (CBAC)

| Tiêu chí | Lab 5 — CBAC | Lab 6 — ZPF (bài này) |
|---|---|---|
| Triết lý | Gắn vào Interface (Interface-Centric): ACL + `ip inspect` áp thẳng lên từng cổng | Gắn vào Vùng (Zone-Centric): nhóm interface cùng mức tin cậy vào 1 zone, chính sách gán giữa zone–zone |
| Hành vi mặc định khi chưa có luật | **Permit** — traffic được phép qua trừ khi có `deny` tường minh | **Deny** — ngay khi gán interface vào zone, mọi traffic liên-zone bị chặn tuyệt đối trừ khi có zone-pair + policy tường minh |
| Lệnh chính | `ip access-list`, `ip inspect name`, áp thẳng lên interface | `zone security`, `class-map type inspect`, `policy-map type inspect`, `zone-pair security`, `zone-member security` |
| Lệnh xem session | `show ip inspect sessions` | `show policy-map type inspect zone-pair sessions` (hiển thị lồng theo zone-pair → class-map → session) |
| Khả năng mở rộng | Phải viết thêm ACL/inspect chéo cho từng cặp interface mới — rối khi router nhiều cổng | Chỉ cần gán interface mới vào đúng zone sẵn có — chính sách tự áp dụng, không cần viết lại lệnh |

Vì Lab 5 và Lab 6 là **2 project GNS3 độc lập** (Lab 5 copy trước, Lab 6 copy sau — đều từ Lab 3 gốc, mỗi project có file `.gns3` và startup-config router riêng), router East ở mỗi project không ảnh hưởng tới nhau. Hai server LAN1 của 2 lab cũng dùng IP khác nhau (`10.10.1.10` ở Lab 5, `10.10.1.100` ở Lab 6) nên khi đối chiếu log/báo cáo giữa 2 lab, không bị nhầm lẫn nguồn gốc dữ liệu.