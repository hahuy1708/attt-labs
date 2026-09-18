# Lab 4 — Triển khai AAA (TACACS+) cho công ty Banana

## 1. Mục tiêu đề bài

Nhân viên công ty Banana khi muốn truy cập **Internet** phải đi qua cơ chế **AAA (Authentication, Authorization, Accounting)** dựa trên giao thức **TACACS+**, gồm hai thành phần chính:

- **TACACS_Client**: thiết bị Router/Switch — đóng vai trò AAA Client, chặn traffic của Clients và bắt xác thực trước khi cho ra Internet.
- **TACACS_SERVER**: máy chủ Windows Server 2003 cài phần mềm **Cisco Secure ACS 4.2** — đóng vai trò AAA Server, thực hiện xác thực (Authentication) và cấp quyền (Authorization).

Yêu cầu: **Clients không xác thực được qua TACACS_SERVER thì không được phép ra Internet.**

**⚠️Lưu ý**: "hệ thống TACACS+" chỉ gồm 2 thành phần kể trên (TACACS_Client = router/switch, TACACS_SERVER = Server 2003+ACS). Nhưng để test được hệ thống này, đề bài còn ngầm định cần thành phần thứ 3: "Clients" — máy trạm Windows XP đại diện nhân viên, là đối tượng bị xác thực (không phải là thành phần của hệ thống TACACS). "TACACS_Client" (router) và "Clients" (máy nhân viên) là hai thứ khác nhau, dù tên rất dễ nhầm.

## 2. Sơ đồ mạng & bảng địa chỉ IP

| Thiết bị/Segment      | Interface | Subnet          | Kết nối tới                          |
|-----------------------|-----------|-----------------|----------------------------------------|
| Clients (LAN nội bộ)  | —         | 192.168.1.0/24  | TACACS_Client (f0/0)                  |
| TACACS_Client         | f0/0      | 192.168.1.0/24  | Clients                               |
| TACACS_Client         | f1/0      | 2.2.2.0/24      | Internet (qua vboxnet2)               |
| TACACS_Client         | f2/1      | 10.0.0.0/24     | TACACS_SERVER (qua vboxnet1)          |
| TACACS_SERVER         | —         | 10.0.0.0/24     | TACACS_Client (f2/1)                  |
| Internet              | —         | 2.2.2.0/24      | TACACS_Client (f1/0) qua vboxnet2     |


## 3. Vai trò từng thành phần

- **Clients**: máy trạm (Windows XP) trong mạng 192.168.1.0/24, đại diện nhân viên công ty Banana, cần xác thực trước khi ra Internet.
- **TACACS_Client**: Router/Switch chạy Cisco IOS, cấu hình AAA để gửi yêu cầu xác thực/cấp quyền tới TACACS_SERVER (giao thức TACACS+, thường qua TCP port 49).
- **TACACS_SERVER**: Server 2003 cài **Cisco Secure ACS v4.2.124** — quản lý danh sách user, chính sách xác thực và phân quyền truy cập Internet.
- **Internet**: mạng ngoài (2.2.2.0/24) mà Clients chỉ được truy cập sau khi AAA xác thực + cấp quyền thành công. Chỉ mang tính tượng trưng — chỉ cần gán IP tĩnh cho interface f1/0, không bắt buộc phải bridge ra Internet thật hay có thiết bị nào đứng sau (tương tự cách Gateway ở Lab 3 xử lý cổng ra Internet).

## 4. Cài đặt phần mềm trên TACACS_SERVER (thứ tự bắt buộc)

Tài nguyên nằm trong thư mục **Tài liệu học tập → TACACS**:

1. **Cài Java Runtime**: `jre-6u13-windows-i586-p-s.exe`
   — bắt buộc cài trước vì Cisco Secure ACS 4.2 cần JRE để chạy giao diện quản trị.
2. **Giải nén và cài Cisco Secure ACS**: giải nén `ACSv4.2.124 FULL-K9.zip`, chạy `setup.exe`
   — đây chính là phần mềm TACACS+ Server.
3. **Cài Firefox 2.0**: `Firefox 2.0.exe`
   — dùng để thao tác giao diện quản trị ACS dễ hơn so với IE6 mặc định trên Server 2003.

## 5. Ghi chú về máy ảo

- Nếu chưa có sẵn máy ảo VMware (XP Client & Server 2003), có thể **import trực tiếp file `.ova`** có sẵn trong thư mục Tài liệu học tập → TACACS thay vì cài mới từ đầu.

## 6. Checklist triển khai (đề xuất)

- [ ] Bước 1: Dựng topology (TACACS_Client + TACACS_Server + Clients + Internet) trong công cụ mô phỏng, tạo/bridge `vboxnet1`, `vboxnet2`
- [ ] Bước 2: Gán IP cho các interface của TACACS_Client (f0/0, f1/0, f2/1) và cho Clients, TACACS_Server, Internet gateway theo bảng ở mục 2
- [ ] Bước 3: Import hoặc dựng máy ảo XP Client và Server 2003 (dùng `.ova` nếu có sẵn)
- [ ] Bước 4: Trên Server 2003 — cài Java Runtime → cài Cisco Secure ACS → cài Firefox 2.0 (đúng thứ tự)
- [ ] Bước 5: Cấu hình Cisco Secure ACS — khai báo TACACS_Client là **AAA Client** (địa chỉ 10.0.0.x, shared secret key), tạo user/group cho nhân viên, thiết lập chính sách Authorization (ví dụ: cho phép truy cập Internet)
- [ ] Bước 6: Cấu hình AAA trên TACACS_Client (Router/Switch) — bật `aaa new-model`, khai báo `tacacs-server host 10.0.0.x key <shared-secret>`, cấu hình `aaa authentication login` và `aaa authorization` trỏ về TACACS+
- [ ] Bước 7: Kiểm tra định tuyến/kết nối cơ bản giữa TACACS_Client ↔ TACACS_Server (ping 10.0.0.0/24) và giữa TACACS_Client ↔ Internet (2.2.2.0/24)
- [ ] Bước 8: Test từ Clients — xác nhận: chưa đăng nhập/xác thực thì không ra được Internet; sau khi nhập đúng user/pass được ACS xác thực + authorize thì truy cập Internet bình thường
- [ ] Bước 9: Test trường hợp sai thông tin đăng nhập hoặc user không có quyền — xác nhận bị từ chối truy cập Internet