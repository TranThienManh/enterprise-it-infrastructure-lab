# 08 — Các sự cố kỹ thuật và bài học triển khai

## 1. Tổng quan

Trong quá trình xây dựng **Enterprise IT Infrastructure & System Administration Lab**, việc triển khai các dịch vụ không phải lúc nào cũng diễn ra đúng như thiết kế ban đầu.

Một số sự cố phát sinh từ cấu hình mạng, DNS, Firewall, Group Policy và cơ chế phân quyền tài nguyên.

Các sự cố này được sử dụng để thực hành quy trình **Infrastructure Troubleshooting**, bao gồm:

1. Xác định triệu chứng.
2. Khoanh vùng thành phần liên quan.
3. Thu thập bằng chứng.
4. Đưa ra giả thuyết nguyên nhân.
5. Kiểm tra giả thuyết.
6. Áp dụng biện pháp khắc phục.
7. Kiểm thử lại chức năng.
8. Ghi nhận bài học kỹ thuật.

Tài liệu tổng hợp những vấn đề đã gặp trong quá trình thực hành, phân biệt rõ sự cố đã xử lý và những vấn đề còn cần xác minh.

---

## 2. Tổng hợp các sự cố

| ID | Thành phần | Vấn đề | Kết quả |
|---|---|---|---|
| INC-01 | DNS | Domain phân giải về địa chỉ IP cũ | Đã xử lý |
| INC-02 | Windows Firewall | DC01 không Ping được máy trạm | Đã xử lý |
| INC-03 | Linux Networking | LINUX01 không kết nối được Gateway | Đã xử lý |
| INC-04 | pfSense / SMB | File Share không truy cập được sau khi thay đổi Firewall Rules | Đã ghi nhận; cần đối chiếu cấu hình cuối |
| INC-05 | Active Directory / GPO | Cần xác minh chính sách thực sự được áp dụng | Đã kiểm thử các GPO liên quan |
| INC-06 | File Server | Phân biệt quyền truy cập theo Security Group | Đã kiểm thử |
| INC-07 | Windows Server | Network Profile hiển thị Public/Unidentified | Chưa xác nhận khắc phục |

Các sự cố được mô tả theo dữ liệu đã ghi nhận trong lab, không phải tình huống giả lập được thêm vào để minh họa.

---

## 3. INC-01 — DNS phân giải về địa chỉ IP cũ

### 3.1. Bối cảnh

Trong quá trình triển khai, hệ thống mạng được tổ chức lại thành các subnet:

- SERVER: `10.10.10.0/24`
- USER: `10.10.20.0/24`
- MGMT: `10.10.30.0/24`

DC01 sử dụng địa chỉ:

`10.10.10.10`

Tuy nhiên, tại một thời điểm kiểm tra, tên Domain hoặc tài nguyên liên quan vẫn được phân giải về địa chỉ cũ:

`192.168.20.10`

### 3.2. Triệu chứng

Kết quả phân giải DNS không khớp với địa chỉ IP hiện tại của Domain Controller.

Điều này có thể gây ra các vấn đề như:

- Domain Controller Discovery không chính xác.
- Kết nối đến Domain Services thất bại.
- Group Policy không được xử lý đúng.
- Máy trạm truy cập sai địa chỉ máy chủ.

### 3.3. Phân tích nguyên nhân

Trong môi trường Active Directory, DNS đóng vai trò quan trọng đối với quá trình xác định Domain Controller.

Khi địa chỉ IP của máy chủ thay đổi, những thông tin cũ có thể còn tồn tại trong:

- DNS Client Cache.
- DNS Records.
- Cấu hình DNS Server trên Client.
- Các bản ghi hoặc cấu hình liên quan đến môi trường mạng trước đó.

### 3.4. Hướng xử lý

Các bước chẩn đoán được sử dụng hoặc có thể áp dụng để xác minh gồm:

```powershell
ipconfig /all

nslookup dc01.corp.local

ipconfig /flushdns

nltest /dsgetdc:corp.local
```

Mục tiêu là kiểm tra DNS Server đang sử dụng, kết quả phân giải và khả năng tìm Domain Controller.

### 3.5. Kết quả

Sau khi xử lý thông tin DNS cũ, các bài kiểm thử Domain Controller Discovery và truy cập tài nguyên Domain đã hoạt động.

### 3.6. Bài học kỹ thuật

**DNS Resolution cần được xác minh trước khi kết luận lỗi thuộc về Routing hoặc Firewall.**

Một hệ thống có thể kết nối thành công đến địa chỉ IP của Domain Controller nhưng vẫn gặp lỗi Domain Services nếu DNS không chính xác.

---

## 4. INC-02 — ICMP bị chặn bởi Windows Firewall

### 4.1. Bối cảnh

Trong quá trình kiểm tra kết nối giữa DC01 và một máy trạm Domain, lệnh Ping không nhận được phản hồi.

### 4.2. Triệu chứng

DC01 không thể Ping thành công đến USER02 trong tình huống đã ghi nhận.

Tuy nhiên, việc Ping thất bại không đủ để khẳng định máy trạm hoàn toàn mất kết nối mạng.

### 4.3. Phân tích nguyên nhân

ICMP Echo Request và Echo Reply có thể bị chặn bởi Windows Defender Firewall.

Do đó, một máy trạm vẫn có thể:

- Kết nối đến Domain Controller.
- Sử dụng DNS.
- Truy cập SMB.
- Thực hiện các kết nối TCP được phép.

Trong khi đó, Ping vẫn thất bại.

### 4.4. Hướng xử lý

Kiểm tra Windows Firewall Rules liên quan đến ICMP.

Xác minh Network Profile và phạm vi nguồn truy cập được phép.

Có thể sử dụng các công cụ:

```powershell
Get-NetFirewallRule

Get-NetConnectionProfile

Test-NetConnection 10.10.20.101 -Port 445
```

Lệnh kiểm tra TCP 445 chỉ phù hợp nếu máy đích thực sự cung cấp dịch vụ SMB; nó không phải phép thử thay thế trực tiếp cho ICMP.

### 4.5. Kết quả

Sau khi điều chỉnh quy tắc cho phép ICMP phù hợp, khả năng Ping được khôi phục trong bài kiểm thử.

### 4.6. Bài học kỹ thuật

**Ping là một công cụ kiểm tra ICMP, không phải phép kiểm chứng đầy đủ cho mọi giao thức mạng.**

Khi xử lý sự cố, cần phân biệt:

- Host Reachability.
- ICMP Filtering.
- TCP Connectivity.
- Application Availability.

Không nên vô hiệu hóa toàn bộ Windows Firewall chỉ để xử lý một lỗi Ping.

---

## 5. INC-03 — LINUX01 không kết nối được Gateway

### 5.1. Bối cảnh

LINUX01 được triển khai tại SERVER Network:

`10.10.10.30/24`

Default Gateway:

`10.10.10.1`

Trong quá trình điều chỉnh môi trường mạng, LINUX01 từng gặp lỗi kết nối đến Gateway.

### 5.2. Triệu chứng

LINUX01 không Ping được địa chỉ Gateway mặc dù cấu hình mạng được thiết kế để hai thiết bị cùng thuộc SERVER Network.

### 5.3. Phân tích nguyên nhân

Khi xử lý sự cố kết nối trong cùng subnet, cần kiểm tra cả Layer 2 và Layer 3.

Các thành phần cần xem xét:

- Network Interface.
- IPv4 Address và Subnet Mask.
- Routing Table.
- VMware Virtual Network.
- ARP/Neighbor Table.
- Firewall Rules.

Trong tình huống thực tế, thông tin ARP/Neighbor Cache không còn phù hợp là yếu tố được xử lý.

### 5.4. Hướng xử lý

Các công cụ Linux liên quan:

```bash
ip address

ip route

ip neigh

ping -c 4 10.10.10.1
```

Có thể làm mới Neighbor Cache khi đã xác định thông tin hiện tại không phù hợp.

### 5.5. Kết quả

Sau khi xử lý ARP/Neighbor Cache, kết nối đến Gateway được khôi phục.

LINUX01 tiếp tục được kiểm tra kết nối đến DC01 và FS01.

### 5.6. Bài học kỹ thuật

**Kết nối giữa hai thiết bị cùng subnet không chỉ phụ thuộc vào IP Address và Subnet Mask mà còn phụ thuộc vào khả năng phân giải địa chỉ Layer 2.**

Khi thay đổi topology hoặc cấu hình máy ảo, cần xem xét Neighbor Cache và Virtual Network Connectivity.

---

## 6. INC-04 — File Share bị ảnh hưởng sau khi thay đổi Firewall Rules

### 6.1. Bối cảnh

USER01 thuộc USER Network, trong khi FS01 thuộc SERVER Network.

Để truy cập File Server, lưu lượng SMB phải đi qua pfSense.

Luồng kết nối:

```text
USER01
10.10.20.102
     |
     v
pfSense
     |
     v
FS01
10.10.10.20
     |
     v
SMB Service
TCP 445
```

### 6.2. Triệu chứng

Trong quá trình điều chỉnh Firewall Rules, đã xuất hiện tình huống máy trạm vẫn truy cập được một số dịch vụ Domain nhưng không truy cập được File Share.

Một kết quả kiểm tra đã ghi nhận:

```powershell
Test-Path "\\fs01.corp.local\CORP-SHARE"
```

Trả về:

```text
False
```

### 6.3. Phân tích

Kết quả `Test-Path` bằng `False` có thể xuất phát từ nhiều nguyên nhân:

- DNS Resolution không chính xác.
- Firewall chặn lưu lượng SMB.
- SMB Service không khả dụng.
- Share Path không tồn tại.
- Share Permissions không phù hợp.
- NTFS Permissions không phù hợp.
- Tài khoản chưa được xác thực hoặc cấp quyền.

Trong tình huống đã ghi nhận, sự cố xuất hiện sau khi thay đổi Firewall Rules, vì vậy Firewall là một hướng điều tra quan trọng.

Tuy nhiên, chỉ riêng kết quả `Test-Path = False` không đủ để xác định nguyên nhân duy nhất.

### 6.4. Phương pháp khoanh vùng

**Lớp 1 — DNS**

```powershell
Resolve-DnsName fs01.corp.local
```

**Lớp 2 — TCP Connectivity**

```powershell
Test-NetConnection fs01.corp.local -Port 445
```

**Lớp 3 — SMB Share**

```powershell
Test-Path "\\fs01.corp.local\CORP-SHARE"
```

**Lớp 4 — Authorization**

Kiểm tra Security Group Membership, Share Permissions và NTFS Permissions.

**Lớp 5 — Firewall**

Đối chiếu pfSense Firewall Rules và Firewall Logs với thời điểm phát sinh sự cố.

### 6.5. Kết quả

Các bài kiểm thử sau đó đã xác nhận USER01 có thể sử dụng những tài nguyên File Server được cấp quyền.

Tuy nhiên, cần lưu lại bản cấu hình Firewall cuối cùng và kết quả kiểm thử tương ứng để xác minh đầy đủ mối quan hệ giữa thay đổi Rules và trạng thái dịch vụ.

### 6.6. Bài học kỹ thuật

**Một thay đổi ở Network Firewall có thể ảnh hưởng đến ứng dụng dù các dịch vụ khác vẫn hoạt động bình thường.**

Khi triển khai Firewall Rules, cần kiểm thử theo từng dịch vụ thay vì chỉ dựa vào Ping hoặc một kết nối thành công.

---

## 7. INC-05 — Xác minh Group Policy thực sự được áp dụng

### 7.1. Bối cảnh

Project sử dụng Group Policy để quản lý cấu hình người dùng và máy trạm.

Các chính sách liên quan bao gồm:

- `GPO_HR_Disable_ControlPanel`
- `GPO_HR_Drive`
- `GPO_CORP_Users_Common`
- `GPO_CORP_Computers_Security`

### 7.2. Thách thức

Việc tạo Group Policy Object không đồng nghĩa chính sách đang có hiệu lực.

Một GPO có thể không áp dụng do:

- GPO Link bị vô hiệu hóa.
- Đối tượng không nằm trong OU phù hợp.
- Security Filtering.
- WMI Filtering.
- Lỗi Group Policy Processing.
- Cấu hình User hoặc Computer Policy không phù hợp.

### 7.3. Phương pháp xác minh

```powershell
gpupdate /force

gpresult /r
```

Kết quả cần được đối chiếu với OU, GPO Links và cấu hình thực tế.

### 7.4. Kết quả

Các chính sách liên quan đến người dùng và Drive Mapping đã được kiểm tra trong quá trình thực hành.

Riêng `GPO_CORP_Computers_Security` có liên kết được vô hiệu hóa có chủ đích.

Do đó, GPO này không được mô tả là đang áp dụng trên máy trạm.

### 7.5. Bài học kỹ thuật

**Phải phân biệt GPO tồn tại trong hệ thống với GPO đã được xử lý thành công trên đối tượng mục tiêu.**

Kết quả `gpresult` và hành vi thực tế trên Client là những bằng chứng quan trọng.

---

## 8. INC-06 — Phân quyền File Server theo Security Group

### 8.1. Bối cảnh

FS01 cung cấp các tài nguyên chia sẻ:

- Accounting.
- HR.
- IT.
- Public.

Hệ thống sử dụng Active Directory Security Groups kết hợp Share Permissions và NTFS Permissions.

### 8.2. Thách thức

Tên tài khoản người dùng không phải cơ sở đáng tin cậy để xác định quyền truy cập.

Trong dự án, tài khoản:

`corp\hr.kimuyen`

được chủ đích đặt trong cơ cấu Accounting và thuộc nhóm:

`GG_Accounting`

### 8.3. Phương pháp xác minh

Kiểm tra:

- Group Membership.
- Share Permissions.
- NTFS Permissions.
- Khả năng đọc/ghi thực tế.
- Khả năng truy cập các tài nguyên không được cấp quyền.

### 8.4. Kết quả

| Tài nguyên | Kết quả |
|---|---|
| Accounting | Đọc/ghi thành công |
| Public | Truy cập thành công |
| HR | Bị hạn chế |
| IT | Bị hạn chế |

Các ổ đĩa mạng `A:` và `P:` cũng được kiểm thử thành công.

### 8.5. Bài học kỹ thuật

**Authorization phải dựa trên Security Groups và Access Control Lists, không dựa trên quy ước đặt tên tài khoản.**

Cần kiểm thử cả hai trường hợp:

- Authorized Access.
- Unauthorized Access.

Một hệ thống phân quyền chỉ được đánh giá đúng khi xác minh được cả khả năng cho phép và từ chối truy cập.

---

## 9. INC-07 — Network Profile Public/Unidentified trên DC01

### 9.1. Bối cảnh

Trong quá trình kiểm tra DC01, Windows Network Profile từng hiển thị trạng thái Public hoặc Unidentified thay vì trạng thái DomainAuthenticated như mong đợi.

### 9.2. Ảnh hưởng tiềm ẩn

Network Profile có thể ảnh hưởng đến:

- Windows Defender Firewall Rules.
- Khả năng phát hiện mạng.
- Một số chính sách kết nối.
- Hành vi của các dịch vụ phụ thuộc vào Firewall Profile.

### 9.3. Hướng phân tích

Các thành phần cần kiểm tra:

- DNS Client Configuration.
- Network Location Awareness.
- Domain Controller Discovery.
- Trạng thái các dịch vụ Domain.
- Network Interface.
- Windows Firewall Profiles.

Công cụ kiểm tra:

```powershell
Get-NetConnectionProfile

Get-NetIPConfiguration

nltest /dsgetdc:corp.local
```

### 9.4. Trạng thái

Đây là vấn đề đã được ghi nhận nhưng chưa có đủ bằng chứng xác nhận đã khắc phục hoàn toàn.

Không nên công bố sự cố này là đã giải quyết nếu chưa có kết quả Network Profile mới.

### 9.5. Bài học kỹ thuật

**Một dịch vụ Domain hoạt động không đồng nghĩa mọi trạng thái cấu hình của hệ điều hành đều đã tối ưu hoặc chính xác.**

Các vấn đề chưa xử lý cần được ghi nhận minh bạch để tiếp tục kiểm tra.

---

## 10. Phương pháp Troubleshooting áp dụng trong dự án

### 10.1. Kiểm tra theo lớp

Các sự cố được phân tích theo thứ tự từ hạ tầng mạng đến ứng dụng.

```text
Application Function
         |
         v
Authentication / Authorization
         |
         v
Service Availability
         |
         v
DNS Resolution
         |
         v
TCP / UDP Connectivity
         |
         v
IP Address / Routing
         |
         v
Network Interface / Layer 2
```

Không phải mọi sự cố đều cần kiểm tra tất cả các lớp theo thứ tự cố định. Quy trình được điều chỉnh dựa trên triệu chứng và bằng chứng ban đầu.

### 10.2. Phân biệt các loại lỗi

| Triệu chứng | Thành phần cần ưu tiên kiểm tra |
|---|---|
| Không có IP phù hợp | DHCP, Interface, DHCP Relay |
| Không Ping được Gateway | IP, Subnet, Layer 2, Firewall |
| Ping IP được nhưng hostname sai | DNS |
| Tìm không thấy Domain Controller | DNS, AD DS, Network |
| Không truy cập được TCP 445 | Firewall, SMB Service, Routing |
| TCP 445 hoạt động nhưng Share bị từ chối | Authentication, Share/NTFS Permissions |
| GPO không áp dụng | OU, GPO Link, Filtering, Processing |
| Linux Service không phản hồi | systemd, Listening Port, Firewall, Logs |

### 10.3. Công cụ sử dụng

| Công cụ | Mục đích |
|---|---|
| `ping` | Kiểm tra ICMP Connectivity |
| `ipconfig` | Kiểm tra cấu hình mạng Windows |
| `nslookup` | Kiểm tra DNS Resolution |
| `nltest` | Kiểm tra Domain Controller Discovery |
| `Test-NetConnection` | Kiểm tra kết nối TCP |
| `gpresult` | Kiểm tra Group Policy |
| `ip address` | Kiểm tra Linux Network Interfaces |
| `ip route` | Kiểm tra Linux Routing Table |
| `ip neigh` | Kiểm tra Neighbor Cache |
| `systemctl` | Kiểm tra trạng thái Linux Services |
| pfSense Firewall Logs | Phân tích lưu lượng bị chặn hoặc cho phép |

---

## 11. Những nguyên tắc rút ra

### 11.1. Không suy luận từ một kết quả kiểm tra

Ping thành công không chứng minh ứng dụng hoạt động.

TCP Connection thành công không chứng minh người dùng có quyền truy cập.

Dịch vụ đang chạy không chứng minh ứng dụng trả về kết quả chính xác.

### 11.2. Ưu tiên bằng chứng

Mọi kết luận cần dựa trên:

- Trạng thái cấu hình.
- Kết quả lệnh.
- Application Logs.
- Firewall Logs.
- Hành vi thực tế của Client.

### 11.3. Kiểm thử lại sau khi thay đổi

Sau khi thay đổi Firewall, DNS, Group Policy hoặc Permissions, cần kiểm thử lại chức năng liên quan.

Điều này giúp phát hiện những ảnh hưởng không mong muốn đến các dịch vụ khác.

### 11.4. Phân biệt xử lý tạm thời và khắc phục triệt để

Một biện pháp như xóa DNS Cache hoặc ARP Cache có thể khôi phục hoạt động trong tình huống cụ thể.

Tuy nhiên, nếu sự cố tái diễn, cần kiểm tra nguyên nhân gốc thay vì chỉ lặp lại thao tác xử lý tạm thời.

### 11.5. Ghi nhận các vấn đề chưa giải quyết

Những vấn đề chưa được xác minh hoàn toàn cần được giữ lại trong tài liệu.

Điều này giúp tránh đánh giá sai trạng thái hệ thống và hỗ trợ công việc kiểm tra sau này.

---

## 12. Đánh giá năng lực kỹ thuật

Thông qua các tình huống trên, dự án thể hiện những nhóm kỹ năng:

**Network Troubleshooting**

Phân tích IP Addressing, Routing, DNS, ICMP, ARP và Firewall.

**Windows Server Administration**

Kiểm tra Domain Services, Group Policy và Windows Firewall.

**Identity and Access Management**

Xác minh Security Group Membership và quyền truy cập tài nguyên.

**Linux System Administration**

Kiểm tra Linux Network Configuration và Neighbor Cache.

**System Integration**

Xác định mối quan hệ phụ thuộc giữa Network, Domain Services và File Sharing.

**Technical Documentation**

Ghi nhận triệu chứng, nguyên nhân, phương pháp xử lý, kết quả và giới hạn.

---

## 13. Kết luận

Quá trình triển khai Project cung cấp nhiều tình huống thực tế để thực hành xử lý sự cố hạ tầng.

Các vấn đề liên quan đến DNS, Windows Firewall, ARP/Neighbor Cache, SMB Access và Group Policy cho thấy một hệ thống có thể gặp lỗi ở nhiều lớp khác nhau dù kiến trúc tổng thể đã được thiết kế phù hợp.

Thông qua việc phân tích và xử lý các tình huống này, dự án thể hiện khả năng:

- Khoanh vùng nguyên nhân sự cố.
- Sử dụng công cụ chẩn đoán phù hợp.
- Phân biệt lỗi mạng và lỗi ứng dụng.
- Kiểm tra Authentication và Authorization.
- Đánh giá ảnh hưởng của thay đổi cấu hình.
- Xác minh lại chức năng sau khi xử lý.