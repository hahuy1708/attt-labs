# Lab 4 — TACACS+ với GNS3, VirtualBox và VMware

## Mục lục

- [1. Import máy ảo Client](#1-cài-đặt-virtualbox-và-import-máy-ảo-client)
- [2. Dựng topology trong GNS3](#2-dựng-topology-trong-gns3)

## 1. Import máy ảo Client

1. Mở **VMware Workstation Pro**, chọn **File > Open** và trỏ đến file `XP Pro SP3.ova` (hoặc file OVA của Server 2003).
2. Chọn thư mục lưu trữ máy ảo và bấm **Import** (Nếu hiện cảnh báo OVF specification, chọn **Relax constraint**).
3. Chỉnh lại cấu hình phần cứng trước khi bật:
   - **Gán VMnet riêng biệt:**
     - Máy **XP Pro SP3** (Clients): Vào Settings > Network Adapter > Chọn **Custom** > Gán vào `VMnet2 (Host-only)`.
     - Máy **Server 2003** (TACACS_SERVER): Vào Settings > Network Adapter > Chọn **Custom** > Gán vào `VMnet3 (Host-only)`.
4. Bật máy ảo Win XP, đăng nhập user `Admin` (không mật khẩu).

## 2. Dựng topology trong GNS3

### 2.1. Thêm thiết bị vào topology

- Thêm một router làm `TACACS_Client`. Thiết bị cần tối thiểu ba interface: `f0/0`, `f1/0` và `f2/1`.
- Thêm 01 node Cloud, đổi tên thành `Clients` đại diện cho VM Win XP.
  - Click chuột phải `Clients` > **Configure** > Tab **Ethernet interfaces** > Chọn `VMware Network Adapter VMnet2` > Bấm **Add** > **Apply**.
- Thêm 01 node Cloud khác, đổi tên thành `TACACS_SERVER` đại diện cho VM Server 2003.
  - Click chuột phải `TACACS_SERVER` > **Configure** > Tab **Ethernet interfaces** > Chọn `VMware Network Adapter VMnet3` > Bấm **Add** > **Apply**.
- **Tuỳ chọn:** Nếu cần mô phỏng Internet phía sau `f1/0`, thêm một Cloud node. Không bắt buộc phải có thiết bị thật phía sau vì trọng tâm của lab là AAA.


> **Lưu ý:** Nếu VM Server đang dùng chung VMnet với dải IP `10.10.x.x` của Lab 3, nên tạo một VMnet mới riêng cho Lab 4 để tránh xung đột subnet.

### 2.2. Nối dây theo sơ đồ

| Thiết bị từ (`TACACS_Client`) | Cổng nối | Thiết bị tới (Node Cloud) | Cổng chọn trong Cloud |
| :--- | :--- | :--- | :--- |
| `TACACS_Client` | `f0/0` | `Clients` | `VMnet2` |
| `TACACS_Client` | `f0/1` | `TACACS_SERVER` | `VMnet3` |

*Sau khi cắm dây, bấm **Start** tất cả thiết bị trên GNS3. Đảm bảo các chấm kết nối chuyển sang màu xanh.*

### 2.3. Gán IP cho router `TACACS_Client`

```text
configure terminal
interface FastEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
exit

interface FastEthernet0/1
 ip address 10.0.0.1 255.255.255.0
 no shutdown
exit

interface Loopback0
 ip address 2.2.2.1 255.255.255.0
exit
end
write memory
```

### 2.4. Gán IP tĩnh cho hai VM

Control Panel -> Network Connections -> Local Area Connection -> Properties -> Internet Protocol (TCP/IP)

| Máy | Địa chỉ IP | Subnet Mask | Gateway |
| --- | --- | --- | --- |
| VM XP (Clients) | `192.168.1.100/24` | `255.255.255.0` | `192.168.1.1` |
| VM Server 2003 (`TACACS_SERVER`) | `10.0.0.100/24` | `255.255.255.0` | `10.0.0.1` |


### 2.5. Kiểm tra kết nối cơ bản

Thực hiện các kiểm tra sau trước khi cấu hình AAA:

- Từ VM XP, ping `192.168.1.1` (gateway), sau đó ping `10.0.0.100` (server). Cả hai lệnh phải thành công.
- Từ VM Server, ping `10.0.0.1` (gateway). Lệnh phải thành công.

Nếu ping thành công theo cả hai chiều, topology đã sẵn sàng để cài ACS trên server và cấu hình AAA trên router.

## 3. Công việc tiếp theo

- Cài JRE, ACS và Firefox trên VM Server. Có thể kéo thả trực tiếp vì đây là VM VMware.
- Cấu hình `aaa new-model`, `tacacs-server host 10.0.0.100 key <shared-secret>`, cùng các lệnh `aaa authentication` và `aaa authorization` trên `TACACS_Client`.
- Khai báo `TACACS_Client` là AAA Client trong giao diện quản trị ACS.
- Tạo một user kiểm thử trên ACS.
- Từ VM XP, kiểm tra quyền truy cập:
	- Chưa xác thực: không truy cập được Internet.
	- Xác thực đúng: truy cập được Internet.