# Enterprise IT Infrastructure & System Administration Lab

**Dự án mô phỏng hạ tầng CNTT doanh nghiệp trên VMware Workstation**, tập trung vào quản trị hệ thống Windows–Linux, phân đoạn mạng, quản lý danh tính tập trung, phân quyền dữ liệu và kiểm thử tích hợp dịch vụ.

**Project Type:** Personal Infrastructure Lab  
**Platform:** VMware Workstation  
**Domain:** `corp.local`  
**Focus Areas:** System Administration · Network Infrastructure · Active Directory · File Services · Linux Administration · Troubleshooting

---

## 1. Project Overview

Trong môi trường doanh nghiệp, hệ thống CNTT cần cung cấp khả năng quản lý tập trung đối với người dùng, thiết bị, tài nguyên lưu trữ và các dịch vụ mạng.

Dự án được xây dựng nhằm mô phỏng một hệ thống hạ tầng nội bộ có nhiều phòng ban, trong đó các thành phần phải phối hợp để cung cấp dịch vụ cho người dùng.

Các yêu cầu chính:

- Phân tách hệ thống thành các vùng mạng SERVER, USER và MGMT.
- Quản lý tài khoản và máy tính thông qua Active Directory.
- Cung cấp DNS và DHCP tập trung.
- Áp dụng cấu hình máy trạm bằng Group Policy.
- Chia sẻ dữ liệu theo phòng ban và kiểm soát quyền truy cập.
- Bổ sung Linux Server để thực hành quản trị các dịch vụ hạ tầng.
- Kiểm thử khả năng kết nối, xác thực và phân quyền giữa các thành phần.

Dự án ưu tiên **thiết kế hệ thống, triển khai chức năng và xác minh kết quả thực tế**, thay vì chỉ cài đặt các dịch vụ độc lập.

## 2. Infrastructure Architecture

### Network Topology

```text
                     External Network
                            |
                            |
                    +----------------+
                    |    pfSense     |
                    | Routing/Firewall
                    +----------------+
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
    SERVER Network     USER Network      MGMT Network
    10.10.10.0/24      10.10.20.0/24     10.10.30.0/24
          |                 |                 |
     +----+----+            |                 |
     |    |    |            |                 |
     v    v    v            v                 v
   DC01 FS01 LINUX01      USER01/02         MGMT01
     |    |    |            |                 |
   AD DS SMB Linux      Domain Client       RSAT
   DNS       Services
   DHCP
```

Ba vùng mạng được tổ chức thành các subnet logic riêng biệt, sử dụng pfSense để định tuyến và kiểm soát lưu lượng.

> Kiến trúc được triển khai trên mạng ảo VMware. Dự án không tuyên bố đã kiểm thử VLAN tagging 802.1Q trên thiết bị Switch vật lý.

### Virtual Machine Inventory

| Hostname | Operating System | IP Address | Role |
|---|---|---|---|
| pfSense | pfSense CE 2.5.2 | Multiple Interfaces | Router, Firewall, DHCP Relay |
| DC01 | Windows Server 2022 | `10.10.10.10` | AD DS, DNS, DHCP |
| FS01 | Windows Server 2022 | `10.10.10.20` | SMB File Server |
| LINUX01 | Ubuntu Server 22.04.5 LTS | `10.10.10.30` | Linux Infrastructure Services |
| USER01 | Windows 10 | `10.10.20.102` | Domain Workstation |
| MGMT01 | Windows 10 | `10.10.30.10` | Administration Workstation |

Môi trường được triển khai trên máy Host có 16 GB RAM. Các máy ảo có thể được vận hành theo từng nhóm chức năng để phù hợp với giới hạn tài nguyên.

## 3. Technology Stack

| Category | Technologies |
|---|---|
| Virtualization | VMware Workstation |
| Networking | pfSense, IPv4 Subnetting, Routing, Firewall, DHCP Relay |
| Identity Management | Active Directory Domain Services, Organizational Units, Security Groups |
| Network Services | Windows DNS Server, Windows DHCP Server |
| Endpoint Management | Group Policy, Domain Join, RSAT |
| File Services | SMB, NTFS Permissions, FSRM, Shadow Copies |
| Linux | Ubuntu Server, OpenSSH, Nginx, Samba, PostgreSQL, LVM, Zabbix |
| Validation | PowerShell, Windows Administrative Tools, Linux CLI, Network Diagnostics |

Các công nghệ thuộc phạm vi thực hành không đồng nghĩa mọi chức năng đều đã được xác minh hoàn chỉnh. Trạng thái kiểm thử được ghi rõ trong tài liệu tương ứng.

## 4. Key Implemented Features

### Network Segmentation & Routing

Thiết kế ba vùng mạng:

- **SERVER:** Dành cho Domain Controller, File Server và Linux Server.
- **USER:** Dành cho máy trạm người dùng.
- **MGMT:** Dành cho máy trạm quản trị.

pfSense cung cấp khả năng định tuyến giữa các subnet và là điểm kiểm soát lưu lượng mạng.

Các bài kiểm thử xác nhận máy trạm USER có thể sử dụng những dịch vụ cần thiết tại SERVER Network.

[Chi tiết Network Infrastructure](docs/03-network-infrastructure.md)

### Centralized Identity Management

Triển khai Domain `corp.local` trên DC01 với Active Directory Domain Services.

Các chức năng chính:

- Quản lý Domain Users và Computers.
- Tổ chức người dùng theo phòng ban.
- Sử dụng Security Groups để quản lý quyền.
- Domain Join cho máy trạm.
- Group Policy Processing.
- DNS và DHCP tập trung.

Các bài kiểm thử Domain Controller Discovery, SYSVOL, NETLOGON và Group Policy đã được thực hiện.

[Chi tiết Active Directory Services](docs/04-active-directory-services.md)

### File Services & Access Control

Triển khai Windows File Server trên FS01 với các tài nguyên:

- Accounting.
- HR.
- IT.
- Public.

Sử dụng Active Directory Security Groups, Share Permissions và NTFS Permissions để quản lý truy cập.

Trong bài kiểm thử, tài khoản thuộc `GG_Accounting` có thể đọc/ghi dữ liệu Accounting, truy cập Public và bị hạn chế truy cập tài nguyên HR/IT không được cấp quyền.

Group Policy Drive Mapping cung cấp các ổ đĩa mạng `A:` và `P:` cho tài khoản kiểm thử.

FSRM và Shadow Copies cũng được đưa vào phạm vi quản trị lưu trữ.

[Chi tiết File Services & Access Control](docs/05-file-services-access-control.md)

### Linux Infrastructure

Triển khai LINUX01 nhằm bổ sung môi trường quản trị Linux vào kiến trúc Windows Server.

Phạm vi công nghệ gồm:

- SSH Remote Administration.
- Nginx Web Services.
- Samba File Sharing.
- PostgreSQL Database Services.
- LVM Storage Management.
- Zabbix Monitoring.

Kết nối mạng giữa LINUX01 với DC01 và FS01 đã được kiểm thử. Kết quả vận hành của từng dịch vụ Linux cần được đối chiếu với Evidence tương ứng.

[Chi tiết Linux Infrastructure Services](docs/06-linux-infrastructure-services.md)

## 5. Engineering Challenges & Troubleshooting

Trong quá trình triển khai, một số vấn đề kỹ thuật đã được ghi nhận:

| Challenge | Technical Learning |
|---|---|
| DNS phân giải về địa chỉ IP cũ | Kiểm tra DNS Records, Client Cache và Domain Discovery |
| ICMP bị chặn bởi Windows Firewall | Phân biệt Ping với TCP/Application Connectivity |
| Linux không kết nối được Gateway | Phân tích IP, Routing và ARP/Neighbor Cache |
| SMB bị ảnh hưởng sau khi thay đổi Firewall Rules | Kiểm tra kết nối theo từng dịch vụ và phân biệt lỗi mạng với lỗi quyền |
| Group Policy không có hiệu lực như mong đợi | Xác minh GPO Links, Scope và `gpresult` |
| Quyền truy cập không tương ứng với tên tài khoản | Xác minh Security Group Membership và ACLs |

Dự án cũng ghi nhận một số vấn đề chưa được xác minh hoàn toàn, bao gồm Network Profile Public/Unidentified trên DC01 và trạng thái cấu hình Firewall cuối cùng.

[Chi tiết Engineering Challenges](docs/08-engineering-challenges.md)

## 6. Documentation

Tài liệu được tổ chức theo chức năng và kết quả kỹ thuật:

| Document | Description |
|---|---|
| [01 — Project Overview](docs/01-project-overview.md) | Bài toán, mục tiêu và phạm vi dự án |
| [02 — Infrastructure Architecture](docs/02-infrastructure-architecture.md) | Kiến trúc hệ thống và thiết kế các thành phần |
| [03 — Network Infrastructure](docs/03-network-infrastructure.md) | Subnet, pfSense, Routing, Firewall, DHCP Relay |
| [04 — Active Directory Services](docs/04-active-directory-services.md) | AD DS, DNS, DHCP, OU, Groups và Group Policy |
| [05 — File Services & Access Control](docs/05-file-services-access-control.md) | SMB, Permissions, Drive Mapping và Storage |
| [06 — Linux Infrastructure Services](docs/06-linux-infrastructure-services.md) | Linux Administration và các dịch vụ hạ tầng |
| [07 — Integration Validation](docs/07-integration-validation.md) | Kịch bản và kết quả kiểm thử tích hợp |
| [08 — Engineering Challenges](docs/08-engineering-challenges.md) | Sự cố, phương pháp xử lý và bài học kỹ thuật |

## 7. Project Outcomes

Thông qua dự án, các nhóm kỹ năng được thực hành gồm:

**Infrastructure Design**

Thiết kế kiến trúc máy chủ, mạng nội bộ và các vùng quản trị.

**Windows Server Administration**

Triển khai AD DS, DNS, DHCP, Group Policy và File Services.

**Network Administration**

Quản lý IP Addressing, Routing, Firewall và DHCP Relay.

**Identity & Access Management**

Quản lý tài khoản Domain, Security Groups và quyền truy cập dữ liệu.

**Linux Administration**

Quản trị Linux Server và nghiên cứu các dịch vụ hạ tầng phổ biến.

**System Integration & Troubleshooting**

Phân tích sự cố và xác minh chức năng giữa nhiều thành phần của hệ thống.

## 8. Limitations

Đây là môi trường **homelab phục vụ học tập và xây dựng năng lực kỹ thuật**, không phải hệ thống Production.

Các giới hạn hiện tại:

- Chỉ có một Domain Controller.
- Chưa triển khai High Availability.
- Chưa xác minh quy trình Disaster Recovery hoàn chỉnh.
- Chưa có Performance Testing ở quy mô doanh nghiệp.
- Một số Linux Services cần bổ sung kết quả kiểm thử.
- Chưa có Security Audit toàn diện.

Các giới hạn được ghi nhận để phân biệt rõ phạm vi đã triển khai và những nội dung có thể mở rộng.

---

## 9. Project Summary

**Enterprise IT Infrastructure & System Administration Lab** là dự án thực hành xây dựng hệ thống CNTT doanh nghiệp mô phỏng, kết hợp Networking, Windows Server, Active Directory, File Services và Linux Infrastructure.

Điểm trọng tâm của dự án là khả năng thiết kế các thành phần có liên kết với nhau, triển khai dịch vụ phục vụ người dùng và xác minh kết quả thông qua các tình huống thực tế.