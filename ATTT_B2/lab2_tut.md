# Lab 2 — VLSM + RIPv2 + PPP CHAP Authentication

## 1. Mục tiêu

- Chia VLSM đúng thứ tự từ `172.16.0.0/16` cho 3 vùng LAN có nhu cầu host khác nhau.
- Định tuyến động bằng RIPv2 giữa 3 router.
- Bảo mật 3 link Serial bằng PPP encapsulation + CHAP authentication.
- Kiểm chứng bằng `show ip route` và Wireshark.

## 2. Tính VLSM (thứ tự bắt buộc: lớn → nhỏ)

| Vùng | Yêu cầu host | Bit host cần (2ⁿ−2 ≥ N) | Prefix | Subnet Mask |
|---|---|---|---|---|
| VN (R2) | 6969 | n=13 (8190) | /19 | 255.255.224.0 |
| LAO (R1) | 3969 | n=12 (4094) | /20 | 255.255.240.0 |
| CAM (R3) | 1969 | n=11 (2046) | /21 | 255.255.248.0 |
| Mỗi link Serial | 2 | n=2 (2) | /30 | 255.255.255.252 |

### Quá trình cắt tuần tự

```
172.16.0.0/16
 └─ VN   : 172.16.0.0/19   (172.16.0.0   – 172.16.31.255)
 └─ LAO  : 172.16.32.0/20  (172.16.32.0  – 172.16.47.255)
 └─ CAM  : 172.16.48.0/21  (172.16.48.0  – 172.16.55.255)
 └─ Serial R1–R2 : 172.16.56.0/30
 └─ Serial R2–R3 : 172.16.56.4/30
 └─ Serial R1–R3 : 172.16.56.8/30
```

> Bắt buộc cắt từ nhu cầu host lớn nhất trước để tránh chồng lấn địa chỉ giữa các dải.

## 3. Topology

```
                         R2 (172.16.0.1/19)
                        /s1/0          s1/1\
             172.16.56.1              172.16.56.5
                  /                          \
         172.16.56.0/30              172.16.56.4/30
                /                              \
      172.16.56.2                        172.16.56.6
          s1/0                               s1/1
   R1 (172.16.32.1/20) ---- 172.16.56.8/30 ---- R3 (172.16.48.1/21)
        s1/2  172.16.56.9         172.16.56.10  s1/2
```

## 4. Bảng địa chỉ đầy đủ

| Router | Interface | IP Address | Subnet |
|---|---|---|---|
| R1 | LAN (LAO) | 172.16.32.1/20 | 172.16.32.0/20 |
| R1 | Serial1/0 → R2 | 172.16.56.2/30 | 172.16.56.0/30 |
| R1 | Serial1/2 → R3 | 172.16.56.9/30 | 172.16.56.8/30 |
| R2 | LAN (VN) | 172.16.0.1/19 | 172.16.0.0/19 |
| R2 | Serial1/0 → R1 | 172.16.56.1/30 | 172.16.56.0/30 |
| R2 | Serial1/1 → R3 | 172.16.56.5/30 | 172.16.56.4/30 |
| R3 | LAN (CAM) | 172.16.48.1/21 | 172.16.48.0/21 |
| R3 | Serial1/1 → R2 | 172.16.56.6/30 | 172.16.56.4/30 |
| R3 | Serial1/2 → R1 | 172.16.56.10/30 | 172.16.56.8/30 |

**Mật khẩu CHAP dùng chung cho cả 3 cặp router: `Sinch@u`**

## 5. Cấu hình đầy đủ

### R1
```
enable
configure terminal
hostname R1
username R2 password Sinch@u
username R3 password Sinch@u

interface Loopback1
 ip address 172.16.32.1 255.255.240.0
 no shutdown
exit

interface Serial1/0
 ip address 172.16.56.2 255.255.255.252
 encapsulation ppp
 ppp authentication chap
 no shutdown
exit

interface Serial1/2
 ip address 172.16.56.9 255.255.255.252
 encapsulation ppp
 ppp authentication chap
 clock rate 2000000
 no shutdown
exit

router rip
 version 2
 network 172.16.0.0
 no auto-summary
end
write memory
```

### R2
```
enable
configure terminal
hostname R2
username R1 password Sinch@u
username R3 password Sinch@u

interface Loopback2
 ip address 172.16.0.1 255.255.224.0
 no shutdown
exit

interface Serial1/0
 ip address 172.16.56.1 255.255.255.252
 encapsulation ppp
 ppp authentication chap
 clock rate 2000000
 no shutdown
exit

interface Serial1/1
 ip address 172.16.56.5 255.255.255.252
 encapsulation ppp
 ppp authentication chap
 clock rate 2000000
 no shutdown
exit

router rip
 version 2
 network 172.16.0.0
 no auto-summary
end
write memory
```

### R3
```
enable
configure terminal
hostname R3
username R1 password Sinch@u
username R2 password Sinch@u

interface Loopback0
 ip address 172.16.48.1 255.255.248.0
 no shutdown
exit

interface Serial1/1
 ip address 172.16.56.6 255.255.255.252
 encapsulation ppp
 ppp authentication chap
 no shutdown
exit

interface Serial1/2
 ip address 172.16.56.10 255.255.255.252
 encapsulation ppp
 ppp authentication chap
 no shutdown
exit

router rip
 version 2
 network 172.16.0.0
 no auto-summary
end
write memory
```

> Chỉ cần **1 dòng** `network 172.16.0.0` trên cả 3 router — RIP tự quy mọi subnet về đúng major network lớp B, không cần khai riêng từng dải /19, /20, /21, /30.

## 6. Kiểm tra (1): `show ip route` — đủ 6 subnet

Mỗi router phải thấy đủ: `172.16.0.0/19`, `172.16.32.0/20`, `172.16.48.0/21`, `172.16.56.0/30`, `172.16.56.4/30`, `172.16.56.8/30` (3 dòng dạng `C`, 3 dòng dạng `R` tùy vị trí router).

> **Lưu ý:** Nếu chạy PPP đúng, IOS sẽ tự thêm thêm các **host route /32** (địa chỉ IP đầu bên kia của mỗi link) do quá trình IPCP negotiation ở giai đoạn NCP của PPP. Đây là hành vi chuẩn (xem RFC 1661), không phải lỗi — nếu muốn ẩn để trình bày đúng 6 dòng theo yêu cầu:
> ```
> show ip route | exclude /32
> ```

## 7. Kiểm tra (2): Wireshark — CHAP hash

Vì Challenge/Response/Success chỉ xảy ra **1 lần duy nhất lúc LCP khởi tạo**, cần ép link tái đàm phán trong lúc đang capture:

1. Right-click link trên GNS3 → Start capture → chọn đúng interface Serial.
2. Trên 1 trong 2 router của link đó:
   ```
   configure terminal
   interface serial1/X
   shutdown
   ```
   Đợi 3–5 giây:
   ```
   no shutdown
   ```
3. Dừng capture, filter `ppp`.
4. Tìm đúng cặp gói: `Challenge (NAME=...)` × 2 → `Response (NAME=..., VALUE=<hash 16 byte>)` × 2 → `Success` × 2 (two-way authentication vì `ppp authentication chap` được bật trên cả 2 đầu).
5. Kiểm tra trường **Value** ở gói Response là chuỗi hex, không phải chữ `Sinch@u` đọc được.
6. Lưu file: `lab2_r12_chap.pcapng`, `lab2_r13_chap.pcapng`, `lab2_r23_chap.pcapng`.

## 8. Sự cố thường gặp

| Hiện tượng | Nguyên nhân | Cách sửa |
|---|---|---|
| Thiếu 1 subnet /2x trong `show ip route` | LAN interface của router đó gõ nhầm IP (VD: gõ `192.16.x.x` thay vì `172.16.x.x`) | `show ip interface brief` để soi đúng IP, sửa lại `ip address` |
| `SLARP` xuất hiện thay vì `LCP Echo` trong Wireshark | Một đầu link chưa đổi `encapsulation ppp` (vẫn HDLC) | `show interfaces serialX/Y` kiểm tra Encapsulation, sửa đầu còn thiếu |
| `PPP LCP Configuration Request` lặp vô hạn, không có `Ack` | 2 đầu đang ở 2 encapsulation khác nhau (1 bên HDLC, 1 bên PPP) | Đồng bộ `encapsulation ppp` cả 2 đầu |
| Link không lên `up/up` dù đã cấu hình đủ | Thiếu `clock rate` ở đầu DCE | Chạy `show controllers serialX/Y` (lưu ý: GNS3/Dynamips thường báo "DCE cable" cả 2 đầu — không đáng tin 100%), thử gõ clock rate ở 1 bên, không lên thì đổi bên kia |

## 9. Lý thuyết PPP ↔ lệnh cấu hình

| Lệnh | Giai đoạn PPP | Vai trò |
|---|---|---|
| `encapsulation ppp` | Trước LCP | Chuyển từ HDLC sang PPP |
| `ppp authentication chap` | LCP quyết định | Yêu cầu chuyển sang giai đoạn Authentication bằng CHAP |
| `username <peer> password ...` | Authentication (CHAP) | Bảng tra cứu để tính hash so sánh khi nhận Challenge/Response |
| `ip address ...` trên Serial | NCP (IPCP) | Giá trị IPCP trao đổi/xác nhận ở giai đoạn cuối — sinh ra host route /32 |