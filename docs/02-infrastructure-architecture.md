# 02 — Thiết kế kiến trúc hạ tầng tổng thể

## 1. Tổng quan kiến trúc

Dự án **Enterprise IT Infrastructure & System Administration Lab** được thiết kế nhằm mô phỏng hạ tầng công nghệ thông tin của một doanh nghiệp quy mô nhỏ, bao gồm các thành phần mạng, quản lý danh tính, quản lý máy trạm, lưu trữ dữ liệu và các dịch vụ máy chủ.

Kiến trúc được triển khai trên nền tảng VMware Workstation, sử dụng pfSense làm thiết bị định tuyến và kiểm soát lưu lượng mạng, Windows Server cung cấp các dịch vụ quản trị tập trung, Ubuntu Server đảm nhiệm các dịch vụ Linux bổ trợ.

Hệ thống được tổ chức theo mô hình **Centralized Infrastructure Management**, trong đó các chức năng quản trị quan trọng được cung cấp tập trung thay vì cấu hình độc lập trên từng máy trạm.

### Các mục tiêu thiết kế

- Phân tách hệ thống theo vai trò mạng và chức năng sử dụng.
- Quản lý tập trung tài khoản, máy tính và chính sách.
- Cung cấp các dịch vụ mạng và tài nguyên nội bộ.
- Kiểm soát truy cập đến tài nguyên theo quyền của người dùng.
- Hỗ trợ quản trị Windows và Linux trong cùng môi trường.
- Tạo điều kiện kiểm thử, giám sát và xử lý sự cố giữa các thành phần.

---

## 2. Kiến trúc tổng thể

### 2.1. Sơ đồ hạ tầng

```text
                              INTERNET
                                 |
                                 |
                          pfSense Firewall
                            WAN: Bridged
                                 |
                +----------------+----------------+
                |                |                |
                |                |                |
           SERVER NETWORK    USER NETWORK     MGMT NETWORK
           10.10.10.0/24     10.10.20.0/24    10.10.30.0/24
                |                |                |
           10.10.10.1       10.10.20.1       10.10.30.1
                |                |                |
        +-------+-------+        |                |
        |       |       |        |                |
       DC01    FS01   LINUX01  USER01/02        MGMT01
        |       |       |        |                |
       AD DS    SMB    Ubuntu  Windows 10       Windows 10
       DNS      NTFS   Nginx   Domain Client    RSAT
       DHCP     FSRM   Samba
                VSS    PostgreSQL
                       LVM
                       Zabbix
```

**Sơ đồ 2.1 — Kiến trúc logic của hệ thống**

Ba vùng mạng sử dụng các subnet khác nhau và được kết nối thông qua pfSense. Việc lưu lượng có được phép đi qua giữa các vùng mạng hay không phụ thuộc vào các Firewall Rules được cấu hình.

Các dịch vụ liệt kê dưới LINUX01 thể hiện phạm vi triển khai và quản trị của dự án; trạng thái hoạt động thực tế cần được xác nhận bằng Evidence tương ứng.

### 2.2. Các lớp chức năng

Kiến trúc có thể được phân thành bốn lớp chức năng:

| Lớp | Thành phần | Vai trò |
|---|---|---|
| Network Infrastructure | pfSense, VMware Virtual Networks | Kết nối và kiểm soát lưu lượng giữa các mạng |
| Core Infrastructure Services | DC01 | Danh tính, DNS, DHCP và chính sách Domain |
| Resource & Application Services | FS01, LINUX01 | Chia sẻ dữ liệu và cung cấp dịch vụ bổ trợ |
| Endpoint & Administration | USER01, MGMT01 | Sử dụng tài nguyên và quản trị hệ thống |

Việc phân lớp giúp xác định trách nhiệm của từng thành phần và phân tích các phụ thuộc khi một dịch vụ gặp sự cố.

---

## 3. Thiết kế phân vùng mạng

### 3.1. Lý do phân chia mạng

Trong hệ thống doanh nghiệp, việc đặt toàn bộ máy chủ và máy trạm trong cùng một subnet làm giảm khả năng tổ chức và kiểm soát lưu lượng theo chức năng.

Vì vậy, dự án sử dụng ba vùng mạng riêng biệt.

| Vùng mạng | Subnet | Gateway | Chức năng |
|---|---|---|---|
| SERVER | `10.10.10.0/24` | `10.10.10.1` | Chứa các dịch vụ hạ tầng |
| USER | `10.10.20.0/24` | `10.10.20.1` | Chứa máy trạm người dùng |
| MGMT | `10.10.30.0/24` | `10.10.30.1` | Chứa máy trạm quản trị |

### 3.2. SERVER Network

Vùng SERVER chứa các máy chủ cung cấp dịch vụ cho toàn hệ thống:

- DC01: Identity, DNS và DHCP.
- FS01: File Sharing và Storage.
- LINUX01: Linux Infrastructure Services.

Việc tập trung các máy chủ vào một vùng mạng giúp xác định rõ các dịch vụ cần được truy cập từ những vùng mạng khác.

### 3.3. USER Network

Vùng USER đại diện cho mạng làm việc của nhân viên.

USER01 sử dụng các dịch vụ được cung cấp từ vùng SERVER như xác thực Domain, phân giải DNS, nhận cấu hình DHCP và truy cập tài nguyên chia sẻ.

Máy trạm không cần được đặt cùng subnet với Domain Controller hoặc File Server để sử dụng các dịch vụ này.

### 3.4. MGMT Network

Vùng MGMT phục vụ công việc quản trị hệ thống.

MGMT01 được bố trí trong mạng riêng và sử dụng các công cụ như RSAT để quản trị các thành phần Windows Server.

Mục tiêu thiết kế là phân biệt hoạt động quản trị với hoạt động sử dụng dịch vụ thông thường.

Tuy nhiên, việc tách subnet không tự động bảo đảm chỉ máy quản trị mới có quyền truy cập các dịch vụ quản trị. Điều này còn phụ thuộc vào Firewall Rules và quyền của tài khoản sử dụng.

### 3.5. Vai trò của pfSense

pfSense đảm nhiệm:

- Default Gateway cho ba vùng mạng.
- Định tuyến giữa các subnet.
- Kiểm soát lưu lượng theo Firewall Rules.
- DHCP Relay để chuyển tiếp yêu cầu DHCP từ vùng USER đến DHCP Server.
- Kết nối mạng lab với mạng bên ngoài thông qua WAN.

Trong dự án, các subnet được triển khai thông qua các giao diện mạng logic của môi trường ảo hóa.

Thiết kế này mô phỏng phân vùng mạng doanh nghiệp, nhưng không mặc định tương đương với cấu hình IEEE 802.1Q VLAN Tagging trên thiết bị Switch vật lý.

---

## 4. Thiết kế các máy chủ hạ tầng

### 4.1. DC01 — Domain Controller

| Thuộc tính | Giá trị |
|---|---|
| Hostname | DC01 |
| Operating System | Windows Server 2022 |
| IP Address | `10.10.10.10` |
| Domain | `corp.local` |
| Vai trò | AD DS, DNS, DHCP |

DC01 là thành phần trung tâm của hệ thống quản lý danh tính và cấu hình.

**Active Directory Domain Services**

Cung cấp môi trường quản lý tập trung các tài khoản người dùng, máy tính, Organizational Units và Security Groups.

**DNS Server**

Phân giải tên miền nội bộ và hỗ trợ các hoạt động tìm kiếm Domain Controller, xác thực và truy cập tài nguyên bằng hostname.

**DHCP Server**

Cấp phát thông tin cấu hình mạng cho các máy trạm. Đối với các subnet khác mạng với DC01, yêu cầu DHCP được chuyển tiếp thông qua DHCP Relay.

**Lý do tích hợp AD DS, DNS và DHCP trên DC01**

Đối với một môi trường lab quy mô nhỏ, việc tích hợp ba dịch vụ giúp giảm số lượng máy ảo cần vận hành và thuận tiện cho quá trình nghiên cứu mối quan hệ giữa các dịch vụ.

Tuy nhiên, thiết kế này cũng tạo ra điểm phụ thuộc tập trung: khi DC01 không khả dụng, nhiều chức năng như xác thực Domain, DNS nội bộ và cấp phát DHCP có thể bị ảnh hưởng.

Dự án chưa triển khai Domain Controller dự phòng.

---

### 4.2. FS01 — Windows File Server

| Thuộc tính | Giá trị |
|---|---|
| Hostname | FS01 |
| Operating System | Windows Server 2022 |
| IP Address | `10.10.10.20` |
| Domain Membership | `corp.local` |
| Vai trò | Centralized File Services |

FS01 được triển khai riêng biệt với Domain Controller nhằm tách biệt chức năng quản lý danh tính và chức năng lưu trữ dữ liệu.

Hệ thống cung cấp tài nguyên chia sẻ cho các nhóm:

- Accounting.
- HR.
- IT.
- Public.

Các cơ chế kiểm soát truy cập gồm:

- Active Directory Security Groups.
- SMB Share Permissions.
- NTFS Permissions.

Ngoài ra, phạm vi quản trị lưu trữ bao gồm FSRM và Shadow Copies.

**Lý do tách FS01 khỏi DC01**

Việc tách hai máy chủ giúp:

- Phân định trách nhiệm của từng máy chủ.
- Hạn chế việc lưu trữ dữ liệu người dùng trực tiếp trên Domain Controller.
- Quản lý cấu hình và quyền truy cập File Server độc lập hơn.
- Tạo môi trường thực hành xử lý sự cố riêng cho dịch vụ SMB và Storage.

FS01 vẫn phụ thuộc vào các dịch vụ Domain để thực hiện những chức năng xác thực và phân quyền dựa trên tài khoản Domain.

---

### 4.3. LINUX01 — Linux Infrastructure Server

| Thuộc tính | Giá trị |
|---|---|
| Hostname | LINUX01 |
| Operating System | Ubuntu Server 22.04.5 LTS |
| IP Address | `10.10.10.30` |
| Vai trò | Linux Infrastructure Services |

LINUX01 được triển khai nhằm mở rộng hệ thống sang môi trường quản trị Linux.

Các nhóm dịch vụ thuộc phạm vi dự án:

| Công nghệ | Mục đích |
|---|---|
| SSH | Quản trị Linux từ xa |
| Nginx | Cung cấp Web Service |
| Samba | Chia sẻ tài nguyên thông qua giao thức SMB |
| PostgreSQL | Quản trị cơ sở dữ liệu |
| LVM | Quản lý lưu trữ logic |
| Zabbix | Giám sát hạ tầng |

**Lý do sử dụng Linux Server**

Trong môi trường doanh nghiệp, hệ thống Windows thường hoạt động cùng các dịch vụ Linux phục vụ ứng dụng, cơ sở dữ liệu hoặc giám sát.

Việc bổ sung LINUX01 giúp dự án thể hiện khả năng quản trị nhiều hệ điều hành, xử lý kết nối mạng và kiểm tra dịch vụ trong cùng một hạ tầng.

LINUX01 không được mặc định là thành viên của Active Directory Domain. Tích hợp Domain, nếu có, phải được xác nhận bằng cấu hình thực tế.

---

## 5. Thiết kế máy trạm

### 5.1. USER01 — Domain Client

| Thuộc tính | Giá trị |
|---|---|
| Hostname | USER01 |
| Operating System | Windows 10 |
| IP Address | `10.10.20.102` |
| Network | USER |
| Domain | `corp.local` |

USER01 mô phỏng máy trạm của nhân viên trong doanh nghiệp.

Máy được sử dụng để kiểm tra:

- Nhận cấu hình mạng qua DHCP.
- Domain Join.
- Đăng nhập bằng tài khoản Domain.
- Group Policy Processing.
- Truy cập File Server.
- Phân quyền theo Security Group.

Trong bài kiểm thử phân quyền, tài khoản `corp\hr.kimuyen` được chủ đích bố trí trong nhóm `GG_Accounting` và cơ cấu Accounting.

Điều này cho phép kiểm tra quyền truy cập dựa trên nhóm bảo mật thay vì dựa vào tên tài khoản.

### 5.2. MGMT01 — Administration Workstation

| Thuộc tính | Giá trị |
|---|---|
| Hostname | MGMT01 |
| Operating System | Windows 10 |
| IP Address | `10.10.30.10` |
| Network | MGMT |
| Công cụ | RSAT |

MGMT01 đại diện cho máy trạm của người quản trị hệ thống.

Máy được bố trí riêng tại vùng MGMT để phục vụ các hoạt động quản trị hạ tầng.

Các quyền quản trị thực tế phụ thuộc vào tài khoản và Security Groups được cấp, không phụ thuộc riêng vào việc máy nằm trong MGMT Network.

---

## 6. Quan hệ phụ thuộc giữa các dịch vụ

Một hệ thống hạ tầng không chỉ được đánh giá bằng khả năng hoạt động độc lập của từng máy chủ mà còn bởi khả năng phối hợp giữa các dịch vụ.

### 6.1. Sơ đồ phụ thuộc

```text
           USER01
             |
             v
       Network Access
             |
             v
          pfSense
             |
       +-----+------+
       |            |
       v            v
      DC01         FS01
       |            |
       |            |
  +----+----+       |
  |    |    |       |
  AD  DNS  DHCP     |
  |    |            |
  +----+------------+
       |
       v
  Domain Authentication
       |
       v
  Resource Authorization
       |
       v
  SMB File Access
```

Sơ đồ thể hiện các phụ thuộc chức năng chính, không biểu diễn toàn bộ các kết nối mạng hoặc thứ tự giao thức bắt buộc.

### 6.2. Các phụ thuộc quan trọng

| Dịch vụ | Phụ thuộc | Ảnh hưởng khi gặp sự cố |
|---|---|---|
| Domain Authentication | AD DS, DNS, Network | Đăng nhập hoặc xác thực Domain có thể bị ảnh hưởng |
| Group Policy | AD DS, DNS, SYSVOL, Network | Chính sách có thể không được cập nhật |
| DHCP | DHCP Server, DHCP Relay, Network | Client có thể không nhận được cấu hình IP mới |
| File Sharing | FS01, SMB, DNS, Authentication, Permissions | Người dùng có thể không truy cập được tài nguyên |
| Linux Services | Linux OS, Network, Service Configuration | Dịch vụ tương ứng có thể không khả dụng |

Bảng này giúp xác định hướng kiểm tra khi xảy ra lỗi liên quan đến nhiều thành phần.

---

## 7. Luồng hoạt động tiêu biểu

### 7.1. Máy trạm tham gia Domain

Khi một máy trạm Windows tham gia `corp.local`, hệ thống cần đáp ứng các điều kiện:

1. Máy trạm có cấu hình IP và định tuyến phù hợp.
2. DNS có khả năng phân giải các bản ghi liên quan đến Domain Controller.
3. Máy trạm kết nối được đến DC01 qua các dịch vụ cần thiết.
4. Tài khoản thực hiện Domain Join có đủ quyền.
5. Quá trình Domain Join hoàn tất và máy tính được đăng ký trong Active Directory.

Luồng này được sử dụng để đánh giá khả năng phối hợp giữa Networking, DNS và Active Directory.

### 7.2. Người dùng truy cập File Server

Khi người dùng đăng nhập USER01 và truy cập tài nguyên chia sẻ trên FS01:

1. USER01 sử dụng DNS để phân giải tên File Server.
2. Lưu lượng được định tuyến qua pfSense đến SERVER Network.
3. Người dùng thực hiện xác thực bằng thông tin Domain.
4. FS01 đánh giá Share Permissions và NTFS Permissions.
5. Người dùng được cấp quyền hoặc từ chối truy cập tùy theo chính sách.

Đây là một trong những luồng tích hợp chính của dự án.

### 7.3. Quản trị hạ tầng từ MGMT01

MGMT01 được sử dụng để truy cập các công cụ quản trị phù hợp.

Các hoạt động quản trị phải đáp ứng đồng thời:

- Kết nối mạng đến máy chủ đích.
- Firewall Rules cho phép giao thức cần thiết.
- Tài khoản có quyền quản trị tương ứng.
- Dịch vụ quản trị trên máy chủ đích đang hoạt động.

Việc đặt MGMT01 trong mạng quản trị không tự động cấp quyền quản trị hệ thống.

---

## 8. Các quyết định thiết kế chính

| Quyết định | Lý do lựa chọn | Đánh đổi |
|---|---|---|
| Sử dụng VMware Workstation | Triển khai nhiều máy chủ trên một máy vật lý | Giới hạn RAM và CPU |
| Sử dụng pfSense | Tập trung Routing và Firewall | Phụ thuộc vào một Router/Firewall |
| Chia ba subnet | Phân tách chức năng Server, User và Management | Cần cấu hình Routing và Firewall |
| Triển khai AD DS | Quản lý tập trung danh tính | Phụ thuộc vào Domain Controller |
| Tích hợp DNS/DHCP trên DC01 | Giảm số lượng VM, thuận tiện quản trị lab | Tập trung nhiều dịch vụ trên một máy |
| Tách FS01 khỏi DC01 | Phân tách danh tính và lưu trữ | Tăng nhu cầu tài nguyên |
| Bổ sung LINUX01 | Thực hành quản trị đa nền tảng | Tăng độ phức tạp vận hành |
| Sử dụng Security Groups | Quản lý quyền truy cập theo vai trò | Cần quản lý Membership chính xác |

Các quyết định này phản ánh sự cân bằng giữa tính thực tế của mô hình doanh nghiệp và giới hạn tài nguyên của môi trường lab.

---

## 9. Đánh giá kiến trúc

### 9.1. Ưu điểm

**Quản lý tập trung:** Active Directory, DNS, DHCP và Group Policy hỗ trợ quản trị người dùng, máy tính và cấu hình từ hệ thống trung tâm.

**Phân tách chức năng:** Các máy chủ DC01, FS01 và LINUX01 có vai trò tương đối rõ ràng.

**Phân vùng mạng:** SERVER, USER và MGMT được bố trí trên các subnet riêng.

**Khả năng tích hợp:** Máy trạm có thể sử dụng nhiều dịch vụ hạ tầng thông qua mạng nội bộ.

**Khả năng thực hành:** Kiến trúc hỗ trợ các bài kiểm thử về Networking, Authentication, Authorization, Storage và Linux Services.

### 9.2. Giới hạn

**Single Points of Failure:** pfSense và DC01 chưa có thành phần dự phòng.

**Giới hạn tài nguyên:** Máy Host 16 GB RAM không phù hợp để vận hành đồng thời nhiều VM cấu hình lớn.

**Chưa có High Availability:** Các dịch vụ không được triển khai theo mô hình Cluster hoặc Failover.

**Chưa có kiểm thử hiệu năng quy mô lớn:** Dự án tập trung vào chức năng và tích hợp dịch vụ.

**Chưa xác nhận Disaster Recovery:** Cần kiểm thử phục hồi thực tế trước khi đánh giá khả năng khôi phục toàn hệ thống.

---

## 10. Kết quả thiết kế

Kiến trúc Project 1 cung cấp nền tảng để triển khai các chức năng hạ tầng doanh nghiệp trong cùng một môi trường ảo hóa.

Các kết quả chính gồm:

- Thiết kế ba vùng mạng theo vai trò.
- Sử dụng pfSense làm trung tâm định tuyến và kiểm soát lưu lượng.
- Triển khai DC01 cung cấp AD DS, DNS và DHCP.
- Tách FS01 để cung cấp dịch vụ chia sẻ dữ liệu.
- Bổ sung LINUX01 cho các dịch vụ Linux.
- Sử dụng USER01 để kiểm thử hoạt động của người dùng Domain.
- Sử dụng MGMT01 làm máy trạm phục vụ quản trị.
- Xác định các phụ thuộc giữa Network, Identity, Configuration và Resource Services.

Kiến trúc này là cơ sở cho các tài liệu triển khai chức năng và kiểm thử tích hợp tiếp theo.

---

## 11. Kết luận

Dự án **Enterprise IT Infrastructure & System Administration Lab** được thiết kế theo hướng quản lý tập trung, phân chia chức năng và tích hợp các dịch vụ Windows–Linux.

Các quyết định kiến trúc không chỉ dựa trên khả năng cài đặt công nghệ mà còn xem xét mục đích sử dụng, mối quan hệ phụ thuộc, khả năng quản trị và giới hạn tài nguyên.

Thiết kế này tạo nền tảng để triển khai các chức năng Networking, Active Directory, File Services và Linux Infrastructure trong một hệ thống thống nhất, đồng thời hỗ trợ việc kiểm thử và phân tích sự cố hạ tầng.