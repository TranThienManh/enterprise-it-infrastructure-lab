# 04 — Triển khai Active Directory và các dịch vụ Domain

## 1. Tổng quan

Trong môi trường doanh nghiệp, việc quản lý tài khoản và cấu hình riêng lẻ trên từng máy tính gây khó khăn khi số lượng nhân viên và thiết bị tăng lên.

Một hệ thống quản lý tập trung cần đáp ứng các yêu cầu:

- Quản lý danh tính người dùng và máy tính.
- Xác thực người dùng khi đăng nhập hệ thống.
- Phân quyền truy cập tài nguyên theo phòng ban.
- Quản lý và áp dụng chính sách cấu hình.
- Cung cấp dịch vụ phân giải tên nội bộ.
- Cấp phát cấu hình mạng cho máy trạm.
- Hỗ trợ quản trị hệ thống từ một vị trí tập trung.

Để đáp ứng các yêu cầu này, Project 1 triển khai **Active Directory Domain Services (AD DS)** trên Windows Server 2022, kết hợp DNS, DHCP và Group Policy.

Domain nội bộ được sử dụng là:

`corp.local`

Domain Controller chính là:

`DC01 — 10.10.10.10`

Mục tiêu không chỉ là triển khai một Domain Controller mà còn xây dựng một hệ thống quản lý danh tính có khả năng tích hợp với các dịch vụ mạng và lưu trữ nội bộ.

---

## 2. Mục tiêu và phạm vi triển khai

| Thành phần | Mục đích | Chức năng |
|---|---|---|
| Active Directory Domain Services | Quản lý danh tính tập trung | Domain Users, Computers, OUs, Groups |
| DNS Server | Phân giải tên và hỗ trợ tìm Domain Controller | Domain Name Resolution, SRV Records |
| DHCP Server | Cấp phát IP tập trung | DHCP Scopes, Leases, Options |
| Group Policy | Quản lý cấu hình theo Domain | User Policies, Computer Policies |
| Domain Join | Đưa máy trạm vào hệ thống quản lý | Domain Authentication |
| RSAT | Quản trị dịch vụ từ máy trạm | Remote Administration |
| Security Groups | Quản lý quyền theo vai trò | Role-Based Access Control |

Các thành phần được triển khai để phục vụ hệ thống gồm máy chủ DC01, FS01 và các máy trạm USER01, MGMT01.

---

## 3. Kiến trúc Active Directory

### 3.1. Thông tin hệ thống

| Thuộc tính | Giá trị |
|---|---|
| Domain | `corp.local` |
| Domain Controller | DC01 |
| Operating System | Windows Server 2022 |
| IP Address | `10.10.10.10` |
| Network | SERVER — `10.10.10.0/24` |
| Default Gateway | `10.10.10.1` |
| Core Services | AD DS, DNS, DHCP |

DC01 đảm nhiệm vai trò cung cấp danh tính và các dịch vụ mạng cốt lõi cho Domain.

### 3.2. Sơ đồ kiến trúc

```text
                        corp.local
                            |
                            v
                    +----------------+
                    |      DC01      |
                    | 10.10.10.10    |
                    | Windows Server |
                    +----------------+
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
            AD DS          DNS           DHCP
              |             |             |
              |             |             |
        Users/Groups    Name Resolution  IP Allocation
              |             |             |
              +-------------+-------------+
                            |
                         pfSense
                            |
                +-----------+-----------+
                |                       |
                v                       v
             USER01                   MGMT01
        Domain Workstation       Administration
                |
                v
              FS01
        Domain File Server
```

**Hình 4.1 — Kiến trúc các dịch vụ Domain trong Project 1.**

Sơ đồ thể hiện quan hệ chức năng giữa các thành phần, không phải toàn bộ các kết nối giao thức.

### 3.3. Lý do tích hợp AD DS, DNS và DHCP trên DC01

Dự án sử dụng một Domain Controller duy nhất để giảm số lượng máy ảo cần vận hành.

Việc tích hợp các dịch vụ giúp:

- Quản lý tập trung hệ thống Domain.
- Đơn giản hóa kiến trúc trong môi trường lab.
- Thuận tiện kiểm thử mối quan hệ giữa DNS, DHCP và AD DS.
- Giảm nhu cầu tài nguyên của VMware Host.

Tuy nhiên, thiết kế này tạo ra một **Single Point of Failure**.

Nếu DC01 không khả dụng, hoạt động xác thực Domain, phân giải DNS nội bộ và cấp phát DHCP mới có thể bị ảnh hưởng.

Dự án chưa triển khai Domain Controller dự phòng.

---

## 4. Active Directory Domain Services

### 4.1. Chức năng

Active Directory Domain Services cung cấp cơ sở dữ liệu tập trung để quản lý các đối tượng trong Domain.

Các đối tượng chính gồm:

- User Accounts.
- Computer Accounts.
- Organizational Units.
- Security Groups.
- Group Policy Objects.

Thay vì tạo và quản lý tài khoản riêng biệt trên từng máy trạm, người quản trị sử dụng Active Directory để quản lý người dùng trong cùng một hệ thống.

### 4.2. Cấu trúc tổ chức

Dự án mô phỏng môi trường doanh nghiệp với các nhóm chức năng:

- HR.
- Accounting.
- IT.
- Computer Management.

Các Organizational Units được sử dụng để tổ chức đối tượng và xác định phạm vi áp dụng Group Policy.

Ví dụ về mô hình tổ chức logic:

```text
corp.local
│
├── HR
│   └── User Accounts
│
├── Accounting
│   └── User Accounts
│
├── IT
│   └── User Accounts
│
└── CORP_Computers
    └── Domain Computers
```

Đây là sơ đồ khái quát theo chức năng, không phải bản xuất đầy đủ Distinguished Names hoặc cây OU thực tế.

### 4.3. Lý do sử dụng Organizational Units

Organizational Units hỗ trợ:

- Phân loại đối tượng theo phòng ban.
- Tổ chức phạm vi quản trị.
- Liên kết Group Policy theo nhóm đối tượng.
- Phân tách cấu hình giữa người dùng và máy tính.

OU không tự động tạo ra quyền truy cập File Server.

Việc phân quyền tài nguyên được thực hiện thông qua Security Groups và cơ chế Authorization của dịch vụ đích.

---

## 5. Security Groups và phân quyền theo vai trò

### 5.1. Mục đích

Trong môi trường doanh nghiệp, quyền truy cập nên được quản lý theo vai trò hoặc phòng ban thay vì cấu hình riêng cho từng người dùng.

Project 1 sử dụng Security Groups để quản lý quyền truy cập các tài nguyên chia sẻ trên FS01.

Ví dụ:

| Nhóm quyền | Phạm vi sử dụng |
|---|---|
| `GG_Accounting` | Tài nguyên Accounting |
| Nhóm HR | Tài nguyên HR |
| Nhóm IT | Tài nguyên IT |
| Các nhóm được cấp quyền chung | Tài nguyên Public |

Tên chính xác của các nhóm ngoài `GG_Accounting` cần được đối chiếu với Active Directory thực tế trước khi công bố.

### 5.2. Nguyên tắc phân quyền

Quyền truy cập được thiết kế theo luồng:

```text
User Account
     |
     v
Security Group Membership
     |
     v
Share Permissions
     |
     v
NTFS Permissions
     |
     v
Authorized File Access
```

Cách tổ chức này giúp giảm số lượng quyền được gán trực tiếp cho từng tài khoản.

### 5.3. Trường hợp kiểm thử thực tế

Trong Project 1, tài khoản:

`corp\hr.kimuyen`

được chủ đích đặt trong cơ cấu Accounting và là thành viên của:

`GG_Accounting`

Mặc dù tên tài khoản có tiền tố `hr.`, quyền truy cập được xác định dựa trên Group Membership và Permissions thực tế.

Kết quả kiểm thử cho thấy tài khoản có thể truy cập tài nguyên Accounting và Public được cấp quyền, đồng thời bị hạn chế truy cập tài nguyên HR và IT.

**Giá trị kỹ thuật:** Cơ chế Authorization dựa trên Security Group, không dựa trên cách đặt tên tài khoản.

---

## 6. DNS — Dịch vụ phân giải tên nội bộ

### 6.1. Vai trò của DNS trong Active Directory

DNS là một thành phần quan trọng của hạ tầng Domain.

Ngoài việc phân giải hostname thành địa chỉ IP, DNS còn hỗ trợ máy trạm xác định vị trí các dịch vụ Active Directory thông qua các bản ghi dịch vụ.

Các chức năng chính:

- Phân giải tên Domain `corp.local`.
- Phân giải tên DC01 và FS01.
- Hỗ trợ Domain Controller Discovery.
- Hỗ trợ Domain Join.
- Hỗ trợ xác thực và xử lý Group Policy.

### 6.2. Thiết kế DNS

DC01 được sử dụng làm DNS Server nội bộ.

Máy trạm Domain cần sử dụng DNS có khả năng phân giải các bản ghi Active Directory.

Ví dụ về các tài nguyên nội bộ:

| Tên máy chủ | IP Address |
|---|---|
| `dc01.corp.local` | `10.10.10.10` |
| `fs01.corp.local` | `10.10.10.20` |

Việc sử dụng hostname giúp các dịch vụ được truy cập theo tên thay vì phụ thuộc trực tiếp vào địa chỉ IP trong thao tác của người dùng.

### 6.3. Kết quả chức năng

Trong quá trình thực hành:

- Máy trạm tìm được Domain Controller.
- Domain Name Resolution hoạt động.
- SYSVOL và NETLOGON được truy cập thành công.
- Máy trạm sử dụng được các dịch vụ Domain liên quan.

### 6.4. Sự cố đã gặp

Trong quá trình điều chỉnh mạng, một kết quả phân giải tên từng trỏ đến địa chỉ cũ `192.168.20.10`.

Việc kiểm tra cấu hình DNS và làm mới DNS Cache giúp xử lý vấn đề trong tình huống đã ghi nhận.

Sự cố này cho thấy kết nối IP thành công không đủ để kết luận hệ thống Domain hoạt động chính xác nếu DNS Resolution bị sai.

---

## 7. DHCP — Quản lý địa chỉ IP tập trung

### 7.1. Bài toán triển khai

DC01 được đặt tại SERVER Network:

`10.10.10.0/24`

Trong khi máy trạm USER01 hoạt động tại USER Network:

`10.10.20.0/24`

Do DHCP Discovery ban đầu sử dụng Broadcast, yêu cầu từ USER Network cần được chuyển tiếp đến DHCP Server ở subnet khác.

### 7.2. Giải pháp

Project 1 triển khai DHCP Server trên DC01 và sử dụng DHCP Relay trên pfSense.

```text
USER01
USER Network
     |
     | DHCP Discovery
     v
pfSense
DHCP Relay
     |
     v
DC01
DHCP Server
     |
     | DHCP Response
     v
USER01
```

### 7.3. Lý do lựa chọn

Thiết kế này cho phép:

- Quản lý cấp phát IP từ một DHCP Server.
- Hỗ trợ các máy trạm thuộc subnet khác.
- Tập trung quản lý DHCP Leases.
- Giảm nhu cầu triển khai DHCP Server riêng trên từng vùng mạng.

### 7.4. Kết quả kiểm thử

Máy trạm tại USER Network đã nhận cấu hình mạng thông qua DHCP.

Khả năng kết nối đến các dịch vụ Domain được kiểm tra sau khi nhận cấu hình IP.

**Kết quả:** DHCP Server và DHCP Relay phối hợp để cung cấp địa chỉ mạng cho máy trạm khác subnet.

---

## 8. Domain Join và quản lý máy trạm

### 8.1. Mục đích

Domain Join cho phép máy trạm trở thành đối tượng được quản lý trong Active Directory.

Sau khi tham gia Domain, máy trạm có thể:

- Sử dụng Domain Authentication.
- Nhận Group Policy phù hợp.
- Truy cập tài nguyên dựa trên tài khoản Domain.
- Được quản lý thông qua các công cụ quản trị tập trung.

### 8.2. Triển khai trên USER01

| Thuộc tính | Giá trị |
|---|---|
| Hostname | USER01 |
| Operating System | Windows 10 |
| Network | USER |
| IP Address | `10.10.20.102` |
| Domain | `corp.local` |

USER01 được sử dụng để kiểm thử hoạt động của Domain Client.

Các chức năng đã kiểm tra:

- Domain Controller Discovery.
- Truy cập SYSVOL.
- Truy cập NETLOGON.
- Group Policy Processing.
- Truy cập tài nguyên chia sẻ.

### 8.3. Kết quả

Máy trạm có thể sử dụng dịch vụ Domain từ SERVER Network thông qua kết nối được định tuyến bởi pfSense.

Điều này thể hiện sự tích hợp giữa Networking, DNS và Active Directory.

---

## 9. Group Policy — Quản lý cấu hình tập trung

### 9.1. Mục đích

Group Policy được sử dụng để quản lý cấu hình người dùng và máy tính trong Domain.

Thay vì cấu hình thủ công trên từng máy trạm, người quản trị có thể xây dựng các chính sách và liên kết chúng với phạm vi đối tượng phù hợp.

### 9.2. Các nhóm chính sách

Trong Project 1, Group Policy được sử dụng cho những chức năng như:

- Hạn chế một số thao tác cấu hình của người dùng.
- Ánh xạ ổ đĩa mạng.
- Áp dụng các cấu hình chung.
- Quản lý một số thiết lập máy trạm.

Một số GPO đã được xây dựng trong quá trình thực hành:

| Group Policy | Mục đích |
|---|---|
| `GPO_HR_Disable_ControlPanel` | Hạn chế truy cập Control Panel theo phạm vi áp dụng |
| `GPO_HR_Drive` | Cấu hình ánh xạ ổ đĩa mạng |
| `GPO_CORP_Users_Common` | Cấu hình chung cho người dùng Domain |
| `GPO_CORP_Computers_Security` | GPO bảo mật máy tính, liên kết đang được vô hiệu hóa có chủ đích |

Các GPO chỉ có hiệu lực khi đáp ứng điều kiện liên kết, phạm vi áp dụng, Security Filtering và quá trình xử lý chính sách.

Không phải mọi GPO được tạo đều đang có hiệu lực.

### 9.3. Drive Mapping

Một chức năng quan trọng được kiểm thử là ánh xạ ổ đĩa mạng thông qua Group Policy.

Đối với tài khoản kiểm thử Accounting:

| Drive | Tài nguyên | Kết quả |
|---|---|---|
| `A:` | Accounting | Truy cập được |
| `P:` | Public | Truy cập được |

Việc ánh xạ ổ đĩa giúp người dùng truy cập tài nguyên được cấp quyền thông qua ký tự ổ đĩa quen thuộc.

Drive Mapping không thay thế NTFS Permissions hoặc Share Permissions.

### 9.4. Kết quả kiểm thử

Các công cụ như `gpresult` được sử dụng để xác minh chính sách được xử lý trên máy trạm.

Kết quả thực hành cho thấy các chính sách và tài nguyên được cấu hình phù hợp có thể được áp dụng cho Domain Client.

**Giá trị kỹ thuật:** Cấu hình máy trạm và truy cập tài nguyên được quản lý tập trung thay vì thiết lập độc lập trên từng thiết bị.

---

## 10. Quản trị từ xa bằng RSAT

### 10.1. Mục đích

Trong môi trường doanh nghiệp, người quản trị không nhất thiết phải đăng nhập trực tiếp vào Domain Controller để thực hiện mọi tác vụ quản lý.

Project 1 bố trí MGMT01 tại MGMT Network để phục vụ công việc quản trị.

### 10.2. Kiến trúc

| Thành phần | Giá trị |
|---|---|
| Hostname | MGMT01 |
| Operating System | Windows 10 |
| IP Address | `10.10.30.10` |
| Network | MGMT |
| Công cụ | RSAT |

RSAT hỗ trợ truy cập các công cụ quản trị Windows Server từ máy trạm.

### 10.3. Ý nghĩa thiết kế

Việc bố trí máy trạm quản trị riêng giúp:

- Phân biệt máy người dùng và máy quản trị.
- Tổ chức lưu lượng quản trị theo vùng mạng.
- Thực hành quản lý hệ thống từ xa.
- Hỗ trợ mô hình phân quyền quản trị theo vai trò.

Tuy nhiên, việc sử dụng MGMT Network không tự động cấp quyền quản trị.

Quyền thực tế vẫn phụ thuộc vào tài khoản, Security Groups và chính sách truy cập.

---

## 11. Tích hợp Active Directory với File Server

### 11.1. Mục tiêu

Một trong những chức năng chính của dự án là sử dụng danh tính Domain để kiểm soát quyền truy cập tài nguyên chia sẻ.

FS01 được tham gia Domain `corp.local`.

Tài khoản người dùng và Security Groups được quản lý trên DC01.

### 11.2. Luồng truy cập

```text
USER01
Domain Client
     |
     v
Domain Authentication
DC01
     |
     v
Security Group Membership
     |
     v
FS01
SMB File Server
     |
     v
Share + NTFS Permissions
     |
     v
Allow / Deny Access
```

### 11.3. Kết quả chức năng

Trong bài kiểm thử với tài khoản Accounting:

- Truy cập Accounting Share thành công.
- Truy cập Public Share thành công.
- Đọc và ghi dữ liệu trên tài nguyên được cấp quyền.
- Truy cập tài nguyên không được cấp quyền bị hạn chế.

Điều này chứng minh khả năng tích hợp giữa Active Directory Security Groups và cơ chế phân quyền File Server.

---

## 12. Các tình huống kỹ thuật đã xử lý

### 12.1. Domain Controller Discovery

**Vấn đề:** Máy trạm cần tìm đúng Domain Controller trong môi trường nhiều subnet.

**Hướng giải quyết:** Kiểm tra DNS Configuration, Routing và các dịch vụ Domain.

**Kết quả:** Domain Controller Discovery hoạt động trong các bài kiểm thử.

### 12.2. DNS Cache không còn phù hợp

**Vấn đề:** Kết quả phân giải tên có thể chứa thông tin địa chỉ cũ.

**Hướng giải quyết:** Kiểm tra bản ghi DNS, DNS Server đang sử dụng và DNS Cache.

**Bài học:** DNS là thành phần thiết yếu đối với hoạt động của Active Directory.

### 12.3. Group Policy không áp dụng như mong đợi

**Vấn đề:** Chính sách có thể không có hiệu lực do phạm vi áp dụng, liên kết GPO hoặc quá trình xử lý chính sách.

**Hướng giải quyết:** Kiểm tra OU, GPO Links, Security Filtering và kết quả `gpresult`.

**Bài học:** Việc tạo GPO không đồng nghĩa chính sách đã được áp dụng thành công.

### 12.4. File Server Access

**Vấn đề:** Máy trạm có thể kết nối đến File Server nhưng vẫn không truy cập được một số tài nguyên.

**Hướng phân tích:** Kiểm tra Security Group Membership, Share Permissions và NTFS Permissions.

**Bài học:** Authentication và Authorization là hai chức năng khác nhau.

---

## 13. Đánh giá kết quả triển khai

| Chức năng | Kết quả | Trạng thái |
|---|---|---|
| Active Directory Domain | `corp.local` trên DC01 | Đã triển khai |
| DNS Server | Phân giải tên và tìm Domain Controller | Đã kiểm thử |
| DHCP Server | Cấp phát cấu hình mạng cho Client | Đã kiểm thử |
| DHCP Relay | Chuyển tiếp DHCP giữa USER và SERVER | Đã kiểm thử |
| Domain Join | Máy trạm tham gia Domain | Đã kiểm thử |
| SYSVOL / NETLOGON | Truy cập được từ Domain Client | Đã kiểm thử |
| Group Policy | Xử lý chính sách và Drive Mapping | Đã kiểm thử |
| Security Groups | Phân quyền truy cập File Server | Đã kiểm thử |
| RSAT | Máy trạm quản trị được bố trí riêng | Đã triển khai; cần bổ sung Evidence quản trị |
| Domain Controller Redundancy | Domain Controller dự phòng | Chưa triển khai |
| AD Backup & Recovery | Kiểm thử khôi phục Domain Controller | Chưa xác nhận |

### Bằng chứng cần đưa lên GitHub

Các Evidence phù hợp gồm:

- Active Directory Users and Computers.
- Cấu trúc Organizational Units.
- Security Group Membership.
- DNS Manager và bản ghi liên quan.
- DHCP Scope và DHCP Lease.
- Domain Join trên USER01.
- Kết quả Domain Controller Discovery.
- Kết quả SYSVOL và NETLOGON.
- Group Policy Management.
- Kết quả `gpresult`.
- Drive Mapping trên USER01.
- Kết quả kiểm thử quyền truy cập File Server.

Lưu tại:

`evidence/active-directory/`

Trước khi công khai, cần kiểm tra và che các thông tin nhạy cảm trong ảnh chụp hoặc kết quả lệnh.

---

## 14. Giới hạn của hệ thống

### Single Domain Controller

Hệ thống chỉ có một Domain Controller, chưa triển khai cơ chế dự phòng.

### Centralized Service Dependency

AD DS, DNS và DHCP được triển khai trên DC01. Sự cố của DC01 có thể ảnh hưởng đồng thời đến nhiều dịch vụ.

### Disaster Recovery

Chưa có bằng chứng xác nhận quá trình khôi phục Domain Controller hoặc Active Directory từ bản sao lưu.

### Group Policy Coverage

Các GPO được triển khai phục vụ những tình huống thực hành cụ thể, chưa xây dựng đầy đủ bộ chính sách bảo mật dành cho môi trường Production.

### Administrative Security

Chưa có kết quả đánh giá đầy đủ về phân quyền quản trị, tài khoản đặc quyền và các cơ chế bảo vệ Domain Controller ở mức doanh nghiệp.

---

## 15. Kết luận

Project 1 triển khai Active Directory Domain Services kết hợp DNS, DHCP và Group Policy nhằm xây dựng hệ thống quản lý danh tính và cấu hình tập trung.

DC01 cung cấp các dịch vụ cốt lõi cho Domain `corp.local`, trong khi các máy trạm và File Server sử dụng những dịch vụ này để thực hiện xác thực, áp dụng chính sách và kiểm soát truy cập tài nguyên.

Các chức năng Domain Join, DNS Resolution, DHCP Address Assignment, Group Policy Processing và File Server Authorization đã được thực hành và kiểm thử.

Thông qua quá trình triển khai, dự án thể hiện khả năng tích hợp các dịch vụ Windows Server trong một hệ thống hạ tầng doanh nghiệp, đồng thời phát triển kỹ năng quản trị Active Directory, quản lý người dùng, chính sách và xử lý sự cố Domain.