# Lab buổi 2: Group Policy Object (GPO) và IIS (Web, FTP)
 
> **Prerequisite: Lab 1.** Lab này dùng lại domain, DC, Client, DNS và mạng đã dựng ở Lab 1.
 
---
 
## 0. Điều kiện tiên quyết (từ Lab 1)
 
Kiểm tra các mục sau trước khi bắt đầu. Nếu một mục không đạt, quay lại Lab 1 để sửa.
 
| Thành phần | Trạng thái cần có | Kiểm tra |
|---|---|---|
| Domain | `dut.lab`, DC là `QNg-SVR12-01` | `Get-ADDomain` trên DC |
| IP DC | `16.19.16.20/24`, GW `16.19.16.1`, DNS `16.19.16.20` | `ipconfig /all` trên DC |
| IP Client | `172.16.0.10/19`, GW `172.16.0.1`, DNS `16.19.16.20` | `ipconfig /all` trên Client |
| Mạng | Client ping được `16.19.16.20` | `ping 16.19.16.20` từ Client |
| DNS | `dut.lab` phân giải ra `16.19.16.20` | `nslookup dut.lab` |
| Join domain | Máy `QNG-WS8-01` đã có trong AD | `Get-ADComputer QNG-WS8-01` trên DC |
| Firewall | Đã tắt (chỉ dùng cho lab) | `Get-NetFirewallProfile` |
| Routing RIP | Các route `R` và `R*` đầy đủ | `show ip route` trên R1, R2, R3 |
 
> **Lưu ý về tên miền:** đề PDF dùng `quangngai.vn`, còn lab này dùng `dut.lab`. Toàn bộ lệnh trong tài liệu này dùng `dut.lab`. Không trộn hai tên trong cùng một bài.
 
Kiểm tra nhanh trên DC (PowerShell Admin):
 
```powershell
Get-ADDomain | Select Name, DNSRoot
Get-ADComputer QNG-WS8-01 | Select Name, Enabled
Get-WindowsFeature Web-Server, Web-Ftp-Server | Select Name, InstallState
```
 
Nếu `Web-Ftp-Server` chưa `Installed`:
 
```powershell
Install-WindowsFeature Web-Server, Web-Ftp-Server -IncludeAllSubFeature -IncludeManagementTools
```
 
---
 
## Phần 1: GPO (chính sách mật khẩu)
 
### 1.1 Yêu cầu
 
Có hai nhóm đối tượng với hai chính sách khác nhau:
 
| Thuộc tính | Nhân viên (Default Domain Policy) | Ban giám đốc (PSO) |
|---|---|---|
| Độ dài tối thiểu | 8 ký tự | 15 ký tự |
| Phải phức tạp | Không | Có |
| Lịch sử mật khẩu | Không yêu cầu | 5 lần liên tiếp không trùng |
| Tuổi thọ tối thiểu | 5 ngày | 5 ngày |
| Tuổi thọ tối đa | 60 ngày | 30 ngày |
| Khóa sau số lần sai | 5 lần trong 180 phút | 3 lần trong 1 ngày |
| Thời gian khóa | 1 ngày (1440 phút) | 3 ngày |
 
Nguyên tắc áp dụng:
- **Nhân viên** dùng chính sách mặc định của domain (Default Domain Policy), theo Bài 4 phần 1 của đề.
- **Ban giám đốc** dùng Password Settings Object (PSO), theo Bài 4 phần 2 của đề. PSO có độ ưu tiên cao hơn chính sách mặc định.
### 1.2 Tạo nhóm và người dùng
 
Thực hiện trên DC, PowerShell chạy với quyền Administrator.
 
**Bước 1: khai báo mật khẩu ban đầu.** Biến này chỉ tồn tại trong phiên PowerShell hiện tại. Nếu đóng cửa sổ hoặc mở phiên mới, phải chạy lại dòng này trước khi tạo user.
 
```powershell
$init = ConvertTo-SecureString "Initial@Pass2026!" -AsPlainText -Force
```
 
Mật khẩu này có 17 ký tự, đạt cả hai chính sách (chính sách của giám đốc yêu cầu tối thiểu 15).
 
**Bước 2: tạo 2 nhóm.**
 
```powershell
New-ADGroup -Name G_Nhanvien -GroupScope Global -GroupCategory Security -Path "CN=Users,DC=dut,DC=lab"
New-ADGroup -Name G_Giamdoc  -GroupScope Global -GroupCategory Security -Path "CN=Users,DC=dut,DC=lab"
```
 
**Bước 3: tạo 2 nhân viên và thêm vào G_Nhanvien.**
 
```powershell
foreach ($u in "nhanvien1","nhanvien2") {
  New-ADUser -Name $u -SamAccountName $u -UserPrincipalName "$u@dut.lab" `
    -AccountPassword $init -Enabled $true -ChangePasswordAtLogon $true
  Add-ADGroupMember -Identity G_Nhanvien -Members $u
}
```
 
**Bước 4: tạo 2 giám đốc và thêm vào G_Giamdoc.**
 
```powershell
foreach ($u in "giamdoc1","giamdoc2") {
  New-ADUser -Name $u -SamAccountName $u -UserPrincipalName "$u@dut.lab" `
    -AccountPassword $init -Enabled $true -ChangePasswordAtLogon $true
  Add-ADGroupMember -Identity G_Giamdoc -Members $u
}
```
 
`-ChangePasswordAtLogon $true` tương ứng với tùy chọn "User must change password at next logon" trong đề, để kiểm tra chính sách khi đăng nhập lần đầu.
 
Nếu gõ từng dòng, bỏ dấu `` ` `` ở cuối dòng và viết liền một dòng.
 
**Kiểm tra:**
 
```powershell
Get-ADGroupMember G_Nhanvien | Select Name
Get-ADGroupMember G_Giamdoc  | Select Name
```
 
`G_Nhanvien` phải có `nhanvien1`, `nhanvien2`. `G_Giamdoc` phải có `giamdoc1`, `giamdoc2`.
 
Nếu báo lỗi *already exists*, nhóm hoặc user đã tồn tại từ trước, bỏ qua và chạy tiếp.
 
### 1.3 Cấu hình chính sách mặc định cho nhân viên
 
**Cách 1: PowerShell (nhanh)**
 
```powershell
Set-ADDefaultDomainPasswordPolicy -Identity dut.lab `
  -MinPasswordLength 8 `
  -ComplexityEnabled $false `
  -MinPasswordAge (New-TimeSpan -Days 5) `
  -MaxPasswordAge (New-TimeSpan -Days 60) `
  -LockoutThreshold 5 `
  -LockoutObservationWindow (New-TimeSpan -Minutes 180) `
  -LockoutDuration (New-TimeSpan -Minutes 1440)
```
 
**Cách 2: GUI**
 
1. Mở **Group Policy Management** (Server Manager → Tools).
2. Chuột phải **Default Domain Policy** → **Edit**.
3. Đi theo đường dẫn: Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy.
4. Đặt các giá trị: Minimum password length = 8; Password must meet complexity requirements = Disabled; Minimum password age = 5 days; Maximum password age = 60 days.
5. Chọn Account Lockout Policy: Account lockout threshold = 5; Reset account lockout counter after = 180 minutes; Account lockout duration = 1440 minutes.
6. Đóng cửa sổ, chạy `gpupdate /force` trong PowerShell.
**Kiểm tra:**
 
```powershell
Get-ADDefaultDomainPasswordPolicy | Format-List MinPasswordLength, ComplexityEnabled, MinPasswordAge, MaxPasswordAge, LockoutThreshold, LockoutObservationWindow, LockoutDuration
```
 
Kết quả đúng: `MinPasswordLength 8`, `ComplexityEnabled False`, `LockoutThreshold 5`, `LockoutObservationWindow 00:03:00:00`, `LockoutDuration 1.00:00:00`.
 
Lệnh `net accounts /domain` cũng hiển thị các giá trị này dưới dạng dễ đọc.
 
### 1.4 Tạo PSO cho Ban giám đốc
 
**Cách 1: PowerShell**
 
```powershell
New-ADFineGrainedPasswordPolicy -Name PSO_BanGiamDoc -Precedence 1 `
  -MinPasswordLength 15 `
  -ComplexityEnabled $true `
  -PasswordHistoryCount 5 `
  -MinPasswordAge (New-TimeSpan -Days 5) `
  -MaxPasswordAge (New-TimeSpan -Days 30) `
  -LockoutThreshold 3 `
  -LockoutObservationWindow (New-TimeSpan -Days 1) `
  -LockoutDuration (New-TimeSpan -Days 3) `
  -ReversibleEncryptionEnabled $false
 
Add-ADFineGrainedPasswordPolicySubject -Identity PSO_BanGiamDoc -Subjects G_Giamdoc
```
 
`Precedence 1` là độ ưu tiên cao nhất. Giá trị càng nhỏ càng ưu tiên. PSO chỉ áp dụng được cho user hoặc global security group, nên áp cho nhóm `G_Giamdoc`.
 
**Cách 2: ADSI Edit (theo Bài 4 của đề)**
 
1. Mở **ADSI Edit**, chuột phải → **Connect to…** → OK (dùng Default naming context).
2. Đi đến `CN=Password Settings Container`, chuột phải vùng bên phải → **New → Object**, chọn `msDS-PasswordSettings`.
3. Nhập lần lượt: cn = `Ban Giam Doc`; msDS-PasswordSettingsPrecedence = `1`; msDS-PasswordReversibleEncryptionEnabled = `false`; msDS-PasswordHistoryLength = `5`; msDS-PasswordComplexityEnabled = `true`; msDS-MinimumPasswordLength = `15`; msDS-MinimumPasswordAge = `5:0:0:0`; msDS-MaximumPasswordAge = `30:0:0:0`; msDS-LockoutThreshold = `3`; msDS-LockoutObservationWindow = `1:0:0:0`; msDS-LockoutDuration = `3:0:0:0`.
4. Mở thuộc tính đối tượng, chọn **msDS-PSOAppliesTo → Edit → Add Windows Account…**, nhập `G_Giamdoc` → OK.
Nếu dùng ADSI Edit, giá trị Duration có dạng `Ngày:Giờ:Phút:Giây`.
 
### 1.5 Kiểm tra PSO đã áp dụng
 
```powershell
gpupdate /force
 
Get-ADUserResultantPasswordPolicy -Identity giamdoc1 |
  Format-List Name, MinPasswordLength, ComplexityEnabled, MaxPasswordAge, LockoutThreshold
```
 
Kết quả đúng:
 
```
Name              : PSO_BanGiamDoc
MinPasswordLength : 15
ComplexityEnabled : True
MaxPasswordAge    : 30.00:00:00
LockoutThreshold  : 3
```
 
**Về nhân viên:** lệnh `Get-ADUserResultantPasswordPolicy` **không trả về gì** cho `nhanvien1`. Đây là đúng, vì lệnh này chỉ hiển thị PSO. Nhân viên dùng chính sách mặc định, nên kiểm tra bằng:
 
```powershell
Get-ADDefaultDomainPasswordPolicy | Select MinPasswordLength, ComplexityEnabled
Get-ADUser nhanvien1 -Properties msDS-ResultantPSO | Select Name, msDS-ResultantPSO
```
 
`msDS-ResultantPSO` của `nhanvien1` phải **trống**. Nếu có giá trị, user đang bị PSO khác áp vào, cần kiểm tra lại nhóm.
 
Hai kết quả nên chụp để nộp: `Get-ADUserResultantPasswordPolicy` của giám đốc và `Get-ADDefaultDomainPasswordPolicy` cho nhân viên.
 
### 1.6 Test chính sách trên Client (Windows 8)
 
**Đăng nhập lần đầu:**
 
1. Trên Client, đăng xuất tài khoản hiện tại.
2. Ở màn hình đăng nhập, bấm **Other user**.
3. Gõ `DUT\giamdoc1` (dấu `\`, không phải `/`).
4. Chuyển bàn phím sang **EN** (`Win + Space`) trước khi gõ mật khẩu.
5. Nhập `Initial@Pass2026!`. Windows sẽ yêu cầu đổi mật khẩu.
**Kết quả thử nghiệm đổi mật khẩu:**
 
| Tài khoản | Mật khẩu mới thử | Số ký tự | Kết quả mong đợi |
|---|---|---|---|
| giamdoc1 | `1234qwer!@#$` | 12 | Báo không đúng chính sách |
| giamdoc1 | `12345qwert!@#$%` | 15 | Đổi thành công, đăng nhập được |
| nhanvien1 | `1234567` | 7 | Báo không đúng chính sách |
| nhanvien1 | `12345678` | 8 | Đổi thành công, đăng nhập được |
 
Ghi chú khi test:
- Mật khẩu `1234qwer!@#$` có đủ chữ, số và ký tự đặc biệt nhưng chỉ 12 ký tự, nên bị từ chối vì chính sách yêu cầu 15.
- Sau khi giám đốc đổi thành công, tuổi thọ tối thiểu 5 ngày nghĩa là không đổi lại được trong 5 ngày. Nếu cần đổi lại để làm tiếp, admin reset trên DC:
```powershell
$new = ConvertTo-SecureString "MatKhauMoi@2026xyz" -AsPlainText -Force
Set-ADAccountPassword -Identity giamdoc1 -Reset -NewPassword $new
Set-ADUser -Identity giamdoc1 -ChangePasswordAtLogon $true
```
 
 
---
 
## Phần 2: IIS (Web và FTP)
 
Phần này làm trên DC. Test từ Client.
 
### 2.1 Web: gán IP cho Default Web Site
 
Theo Bài 6 mục 6.1 bước 6, site phải lắng nghe đúng IP của server.
 
**Cách GUI:**
 
1. Mở **IIS Manager** (Server Manager → Tools → Internet Information Services (IIS) Manager).
2. Mở **DC → Sites → Default Web Site**.
3. Ở khung bên phải, bấm **Bindings…**.
4. Chọn dòng `http`, bấm **Edit…**.
5. IP address chọn `16.19.16.20`, Port giữ `80`, bấm **OK** → **Close**.
**Cách PowerShell:**
 
```powershell
Import-Module WebAdministration
Set-WebBinding -Name "Default Web Site" -BindingInformation "*:80:" -PropertyName IPAddress -Value "16.19.16.20"
Start-Website -Name "Default Web Site"
Get-WebBinding -Name "Default Web Site"
```
 
Kết quả `Get-WebBinding` phải có `16.19.16.20:80`.
 
**Kiểm tra trên DC:**
 
```powershell
Invoke-WebRequest -UseBasicParsing http://16.19.16.20 | Select StatusCode
```
 
Kết quả `200` là đạt.
 
**Kiểm tra trên Client:** mở trình duyệt, gõ `http://16.19.16.20`, phải thấy trang chào IIS 8. Hoặc trong PowerShell trên Client chạy `Invoke-WebRequest -UseBasicParsing http://16.19.16.20 | Select StatusCode`.
 
### 2.2 FTP: tạo user userftp
 
```powershell
$init = ConvertTo-SecureString "Initial@Pass2026!" -AsPlainText -Force
 
New-ADUser -Name userftp -SamAccountName userftp -UserPrincipalName "userftp@dut.lab" `
  -AccountPassword $init -Enabled $true -ChangePasswordAtLogon $false
```
 
Phải để `-ChangePasswordAtLogon $false`. Nếu bật, FTP sẽ từ chối đăng nhập vì không có phiên tương tác để đổi mật khẩu.
 
Kiểm tra:
 
```powershell
Get-ADUser userftp -Properties Enabled | Select Name, Enabled
```
 
Nếu tài khoản đã có sẵn với `ChangePasswordAtLogon` đang bật, sửa lại:
 
```powershell
Set-ADUser userftp -ChangePasswordAtLogon $false
```
 
### 2.3 FTP: thư mục, file kiểm tra và quyền NTFS
 
```powershell
New-Item -ItemType Directory -Path C:\FTP -Force
Set-Content -Path C:\FTP\Test.txt -Value "Dich vu FTP chay thanh cong"
icacls C:\FTP /grant "DUT\userftp:(OI)(CI)(RX)"
icacls C:\FTP
```
 
- `(RX)` là quyền đọc và thực thi, kèm liệt kê thư mục. Tương đương Bài 6 mục 6.3 bước 3.
- `(OI)(CI)` áp quyền cho các file và thư mục con.
Kết quả của `icacls C:\FTP` phải có dòng `DUT\userftp:(OI)(CI)(RX)`.
 
Nếu báo *No mapping between account names and security IDs*, nghĩa là `userftp` chưa tồn tại trong AD. Quay lại mục 2.2.
 
### 2.4 FTP: tạo FTP site (GUI)
 
Theo Bài 6 mục 6.3 bước 4 đến 6.
 
1. Trong IIS Manager, chuột phải **Sites** → **Add FTP Site…**.
2. **Site Information**: FTP site name = `FTP`; Physical path = `C:\FTP`. Bấm **Next**.
3. **Binding and SSL Settings**: IP Address = `16.19.16.20`; Port = `21`; tick **Start FTP site automatically**; SSL = **No SSL**. Bấm **Next**.
4. **Authentication and Authorization Information**:
   - Authentication: tick **Basic**, bỏ tick **Anonymous**.
   - Authorization, Allow access to: **Specified users**, nhập `DUT\userftp`.
   - Permissions: tick **Read**, bỏ tick **Write**.
   - Bấm **Finish**.
Nếu IIS không chấp nhận `DUT\userftp`, thử `userftp@dut.lab`. Dùng một cách viết nhất quán ở cả bước cấp quyền lẫn bước đăng nhập.
 
**Kiểm tra quy tắc ủy quyền:** chọn site **FTP**, bấm đúp **FTP Authorization Rules**.
- Chỉ có một quy tắc: **Allow**, **Specified users**, `DUT\userftp`, quyền **Read**.
- Nếu có quy tắc **All Users** hoặc **Anonymous Users**, xóa đi.
Nếu thiếu quy tắc: chuột phải trong khung → **Add Allow Rule…** → Specified users = `DUT\userftp`, tick **Read** → **OK**.
 
**Kiểm tra dịch vụ:**
 
```powershell
Get-Service ftpsvc | Select Name, Status
netstat -an | findstr ":21"
```
 
`ftpsvc` phải `Running`, và `netstat` phải có dòng `16.19.16.20:21 ... LISTENING`. Nếu site dừng: `Start-WebItem "IIS:\Sites\FTP"`.
 
### 2.5 Test FTP trên Client (Windows 8)
 
Mở **Command Prompt** (không cần quyền admin):
 
```
ftp 16.19.16.20
User (16.19.16.20:(none)): userftp@dut.lab
Password: Initial@Pass2026!
```
 
Sau khi đăng nhập được (`230 User logged in`):
 
```
ftp> dir
ftp> lcd C:\Users\giamdoc1
ftp> get Test.txt
ftp> quit
```
 
Thay `giamdoc1` bằng tên user đang đăng nhập trên Client.
 
**Vì sao không tải thẳng vào `C:\`:** tài khoản người dùng thường không có quyền ghi vào thư mục gốc `C:\`. Lệnh `lcd C:\` vẫn thành công vì chỉ đổi thư mục hiện tại, nhưng `get` sẽ báo `R:I/O Error` khi ghi file. Lưu vào profile của user (`C:\Users\<tên>`) là cách không cần quyền admin.
 
Nếu thầy yêu cầu file nằm đúng ở `C:\`, mở **Command Prompt as Administrator** rồi chạy lại các lệnh trên với `lcd C:\`.
 
**Kiểm tra file trên Client:**
 
```
C:\> dir C:\Users\giamdoc1\Test.txt
C:\> type C:\Users\giamdoc1\Test.txt
```
 
Phải thấy file 29 bytes và nội dung `Dich vu FTP chay thanh cong`.
 
Nếu `dir` báo `425 Can't open data connection`, gõ `passive` rồi chạy lại `dir`.
 
---
 
## Phần 3: Các lỗi đã gặp và cách xử lý
 
| Triệu chứng | Nguyên nhân | Cách xử lý |
|---|---|---|
| `New-ADUser` báo *password does not meet the length, complexity, or history requirement* | Biến `$init` trống do phiên PowerShell đã đóng hoặc mở lại | Chạy lại `$init = ConvertTo-SecureString ...` trong cùng khối lệnh với `New-ADUser` |
| `Get-ADUser userftp` báo *Cannot find an object* | User chưa được tạo do lỗi trên | Tạo lại user sau khi sửa `$init` |
| `icacls` báo *No mapping between account names and security IDs* | `DUT\userftp` chưa có trong domain | Tạo user trước, rồi chạy `icacls` |
| Đăng nhập Client báo sai mật khẩu nhưng đúng mật khẩu | Đang đăng nhập vào tài khoản local, hoặc gõ `/` thay `\` | Chọn **Other user**, gõ `DUT\giamdoc1` |
| Đăng nhập báo *account is locked out* | Nhập sai quá 3 lần (PSO giám đốc) | `Unlock-ADAccount -Identity giamdoc1` |
| Đổi mật khẩu bị từ chối dù đủ ký tự | Chưa đạt độ phức tạp hoặc độ dài của chính sách áp dụng | Kiểm tra lại số ký tự và các loại ký tự theo bảng 1.1 |
| Không đổi được mật khẩu, báo *too soon* | Chưa qua tuổi thọ tối thiểu 5 ngày | Admin reset bằng `Set-ADAccountPassword -Reset` |
| `Get-ADUserResultantPasswordPolicy` của nhân viên không có kết quả | Đây là hành vi bình thường, nhân viên không có PSO | Dùng `Get-ADDefaultDomainPasswordPolicy` để kiểm tra |
| `530 Login incorrect` khi FTP | Sai tên hoặc mật khẩu, chưa bật Basic, hoặc user bị bắt đổi mật khẩu | Thử `userftp@dut.lab`, bật Basic, đặt `ChangePasswordAtLogon $false` |
| `550 Access denied` khi FTP | Thiếu quyền NTFS | Chạy lại `icacls C:\FTP /grant "DUT\userftp:(OI)(CI)(RX)"` |
| `R:I/O Error` khi `get` | Tài khoản không có quyền ghi vào thư mục đích | Dùng `lcd C:\Users\<tên user>` |
| `425 Can't open data connection` | Chế độ kết nối dữ liệu | Gõ `passive` trong ftp |
| `Connection refused` ở cổng 21 | FTP site dừng, hoặc sai IP binding | `netstat -an \| findstr ":21"`, `Start-WebItem "IIS:\Sites\FTP"` |
| Web không mở từ Client | Binding sai hoặc W3SVC dừng | `Get-WebBinding`, `Start-Website "Default Web Site"` |
| Không kết nối được dù service đã chạy | Firewall còn bật | `Set-NetFirewallProfile -All -Enabled False` trên DC và Client |
 
Để xem chi tiết lỗi FTP, mở file log mới nhất trong `C:\inetpub\logs\LogFiles\FTPSVC1\` trên DC.
 
---
 
## Ghi chú
 
- Mật khẩu `Initial@Pass2026!` và các mật khẩu thử nghiệm chỉ dùng trong lab. Không dùng trong môi trường thật.
- Việc tắt firewall và cho phép Basic authentication FTP chỉ phù hợp với lab. Ghi chú điều này trong báo cáo.
- Các lệnh trong tài liệu dùng tên miền `dut.lab` và tên máy `QNG-WS8-01`, khớp với Lab 1.