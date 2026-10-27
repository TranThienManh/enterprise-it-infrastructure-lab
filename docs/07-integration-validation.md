# 07 — Kiểm thử tích hợp và xác minh chức năng hệ thống

## 1. Tổng quan

Project triển khai nhiều thành phần hạ tầng trên VMware Workstation, bao gồm pfSense, Windows Server, Active Directory, DNS, DHCP, Group Policy, File Server và Linux Server.

Tuy nhiên, việc triển khai thành công từng thành phần không đồng nghĩa toàn bộ hệ thống hoạt động chính xác.

Ví dụ, một máy trạm có thể kết nối được đến Domain Controller nhưng vẫn không thể đăng nhập Domain nếu DNS hoặc các dịch vụ xác thực gặp sự cố. Tương tự, một máy trạm có thể kết nối đến File Server qua TCP 445 nhưng không có quyền truy cập thư mục chia sẻ.

Vì vậy, dự án thực hiện **Integration Validation** nhằm xác minh sự phối hợp giữa các thành phần trong những tình huống sử dụng thực tế.

Các bài kiểm thử tập trung vào:

- Kết nối mạng giữa các subnet.
- Cấp phát địa chỉ IP và phân giải DNS.
- Domain Controller Discovery.
- Domain Authentication và Group Policy.
- Truy cập File Server theo Security Groups.
- Quản lý tài nguyên qua Drive Mapping.
- Kết nối Linux Server với hạ tầng Windows.
- Đánh giá các giới hạn chưa được kiểm thử đầy đủ.

---

## 2. Mục tiêu kiểm thử

Các bài kiểm thử được thiết kế để trả lời bốn câu hỏi chính:

1. **Connectivity:** Các thiết bị có kết nối được đến dịch vụ cần thiết không?
2. **Service Availability:** Dịch vụ có sẵn sàng tiếp nhận yêu cầu không?
3. **Functional Correctness:** Dịch vụ có thực hiện đúng chức năng mong muốn không?
4. **Access Control:** Người dùng có được cấp đúng quyền và bị từ chối khi không có quyền không?

## 3. Môi trường kiểm thử

### 3.1. Hạ tầng

| Thành phần | IP Address | Vai trò |
|---|---|---|
| pfSense | `10.10.10.1`, `10.10.20.1`, `10.10.30.1` | Routing và Firewall |
| DC01 | `10.10.10.10` | AD DS, DNS, DHCP |
| FS01 | `10.10.10.20` | File Server |
| LINUX01 | `10.10.10.30` | Linux Services |
| USER01 | `10.10.20.102` | Domain Client |
| MGMT01 | `10.10.30.10` | Administration Client |

Domain nội bộ:

`corp.local`

### 3.2. Các vùng mạng

| Network | Subnet | Chức năng |
|---|---|---|
| SERVER | `10.10.10.0/24` | Máy chủ |
| USER | `10.10.20.0/24` | Máy trạm |
| MGMT | `10.10.30.0/24` | Quản trị |

### 3.3. Tài khoản kiểm thử

Một tài khoản được sử dụng để xác minh phân quyền là:

`corp\hr.kimuyen`

Tài khoản thuộc Security Group:

`GG_Accounting`

Việc đặt tên tài khoản không tương ứng với tên nhóm là có chủ đích, nhằm kiểm tra cơ chế phân quyền dựa trên Group Membership.

---

## 4. Kịch bản 1 — Kết nối liên mạng

### 4.1. Mục tiêu

Xác minh máy trạm tại USER Network có thể kết nối đến các máy chủ thuộc SERVER Network thông qua pfSense.

### 4.2. Luồng kiểm thử

```text
USER01
10.10.20.102
     |
     v
pfSense
10.10.20.1
     |
     v
SERVER Network
     |
     +---- DC01: 10.10.10.10
     |
     +---- FS01: 10.10.10.20
```

### 4.3. Các bước xác minh

Trên USER01:

```powershell
ipconfig /all

ping 10.10.20.1
ping 10.10.10.10
ping 10.10.10.20

Test-NetConnection 10.10.10.10 -Port 445
Test-NetConnection 10.10.10.20 -Port 445
```

Các lệnh trên là phương pháp kiểm tra và không phải bản ghi kết quả thực thi.

### 4.4. Kết quả đã ghi nhận

Trong quá trình triển khai:

- Routing giữa USER và SERVER hoạt động.
- USER01 truy cập được các dịch vụ Domain.
- Kết nối đến File Server đã được kiểm tra.
- SYSVOL và NETLOGON có thể truy cập từ Domain Client.

**Đánh giá:** Các luồng mạng cần thiết cho hoạt động Domain và File Services đã được kiểm thử

---

## 5. Kịch bản 2 — DHCP và DNS

### 5.1. Mục tiêu

Xác minh USER01 có thể nhận cấu hình IP từ DHCP Server trên DC01 và phân giải các tên máy chủ nội bộ.

### 5.2. Luồng DHCP

```text
USER01
USER Network
     |
     v
pfSense DHCP Relay
     |
     v
DC01 DHCP Server
     |
     v
DHCP Lease
     |
     v
USER01
```

### 5.3. Phương pháp xác minh

```powershell
ipconfig /all
ipconfig /renew
```

Kiểm tra các thông tin:

- IPv4 Address.
- Subnet Mask.
- Default Gateway.
- DHCP Server.
- DNS Servers.
- DHCP Lease.

Đối với DNS:

```powershell
nslookup dc01.corp.local
nslookup fs01.corp.local
```

### 5.4. Kết quả đã ghi nhận

- Máy trạm nhận cấu hình mạng qua DHCP.
- DNS nội bộ hỗ trợ Domain Controller Discovery.
- Tên máy chủ nội bộ được sử dụng để truy cập các dịch vụ Domain và File Server.

**Đánh giá:** DHCP Relay, DHCP Server và DNS hoạt động phối hợp trong các bài kiểm thử đã thực hiện.

---

## 6. Kịch bản 3 — Domain Controller Discovery

### 6.1. Mục tiêu

Xác minh máy trạm USER01 có thể tìm Domain Controller của `corp.local`.

### 6.2. Phương pháp xác minh

```powershell
nltest /dsgetdc:corp.local
```

### 6.3. Kết quả đã ghi nhận

Domain Controller được tìm thấy:

```text
DC: \\DC01.corp.local
Address: \\10.10.10.10
Dom Name: corp.local
Forest Name: corp.local
```

### 6.4. Ý nghĩa

Kết quả chứng minh máy trạm có thể xác định Domain Controller thông qua hạ tầng Domain hiện có.

Đây là điều kiện quan trọng để thực hiện các hoạt động xác thực và quản trị Domain.

---

## 7. Kịch bản 4 — Truy cập SYSVOL và NETLOGON

### 7.1. Mục tiêu

Xác minh Domain Client có thể truy cập các tài nguyên chia sẻ quan trọng trên Domain Controller.

SYSVOL được sử dụng để lưu trữ và phân phối các thành phần liên quan đến Group Policy.

NETLOGON phục vụ những chức năng chia sẻ liên quan đến Domain.

### 7.2. Phương pháp xác minh

```powershell
Test-Path "\\dc01.corp.local\SYSVOL"
Test-Path "\\dc01.corp.local\NETLOGON"
```

### 7.3. Kết quả đã ghi nhận

```text
SYSVOL  : True
NETLOGON: True
```

### 7.4. Đánh giá

USER01 có thể truy cập các tài nguyên chia sẻ Domain cần thiết.

Kết quả này hỗ trợ xác minh khả năng kết nối giữa Domain Client và DC01.

---

## 8. Kịch bản 5 — Group Policy Processing

### 8.1. Mục tiêu

Xác minh các Group Policy được áp dụng đến người dùng hoặc máy tính thuộc phạm vi tương ứng.

### 8.2. Phương pháp xác minh

```powershell
gpupdate /force
gpresult /r
```

Các thông tin cần kiểm tra:

- Domain của người dùng và máy tính.
- Organizational Unit.
- Applied Group Policy Objects.
- Chính sách bị từ chối hoặc không áp dụng.
- Kết quả cấu hình trên máy trạm.

### 8.3. Kết quả đã ghi nhận

Trong quá trình thực hành, các GPO liên quan đến người dùng và Drive Mapping đã được xử lý thành công.

Các chính sách được áp dụng trong phạm vi phù hợp.

Lưu ý: `GPO_CORP_Computers_Security` có liên kết được vô hiệu hóa có chủ đích, vì vậy không được tính là một chính sách đang áp dụng.

### 8.4. Đánh giá

Group Policy cho phép quản lý một số cấu hình máy trạm và tài nguyên tập trung.

---

## 9. Kịch bản 6 — File Server Access Control

### 9.1. Mục tiêu

Xác minh quyền truy cập tài nguyên trên FS01 dựa vào Active Directory Security Groups và Permissions.

### 9.2. Môi trường

| Thành phần | Giá trị |
|---|---|
| Client | USER01 |
| File Server | FS01 |
| Domain | `corp.local` |
| Tài khoản | `corp\hr.kimuyen` |
| Security Group | `GG_Accounting` |

### 9.3. Ma trận kiểm thử

| Test ID | Tài nguyên | Hành động | Kết quả |
|---|---|---|---|
| FS-01 | Accounting | Truy cập | Thành công |
| FS-02 | Accounting | Đọc dữ liệu | Thành công |
| FS-03 | Accounting | Ghi dữ liệu | Thành công |
| FS-04 | Public | Truy cập | Thành công |
| FS-05 | HR | Truy cập không được cấp quyền | Bị hạn chế |
| FS-06 | IT | Truy cập không được cấp quyền | Bị hạn chế |

### 9.4. Phương pháp xác minh

Kiểm tra kết nối SMB:

```powershell
Test-NetConnection fs01.corp.local -Port 445
```

Kiểm tra đường dẫn chia sẻ:

```powershell
Test-Path "\\fs01.corp.local\CORP-SHARE"
```

Sau đó, kiểm tra quyền đọc và ghi trực tiếp bằng tài khoản Domain tương ứng.

### 9.5. Kết quả

Tài khoản Accounting có thể sử dụng tài nguyên được cấp quyền và bị hạn chế truy cập các tài nguyên không được cấp quyền.

**Đánh giá:** Cơ chế phân quyền theo Security Group hoạt động trong những trường hợp đã kiểm thử.

---

## 10. Kịch bản 7 — Group Policy Drive Mapping

### 10.1. Mục tiêu

Xác minh Group Policy có thể cung cấp ổ đĩa mạng tương ứng với tài nguyên được cấp quyền.

### 10.2. Kết quả kiểm thử

| Drive | Tài nguyên | Kết quả |
|---|---|---|
| `A:` | Accounting | Thành công |
| `P:` | Public | Thành công |

### 10.3. Ý nghĩa

Drive Mapping giúp người dùng truy cập tài nguyên thuận tiện mà không cần nhập đường dẫn UNC mỗi lần sử dụng.

Việc ổ đĩa xuất hiện không thay thế kiểm tra Share Permissions và NTFS Permissions.

---

## 11. Kịch bản 8 — Linux Network Integration

### 11.1. Mục tiêu

Xác minh LINUX01 có thể giao tiếp với các máy chủ Windows trong SERVER Network.

### 11.2. Phương pháp xác minh

```bash
ip address
ip route

ping -c 4 10.10.10.1
ping -c 4 10.10.10.10
ping -c 4 10.10.10.20
```

### 11.3. Kết quả đã ghi nhận

LINUX01 đã được kiểm tra kết nối đến:

- pfSense Gateway.
- DC01.
- FS01.

Trong quá trình điều chỉnh cấu hình mạng, một sự cố liên quan đến ARP/Neighbor Cache đã được xử lý.

### 11.4. Đánh giá

LINUX01 đã được tích hợp vào mạng nội bộ và có khả năng giao tiếp với các thành phần Windows liên quan.

---

## 12. Kịch bản 9 — Linux Application Services

### 12.1. Mục tiêu

Đánh giá khả năng hoạt động của các dịch vụ được triển khai trên LINUX01.

### 12.2. Ma trận xác minh

| Test ID | Dịch vụ | Tiêu chí |
|---|---|---|
| LNX-01 | SSH | Kết nối và xác thực từ máy quản trị |
| LNX-02 | Nginx | HTTP Request trả về nội dung hợp lệ |
| LNX-03 | Samba | Client truy cập được Share |
| LNX-04 | PostgreSQL | Database Connection và SQL Query thành công |
| LNX-05 | LVM | Logical Volume và Filesystem hoạt động |
| LNX-06 | Zabbix | Thu thập và hiển thị Metrics |
| LNX-07 | Linux Security | Permissions và Firewall hoạt động theo cấu hình |

## 13. Kịch bản 10 — Shadow Copies và khôi phục phiên bản tệp

### 13.1. Mục tiêu

Xác minh khả năng phục hồi phiên bản trước của tệp trên FS01.

### 13.2. Phương pháp kiểm thử

Quy trình kiểm thử gồm:

1. Tạo hoặc lựa chọn tệp thử nghiệm.
2. Tạo Shadow Copy.
3. Thay đổi hoặc xóa dữ liệu thử nghiệm.
4. Truy cập Previous Versions.
5. Khôi phục phiên bản trước.
6. Kiểm tra nội dung sau khi phục hồi.

### 13.3. Kết quả đã ghi nhận

Trong quá trình thực hành, chức năng snapshot và restore đã được kiểm tra.

### 13.4. Đánh giá

Shadow Copies hỗ trợ phục hồi phiên bản dữ liệu trong những tình huống phù hợp.

Tuy nhiên, kết quả này không chứng minh khả năng Disaster Recovery khi mất toàn bộ máy chủ hoặc volume lưu trữ.

---

## 14. Ma trận tổng hợp kết quả kiểm thử

| ID | Nhóm | Nội dung | Trạng thái |
|---|---|---|---|
| NET-01 | Networking | Inter-Subnet Routing | Đã kiểm thử |
| NET-02 | Networking | USER → SERVER Connectivity | Đã kiểm thử |
| NET-03 | Networking | DHCP Relay | Đã kiểm thử |
| AD-01 | Active Directory | Domain Controller Discovery | Đã kiểm thử |
| AD-02 | Active Directory | DNS Resolution | Đã kiểm thử |
| AD-03 | Active Directory | SYSVOL Access | Đã kiểm thử |
| AD-04 | Active Directory | NETLOGON Access | Đã kiểm thử |
| AD-05 | Group Policy | Policy Processing | Đã kiểm thử |
| FS-01 | File Server | SMB Connectivity | Đã kiểm thử |
| FS-02 | File Server | Authorized Read/Write | Đã kiểm thử |
| FS-03 | File Server | Unauthorized Access Restriction | Đã kiểm thử |
| FS-04 | File Server | Drive Mapping | Đã kiểm thử |
| FS-05 | File Server | Shadow Copy Restore | Đã thực hành |
| LNX-01 | Linux | Connectivity to DC01/FS01 | Đã kiểm thử |
| LNX-02 | Linux | SSH Remote Administration | Cần Evidence |
| LNX-03 | Linux | Nginx HTTP Service | Cần Evidence |
| LNX-04 | Linux | Samba File Sharing | Cần Evidence |
| LNX-05 | Linux | PostgreSQL Database | Cần Evidence |
| LNX-06 | Linux | LVM Storage | Cần Evidence |
| LNX-07 | Linux | Zabbix Monitoring | Cần Evidence |
| SEC-01 | Security | Firewall Rule Review | Cần rà soát |
| BAK-01 | Recovery | Full Backup and Restore | Chưa xác nhận |

Bảng tổng hợp phản ánh phạm vi kết quả đã ghi nhận, không thay thế các bằng chứng kiểm thử gốc.

---

## 15. Đánh giá kết quả tích hợp

### 15.1. Các chức năng đã được chứng minh

**Network Integration**

Các subnet liên quan có thể giao tiếp thông qua pfSense theo những luồng đã kiểm thử.

**Domain Infrastructure**

USER01 có thể tìm Domain Controller và truy cập các tài nguyên Domain cần thiết.

**Centralized Configuration**

Group Policy được sử dụng để áp dụng cấu hình và ánh xạ tài nguyên mạng.

**Role-Based Authorization**

Quyền truy cập File Server được kiểm soát dựa trên Security Group và Permissions.

**Windows–Linux Connectivity**

LINUX01 có khả năng kết nối đến các máy chủ Windows trong hạ tầng.

### 15.2. Những chức năng chưa được xác minh đầy đủ

- Toàn bộ Linux Application Services.
- Chính sách Firewall theo nguyên tắc Least Privilege.
- Khả năng quản trị từ MGMT01 đối với mọi máy chủ.
- Backup và Disaster Recovery hoàn chỉnh.
- Monitoring Coverage của Zabbix.
- High Availability.
- Performance Testing.

Các giới hạn này được ghi nhận để phân biệt kết quả triển khai thực tế với khả năng mở rộng trong tương lai.

---

## 16. Kết luận

Các bài kiểm thử tích hợp trong Project tập trung xác minh khả năng phối hợp giữa Networking, Active Directory, DNS, DHCP, Group Policy, File Server và Linux Infrastructure.

Những kết quả đã ghi nhận cho thấy hệ thống có khả năng cung cấp các chức năng quan trọng của một môi trường IT doanh nghiệp mô phỏng, bao gồm:

- Kết nối giữa các vùng mạng.
- Cấp phát IP và phân giải tên nội bộ.
- Domain Controller Discovery.
- Truy cập tài nguyên Domain.
- Áp dụng Group Policy.
- Phân quyền File Server theo Security Group.
- Truy cập dữ liệu qua SMB.
- Tích hợp kết nối giữa Windows và Linux.

Việc phân biệt rõ kết quả đã kiểm thử và chức năng chưa được xác minh giúp tài liệu phản ánh chính xác mức độ hoàn thành của dự án.