# Lab 7 — IPsec Site-to-Site VPN (WEST ⇄ EAST qua GATEWAY)

Tài liệu tham khảo: *VPN Slides* (Cisco Networking Academy, CCNA Security — phần "Configuring IPsec", 5 Task cấu hình bằng CLI). Topology/IP kế thừa từ **Lab 3 — ACL**.

---

## 1. Mục tiêu

- Cấu hình **IPsec site-to-site VPN** giữa router **WEST** và **EAST**; đường hầm đi qua **GATEWAY** (GATEWAY chỉ định tuyến gói IP bình thường, không biết gì về VPN).
- Chỉ lưu lượng **LAN1 (10.10.1.0/24) ⇄ LAN3 (10.10.3.0/24)** được mã hóa. Mọi lưu lượng khác (ví dụ LAN1 → LAN4) đi bình thường, không mã hóa.
- Kiểm chứng bằng `show crypto ipsec sa` (bộ đếm gói) và, tùy chọn, bằng Wireshark (thấy ESP, không thấy ICMP bên trong).

## 1.5. Đối chiếu với cấu hình thực tế của Lab 3

Output `show ip interface brief`, `show running-config | section interface` và `show running-config | section router rip` trên 3 router:

| Hạng mục | Thực tế trên Lab 3 | Kết luận |
|---|---|---|
| Serial nối Gateway trên WEST | `Serial0/0` = 172.16.3.1, up/up | Đúng. Crypto map gắn vào `Serial0/0` |
| Serial nối Gateway trên EAST | `Serial0/1` = 172.16.4.1, up/up | Đúng. Crypto map gắn vào `Serial0/1` |
| Serial trên Gateway | `Serial0/0` = 172.16.3.2, `Serial0/1` = 172.16.4.2, up/up | Đúng |
| Cổng LAN | WEST: F0/0 = 10.10.1.1, F0/1 = 10.10.2.1. EAST: F0/0 = 10.10.3.1, F0/1 = 10.10.4.1 | Đúng |
| RIPv2 | WEST và EAST: `version 2`, `network 10.0.0.0`, `network 172.16.0.0`, `no auto-summary`. Gateway: `version 2`, `network 172.16.0.0`, `no auto-summary` | Đúng |
| Cổng Serial không dùng | Serial1/0–1/3 trên cả 3 router đều shutdown. EAST có Serial0/0 shutdown, không gán IP | Không ảnh hưởng |
| ACL trên đường VPN | Không có ACL trên `Serial0/0` (WEST), `Serial0/1` (EAST), `Serial0/0` và `Serial0/1` (Gateway) | Không cần thêm lệnh `permit esp`, `permit ahp`, `permit udp ... eq isakmp` như Task 1 trong slide |
| ACL_LAN2_LAN3 trên WEST F0/1 | Chỉ áp cho LAN2 | Không nằm trên đường LAN1 ↔ LAN3, giữ nguyên |
| IOS image | `c3725-adventerprisek9` | Có tính năng crypto, dùng được `esp-3des` và `esp-sha-hmac` |

Hai khác biệt nhỏ, không cần sửa cấu hình:
- Gateway không có `passive-interface FastEthernet0/0` như tutorial Lab 3. Lệnh này không có tác dụng vì F0/0 (16.19.16.19) không thuộc 172.16.0.0/16 hay 10.0.0.0/8, nên RIP không chạy trên cổng đó.
- Cả hai đầu Serial trên Gateway đều có `clock rate 2000000`. Link vẫn up/up nên không cần sửa.


Các thông số của Lab 7 giữ nguyên: IP peer của WEST là `172.16.4.1`, của EAST là `172.16.3.1`. Crypto map `VPN-MAP` gắn vào `Serial0/0` (WEST) và `Serial0/1` (EAST).

---

## 2. Chuẩn bị

### 2.1. Tạo project Lab 7 từ Lab 3

1. Mở project **Lab 3**, **Stop** toàn bộ node.
2. **File → Save Project As...** → đặt tên `lab7`.
3. Kiểm tra thanh tiêu đề đang là `lab7`, rồi Start lại các router.

> Dùng **Lab 3**, không dùng Lab 5/6: East ở Lab 3 chưa có firewall nào, nên ESP (giao thức 50) và ISAKMP (UDP 500) đi qua tự do. Slide VPN (Task 1) cũng nhấn mạnh phải đảm bảo ACL trên interface không chặn ESP/AH/UDP 500.

### 2.2. Bảng địa chỉ IP (đã có từ Lab 3)

| Router | Interface | IP | Nối tới |
|---|---|---|---|
| Gateway | F0/0 | 16.19.16.19/24 | Internet |
| Gateway | S0/0 | 172.16.3.2/24 | West S0/0 |
| Gateway | S0/1 | 172.16.4.2/24 | East S0/1 |
| West | S0/0 | **172.16.3.1/24** | Gateway S0/0 — đầu ra của VPN |
| West | F0/0 | 10.10.1.1/24 | LAN1 |
| West | F0/1 | 10.10.2.1/24 | LAN2 |
| East | S0/1 | **172.16.4.1/24** | Gateway S0/1 — đầu ra của VPN |
| East | F0/0 | 10.10.3.1/24 | LAN3 |
| East | F0/1 | 10.10.4.1/24 | LAN4 |

Các host dùng trong bài:

| Host | IP | Gateway | Ghi chú |
|---|---|---|---|
| PC-LAN1 | 10.10.1.100/24 | 10.10.1.1 | Thêm mới (VPCS), nối Switch1 |
| PC-LAN3 | 10.10.3.100/24 | 10.10.3.1 | Server2003R2-2 có sẵn từ Lab 3 |
| PC-LAN4 | 10.10.4.100/24 | 10.10.4.1 | Thêm mới (VPCS), nối Switch4 |

#### Tại sao lai dùng VPCS 

- Không cần tạo thêm máy ảo Windows, cài đặt hệ điều hành hay cấu hình card mạng.

- Tiết kiệm RAM và CPU, đặc biệt hữu ích khi chạy nhiều router trong GNS3.

- Đủ để thực hiện các bài kiểm tra ping mà đề yêu cầu.

### 2.3. Thêm PC-LAN1 và PC-LAN4 (VPCS)

1. Kéo 2 node **VPCS** vào canvas, đặt tên `PC-LAN1`, `PC-LAN4`.
2. Nối `PC-LAN1` (e0) vào **Switch1**, `PC-LAN4` (e0) vào **Switch4**.
3. Start, mở console từng VPCS và gõ:

```
PC-LAN1> ip 10.10.1.100/24 10.10.1.1
PC-LAN1> save
```
```
PC-LAN4> ip 10.10.4.100/24 10.10.4.1
PC-LAN4> save
```

> Nếu không muốn thêm máy, có thể ping ngay từ router với địa chỉ nguồn trong LAN: `West# ping 10.10.3.100 source FastEthernet0/0`. Địa chỉ nguồn phải nằm trong 10.10.1.0/24 thì mới khớp ACL 110.

### 2.4. Đặt mật khẩu cho các router (theo đề)

| Mật khẩu | Giá trị |
|---|---|
| Console | `ciscoconpa55` |
| Telnet (vty) | `ciscovtypa55` |
| Enable | `ciscoenpa55` |

Chạy khối lệnh sau trên **cả 3 router** (Gateway, West, East):

```
enable
configure terminal
enable secret ciscoenpa55
line console 0
 password ciscoconpa55
 login
 exit
line vty 0 4
 password ciscovtypa55
 login
 exit
end
write memory
```

---

## 3. Bảng tham số chính sách (đã chỉnh lại peer cho đúng)

### Phase 1 — ISAKMP (IKE)

| Tham số | WEST | EAST |
|---|---|---|
| Phương thức phân phối khóa | ISAKMP | ISAKMP |
| Thuật toán mã hóa | AES | AES |
| Thuật toán băm | SHA-1 (`hash sha`) | SHA-1 |
| Phương thức xác thực | Pre-shared key | Pre-shared key |
| Trao đổi khóa | Diffie-Hellman nhóm 2 | DH nhóm 2 |
| Thời gian sống SA IKE | 86400 giây | 86400 giây |
| Khóa ISAKMP | `vpnpa55` | `vpnpa55` |

### Phase 2 — IPsec

| Tham số | WEST | EAST |
|---|---|---|
| Transform set | `VPN-SET` (esp-3des, esp-sha-hmac) | `VPN-SET` |
| Tên đồng cấp | EAST | WEST |
| **IP đồng cấp (peer) — đã sửa** | **172.16.4.1** | **172.16.3.1** |
| Mạng cần mã hóa | 10.10.1.0/24 → 10.10.3.0/24 | 10.10.3.0/24 → 10.10.1.0/24 |
| Crypto map | `VPN-MAP` (seq 10) | `VPN-MAP` (seq 10) |
| Thiết lập SA | `ipsec-isakmp` | `ipsec-isakmp` |
| Interface gắn crypto map | **Serial0/0** | **Serial0/1** |

---

## 4. Nhiệm vụ 1 — Kiểm tra kết nối và cấu hình WEST

### Bước 1. Kiểm tra kết nối (làm TRƯỚC khi cấu hình VPN)

```
PC-LAN1> ping 10.10.3.100
```
Phải có reply (RIPv2 từ Lab 3 đã thông). Nếu không, kiểm tra `show ip route` trên West/Gateway/East trước, đừng cấu hình VPN khi mạng nền chưa thông.

### Bước 2 → 5. Cấu hình WEST (toàn bộ lệnh, chạy liên tiếp)

```
enable
configure terminal

! ===== Bước 2: ACL 110 xác định lưu lượng VPN (LAN1 -> LAN3) =====
access-list 110 permit ip 10.10.1.0 0.0.0.255 10.10.3.0 0.0.0.255

! ===== Bước 3: Phase 1 - chính sách ISAKMP =====
crypto isakmp policy 10
 encryption aes
 authentication pre-share
 group 2
 hash sha
 lifetime 86400
 exit
crypto isakmp key vpnpa55 address 172.16.4.1

! ===== Bước 4: Phase 2 - transform set + crypto map =====
crypto ipsec transform-set VPN-SET esp-3des esp-sha-hmac
 exit
crypto map VPN-MAP 10 ipsec-isakmp
 description VPN tunnel to EAST
 set peer 172.16.4.1
 set transform-set VPN-SET
 match address 110
 exit

! ===== Bước 5: Gắn crypto map vào interface đầu ra =====
interface Serial0/0
 crypto map VPN-MAP
 exit

end
write memory
```

Thông báo bình thường khi gõ:
- `% NOTE: This new crypto map will remain disabled until a peer and a valid access list have been configured.` — IOS nhắc khi mới tạo map, biến mất sau khi có `set peer` và `match address`.
- `%CRYPTO-6-ISAKMP_ON_OFF: ISAKMP is ON` — xuất hiện khi gắn crypto map vào interface.

Giải thích nhanh:

| Lệnh | Ý nghĩa |
|---|---|
| `access-list 110 permit ip ...` | "Crypto ACL": traffic **khớp** thì bị mã hóa; không khớp thì đi rõ (deny ngầm cuối ACL, nên không cần viết `deny`) |
| `encryption aes` / `group 2` / `authentication pre-share` | Ba thông số **khác mặc định** (mặc định là des / group 1 / rsa-sig) nên bắt buộc phải đặt. `hash sha` và `lifetime 86400` vốn là mặc định, ghi ra cho rõ |
| `crypto isakmp key vpnpa55 address 172.16.4.1` | Khóa chia sẻ dùng khi nói chuyện với peer có IP `172.16.4.1` |
| `crypto ipsec transform-set VPN-SET esp-3des esp-sha-hmac` | Phase 2: ESP mã hóa 3DES + xác thực/toàn vẹn HMAC-SHA |
| `set peer` / `set transform-set` / `match address` | Crypto map gắn 3 thứ lại: **đối tác nào**, **thuật toán nào**, **traffic nào** |
| `crypto map VPN-MAP` trên Serial0/0 | Chỉ khi gắn vào interface thì chính sách mới có hiệu lực; mỗi interface chỉ gắn **một** crypto map |

---

## 5. Nhiệm vụ 2 — Cấu hình EAST

Cấu hình **đối xứng** với WEST: ACL đảo chiều nguồn/đích, peer là IP của WEST, gắn vào `Serial0/1`.

```
enable
configure terminal

! ===== Bước 1: ACL 110 (LAN3 -> LAN1), đối xứng với WEST =====
access-list 110 permit ip 10.10.3.0 0.0.0.255 10.10.1.0 0.0.0.255

! ===== Bước 2: Phase 1 - ISAKMP =====
crypto isakmp policy 10
 encryption aes
 authentication pre-share
 group 2
 hash sha
 lifetime 86400
 exit
crypto isakmp key vpnpa55 address 172.16.3.1

! ===== Bước 3: Phase 2 - transform set + crypto map =====
crypto ipsec transform-set VPN-SET esp-3des esp-sha-hmac
 exit
crypto map VPN-MAP 10 ipsec-isakmp
 description VPN tunnel to WEST
 set peer 172.16.3.1
 set transform-set VPN-SET
 match address 110
 exit

! ===== Bước 4: Gắn crypto map vào interface đầu ra =====
interface Serial0/1
 crypto map VPN-MAP
 exit

end
write memory
```

Kiểm tra cấu hình đã nhận (trên cả hai router):

```
show crypto isakmp policy
show crypto ipsec transform-set
show crypto map
show access-lists 110
```

Điểm phải khớp giữa hai router: thuật toán mã hóa/băm, phương thức xác thực, nhóm DH, khóa `vpnpa55`, transform set (cùng thuật toán), và ACL 110 phải là **ảnh gương** của nhau. Số policy (10) và tên transform set (VPN-SET) chỉ có ý nghĩa cục bộ, không bắt buộc trùng, nhưng đề yêu cầu đặt giống nhau.

---

## 6. Nhiệm vụ 3 — Xác minh VPN

### Bước 1. Trước khi có lưu lượng VPN

```
WEST# show crypto ipsec sa
```
Phải thấy `interface: Serial0/0`, `Crypto map tag: VPN-MAP`, `local addr 172.16.3.1`, và các bộ đếm đều bằng 0:

```
#pkts encaps: 0, #pkts encrypt: 0, #pkts digest: 0
#pkts decaps: 0, #pkts decrypt: 0, #pkts verify: 0
```

**Chụp ảnh màn hình** làm bằng chứng "trước".

### Bước 2. Tạo lưu lượng VPN

```
PC-LAN1> ping 10.10.3.100
```
Gói đầu tiên thường **timeout** vì router phải thương lượng IKE (Phase 1 + Phase 2) trước khi mã hóa được. Ping tiếp 2–3 lần cho đến khi có reply.

### Bước 3. Xác minh sau khi có lưu lượng VPN

```
WEST# show crypto ipsec sa
WEST# show crypto isakmp sa
```
- `show crypto ipsec sa`: các bộ đếm `#pkts encaps/encrypt` và `#pkts decaps/decrypt` **lớn hơn 0** → đường hầm đang hoạt động. Các con số có thể lệch nhau 1 gói do gói ping đầu bị mất.
- `show crypto isakmp sa`: trạng thái **`QM_IDLE`** (cột `state`), `ACTIVE` → Phase 1 đã thành công. Dạng kết quả:

```
dst             src             state          conn-id slot status
172.16.4.1      172.16.3.1      QM_IDLE           1001    0 ACTIVE
```

Chụp ảnh bằng chứng "sau". Làm tương tự trên EAST nếu cần đối chiếu (gói `decaps` của EAST tương ứng `encaps` của WEST).

### Bước 4. Tạo lưu lượng không phải VPN

```
PC-LAN1> ping 10.10.4.100
```
Ping này đi LAN1 → LAN4, **không khớp ACL 110** nên không được mã hóa.

### Bước 5. Xác minh đường hầm

```
WEST# show crypto ipsec sa
```
Các bộ đếm **giữ nguyên** so với Bước 3 → xác nhận lưu lượng không thuộc VPN đi rõ, không qua đường hầm.

### Reset VPN: nếu filter isakmp trên Wireshark hoặc muốn kiểm tra lại từ đầu, xóa SA cũ:

```
WEST# clear crypto isakmp
WEST# clear crypto sa
EAST# clear crypto isakmp
EAST# clear crypto sa
```

---

## 7. Wireshark & Tunnel Mode

### 7.1 Wireshark Capture

1. Right-click link **West S0/0 ⇄ Gateway S0/0** trên canvas GNS3 → **Start capture** → Cisco HDLC
2. Chạy lần lượt:
   - `ping 10.10.3.100` từ PC-LAN1 (lưu lượng VPN).
   - `ping 10.10.4.100` từ PC-LAN1 (lưu lượng không phải VPN).
3. Dừng capture và lọc:

| Filter | Kỳ vọng |
|---|---|
| `isakmp` | Gói IKE (UDP 500) giữa `172.16.3.1` ⇄ `172.16.4.1` — quá trình thương lượng Phase 1 (Main Mode) và Phase 2 (Quick Mode) |
| `esp` | Gói ESP giữa `172.16.3.1` ⇄ `172.16.4.1`, có SPI và Sequence number, **phần dữ liệu không đọc được** |
| `icmp` | Chỉ thấy ICMP của ping **LAN1 → LAN4** (`10.10.1.100` ⇄ `10.10.4.100`). **Không** thấy ICMP giữa `10.10.1.100` và `10.10.3.100` |

> Vì sao không thấy ICMP LAN1↔LAN3: gói gốc đã bị mã hóa và bọc trong gói ESP có IP ngoài `172.16.3.1 → 172.16.4.1`. GATEWAY (và Wireshark trên link này) chỉ nhìn thấy IP ngoài.

### 7.2 Tunnel Mode

1. Kiểm tra tunnel mode trên router:

```
show crypto ipsec sa
show crypto ipsec transform-set
show crypto isakmp sa
```
Tập trung vào các dòng:
- in use settings ={Tunnel, }: xác nhận SA đang sử dụng Tunnel Mode.
- #pkts encrypt và #pkts decrypt: bộ đếm mã hóa/giải mã.
- #pkts digest và #pkts verify: bộ đếm tính toán/xác minh giá trị xác thực ESP.
- QM_IDLE: trạng thái IKE thường thấy khi thương lượng đã hoàn tất trên IOS của lab này.

Nếu muốn xem nhanh các bộ đếm:
```
show crypto ipsec sa | include pkts
```

2. Bật kiểm tra IPv4 header checksum để Wireshark kiểm tra checksum IPv4 thay vì báo Unverified:

- Trong Wireshark: Edit → Preferences → Protocols → IPv4.
- Tick Validate the IPv4 checksum if possible.
- OK, rồi mở lại gói. Dòng sẽ đổi thành header checksum status: Good.

Lưu ý: đây chỉ là kiểm tra lỗi truyền. Đừng trình bày nó như bằng chứng xác thực.

3. Để IP ngoài được xác thực: thêm AH

- AH (giao thức 51) xác thực toàn bộ IP header ngoài, trừ các trường thay đổi được như TTL. Trên WEST và EAST, sửa transform set:

```
enable
configure terminal
no crypto ipsec transform-set VPN-SET
crypto ipsec transform-set VPN-SET ah-sha-hmac esp-3des esp-sha-hmac
 mode tunnel
 exit
end
write memory
```

- Làm tương tự trên EAST. ACL 110, crypto map và peer giữ nguyên. Gateway không cần đổi.

- Sau đó:

```
clear crypto sa
```

- Ping LAN3 từ PC-LAN1, trên WEST kiểm tra 

```
show crypto ipsec sa | include in use 
show crypto ipsec sa | include pkts.
```

- Trong Wireshark, filter ah || esp: sẽ có gói protocol 51 (AH) với Authentication Data, đi trước phần ESP. Khi đó IP ngoài cũng được xác thực.



---

## 8. Xử lý sự cố

| Hiện tượng | Nguyên nhân thường gặp | Cách kiểm tra / sửa |
|---|---|---|
| Ping LAN1→LAN3 luôn timeout, `show crypto isakmp sa` trống | Mạng nền chưa thông, hoặc traffic không khớp ACL 110 | `show ip route`, `ping 172.16.4.1` từ West; `show access-lists 110` xem hit counter có tăng không |
| `show crypto isakmp sa` ở trạng thái `MM_NO_STATE` | Phase 1 thất bại: sai/lệch policy (aes/group/auth), sai khóa `vpnpa55`, hoặc sai IP peer (đặc biệt lỗi **peer ghi ngược** ở đề) | `show crypto isakmp policy` hai bên; `debug crypto isakmp` — thông báo kiểu `atts are not acceptable` nghĩa là policy Phase 1 không khớp |
| Phase 1 lên (`QM_IDLE`) nhưng bộ đếm không tăng | ACL 110 hai bên không đối xứng, hoặc transform set khác nhau (Phase 2 thất bại) | `show crypto ipsec transform-set`, `debug crypto ipsec` |
| Không có gì xảy ra dù cấu hình đúng | Quên gắn `crypto map` vào interface, hoặc gắn nhầm cổng (Serial0/1 trên West) | `show crypto map` xem mục "Interfaces using crypto map VPN-MAP" |
| VPN lên rồi nhưng đổi cấu hình không có tác dụng | SA cũ vẫn còn | `clear crypto isakmp` và `clear crypto sa` trên cả hai router, rồi ping lại |
| VPN không lên trong project cũ | East đang có CBAC/ZPF (Lab 5/6) chặn ESP/UDP 500 | Dùng project tạo từ Lab 3, hoặc thêm vào ACL interface: `permit esp host <peer> host <local>` và `permit udp host <peer> host <local> eq isakmp` |

Lệnh debug nên dùng ngắn gọn rồi tắt: `debug crypto isakmp`, `debug crypto ipsec`, kết thúc bằng `undebug all`.

---

## 9. Lý thuyết và luồng gói tin

### 9.1. IPsec gồm những gì

| Thành phần | Vai trò |
|---|---|
| **AH** (giao thức 51) | Xác thực + toàn vẹn, **không mã hóa** (dữ liệu vẫn là bản rõ) |
| **ESP** (giao thức 50) | Mã hóa + xác thực + toàn vẹn. Lab này dùng **ESP** (`esp-3des` mã hóa, `esp-sha-hmac` xác thực) |
| **IKE / ISAKMP** (UDP 500) | Giao thức thương lượng thông số và khóa, tạo ra các SA |
| **SA** (Security Association) | "Thỏa thuận" một chiều giữa hai peer: thuật toán, khóa, SPI. IKE SA (Phase 1) là hai chiều; IPsec SA (Phase 2) là **một chiều** — nên có một SA cho mỗi hướng |
| **Tunnel mode** | Gói gốc được mã hóa trọn vẹn rồi bọc thêm IP header mới (`172.16.3.1 → 172.16.4.1`). Đây là chế độ dùng cho VPN site-to-site |

Cấu trúc gói ESP tunnel mode: `[IP mới | ESP header | IP gốc + dữ liệu (đã mã hóa) | ESP trailer | ESP auth]`.

### 9.2. Hai pha của IKE

**Phase 1 — tạo kênh IKE an toàn (IKE SA)**, qua 3 lần trao đổi (Main Mode):
1. Thương lượng chính sách IKE (mã hóa, băm, xác thực, DH, lifetime) — hai bên phải tìm được **một policy khớp hoàn toàn**.
2. Trao đổi Diffie-Hellman để tạo khóa chung mà không gửi khóa qua đường truyền.
3. Xác thực peer (ở lab này bằng **pre-shared key** `vpnpa55`).

**Phase 2 (Quick Mode)** — chạy bên trong kênh IKE đã an toàn: thương lượng transform set (esp-3des + esp-sha-hmac), ACL/proxy identities, rồi tạo hai IPsec SA một chiều để mã hóa dữ liệu thật.

### 9.3. Luồng gói tin khi PC-LAN1 ping PC-LAN3

```
PC-LAN1 (10.10.1.100)
   │  ICMP Echo: 10.10.1.100 -> 10.10.3.100
   ▼
WEST F0/0 (nhận gói, quyết định định tuyến: đi ra S0/0)
   │  Crypto map VPN-MAP trên S0/0: gói khớp ACL 110?  -> CÓ ("interesting traffic")
   │  Chưa có SA -> khởi động IKE: Phase 1 (UDP 500 tới 172.16.4.1) rồi Phase 2
   │  (gói ping đầu tiên thường bị mất trong lúc thương lượng)
   ▼
Có SA: WEST mã hóa + đóng gói ESP
   │  [ IP 172.16.3.1 -> 172.16.4.1 | ESP | (gói ICMP gốc đã mã hóa) | auth ]
   ▼
GATEWAY: chỉ đọc IP ngoài, định tuyến bình thường theo RIP tới 172.16.4.1
   │  (không biết bên trong là gì, không có cấu hình VPN)
   ▼
EAST S0/1: nhận ESP -> tra SPI -> xác thực + giải mã -> ra gói gốc 10.10.1.100 -> 10.10.3.100
   ▼
EAST F0/0 -> PC-LAN3 (10.10.3.100) trả lời
   │  Gói trả lời 10.10.3.100 -> 10.10.1.100 khớp ACL 110 của EAST -> EAST mã hóa
   │  bằng IPsec SA chiều ngược lại -> WEST giải mã -> PC-LAN1
```

Gói ping LAN1 → LAN4 thì sao: WEST định tuyến ra S0/0 như trên, nhưng **không khớp ACL 110** nên crypto map bỏ qua, gói đi nguyên dạng (bản rõ) → bộ đếm `show crypto ipsec sa` không đổi.

### 9.4. Câu hỏi thường gặp

**Q: "Interesting traffic" là gì? Ai quyết định?**
A: Là lưu lượng khớp crypto ACL (ACL 110) gắn vào crypto map. Khi có gói khớp, router mới khởi tạo IKE và mã hóa. Gói không khớp đi bản rõ.

**Q: Vì sao ACL 110 của hai router phải là ảnh gương?**
A: WEST mã hóa traffic 10.10.1.0→10.10.3.0; EAST phải coi traffic 10.10.3.0→10.10.1.0 là cần bảo vệ theo chiều của nó. Nếu lệch, Phase 2 thương lượng thất bại vì hai bên không đồng ý về "traffic nào được bảo vệ".

**Q: Vì sao GATEWAY không cần cấu hình gì mà VPN vẫn chạy?**
A: Gói ESP có IP ngoài là `172.16.3.1 → 172.16.4.1`, Gateway chỉ cần có route tới hai địa chỉ này (đã có qua RIPv2). Nó không giải mã, không cần biết nội dung.

**Q: Vì sao ping đầu tiên thường timeout?**
A: Chưa có SA. Router giữ gói, chạy IKE Phase 1 và Phase 2 rồi mới mã hóa. Các gói sau đi qua SA đã tạo nên bình thường.

**Q: Phase 1 và Phase 2 khác nhau thế nào? Vì sao phải có hai pha?**
A: Phase 1 dựng kênh quản lý an toàn (IKE SA) giữa hai router bằng cách xác thực và trao khóa DH. Phase 2 chạy trong kênh đó để thương lượng thông số bảo vệ dữ liệu và tạo IPsec SA. Tách pha giúp dùng một IKE SA để tạo nhiều IPsec SA khác nhau và đổi khóa dữ liệu mà không phải xác thực lại.

**Q: Vì sao Phase 1 dùng AES nhưng Phase 2 dùng 3DES?**
A: Hai pha bảo vệ hai thứ khác nhau (kênh IKE và dữ liệu người dùng), được thương lượng độc lập nên không cần trùng thuật toán. Đây là cấu hình theo đề.

**Q: Diffie-Hellman nhóm 2 để làm gì?**
A: Cho phép hai router tự tạo ra khóa chung bí mật mà không phải truyền khóa qua mạng. Nhóm 2 dùng số nguyên tố 1024-bit; nhóm càng cao càng mạnh. Hai bên phải dùng cùng nhóm.

**Q: AH và ESP khác nhau thế nào? Vì sao chọn ESP?**
A: AH chỉ xác thực và bảo đảm toàn vẹn, không mã hóa. ESP có thêm mã hóa nên giữ được bí mật dữ liệu — là yêu cầu của VPN.

**Q: Thấy gì trong `show crypto ipsec sa` để biết VPN hoạt động?**
A: Các bộ đếm `#pkts encaps/encrypt` (gói đã mã hóa gửi đi) và `#pkts decaps/decrypt` (gói đã giải mã nhận về) lớn hơn 0. Gửi lưu lượng không khớp ACL 110 thì các bộ đếm này không tăng.

**Q: Nếu hai router không khớp chính sách Phase 1 thì sao?**
A: IKE không dựng được SA; `show crypto isakmp sa` không có `QM_IDLE` (thường `MM_NO_STATE`), `debug crypto isakmp` báo `atts are not acceptable` / `SA not acceptable`. Slide VPN xếp lỗi này vào nhóm "Main Mode thất bại — policy Phase 1 không khớp".
