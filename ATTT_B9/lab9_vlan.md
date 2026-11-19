# Lab 9: Bảo mật VLAN (VTP động, Trunk/Access, DHCP Snooping) trên vIOS-L2

Cấu hình này gồm cả Lab 8 và Lab 9. Paste từng thiết bị một, theo đúng thứ tự: R1, S1, S2.

---
Cấu hình key trên S1, S2
```
show crypto key mypubkey rsa
configure terminal
ip domain-name cisco.local
crypto key generate rsa modulus 1024
end
```
Khối 1: R1
```
enable
configure terminal
hostname R1
interface FastEthernet0/0.69
 no ip access-group DHCP_ONLY_R1 in
exit
interface FastEthernet0/0.96
 no ip access-group DHCP_ONLY_R1 in
exit
no ip access-list extended DHCP_ONLY_R1
interface FastEthernet0/0
 no ip address
 no shutdown
exit
interface FastEthernet0/1
 no ip address
 shutdown
exit
interface FastEthernet0/0.1
 encapsulation dot1Q 1 native
 ip address 192.168.1.1 255.255.255.0
exit
interface FastEthernet0/0.69
 encapsulation dot1Q 69
 ip address 192.168.69.1 255.255.255.0
exit
interface FastEthernet0/0.96
 encapsulation dot1Q 96
 ip address 192.168.96.1 255.255.255.0
exit
ip dhcp excluded-address 192.168.69.1 192.168.69.10
ip dhcp excluded-address 192.168.96.1 192.168.96.10
ip dhcp pool VLAN69
 network 192.168.69.0 255.255.255.0
 default-router 192.168.69.1
 dns-server 8.8.8.8
exit
ip dhcp pool VLAN96
 network 192.168.96.0 255.255.255.0
 default-router 192.168.96.1
 dns-server 8.8.8.8
exit
ip access-list extended DHCP_ONLY_R1
 remark Client gui DHCP Discover va Request toi R1
 permit udp any eq bootpc any eq bootps
 remark Chi 2 gateway duoc tra loi DHCP
 permit udp host 192.168.69.1 eq bootps any eq bootpc
 permit udp host 192.168.96.1 eq bootps any eq bootpc
 remark Chan moi server DHCP khac
 deny udp any eq bootps any eq bootpc
 remark Chi cho luu luong tu 2 subnet VLAN
 permit ip 192.168.69.0 0.0.0.255 any
 permit ip 192.168.96.0 0.0.0.255 any
exit
interface FastEthernet0/0.69
 ip access-group DHCP_ONLY_R1 in
exit
interface FastEthernet0/0.96
 ip access-group DHCP_ONLY_R1 in
exit
end
write memory
```

Khối 2: S1 (vios-1)
```
enable
configure terminal
hostname S1
ip domain-name cisco.local
username admin privilege 15 secret cisco123
ip ssh version 2
line vty 0 4
 transport input ssh
 login local
 exec-timeout 10 0
 logging synchronous
exit
vtp domain CISCO-vIOS
vtp mode server
vlan 69
 name VLAN69
exit
vlan 96
 name VLAN96
exit
spanning-tree vlan 1 root primary
spanning-tree portfast bpduguard default
policy-map POLICE_1M
 class class-default
  police cir 1000000
 exit
exit
interface Vlan1
 ip address 192.168.1.2 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.1.1
interface Gi0/0
 description To-R1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 ip dhcp snooping trust
 no shutdown
exit
interface Gi0/1
 description Trunk-to-S2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 ip dhcp snooping trust
 no shutdown
exit
interface Gi0/2
 description PC-A
 switchport host
 switchport access vlan 69
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 service-policy input POLICE_1M
 no shutdown
exit
ip dhcp snooping
ip dhcp snooping vlan 69,96
no ip dhcp snooping information option
end
write memory
```

Khối 3: S2 (vios-2)

Chỉ paste khối này sau khi show vlan brief trên S2 đã thấy VLAN 69 và 96 (VTP đã đồng bộ từ S1).

```
enable
configure terminal
hostname S2
ip domain-name cisco.local
username admin privilege 15 secret cisco123
ip ssh version 2
line vty 0 4
 transport input ssh
 login local
 exec-timeout 10 0
 logging synchronous
exit
vtp domain CISCO-vIOS
vtp mode client
spanning-tree vlan 1 root secondary
spanning-tree portfast bpduguard default
policy-map POLICE_1M
 class class-default
  police cir 1000000
 exit
exit
interface Vlan1
 ip address 192.168.1.3 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.1.1
interface Gi0/1
 description Trunk-to-S1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 ip dhcp snooping trust
 no shutdown
exit
interface Gi0/2
 description PC-B
 switchport host
 switchport access vlan 96
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 service-policy input POLICE_1M
 no shutdown
exit
ip dhcp snooping
ip dhcp snooping vlan 69,96
no ip dhcp snooping information option
end
write memory
```