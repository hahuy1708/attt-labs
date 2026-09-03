# Lab 1 — Routing Protocol Authentication (RIPv2 / EIGRP / OSPF)

## 1. Mục tiêu

Cấu hình và kiểm chứng bằng Wireshark cơ chế xác thực (authentication) trên 3 giao thức định tuyến:

- **RIPv2** — Plain Text & MD5
- **EIGRP** — MD5 (classic mode chỉ hỗ trợ MD5, không có plain text)
- **OSPF** — Plain Text & MD5

## 2. Topology

```
        Serial1/0                    Serial1/0
  R1 (192.168.1.1/24) <---- 6.9.6.0/24 ----> R2 (192.168.2.1/24)
   Loopback1                                  Loopback2
   6.9.6.9                                    6.9.6.10
```

| Router | Interface | IP Address | Vai trò |
|---|---|---|---|
| R1 | Loopback1 | 192.168.1.1/24 | LAN giả lập |
| R1 | Serial1/0 | 6.9.6.9/24 | Link tới R2 (DCE — giữ clock rate) |
| R2 | Loopback2 | 192.168.2.1/24 | LAN giả lập |
| R2 | Serial1/0 | 6.9.6.10/24 | Link tới R1 (DTE) |

## 3. Cấu hình IP

**R1:**
```
enable
configure terminal
hostname R1
interface Loopback1
ip address 192.168.1.1 255.255.255.0
no shutdown
exit
interface Serial1/0
ip address 6.9.6.9 255.255.255.0
clock rate 2000000
no shutdown
exit
end
write memory
```

**R2:**
```
enable
configure terminal
hostname R2
interface Loopback2
ip address 192.168.2.1 255.255.255.0
no shutdown
exit
interface Serial1/0
ip address 6.9.6.10 255.255.255.0
no shutdown
exit
end
write memory
```

Kiểm tra: `show ip interface brief` — cả 2 interface phải `up/up`.

## 4. RIPv2 Authentication

### 4.1 Bật RIP

Trên cả 2 router:
```
router rip
 version 2
 network 192.168.1.0   ! (R1) hoặc network 192.168.2.0 (R2)
 network 6.0.0.0
 no auto-summary
end
```

### 4.2 Plain Text Authentication

**R1 & R2 (giống nhau):**
```
configure terminal
key chain kal
 key 1
  key-string 234
exit
interface Serial1/0
 ip rip authentication key-chain kal
end
write memory
```

Verify: `debug ip rip` → thấy `RIP: received packet with text authentication 234`.

### 4.3 Chuyển sang MD5

**R1 & R2 (giống nhau, chỉ thêm 1 dòng):**
```
configure terminal
interface Serial1/0
 ip rip authentication mode md5
end
write memory
```

Verify: `debug ip rip` → thấy `RIP: received packet with MD5 authentication`.

### 4.4 Wireshark

Filter: `rip`. Mở gói Response → mục **Authentication** → so sánh:
- Plain text: thấy rõ chuỗi `234`.
- MD5: chỉ thấy **Authentication Data** dạng hash 16 byte.

## 5. EIGRP Authentication (MD5 only)

**R1:**
```
configure terminal
key chain EIGRPCHAIN
 key 1
  key-string EigrpPass123
exit
interface Serial1/0
 ip authentication mode eigrp 10 md5
 ip authentication key-chain eigrp 10 EIGRPCHAIN
exit
router eigrp 10
 network 192.168.1.0
 network 6.0.0.0
 no auto-summary
end
write memory
```

**R2:** giống hệt, chỉ đổi `network 192.168.1.0` → `network 192.168.2.0`.

Verify:
```
show ip eigrp neighbors      ! phải thấy đúng neighbor
debug eigrp packets          ! thấy "received packet with MD5 authentication, key id = 1"
```

Wireshark filter: `eigrp`.

## 6. OSPF Authentication

### 6.1 MD5

**R1:**
```
configure terminal
interface Serial1/0
 ip ospf message-digest-key 1 md5 OspfPass123
 ip ospf authentication message-digest
exit
router ospf 1
 network 192.168.1.0 0.0.0.255 area 0
 network 6.9.6.0 0.0.0.255 area 0
end
write memory
```

**R2:** giống hệt, chỉ đổi `network 192.168.1.0` → `network 192.168.2.0`.

Verify:
```
show ip ospf neighbor        ! state phải là FULL
debug ip ospf adj            ! thấy "Send with youngest Key 1"
```

### 6.2 Đổi tạm sang Plain Text để so sánh

```
interface Serial1/0
 no ip ospf authentication message-digest
 no ip ospf message-digest-key 1 md5 OspfPass123
 ip ospf authentication-key OspfPassPlain
 ip ospf authentication
```

Bắt Wireshark filter `ospf`, mở trường **Auth Type** — sẽ thấy `Simple Password` thay vì `Cryptographic`. Sau khi demo xong, đổi lại MD5 như mục 6.1.

## 7. Quy trình bắt Wireshark trong GNS3

1. Right-click vào link R1–R2 trên canvas GNS3 → **Start capture** → chọn interface Serial1/0.
2. Gõ cấu hình cần demo trên router.
3. Dừng capture (nút đỏ), gõ filter tương ứng (`rip` / `eigrp` / `ospf`).
4. File → Save As → đặt tên rõ ràng (`RIPv2_MD5.pcapng`, `EIGRP_MD5.pcapng`, `OSPF_MD5.pcapng`...).
5. **Đóng hẳn cửa sổ Wireshark** trước khi mở capture mới cho giao thức tiếp theo — tránh gói tin của các giao thức chồng lẫn vào 1 file.

## 8. Ghi chú xử lý sự cố thường gặp

- **`erase flash:` lúc mới mở console**: xóa file hệ thống trong flash ảo (không phải xóa cấu hình đang chạy), dùng để đảm bảo lab khởi đầu từ trạng thái sạch — an toàn trên GNS3 vì IOS image không nằm trong flash ảo.
- **Prompt không đổi sau khi gõ `router rip`/`router eigrp`**: thường do gõ đúng lúc log `%LINK-3-UPDOWN` / `%LINEPROTO-5-UPDOWN` đổ ra màn hình làm vỡ dòng lệnh — luôn nhấn Enter chờ prompt sạch trước khi gõ lệnh quan trọng tiếp theo.
- **`network` dưới RIP/EIGRP không cần gõ đúng network address (`.0`)** — dù gõ IP host (`192.168.1.1`) hay network address (`192.168.1.0`), IOS đều tự quy về đúng ranh giới classful khi lưu vào running-config.