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

## Phần 3: Gắn Server 2003 (IIS) vào LAN2, LAN3

## Phần 4: Cấu hình ACL