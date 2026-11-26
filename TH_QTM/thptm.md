# ĐỀ THI THỰC HÀNH QUẢN TRỊ MẠNG

Sử dụng VMWare Workstation Pro 17, GNS3 v2.2.61 và các máy ảo Windows Server 2012 và Windows 8 (Client) triển khai cơ sở hạ tầng mạng như đề bài minh họa.

## Câu 1 (4 điểm)

### 1.1. (1 điểm)
Phân chia các mạng con (subnet) theo kỹ thuật VLSM từ mạng tổng `172.16.0.0/16` với các yêu cầu sau:

- **1.1.1.** LAN chứa Client: 6969 trạm (hosts).
- **1.1.2.** LAN 3 chứa 3969 trạm (hosts).

### 1.2. (1 điểm)
Định tuyến RIPv2 giữa các subnet; kết nối giữa R1 và Server2K12 là tuyến ngầm định (Default Route). Chú ý: LAN3 có thể sử dụng cổng loopback.

### 1.3. (2 điểm)
Thăng cấp Server lên làm Domain Controller và gia nhập Windows 8 vào Domain.

## Câu 2 (4 điểm)

### 2.1. (2 điểm)
Nhằm tăng mức độ an toàn thông tin cho lãnh đạo đơn vị, bạn được yêu cầu xây dựng chính sách mật khẩu dành riêng cho ban giám đốc như sau:

- Mật khẩu phải có ít nhất 15 ký tự.
- Trong 5 lần liên tiếp không được đổi mật khẩu trùng nhau.
- Phải phức tạp.
- Thời gian tối thiểu: 5 ngày.
- Thời gian sử dụng tối đa: 30 ngày.
- Đăng nhập sai 3 lần trong 1 ngày thì tài khoản đó sẽ bị khóa 3 ngày.

Với vai trò là Quản trị viên hãy thực hiện yêu cầu trên.

### 2.2. (2 điểm)
Cấu hình Server cung cấp các dịch vụ Web và FTP cho Client.

## Câu 3 (2 điểm)

Triển khai mạng riêng ảo (VPN) trên Server để Client có thể kết nối đến Server và lấy dữ liệu an toàn, bảo mật.

---

## Sơ đồ mạng trong đề (trang 1)

Sơ đồ minh họa các thiết bị và kết nối:

- **Server 2012** kết nối với R1 qua VMnet1; trên hình, Server được ghi địa chỉ `19.16.16.20/24`.
- **R1** kết nối Server ở phía trên, R2 qua liên kết serial `S1/0`, và R3 qua liên kết serial `S1/1`.
- **R2** kết nối đến **Client** qua VMnet8; phía R2 ghi cổng `F0/0` hướng về Client và `S1/0` hướng về R1.
- **LAN 3** được ghi chú là có thể dùng cổng loopback. Phần ghi chú trong hình hướng dẫn cấu hình loopback (các dòng lệnh nhỏ trong ảnh không đọc rõ hoàn toàn).

