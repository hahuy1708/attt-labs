# Lab 8: Bảo mật Switch lớp 2 (Data Link Layer)

## Phần 1. Đề bài

### 1.1. Bối cảnh

Nhằm ngăn chặn nguy cơ tấn công từ bên trong mạng nội bộ (nhân viên biến chất, gián điệp), thường nhắm vào switch qua các đối tượng: địa chỉ MAC, bảng MAC và giao thức Spanning Tree. Cần triển khai bảo mật switch lớp 2.

### 1.2. Yêu cầu dựng mô hình

- Dùng module **NM-16ESW** để thêm 16 cổng switch vào router Cisco 3725 đã có.
- S1 và S2 là 2 router Cisco 3725 chạy chế độ switch với module 16 cổng.
- Triển khai sơ đồ vật lý, sau đó đặt địa chỉ IP theo bảng sau:

| Thiết bị | Giao diện | Địa chỉ IP | Subnet mask | Gateway | Cổng switch |
|---|---|---|---|---|---|
| R1 | F0/1 | 192.168.1.1 | 255.255.255.0 | N/A | S1 F0/5 |
| S1 | VLAN 1 | 192.168.1.2 | 255.255.255.0 | N/A | N/A |
| S2 | VLAN 1 | 192.168.1.3 | 255.255.255.0 | N/A | N/A |
| PC-A | NIC | 192.168.1.10 | 255.255.255.0 | 192.168.1.1 | S1 F0/6 |
| PC-B | NIC | 192.168.1.11 | 255.255.255.0 | 192.168.1.1 | S2 F0/18 |

### 1.3. Yêu cầu bảo mật

1. **SSH** đến được cả hai switch.
2. **Bảo mật STP**: PortFast, BPDU guard, storm control.
3. **Bảo mật cổng** với địa chỉ MAC chỉ định: port security, sticky MAC.

---

## Phần 2. Hướng dẫn thực hiện

### 2.1. Mục tiêu

- Hiểu vì sao cần bảo mật lớp 2 và các nhóm tấn công tương ứng: STP manipulation, LAN storm, MAC spoofing, MAC table overflow.
- Cấu hình được SSH, PortFast, BPDU guard, storm control trên switch chạy trên router.
- Kiểm tra và ghi lại bằng chứng cho từng yêu cầu.

### 2.2. Điều chỉnh so với đề (bắt buộc đọc)

| Vấn đề | Lý do | Cách xử lý |
|---|---|---|
| PC-B nối **Fa0/18** trong đề | NM-16ESW chỉ có 16 cổng, đánh số 0–15 | Dùng **Fa2/8** trên S2 và ghi chú trong báo cáo |
| Tên cổng trong GNS3 | Module ở slot 2 cho `Fa2/0–2/15`, không phải `Fa0/x` | Lập bảng ánh xạ (mục 2.3) |
| Cổng S1 nối R1 là F0/5 | Trong sơ đồ GNS3 nối vào Fa2/5 | Dùng **Fa2/5** |
| `switchport port-security` | Image IOS hiện tại báo `Unrecognized command` | Không cấu hình được, ghi nhận trong báo cáo (mục 2.7) |

### 2.3. Bảng ánh xạ cổng

| Thiết bị | Cổng GNS3 | Nối với | Vai trò | Bảo mật |
|---|---|---|---|---|
| R1 | Fa0/1 | S1 Fa2/5 | Gateway 192.168.1.1 | Không |
| S1 | Fa2/5 | R1 Fa0/1 | Uplink tới router | Không |
| S1 | Fa2/1 | S2 Fa2/1 | Liên kết S1–S2 | Không |
| S1 | Fa2/6 | PC-A | Cổng người dùng | PortFast, BPDU guard, storm control |
| S2 | Fa2/1 | S1 Fa2/1 | Liên kết S2–S1 | Không |
| S2 | Fa2/8 | PC-B | Cổng người dùng | PortFast, BPDU guard, storm control |

Chỉ đặt bảo mật lên cổng người dùng. Không đặt PortFast, BPDU guard hay port security lên cổng nối S1–S2, vì sẽ làm cổng đó bị shutdown.

### 2.4. Chuẩn bị

1. Tạo node **S1**, **S2** là router c3725 gắn NM-16ESW ở slot 2.
2. Nối theo bảng 2.3.
3. Start toàn bộ thiết bị.
4. Kiểm tra các lệnh có được hỗ trợ:

```
S1# configure terminal
S1(config)# interface fa2/6
S1(config-if)# switchport port-security ?
S1(config-if)# storm-control ?
S1(config)# spanning-tree portfast ?
```

Ghi lại kết quả. Đây là bằng chứng cho phần không thực hiện được.

### 2.5. Cấu hình nền: IP và kết nối

**R1:**
```
enable
configure terminal
hostname R1
interface fa0/1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
exit
end
write memory
```

**S1:**
```
enable
configure terminal
hostname S1
interface vlan 1
 ip address 192.168.1.2 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.1.1
end
write memory
```

**S2:**
```
enable
configure terminal
hostname S2
interface vlan 1
 ip address 192.168.1.3 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.1.1
end
write memory
```

**PC-A** (VPCS nối S1 Fa2/6):
```
ip 192.168.1.10 255.255.255.0 192.168.1.1
save
```

**PC-B** (VPCS nối S2 Fa2/8):
```
ip 192.168.1.11 255.255.255.0 192.168.1.1
save
```

Kiểm tra từ PC-A:
```
ping 192.168.1.1
ping 192.168.1.2
ping 192.168.1.3
ping 192.168.1.11
```
Cả bốn phải trả lời. Nếu không, kiểm tra `no shutdown`, tên cổng và dây nối.

### 2.6. Yêu cầu 1: SSH trên S1 và S2

Chạy trên **S1**, sau đó lặp lại trên **S2** với hostname tương ứng:

```
enable
configure terminal
hostname S1
ip domain-name lab.local
username admin privilege 15 secret Cisco123!
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 transport input ssh
 login local
exit
end
write memory
```

Lưu ý:
- `hostname` và `ip domain-name` phải đặt trước `crypto key generate rsa`, vì khóa mang tên `S1.lab.local`.
- Tạo khóa RSA trong Dynamips có thể mất vài chục giây.

**Kiểm tra:**

Từ R1:
```
R1# ssh -v 2 -l admin 192.168.1.2
```
Nhập `Cisco123!`. Khi vào được `S1>` thì gõ `exit`. Lặp lại với `192.168.1.3` cho S2.

Trên switch, ngay khi phiên còn mở hoặc chụp sau khi test:
```
show ip ssh
show ssh
```
`show ssh` phải có một phiên với trạng thái `Session started`. Nếu chỉ thấy `No SSHv2 server connections running`, nghĩa là chưa có phiên nào đang chạy. Hãy test lại SSH rồi chụp.

### 2.7. Yêu cầu 2: PortFast và BPDU guard

**Trên S1:**
```
enable
configure terminal
spanning-tree portfast bpduguard
interface fa2/6
 switchport mode access
 spanning-tree portfast
end
write memory
```

**Trên S2:**
```
enable
configure terminal
spanning-tree portfast bpduguard
interface fa2/8
 switchport mode access
 spanning-tree portfast
end
write memory
```

Lưu ý:
- Cú pháp đúng trên IOS này là `spanning-tree portfast bpduguard`. Không có từ khóa `default`.
- `switchport mode access` cần thiết để PortFast có tác dụng (cổng phải ở chế độ non-trunking).
- Nếu muốn bật BPDU guard trực tiếp trên cổng, thử thêm `spanning-tree bpduguard enable` trong `interface`. Nếu báo lỗi, bỏ qua, vì cấu hình toàn cục đã bảo vệ cổng PortFast.

**Kiểm tra:**
```
show running-config | include portfast|bpduguard
show spanning-tree interface fa2/6 portfast
show spanning-tree summary
```
Kỳ vọng: dòng `spanning-tree portfast bpduguard` xuất hiện trong cấu hình, cổng người dùng báo PortFast enabled.

### 2.8. Yêu cầu 2 (tiếp): Storm control

**Trên S1:**
```
enable
configure terminal
interface fa2/6
 storm-control broadcast level 20
 storm-control action shutdown
end
write memory
```

**Trên S2:** làm tương tự với `interface fa2/8`.

Lưu ý:
- Mức 20% là giá trị chọn cho lab. Ghi lại trong báo cáo.
- Nếu `level 20` báo lỗi, thử `level 20.00`.
- Lệnh `storm-control` đứng một mình rồi Enter không có tác dụng, và có thể chạy lại khối lệnh trên mà không xung đột.

**Kiểm tra:**
```
show storm-control broadcast
```
Kỳ vọng: mức 20% và action `shutdown` trên cổng người dùng.

Trong Dynamips rất khó tạo broadcast storm thật, nên phần này được chứng minh chủ yếu bằng cấu hình và lệnh `show`.

### 2.9. Yêu cầu 3: Port security với sticky MAC

**Kết quả: không thực hiện được trên image hiện tại.**

Khi gõ `switchport port-security ?` trên S1, IOS báo `Unrecognized command`. Đây là giới hạn của image IOS trên Dynamips/NM-16ESW, không phải lỗi cấu hình.

Cấu hình theo đề, để tham khảo nếu sau này có image hỗ trợ:
```
configure terminal
interface fa2/6
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
end
write memory
```

Khi nộp bài, ghi rõ:
- Đề yêu cầu port security với sticky MAC.
- Lệnh đã thử và kết quả `Unrecognized command` (kèm ảnh chụp).
- Đề xuất thầy kiểm tra image IOS khác, nếu cần.

Kiểm tra image đang dùng:
```
show version
```

### 2.10. Kiểm tra tổng thể

Từ PC-A:
```
ping 192.168.1.11
```
Mạng phải thông sau khi áp các cấu hình bảo mật.

---

## Phần 3. Bằng chứng cần nộp

| Nội dung | Lệnh hoặc cách lấy |
|---|---|
| Sơ đồ GNS3 | Chụp toàn bộ topology, thấy rõ tên node và cổng |
| Bảng ánh xạ cổng | Ghi vào báo cáo (mục 2.3) |
| Cấu hình IP | `show ip interface brief` trên R1, S1, S2 |
| Kết quả thử lệnh port security | Ảnh `Unrecognized command` |
| SSH | Ảnh test từ R1 và `show ip ssh`, `show ssh` (có phiên đang chạy) |
| PortFast, BPDU guard | `show running-config \| include portfast\|bpduguard`, `show spanning-tree summary` |
| Storm control | `show storm-control broadcast` |
| Ping thông mạng | Ping từ PC-A tới PC-B |
| Cấu hình đầy đủ | `show running-config` của R1, S1, S2 (lưu vào file txt) |

Trước khi nộp:
- Đã `write memory` trên R1, S1, S2.
- Đã lưu project GNS3 (File → Save).
- Đã test lại SSH sau khi hoàn thành mọi cấu hình.
- Đã ghi rõ mục port security không thực hiện được và lý do.

---

## Phần 4. Cấu trúc báo cáo gợi ý

1. **Mục tiêu**: ba nhóm bảo mật lớp 2 theo đề.
2. **Sơ đồ và bảng IP** theo đề, kèm bảng ánh xạ cổng (mục 2.3).
3. **Các bước cấu hình** cho từng nhóm, kèm lệnh và kết quả `show`.
4. **Kết quả kiểm tra** cho SSH, STP, storm control, ping.
5. **Phần không thực hiện được**: port security, kèm bằng chứng và lý do.
6. **Phụ lục**: cấu hình đầy đủ của R1, S1, S2.

---

## Phần 5. Xử lý sự cố

| Hiện tượng | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|
| Ping không thông | Quên `no shutdown` trên VLAN 1 hoặc cổng, nối sai cổng | `show ip interface brief`, kiểm tra bảng ánh xạ |
| SSH báo connection refused | Chưa tạo khóa RSA, hoặc thiếu `transport input ssh` | `show ip ssh`, chạy lại `crypto key generate rsa` |
| `Unrecognized command` khi gõ `spanning-tree portfast bpduguard default` | Image không có từ khóa `default` | Dùng `spanning-tree portfast bpduguard` |
| `spanning-tree bpduguard` báo lỗi ở chế độ global | BPDU guard toàn cục chỉ nằm trong `spanning-tree portfast bpduguard` | Dùng đúng cú pháp ở mục 2.7 |
| Cổng S1–S2 bị shutdown | Đã đặt PortFast hoặc BPDU guard lên cổng liên kết | Bỏ cấu hình khỏi Fa2/1, `shutdown` rồi `no shutdown` |
| Lỗi `%PM-4-ERR_DISABLE` | Cổng bị err-disabled do BPDU guard hoặc storm control | `show interfaces status err-disabled`, rồi `shutdown` và `no shutdown` |
| Lệnh `storm-control` không nhận | Gõ thiếu tham số | Gõ đầy đủ `storm-control broadcast level 20` |