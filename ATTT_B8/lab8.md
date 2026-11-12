# Lab 8: Bảo mật Switch Lớp 2 trên vIOS-L2 (GNS3 + QEMU)

---

## 0. Điều chỉnh so với đề gốc

| Hạng mục | Đề gốc (c3725 + NM-16ESW) | Thực hiện trên vIOS-L2 |
| --- | --- | --- |
| Thiết bị switch | Router c3725 gắn NM-16ESW | Node vIOS-L2 (QEMU) |
| Tên cổng | `Fa2/x`, `Fa0/5`, `Fa0/6`, `Fa0/18` | `Gi0/x` (8 cổng: Gi0/0 đến Gi1/3) |
| R1 nối switch | F0/1 | **Fa0/0** (theo topology) |
| PC-B | Fa0/18 (không tồn tại trên NM-16ESW) | vios-2 **Gi0/2** |
| Port security | Không chạy được trên c3725 | **Chạy được** (cú pháp có điều chỉnh, xem mục 6) |
| Storm control | Có trên c3725 | **Không có** trên vIOS-L2, dùng biện pháp thay thế (mục 7) |
| Tên switch | S1, S2 | Node vios-1 = **S1**, vios-2 = **S2** |

---

## 1. Đề bài

1. **SSH** đến được cả hai switch.
2. **STP**: PortFast, BPDU guard, storm control.
3. **Bảo mật cổng**: port security, sticky MAC.

Yêu cầu kiểm tra:

- Từ R1 ping được các VPCS.
- Từ R1 SSH được tới switch.
- `show spanning-tree` để xem switch root và secondary.
- `show running-config` để xem PortFast, BPDU guard, port security, sticky MAC.

---

## 2. Topology và bảng địa chỉ

### 2.1. Sơ đồ

```
R1 Fa0/0 ──── vios-1 (S1) Gi0/0
                 Gi0/1 ──── Gi0/1  vios-2 (S2)
                 Gi0/2 ──── PC1 (PC-A)     Gi0/2 ──── PC2 (PC-B)
```

Ánh xạ adapter GNS3 sang tên cổng: `e0` = `Gi0/0`, `e1` = `Gi0/1`, `e2` = `Gi0/2`.

### 2.2. Bảng IP

| Thiết bị | Node GNS3 | Giao diện | IP | Mask | Gateway |
| --- | --- | --- | --- | --- | --- |
| R1 | R1 | Fa0/0 | 192.168.1.1 | 255.255.255.0 | — |
| S1 | vios-1 | VLAN 1 | 192.168.1.2 | 255.255.255.0 | 192.168.1.1 |
| S2 | vios-2 | VLAN 1 | 192.168.1.3 | 255.255.255.0 | 192.168.1.1 |
| PC-A | PC1 | e0 | 192.168.1.10 | 255.255.255.0 | 192.168.1.1 |
| PC-B | PC2 | e0 | 192.168.1.11 | 255.255.255.0 | 192.168.1.1 |

### 2.3. Bảng cổng switch

| Switch | Cổng | Nối với | Chế độ | Bảo mật |
| --- | --- | --- | --- | --- |
| S1 | Gi0/0 | R1 Fa0/0 | access | Không |
| S1 | Gi0/1 | S2 Gi0/1 | trunk | Không |
| S1 | Gi0/2 | PC-A | access | PortFast, BPDU guard, port security, policing |
| S2 | Gi0/1 | S1 Gi0/1 | trunk | Không |
| S2 | Gi0/2 | PC-B | access | PortFast, BPDU guard, port security, policing |

Chỉ đặt bảo mật lên cổng người dùng (Gi0/2). Không đặt PortFast, BPDU guard hay port security lên Gi0/0 và Gi0/1.

### 2.4. Thứ tự khởi động

1. Start **R1**, đợi xong.
2. Start **vios-1** và **vios-2**. Lần đầu boot có thể mất vài phút, đợi đến khi có dấu nhắc `vIOS-L2-01>`.
3. Start **PC1** và **PC2**.

---

## 3. Cấu hình nền

### 3.1. R1

Nếu R1 còn IP của Lab 3 trên `FastEthernet0/1` (cùng dải 192.168.1.0/24), IOS sẽ báo `overlaps`. Xóa trước:

```
enable
configure terminal
interface FastEthernet0/1
 no ip address
 shutdown
exit
interface FastEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
exit
end
write memory
```

Kiểm tra: `show ip interface brief`. `FastEthernet0/0` phải là `192.168.1.1` và `up/up`.

### 3.2. S1 (vios-1)

```
enable
configure terminal
hostname S1
interface Vlan1
 ip address 192.168.1.2 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.1.1
end
write memory
```

### 3.3. S2 (vios-2)

```
enable
configure terminal
hostname S2
interface Vlan1
 ip address 192.168.1.3 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.1.1
end
write memory
```

### 3.4. PC1 và PC2 (VPCS)

PC1:

```
ip 192.168.1.10 255.255.255.0 192.168.1.1
save
```

PC2:

```
ip 192.168.1.11 255.255.255.0 192.168.1.1
save
```

### 3.5. Kiểm tra kết nối

Trên R1:

```
ping 192.168.1.2
ping 192.168.1.3
ping 192.168.1.10
ping 192.168.1.11
```

Trên PC1:

```
ping 192.168.1.1
ping 192.168.1.11
```

Nếu ping không thông: kiểm tra `show ip interface brief` trên switch (`Vlan1` phải `up/up`), và `show interfaces status` (cổng nối phải `connected`).

---

## 4. Nhiệm vụ 1: SSH trên S1 và S2

Chạy trên S1, rồi lặp lại trên S2:

```
enable
configure terminal
ip domain-name cisco.local
username admin privilege 15 secret cisco123
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 transport input ssh
 login local
 exec-timeout 10 0
 logging synchronous
exit
end
write memory
```

Lưu ý:

- Nếu `crypto key generate rsa` hỏi ghi đè khóa cũ (`[yes/no]`), gõ `yes`.
- Khóa RSA được tạo xong trong vài giây trên vIOS-L2.

**Kiểm tra từ R1:**

```
ssh -l admin 192.168.1.2
```

Nhập mật khẩu `cisco123`. Khi vào được `S1>`, gõ `exit`. Lặp lại với `192.168.1.3` cho S2.

**Kiểm tra trên switch:**

```
show ip ssh
show ssh
```

Phải thấy `SSH Enabled - version 2.0`.

---

## 5. Nhiệm vụ 2: STP, PortFast và BPDU guard

### 5.1. Root và secondary

Trên S1:

```
enable
configure terminal
spanning-tree vlan 1 root primary
end
write memory
```

Trên S2:

```
enable
configure terminal
spanning-tree vlan 1 root secondary
end
write memory
```

Kiểm tra trên cả hai switch:

```
show spanning-tree vlan 1
```

Kỳ vọng:

- **S1:** dòng `This bridge is the root`, Bridge ID priority `24577`.
- **S2:** `Root ID` có địa chỉ MAC của S1 (`0c73.97fc.0000` trong lần chạy này), priority của S2 cao hơn (`28673`), cổng Gi0/1 là `Root`.

### 5.2. BPDU guard toàn cục

Trên cả S1 và S2:

```
enable
configure terminal
spanning-tree portfast bpduguard default
end
write memory
```

Kiểm tra: `show spanning-tree summary` phải có dòng `PortFast BPDU Guard Default is enabled`.

### 5.3. Trunk Gi0/1

Trên cả S1 và S2:

```
enable
configure terminal
interface Gi0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
exit
end
write memory
```

Phải đặt `encapsulation dot1q` trước `switchport mode trunk`. Nếu bỏ qua, IOS báo `Command rejected: ... trunk encapsulation is "Auto"`.

Kiểm tra: `show interfaces trunk`. Gi0/1 phải ở trạng thái `trunking`, encapsulation `802.1q`.

### 5.4. PortFast trên cổng người dùng (Gi0/2)

Trên S1 và S2:

```
enable
configure terminal
interface Gi0/2
 description PC
 switchport host
 spanning-tree portfast
 no shutdown
exit
end
write memory
```

`switchport host` tự đặt cổng ở chế độ access và bật PortFast. Dùng lệnh này để tránh lỗi `dynamic port` khi bật port security (xem mục 6).

Không đặt PortFast lên Gi0/0 hay Gi0/1.

---

## 6. Nhiệm vụ 3: Port security và sticky MAC

Trên cả S1 và S2, cổng Gi0/2:

```
enable
configure terminal
interface Gi0/2
 switchport host
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 no shutdown
exit
end
write memory
```

Ghi chú về cú pháp (đã kiểm chứng):

- `switchport mode access` đứng riêng không đủ, cổng vẫn báo `dynamic port`. Phải dùng `switchport host`.
- `switchport port-security mac-address sticky` **không kèm địa chỉ MAC**. Dạng `sticky 0050.7966.6800` báo `Invalid input` trên image này. MAC sẽ được học tự động.
- `violation restrict` (đã dùng) không khóa cổng, chỉ bỏ frame lạ và tăng bộ đếm vi phạm. Có thể dùng `violation shutdown` nếu muốn khóa cổng.

**Sinh traffic để học MAC.** Từ PC1 chạy `ping 192.168.1.1`, từ PC2 chạy `ping 192.168.1.1`.

**Kiểm tra:**

```
show port-security interface Gi0/2
show port-security address
show running-config interface Gi0/2
```

Kỳ vọng:

- `Port Security: Enabled`, `Port Status: Secure-up`, `Maximum MAC Addresses: 1`, `Violation Mode: Restrict`.
- `show port-security address` có MAC của PC (`0050.7966.6800` cho PC1) với loại `SecureSticky`.
- `running-config` có dòng `switchport port-security mac-address sticky`.

---

## 7. Storm control và biện pháp thay thế

### 7.1. Kiểm tra storm control

Trên S1:

```
enable
configure terminal
interface Gi0/2
 storm-control ?
end
```

Kết quả: `Unrecognized command`. Image vIOS-L2 không có storm control. Ghi lại kèm ảnh chụp.

### 7.2. Policing trên cổng người dùng (thay thế)

Trên cả S1 và S2:

```
enable
configure terminal
policy-map POLICE_1M
 class class-default
  police cir 1000000
 exit
exit
interface Gi0/2
 service-policy input POLICE_1M
exit
end
write memory
```

Lưu ý: `police` chỉ nhận dạng `police cir <bps>`. Các dạng có `exceed-action drop` và `burst` báo `Invalid input` trên image này.

Kiểm tra:

```
show policy-map interface Gi0/2
```

Kỳ vọng: `Service-policy input: POLICE_1M`, `cir 1000000 bps`, `conformed ... actions: transmit`, `exceeded ... actions: drop`.

### 7.3. IGMP snooping

Kiểm tra:

```
show ip igmp snooping
```

Kỳ vọng: `IGMP snooping: Enabled` ở phần global và ở VLAN 1. Image đã bật sẵn, không cần cấu hình thêm.

---

## 8. Kiểm tra theo hướng dẫn

### 8.1. Từ R1: ping và SSH

```
ping 192.168.1.10
ping 192.168.1.11
ssh -l admin 192.168.1.2
ssh -l admin 192.168.1.3
```

### 8.2. Trên S1 và S2: root và secondary

```
show spanning-tree
```

### 8.3. Trên S1 và S2: cấu hình

```
show running-config | include portfast|bpduguard|port-security|sticky|police
```

Phải thấy:

- `spanning-tree portfast bpduguard default`
- `spanning-tree portfast` trên Gi0/2
- `switchport port-security`, `violation restrict`, `mac-address sticky`
- `service-policy input POLICE_1M` trên Gi0/2

### 8.4. Trên S1 và S2: trạng thái thực tế

```
show port-security interface Gi0/2
show port-security address
show spanning-tree interface Gi0/2 portfast
show interfaces trunk
show policy-map interface Gi0/2
show ip igmp snooping
```

---

## 9. Test vi phạm port security

Mục đích: chứng minh cổng từ chối MAC lạ.

1. Đảm bảo PC1 đã được học (`show port-security address` có MAC PC1).
2. Tháo dây PC1 khỏi S1 Gi0/2.
3. Tháo dây PC2 khỏi S2 Gi0/2, cắm vào S1 Gi0/2.
4. Từ PC2 chạy `ping 192.168.1.1`. Kết quả: không có reply.
5. Trên S1:

   ```
   show port-security interface Gi0/2
   ```

   `Security Violation Count` phải lớn hơn 0. `Port Status` vẫn là `Secure-up` (vì chế độ `restrict`).
6. Khôi phục: tháo PC2 khỏi S1 Gi0/2, cắm lại PC1 vào S1 Gi0/2, cắm PC2 vào lại S2 Gi0/2. Ping từ PC1 tới `192.168.1.1` để xác nhận.

Nếu dùng `violation shutdown` thay cho `restrict`, cổng sẽ `err-disabled`. Khôi phục bằng:

```
configure terminal
interface Gi0/2
 shutdown
 no shutdown
end
```

---

## 10. Lưu cấu hình và kiểm tra startup-config

Lỗi đã gặp: cấu hình bị mất sau khi node restart, do chưa `write memory` hoặc node bị reset. Sau mỗi nhóm lệnh:

```
write memory
show startup-config | include hostname|username|ip ssh|sticky
```

Chỉ khi `startup-config` có đủ hostname, user và dòng sticky mới đóng project.

---

## 11. Xử lý sự cố (lỗi đã gặp khi làm)

| Thông báo | Nguyên nhân | Cách xử lý |
| --- | --- | --- |
| `% 192.168.1.0 overlaps with FastEthernet0/1` trên R1 | IP Lab 3 còn trên Fa0/1 | Mục 3.1: `no ip address` trên Fa0/1 |
| `Command rejected: ... trunk encapsulation is "Auto"` | Thiếu encapsulation | Đặt `switchport trunk encapsulation dot1q` trước |
| `Command rejected: GigabitEthernet0/2 is a dynamic port` | Cổng chưa tĩnh | Dùng `switchport host` trước |
| `Invalid input` ở `mac-address sticky <mac>` | Image không nhận MAC kèm sticky | Chỉ dùng `mac-address sticky` |
| `Invalid input` ở `switchport nonegotiate` | Image không có lệnh này | Dùng `switchport host` |
| `Unrecognized command` ở `storm-control` | Image không có storm control | Mục 7 |
| `police` báo `Invalid input` với `exceed-action` | Cú pháp không được hỗ trợ | Dùng `police cir 1000000` |
| `show running-config` có hai dòng sticky | Đã nhập dòng có MAC trước đó | `no switchport port-security mac-address sticky <mac>` |
| Cấu hình biến mất sau restart | Chưa `write memory` | Mục 10 |
| Ping `198.168.1.1` báo `Destination host unreachable` | Gõ sai địa chỉ | Ping đúng `192.168.1.1` |
| `show ip interfaces brief` báo lỗi | Gõ sai lệnh | Đúng là `show ip interface brief` |
| `Gi0/0` `down` | R1 chưa start hoặc chưa cấu hình | Start R1, kiểm tra Fa0/0 |
| `Vlan1` `down` | Không có cổng VLAN 1 nào `up` | Kiểm tra `show interfaces status` |