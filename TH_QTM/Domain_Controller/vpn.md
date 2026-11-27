# Lab 3: VPN (RRAS, PPTP) trên Windows Server 2012

> Lab này làm tiếp trên hạ tầng của Lab 1 (VLSM, RIPv2, DC `dut.lab`, Client đã join domain). Không đổi cấu hình của router, Lab 2 hay domain.

---

## 1. Mục tiêu

- Cấu hình Server thành VPN Server bằng RRAS.
- Tạo user VPN có quyền dial-in.
- Client (Windows 8.1) kết nối VPN, nhận IP `192.168.200.x`.
- Client truy cập được thư mục `SHARED` trên Server qua `\\10.0.0.100\SHARED`.

---

## 2. Điều kiện đầu vào (từ Lab 1)

| Thành phần | Trạng thái cần có | Kiểm tra |
|---|---|---|
| Router | R1, R2, R3 có route RIP đầy đủ | `show ip route` trên R2 có `R` tới 172.16.32.0/20 và `R*` cho 0.0.0.0/0 |
| Server (DC) | `16.19.16.20/24`, GW `16.19.16.1`, DNS `16.19.16.20` | `Get-NetIPAddress -InterfaceAlias "Ethernet0 2" -AddressFamily IPv4` |
| Client | `172.16.0.10/19`, GW `172.16.0.1`, DNS `16.19.16.20` | `ipconfig /all` |
| Mạng | Client ping được Server | `ping 16.19.16.20` từ Client |
| Domain | Client đã join `dut.lab` | `whoami` trên Client |
| Firewall | Đã tắt (chỉ dùng cho lab) | `Get-NetFirewallProfile` |

Tên máy có thể khác nhau giữa các tài liệu (`Server2K12-1` hoặc `QNG-SVR12-01`; `Windows8-1` hoặc `QNG-WS8-01`). Kiểm tra bằng `hostname`.

---

## 3. Thiết kế IP cho Lab 3

| Thiết bị | Adapter | IP | Vai trò |
|---|---|---|---|
| Server | Ethernet0 2 | 16.19.16.20/24 | IP chính (từ Lab 1) |
| Server | Ethernet0 2 | 10.0.0.100/24 | IP phụ, làm "Internet address" của VPN |
| Client | Ethernet0 2 | 172.16.0.10/19 | Từ Lab 1 |
| VPN Client | PPP adapter VPN Connection | 192.168.200.150–200 | RRAS cấp động |

**Vì sao gắn 10.0.0.100 làm IP phụ:** PDF Bài 7 yêu cầu thêm NIC thứ hai. Đề Lab 3 yêu cầu giữ nguyên IP của VLSM và RIPv2. Gắn IP phụ trên cùng NIC đáp ứng cả hai, vì Client vẫn đến được `10.0.0.100` qua R2 → R1 → Server mà không phải đổi router hay thêm link GNS3. Nếu thầy yêu cầu đúng hai NIC, cần thêm link riêng và cấu hình lại.

---

## 4. Các bước đã thực hiện

### Bước 1: Gắn IP phụ cho Server

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0 2" -IPAddress 10.0.0.100 -PrefixLength 24
Get-NetIPAddress -InterfaceAlias "Ethernet0 2" -AddressFamily IPv4 | Format-Table IPAddress, PrefixLength
```

Kết quả: có cả `16.19.16.20` và `10.0.0.100`.

Lưu ý: lệnh đúng là `New-NetIPAddress`. Không có cmdlet `Add-NetIPAddress`. Không thêm `-DefaultGateway`.

### Bước 2: Cài RRAS

Server Manager → Add Roles and Features → Role-based → Remote Access → chọn **DirectAccess and VPN (RAS)** và các tính năng đi kèm.

Hoặc PowerShell:
```powershell
Install-WindowsFeature RemoteAccess, DirectAccess-VPN, Routing -IncludeManagementTools
```

### Bước 3: Cấu hình RRAS

Mở **Routing and Remote Access** (Server Manager → Tools).

Nếu menu chuột phải chỉ có **Disable Routing and Remote Access** thì RRAS đã bật từ trước. Chọn Disable, rồi làm lại từ đầu.

Chuột phải tên Server → **Configure and Enable Routing and Remote Access**, sau đó:

1. Màn hình đầu: Next.
2. **Configuration**: chọn **Remote access (dial-up or VPN)** → Next.
3. **Remote Access**: tick **VPN**, bỏ tick **Dial-up** → Next.
4. **VPN Connection**:
   - Chọn card nhận kết nối từ bên ngoài: **Ethernet0 2** (card có `10.0.0.100`).
   - Bỏ tick "Enable security on the selected interface by setting up static packet filters".
   - Next.
5. **Network Selection**: chọn card mạng nội bộ cho client VPN. Đã chọn **Ethernet1 2** (không có IP, Disconnected). Card Ethernet0 2 không hiện ở đây vì đã dùng ở bước trước. → Next.
6. **IP Address Assignment**: chọn **From a specified range of addresses** → Next.
7. **Address Range Assignment**: bấm New, nhập Start `192.168.200.150`, End `192.168.200.200` (51 địa chỉ) → OK → Next.
8. **Managing Multiple Remote Access Servers**: chọn **No, use Routing and Remote Access to authenticate connection requests** → Next.
9. Finish, rồi OK để **Start service**.

Kiểm tra:
```powershell
Get-Service RemoteAccess | Select Name, Status
Get-CimInstance Win32_Service -Filter "Name='RemoteAccess'" | Select Name, State, StartMode
```
Kết quả đúng: `Running`, `Auto`.

Lưu ý: `Get-Service ... | Select StartType` có thể trống trên Server 2012. Dùng lệnh `Get-CimInstance` ở trên.

### Bước 4: Tạo user VPN

```powershell
$init = ConvertTo-SecureString "Initial@Pass2026!" -AsPlainText -Force
New-ADUser -Name vpnuser -SamAccountName vpnuser -UserPrincipalName "vpnuser@dut.lab" -AccountPassword $init -Enabled $true -ChangePasswordAtLogon $false
Set-ADUser vpnuser -Replace @{msNPAllowDialin = $true}
```

Viết `New-ADUser` trên một dòng khi chạy trong PowerShell.

Kiểm tra:
```powershell
Get-ADUser vpnuser -Properties Enabled, PasswordLastSet, pwdLastSet, msNPAllowDialin | Select Name, Enabled, PasswordLastSet, pwdLastSet, msNPAllowDialin
```
Kết quả đúng: `Enabled True`, `PasswordLastSet` có giá trị, `pwdLastSet` khác 0, `msNPAllowDialin True`.

Lưu ý: `ChangePasswordAtLogon` không phải thuộc tính AD, không đưa vào `-Properties` được. `pwdLastSet = 0` nghĩa là user bị bắt đổi mật khẩu và VPN sẽ từ chối.

### Bước 5: Thư mục SHARED

```powershell
New-Item -ItemType Directory -Path C:\SHARED -Force
Set-Content -Path C:\SHARED\test.txt -Value "Ket noi VPN thanh cong"
Get-SmbShare SHARED | Select Name, Path
Get-SmbShareAccess SHARED
```

Share `SHARED` đã tồn tại và trỏ đúng `C:\SHARED`. Quyền hiện tại là `Everyone: Full` và `BUILTIN\Administrators: Full`. Đây là cấu hình quá rộng, chỉ chấp nhận trong lab. Nếu muốn chặt hơn:

```powershell
Revoke-SmbShareAccess -Name SHARED -AccountName "Everyone" -Force
Grant-SmbShareAccess -Name SHARED -AccountName "DUT\Domain Users" -AccessRight Read -Force
```

### Bước 6: Tạo kết nối VPN trên Client

Network and Sharing Center → Set up a new connection or network → Connect to a workplace → Use my Internet connection (VPN):
- Internet address: `10.0.0.100`
- Destination name: `VPN Connection`

Sau đó chỉnh tại **VPN Connection → Properties**:
- Tab **Security**: Type of VPN = **PPTP**; Data encryption = **Optional**; Allow these protocols: tick **Microsoft CHAP Version 2**.
- Tab **Networking**: để mặc định (Use default gateway on remote network đang bật).

Lý do chọn PPTP: mặc định là Automatic, Windows sẽ thử L2TP trước và phải chờ timeout. Lab không có chứng chỉ máy cho L2TP/IPsec.

Tương đương PowerShell:
```powershell
Add-VpnConnection -Name "VPN Connection" -ServerAddress 10.0.0.100 -TunnelType Pptp -AuthenticationMethod MSChapv2 -EncryptionLevel Optional -RememberCredential -PassThru
```

Trước khi kết nối, kiểm tra từ Client:
```powershell
ping 10.0.0.100
Test-NetConnection 10.0.0.100 -Port 1723
```
Phải có Reply và `TcpTestSucceeded : True`.

### Bước 7: Kết nối và kiểm tra

1. Chọn VPN Connection → Connect → nhập `DUT\vpnuser` và `Initial@Pass2026!`. Trạng thái: **Connected**.
2. Chạy `ipconfig` trong cmd. Phải có khối:
   ```
   PPP adapter VPN Connection:
      IPv4 Address. . . . . . . . . . . : 192.168.200.15x
      Default Gateway . . . . . . . . . : 0.0.0.0
   ```
3. Nhấn Win + R, gõ `\\10.0.0.100`, đăng nhập `DUT\vpnuser` nếu được hỏi, mở `SHARED` và lấy `test.txt`. Hoặc trong cmd: `dir \\10.0.0.100\SHARED`.

Yêu cầu hiển thị cmd của thầy: nền trắng, chữ xanh dương, font Consolas cỡ 18. Chỉnh qua biểu tượng góc trái thanh tiêu đề → Properties → tab Font và Colors.

### Bước 8: Kiểm tra phía Server

Routing and Remote Access → **Remote Access Clients** → thấy `vpnuser` với IP `192.168.200.x`. Nếu trống, chuột phải → Refresh.

---

## 5. Lỗi đã gặp và cách xử lý

| Triệu chứng | Nguyên nhân | Cách xử lý |
|---|---|---|
| `Add-NetIPAddress` không được nhận dạng | Không có cmdlet này | Dùng `New-NetIPAddress` |
| `New-ADUser` bị `<<` và `-SamAccount Name` lỗi | Dòng bị ngắt sai | Viết trên một dòng, đúng `-SamAccountName` |
| `Get-ADUser -Properties ChangePasswordAtLogon` báo *properties are invalid* | `ChangePasswordAtLogon` không phải thuộc tính AD | Dùng `pwdLastSet` |
| `New-SmbShare` báo *already been shared* | Share `SHARED` đã có | Không cần tạo lại; kiểm tra bằng `Get-SmbShare` |
| Menu Configure and Enable RRAS bị mờ | RRAS đã bật | Disable Routing and Remote Access, rồi cấu hình lại |
| `Get-Service RemoteAccess` cột StartType trống | PowerShell không trả thuộc tính này | Dùng `Get-CimInstance Win32_Service` |
| Network Selection không có Ethernet0 2 | Card này đã dùng ở bước VPN Connection | Chọn Ethernet1 2 làm mạng nội bộ |
| Connect báo **691** | Sai tên hoặc mật khẩu | Dùng `DUT\vpnuser` |
| Connect báo **678** | Máy đích không phản hồi | Kiểm tra `ping 10.0.0.100` và cổng 1723 |
| Connect báo **800** | Không thiết lập được kết nối | `Restart-Service RemoteAccess`; kiểm tra địa chỉ `10.0.0.100` |
| Connect báo **741/742** | Không khớp mã hóa hoặc giao thức | Type of VPN = PPTP, Encryption = Optional, bật MS-CHAP v2 |
| Đúng mật khẩu nhưng VPN vẫn từ chối | Dial-in đang là Control access through NPS | Đặt `msNPAllowDialin = True` |
| VPN đòi đổi mật khẩu | `pwdLastSet = 0` | `Set-ADUser vpnuser -ChangePasswordAtLogon $false` |
| Connected nhưng không có IP `192.168.200.x` | Pool chưa cấu hình hoặc RRAS chưa khởi động xong | Kiểm tra bước 3; `Restart-Service RemoteAccess` |
| `\\10.0.0.100` báo *Network path not found* | Dịch vụ chia sẻ file hoặc firewall | `Test-NetConnection 10.0.0.100 -Port 445`; tắt firewall |
| `\\10.0.0.100` báo *Access is denied* | Thiếu quyền share | Kiểm tra `Get-SmbShareAccess SHARED` |
