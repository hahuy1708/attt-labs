# GNS3 + Windows Server 2012 DC + Join Windows 8

Domain: `dut.lab` (NetBIOS: `DUT`)

---

## 1. Topology và VLSM

**Nối dây**

| Từ                | Đến     |
| ----------------- | ------- |
| Server2K12-1 (e0) | R1 f0/0 |
| R1 s1/0           | R2 s1/0 |
| R1 s1/1           | R3 s1/1 |
| Windows8-1 (e0)   | R2 f0/0 |

**VLSM từ 172.16.0.0/16**

| Mạng | Host cần | Prefix | Network | Dải host | Broadcast |
|---|---|---|---|---|---|
| LAN Client | 6969 | /19 | 172.16.0.0 | 172.16.0.1 – 172.16.31.254 | 172.16.31.255 |
| LAN3 | 3969 | /20 | 172.16.32.0 | 172.16.32.1 – 172.16.47.254 | 172.16.47.255 |
| Link R1–R2 | 2 | /30 | 172.16.48.0 | .1 – .2 | 172.16.48.3 |
| Link R1–R3 | 2 | /30 | 172.16.48.4 | .5 – .6 | 172.16.48.7 |

Mạng R1–Server: `16.19.16.0/24` (đề cho sẵn, không lấy từ 172.16.0.0/16).

**Bảng IP**

| Thiết bị | Cổng | IP |
|---|---|---|
| R1 | f0/0 | 16.19.16.1/24 |
| R1 | s1/0 | 172.16.48.1/30 |
| R1 | s1/1 | 172.16.48.5/30 |
| R2 | f0/0 | 172.16.0.1/19 |
| R2 | s1/0 | 172.16.48.2/30 |
| R3 | Loopback 3 | 172.16.32.1/20 |
| R3 | s1/1 | 172.16.48.6/30 |
| Server2K12-1 | e0 | 16.19.16.20/24, GW 16.19.16.1, DNS 16.19.16.20 |
| Windows8-1 | e0 | 172.16.0.10/19, GW 172.16.0.1, DNS 16.19.16.20 |

---

## 2. Cấu hình router

**R1**
```
enable
configure terminal
hostname R1
interface f0/0
 ip address 16.19.16.1 255.255.255.0
 no shutdown
exit
interface s1/0
 ip address 172.16.48.1 255.255.255.252
 no shutdown
exit
interface s1/1
 ip address 172.16.48.5 255.255.255.252
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 16.19.16.20
router rip
 version 2
 no auto-summary
 network 172.16.0.0
 network 16.0.0.0
 passive-interface f0/0
 default-information originate
end
write memory
```

**R2**
```
enable
configure terminal
hostname R2
interface f0/0
 ip address 172.16.0.1 255.255.224.0
 no shutdown
exit
interface s1/0
 ip address 172.16.48.2 255.255.255.252
 no shutdown
exit
router rip
 version 2
 no auto-summary
 network 172.16.0.0
 passive-interface f0/0
end
write memory
```

**R3**
```
enable
configure terminal
hostname R3
interface loopback 3
 ip address 172.16.32.1 255.255.240.0
exit
interface s1/1
 ip address 172.16.48.6 255.255.255.252
 no shutdown
exit
router rip
 version 2
 no auto-summary
 network 172.16.0.0
end
write memory
```

**Kiểm tra router**
```
show ip interface brief    ! cổng phải up/up
show ip route              ! R cho các subnet, R* cho 0.0.0.0/0
```
Nếu serial báo `down`, thêm `clock rate 64000` trên đầu DCE (thường là R1 s1/0 và s1/1).

---

## 3. GNS3 và VMware

- **VMware:** không chỉnh VMnet cho VM dùng trong GNS3. Giữ VMnet8 nguyên cho các máy khác.
- **GNS3 router:** platform c3725, slot 1 = NM-4T, bật Idle-PC.
- **GNS3 VM node:** Thêm trực tiếp Server2K12-1 và Windows8-1 vào project.
- **Configure từng VM:** Network → **Number of adapters = 1** (không cần nhiều adapter, GNS3 sẽ gán nhiều VMnet).
- Khởi động router trước, sau đó mới start VM.

---

## 4. Cấu hình IP trên máy ảo

**Server2K12-1 (PowerShell Admin)**
```powershell
Get-NetAdapter
New-NetIPAddress -InterfaceAlias "Ethernet0 2" -IPAddress 16.19.16.20 -PrefixLength 24 -DefaultGateway 16.19.16.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0 2" -ServerAddresses 16.19.16.20
Set-NetFirewallProfile -All -Enabled False
```

**Windows8-1 (PowerShell Admin)**
```powershell
Get-NetAdapter
New-NetIPAddress -InterfaceAlias "Ethernet0 2" -IPAddress 172.16.0.10 -PrefixLength 19 -DefaultGateway 172.16.0.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0 2" -ServerAddresses 16.19.16.20
Set-NetFirewallProfile -All -Enabled False
```

Tên adapter lấy từ `Get-NetAdapter`, cột Name, dòng đang `Up`.

**Thứ tự test**

| Máy | Lệnh |
|---|---|
| Server | `ping 16.19.16.1`, `ping 172.16.0.1`, `ping 172.16.0.10` |
| Client | `ping 172.16.0.1`, `ping 16.19.16.1`, `ping 16.19.16.20` |

Tất cả phải có Reply trước khi làm tiếp phần DC.

---

## 5. Thăng cấp Domain Controller (Server2K12-1)

**Cài AD DS**
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

**Tạo forest**
```powershell
Install-ADDSForest -DomainName "dut.lab" -DomainNetbiosName "DUT" `
  -ForestMode Win2012R2 -DomainMode Win2012R2 -InstallDns `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "<DSRM_password>" -AsPlainText -Force) `
  -Force
```
Máy tự restart. Đăng nhập lại bằng `DUT\Administrator` với mật khẩu vừa đặt.

GUI tương đương: Server Manager → Add Roles and Features → AD DS → sau đó Promote → Add a new forest → nhập `dut.lab`.

**Sửa DNS trên DC sau khi restart**
```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0 2" -ServerAddresses 16.19.16.20
netsh interface teredo set state disabled
netsh interface 6to4 set state disabled
netsh interface isatap set state disabled
ipconfig /registerdns
Restart-Service DNS, Netlogon
```

**Xóa bản ghi AAAA 2002:... sai (nếu có)**
- Hoặc mở DNS Manager → Forward Lookup Zones → `dut.lab`, xóa thủ công các AAAA có địa chỉ `2002:...`.
- Hoặc bằng PowerShell:
```powershell
Get-DnsServerResourceRecord -ZoneName "dut.lab" -RRType AAAA |
  Where-Object { $_.RecordData.IPv6Address.IPAddressToString -like "2002:*" } |
  Remove-DnsServerResourceRecord -ZoneName "dut.lab" -Force
```

**Kiểm tra DC**
```powershell
Get-ADDomain
nslookup dut.lab                 ! phải ra 16.19.16.20 ở dòng Name
dcdiag /test:dns
```

---

## 6. Join Windows 8 vào domain

**Kiểm tra DNS trước khi join (Windows8-1)**
```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet0 2" -AddressFamily IPv4   ! phải là 16.19.16.20
ipconfig /flushdns
nslookup dut.lab                                                              ! phải ra 16.19.16.20
Test-NetConnection 16.19.16.20 -Port 389
Test-NetConnection 16.19.16.20 -Port 88
```

**Join**
```powershell
Add-Computer -DomainName "dut.lab" -Credential DUT\Administrator -Restart
```
GUI: This PC → Properties → Change settings → Change → chọn Domain, nhập `dut.lab`.

**Đăng nhập domain**
1. Ở màn hình đăng nhập, bấm **Other user**.
2. Gõ `DUT\Administrator` (dấu `\`, không phải `/`) hoặc `Administrator@dut.lab`.
3. Chuyển bàn phím EN, nhập mật khẩu Administrator đã dùng khi promote DC.

**Xác nhận**
```powershell
whoami                                          ! dut\administrator
Get-ADComputer -Identity "QNG-WS8-01"           ! trên DC, máy phải tồn tại
```

---

## 7. Lỗi đã gặp và cách xử lý

| Triệu chứng | Nguyên nhân | Cách xử lý |
|---|---|---|
| `No VMnet interface available between vmnet2 and vmnet19` | Tự tạo VMnet2 thủ công, Server có 4 adapter, GNS3 gán hết VMnet | Đóng VM và GNS3, xóa VMnet thủ công, đặt Number of adapters = 1, mở lại GNS3 |
| Server có 4 adapter (Ethernet0 2, Ethernet1 2, Ethernet2, Ethernet3) | Number of adapters đặt = 4 | Đặt lại = 1 trong GNS3 Configure |
| `Windows cannot find 'ncpa.cp1'` | Gõ nhầm số 1 thay chữ l | Gõ `ncpa.cpl` |
| Ping gateway từ VM báo *Destination host unreachable* (reply từ chính IP của máy) | Router chưa cấu hình nên không trả ARP | Cấu hình R1, R2, R3 rồi test lại |
| `nslookup` timeout, Server `Address: ::1` | DNS Server đặt 127.0.0.1 / IPv6, không có IPv4 DNS | Set DNS `16.19.16.20` cho adapter |
| `nslookup dut.lab` ra `2002:1013:1014::...` | Tunnel 6to4/Teredo/ISATAP đăng ký bản ghi AAAA sai | Tắt teredo, 6to4, isatap; xóa AAAA `2002:...`; `ipconfig /registerdns` |
| Client DNS là `fec0:0:0:ffff::1` | Chưa đặt DNS trên Windows 8 | Set DNS `16.19.16.20` trên Client |
| Join báo *AD DC for the domain "dut.lab" could not be contacted* | DNS Client sai | Sửa DNS Client, kiểm tra `nslookup dut.lab` |
| Join báo *The specified network resource or device is no longer available* | Client không liên lạc được DC (DNS hoặc firewall) | Sửa DNS, tắt firewall, `Test-NetConnection` cổng 389, 88, 445 |
| Nghi lệch giờ 1 tiếng | Đã kiểm tra, không phải nguyên nhân chính | Kerberos chỉ cần lệch dưới 5 phút; sửa DNS trước |
| Join thành công nhưng đăng nhập báo sai mật khẩu | Đăng nhập vào tài khoản local, hoặc gõ `dut` thay vì `DUT\Administrator` | Chọn Other user, gõ `DUT\Administrator` |
| Gõ `dut/administrator` | Dùng `/` thay vì `\` | Dùng `DUT\Administrator` |
| Gõ sai lệnh `Set-DnsClientServer Address`, `Get-Dns Server ResourceRecord` | Thừa hoặc thiếu khoảng trắng trong cmdlet | Gõ đúng `Set-DnsClientServerAddress`, `Get-DnsServerResourceRecord` |
| Đề PDF dùng `quangngai.vn` nhưng lab dùng `dut.lab` | Tên domain khác nhau | Dùng nhất quán một tên trong toàn bộ lệnh |

---

**Lưu ý:** Tắt firewall và tunnel IPv6 chỉ phù hợp cho môi trường lab.