# Lab 3 — Tutorial thực hành GNS3

## Phần 1: Nối 3 router với 4 switch

### Mục tiêu
Dựng đúng khung topology vật lý (router, switch, dây nối) trước khi gán IP và cấu hình RIPv2/ACL.

### Danh sách thiết bị
- 3 router **c3725**: Gateway, West, East
- 4 **Ethernet Switch**: SW-LAN1, SW-LAN2 (nối với West), SW-LAN3, SW-LAN4 (nối với East)
- 1 node **Cloud** hoặc **NAT** đại diện Internet
- PC/VPCS hoặc Server 2003 sẽ gắn vào switch ở phần sau

> Lưu ý: nếu lỡ kéo nhầm **ATM Switch** (icon chữ X, tên ATMSW1–4), cần xóa và thay bằng **Ethernet Switch**, vì ATM switch không hoạt động như switch Ethernet thường (không tự học MAC).

### Bước 1 — Thêm cổng Serial (WIC-2T) cho router
Router c3725 mặc định chỉ có FastEthernet0/0 và FastEthernet0/1, chưa có cổng Serial. Vì sơ đồ yêu cầu Gateway–West và Gateway–East nối bằng Serial (S0/0, S0/1), cần thêm module trước:

1. Chuột phải vào Gateway, West, East → **Stop** (nếu đang chạy).
2. Chuột phải Gateway → **Configure** → tab **Slots** → ở dòng "slot0", chọn **WIC-2T** ở ô `wic0` → **Save**. Cổng Serial0/0 và Serial0/1 sẽ xuất hiện.
3. Lặp lại y hệt cho West và East.

### Bước 2 — Nối Gateway ↔ West ↔ East bằng Serial
1. Bật công cụ **Add a link** (icon mũi tên hai chiều ở toolbar dọc bên trái).
2. Click Gateway → chọn **Serial0/0** → click West → chọn **Serial0/0**.
3. Click Gateway → chọn **Serial0/1** → click East → chọn **Serial0/1**.

Kết quả: Gateway–West dùng mạng 172.16.3.0/24, Gateway–East dùng mạng 172.16.4.0/24 — đúng sơ đồ gốc.

### Bước 3 — Nối West và East vào 4 switch LAN
Vẫn dùng công cụ Add a link:

| Từ | Interface | Đến |
|---|---|---|
| West | FastEthernet0/0 | SW-LAN1 |
| West | FastEthernet0/1 | SW-LAN2 |
| East | FastEthernet0/0 | SW-LAN3 |
| East | FastEthernet0/1 | SW-LAN4 |

Click router → chọn interface tương ứng → click switch → chọn 1 cổng bất kỳ còn trống trên switch.

### Bước 4 — (Tùy chọn) Nối Internet
Click Gateway → chọn **FastEthernet0/0** → click node Cloud/NAT → chọn cổng có sẵn.

- Nếu Cloud chỉ hiện adapter VPN, hoặc NAT báo thiếu `vmnet8`: không bắt buộc phải xử lý ngay — phần chấm điểm của lab tập trung ở RIPv2 nội bộ và ACL giữa LAN2/LAN3, kết nối Internet thật là phụ. Có thể bỏ qua hoặc nối tạm, xử lý sau nếu cần (cài VMware để có vmnet8, hoặc chạy GNS3 với quyền Administrator để Cloud thấy card mạng thật).

### Bước 5 — Khởi động và kiểm tra
1. Chọn tất cả node (Ctrl+A) → bấm **Play** (tam giác xanh) để start.
2. Các ô vuông tại điểm nối chuyển từ đỏ sang **xanh lá** nghĩa là interface đã up.
3. Nếu vẫn đỏ: kiểm tra lại đúng cổng đã chọn khi vẽ dây (nhầm Serial/FastEthernet là lỗi phổ biến nhất).

---

## Phần 2: Gán IP + cấu hình RIPv2

### Bảng địa chỉ IP
 
| Router  | Interface | IP Address       | Kết nối tới                |
|---------|-----------|------------------|------------------------------|
| Gateway | F0/0      | 16.19.16.19/24   | Internet (Cloud)            |
| Gateway | S0/0      | 172.16.3.2/24    | West S0/0                   |
| Gateway | S0/1      | 172.16.4.2/24    | East S0/1                   |
| West    | S0/0      | 172.16.3.1/24    | Gateway S0/0                |
| West    | F0/0      | 10.10.1.1/24     | LAN1 — 10.10.1.0/24         |
| West    | F0/1      | 10.10.2.1/24     | LAN2 — 10.10.2.0/24         |
| East    | S0/1      | 172.16.4.1/24    | Gateway S0/1                |
| East    | F0/0      | 10.10.3.1/24     | LAN3 — 10.10.3.0/24         |
| East    | F0/1      | 10.10.4.1/24     | LAN4 — 10.10.4.0/24         |
| Server LAN2 | (Ethernet0) | 10.10.2.100/24, gateway 10.10.2.1 | Switch2 |
| Server LAN3 | (Ethernet0) | 10.10.3.100/24, gateway 10.10.3.1 | Switch3 |
 
### Cấu hình Gateway
```
hostname Gateway
!
interface FastEthernet0/0
 ip address 16.19.16.19 255.255.255.0
 no shutdown
!
interface Serial0/0
 ip address 172.16.3.2 255.255.255.0
 clock rate 64000
 no shutdown
!
interface Serial0/1
 ip address 172.16.4.2 255.255.255.0
 clock rate 64000
 no shutdown
!
router rip
 version 2
 no auto-summary
 network 172.16.0.0
 passive-interface FastEthernet0/0
```
 
### Cấu hình West
```
hostname West
!
interface Serial0/0
 ip address 172.16.3.1 255.255.255.0
 no shutdown
!
interface FastEthernet0/0
 ip address 10.10.1.1 255.255.255.0
 no shutdown
!
interface FastEthernet0/1
 ip address 10.10.2.1 255.255.255.0
 no shutdown
!
router rip
 version 2
 no auto-summary
 network 172.16.0.0
 network 10.0.0.0
```
 
### Cấu hình East
```
hostname East
!
interface Serial0/1
 ip address 172.16.4.1 255.255.255.0
 no shutdown
!
interface FastEthernet0/0
 ip address 10.10.3.1 255.255.255.0
 no shutdown
!
interface FastEthernet0/1
 ip address 10.10.4.1 255.255.255.0
 no shutdown
!
router rip
 version 2
 no auto-summary
 network 172.16.0.0
 network 10.0.0.0
```
 
### Kiểm tra
```
show ip route
show ip protocols
```
Kỳ vọng: Gateway thấy được cả 4 mạng LAN qua RIP (`R`); West thấy LAN3, LAN4; East thấy LAN1, LAN2.
 
---
 
## Phần 3: Gắn Server 2003 (IIS) vào LAN2, LAN3
 
### 3.1 Chuẩn bị VM

1. Cài **VMware Workstation Pro**.

2. Tải file `.ova` Server 2003 R2 về, lưu vào **thư mục riêng ngoài git repo** (ví dụ `E:\VMs\Server2003R2\`) — **không** lưu vào thư mục project git, vì file VM rất nặng và ở dạng nhị phân, không nên track bằng git.

3. Double-click file `.ova` → VMware tự mở wizard Import → chọn nơi lưu (đúng thư mục ở bước 2) → Import.

### 3.2 Thêm VM vào GNS3 — các lỗi thường gặp & cách xử lý
 
| Lỗi gặp phải | Nguyên nhân | Cách xử lý |
|---|---|---|
| `Could not find VMware vmrun` | GNS3 chưa biết đường dẫn tới `vmrun.exe` | Edit > Preferences > VMware > điền path đầy đủ tới file `vmrun.exe` (ví dụ `C:\Program Files\VMware\VMware Workstation\vmrun.exe`), dùng nút Browse để chọn chính xác |
| `Attachment 'bridged' is configured on network adapter 0...` | Card mạng ảo của VM đang khoá cứng chế độ Bridged | Tắt VM trong VMware → VM > Settings > Network Adapter → đổi từ Bridged sang **Host-only** hoặc **Custom** → OK |
| `Sorry a node without the linked clone setting enabled can only be used once...` | Template VM chưa bật linked clone, nên chỉ tạo được 1 node duy nhất | Edit > Preferences > VMware > VMware VMs > chọn template > tab General settings > tick **"Use as a linked base VM (experimental)"** |
| `No VMnet interface available between vmnet2 and vmnet19` | Máy chưa có VMnet tùy chỉnh nào để GNS3 cấp phát | VMware > Edit > Virtual Network Editor > Change Settings > Add Network... thêm vài VMnet mới (VMnet2, VMnet3...) |
 
### 3.3 Kết nối và gán IP

1. Thêm 2 node từ template Server 2003 R2 (đã bật linked clone) — 1 node nối vào **Switch2** (LAN2), 1 node nối vào **Switch3** (LAN3).

2. Start cả 2 node qua GNS3 (nút Play).

3. Double-click từng node để mở màn hình VM → Control Panel > Network Connections > Local Area Connection > Properties > Internet Protocol (TCP/IP) > Properties:

   - Server LAN2: IP `10.10.2.100`, mask `255.255.255.0`, gateway `10.10.2.1`
   - Server LAN3: IP `10.10.3.100`, mask `255.255.255.0`, gateway `10.10.3.1`

4. Kiểm tra: `ping` tới gateway tương ứng để xác nhận đã thông.

### 3.4 Cài và bật IIS (Web + FTP)

1. Control Panel > Add or Remove Programs > **Add/Remove Windows Components**.

2. Tích **Application Server** → **Details...** → tích **Internet Information Services (IIS)** → **Details...** → đảm bảo tích cả **World Wide Web Service** và **File Transfer Protocol (FTP) Service**.

3. Next, cung cấp đường dẫn cài đặt (`i386`) nếu được hỏi → Finish.

4. Administrative Tools > **Internet Information Services (IIS) Manager** → đảm bảo **Default Web Site** và **Default FTP Site** đang **Start**.

 
---
 
## Phần 4: Cấu hình ACL

### Yêu cầu
- Cho phép **FTP** giữa LAN2 (10.10.2.0/24) và LAN3 (10.10.3.0/24), cả 2 chiều.
- Chặn **các kết nối khác** (Ping/ICMP, Web/HTTP...) giữa LAN2 và LAN3.
- Cho phép **mọi kết nối khác** không liên quan LAN2↔LAN3.

### Cấu hình trên Router West

#### Cấu hình Extended ACL trên Router

Để tối ưu hóa tài nguyên xử lý của router, ta sử dụng Extended ACL và đặt tại Router West (áp dụng gần nguồn phát lưu lượng từ LAN2).

Cần lọc các lưu lượng giữa LAN2 (10.10.2.0/24) và LAN3 (10.10.3.0/24):

- Cho phép FTP (TCP port 21 và port 20 cho dữ liệu) từ LAN2 sang LAN3.

- Cho phép FTP chiều ngược lại từ LAN3 sang LAN2.

- Chặn tất cả các lưu lượng khác giữa LAN2 và LAN3 (ICMP/Ping, HTTP port 80,...).

- Cho phép tất cả các lưu lượng còn lại.

Các câu lệnh cấu hình trên Router West:

```
configure terminal
ip access-list extended ACL_LAN2_LAN3

remark
permit tcp 10.10.2.0 0.0.0.255 10.10.3.0 0.0.0.255 eq ftp
permit tcp 10.10.2.0 0.0.0.255 10.10.3.0 0.0.0.255 eq ftp-data

remark
permit tcp 10.10.3.0 0.0.0.255 10.10.2.0 0.0.0.255 eq ftp
permit tcp 10.10.3.0 0.0.0.255 10.10.2.0 0.0.0.255 eq ftp-data

remark
deny ip 10.10.2.0 0.0.0.255 10.10.3.0 0.0.0.255
deny ip 10.10.3.0 0.0.0.255 10.10.2.0 0.0.0.255

remark
permit ip any any
exit
```

#### Gắn ACL vào interface FastEthernet0/1 (LAN2) của Router West

```
configure terminal
interface FastEthernet0/1
ip access-group ACL_LAN2_LAN3 in
exit
end
write memory
```

 
### Kết quả kiểm tra thực tế
 
**Ping (ICMP) — từ server LAN2 (10.10.2.100) đến server LAN3 (10.10.3.100):**
```
Pinging 10.10.3.100 with 32 bytes of data:
Reply from 10.10.2.1: Destination net unreachable. (x4)
```

→ **Bị chặn**
 
**Web (HTTP):** truy cập bị chặn.
 
**FTP — từ server LAN2 đến server LAN3:**

```
C:\...>ftp 10.10.3.100
Connected to 10.10.3.100.
220 Microsoft FTP Service
User (10.10.3.100:(none)): Administrator
331 Password required for Administrator.
Password:
230 User Administrator logged in.
ftp> dir
... (hiển thị ftp_test, iisstart.htm, pagerror.gif)
ftp> get ftp_test
226 Transfer complete.
ftp: 9 bytes received.
ftp> quit
221
 
C:\...>dir
...  9 ftp_test
```
→ **Đăng nhập, `dir`, `get` và file xuất hiện lại trên máy nguồn — FTP hoạt động đúng yêu cầu, được cho phép qua ACL.**
 
**Kết luận:** ACL hoạt động đúng cả 3 tiêu chí — chặn Ping, chặn Web, cho phép FTP giữa LAN2 và LAN3. ✅