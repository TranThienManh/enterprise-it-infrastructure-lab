# 01 — Tổng quan dự án

## Enterprise IT Infrastructure & System Administration Lab

### 1. Giới thiệu dự án

Trong môi trường doanh nghiệp, hạ tầng công nghệ thông tin đóng vai trò quan trọng trong việc quản lý tài khoản nhân viên, kết nối mạng nội bộ, chia sẻ tài nguyên và vận hành các dịch vụ tập trung.

Dự án **Enterprise IT Infrastructure & System Administration Lab** được xây dựng nhằm mô phỏng một hệ thống hạ tầng IT doanh nghiệp quy mô nhỏ trên nền tảng VMware Workstation.

Thay vì triển khai các công nghệ riêng lẻ, dự án tập trung vào việc tích hợp nhiều thành phần thành một hệ thống có khả năng hoạt động phối hợp, bao gồm:

- Hạ tầng mạng và phân chia các vùng mạng.
- Hệ thống quản lý danh tính tập trung.
- Dịch vụ DNS và DHCP.
- Quản lý người dùng và máy tính bằng Group Policy.
- Hệ thống lưu trữ và chia sẻ dữ liệu nội bộ.
- Máy chủ Linux cung cấp các dịch vụ bổ trợ.
- Kiểm thử kết nối, phân quyền và xử lý sự cố.

Mục tiêu chính của dự án là thể hiện năng lực **thiết kế, triển khai, quản trị và kiểm thử hệ thống hạ tầng IT**, phù hợp với các công việc System Administrator, IT Infrastructure Engineer và IT Support.

---

### 2. Mục tiêu triển khai

Dự án được xây dựng nhằm giải quyết năm nhóm yêu cầu chính:

| Nhóm yêu cầu | Mục tiêu | Giải pháp triển khai |
|---|---|---|
| Hạ tầng mạng | Kết nối các hệ thống và phân chia vùng mạng theo chức năng | pfSense, Routing, Network Segmentation |
| Quản lý danh tính | Quản lý tập trung tài khoản và máy tính | Active Directory Domain Services |
| Quản lý cấu hình | Áp dụng chính sách và cấu hình máy trạm tập trung | Group Policy |
| Chia sẻ dữ liệu | Cung cấp tài nguyên lưu trữ theo phòng ban và kiểm soát truy cập | Windows File Server, SMB, NTFS |
| Dịch vụ Linux | Triển khai và quản trị các dịch vụ hạ tầng bổ trợ | Ubuntu Server, Nginx, Samba, PostgreSQL, LVM, Zabbix |

Các thành phần được thiết kế để hoạt động trong cùng một môi trường mạng doanh nghiệp mô phỏng.

---

### 3. Môi trường triển khai

Toàn bộ hệ thống được triển khai trên VMware Workstation với máy vật lý có dung lượng RAM 16 GB.

Hạ tầng được chia thành ba vùng mạng:

| Vùng mạng | Dải địa chỉ | Mục đích |
|---|---|---|
| SERVER | `10.10.10.0/24` | Chứa các máy chủ và dịch vụ hạ tầng |
| USER | `10.10.20.0/24` | Chứa các máy trạm người dùng |
| MGMT | `10.10.30.0/24` | Phục vụ hoạt động quản trị hệ thống |

pfSense đảm nhiệm vai trò Default Gateway và định tuyến giữa các vùng mạng theo chính sách Firewall.

#### Danh sách máy ảo

| Máy ảo | Địa chỉ IP | Vai trò |
|---|---|---|
| pfSense | `10.10.10.1`, `10.10.20.1`, `10.10.30.1` | Firewall và Router |
| DC01 | `10.10.10.10` | Active Directory, DNS, DHCP |
| FS01 | `10.10.10.20` | Windows File Server |
| LINUX01 | `10.10.10.30` | Linux Infrastructure Services |
| USER01 | `10.10.20.102` | Máy trạm tham gia Domain |
| MGMT01 | `10.10.30.10` | Máy trạm quản trị |

Domain nội bộ được sử dụng là `corp.local`.

---

### 4. Các chức năng chính của hệ thống

#### 4.1. Hạ tầng mạng doanh nghiệp

Hệ thống được phân chia thành ba vùng mạng SERVER, USER và MGMT nhằm tổ chức các thiết bị theo vai trò.

pfSense thực hiện chức năng định tuyến giữa các vùng mạng và kiểm soát lưu lượng thông qua Firewall Rules.

**Kết quả chức năng:** Máy trạm tại vùng USER có thể kết nối đến các dịch vụ nội bộ được cho phép tại vùng SERVER.

#### 4.2. Quản lý danh tính tập trung

DC01 được triển khai với Active Directory Domain Services và domain `corp.local`.

Hệ thống hỗ trợ quản lý tập trung:

- Tài khoản người dùng.
- Tài khoản máy tính.
- Organizational Units.
- Security Groups.
- Xác thực đăng nhập Domain.

**Kết quả chức năng:** Người dùng có thể đăng nhập vào máy trạm bằng tài khoản Domain, thay vì phải quản lý tài khoản cục bộ riêng biệt trên từng máy.

#### 4.3. Dịch vụ DNS và DHCP

DC01 cung cấp dịch vụ DNS phục vụ phân giải tên trong Domain và DHCP để cấp phát cấu hình IP cho máy trạm.

DHCP Relay trên pfSense cho phép máy trạm ở vùng USER nhận địa chỉ IP từ DHCP Server đặt tại vùng SERVER.

**Kết quả chức năng:** Máy trạm nhận được cấu hình mạng và có thể phân giải tên các tài nguyên nội bộ.

#### 4.4. Quản lý cấu hình bằng Group Policy

Group Policy được sử dụng để quản lý một số chính sách và cấu hình dành cho người dùng, máy tính trong Domain.

Các chức năng thực hành bao gồm áp dụng chính sách theo Organizational Unit và ánh xạ ổ đĩa mạng.

**Kết quả chức năng:** Một số cấu hình và tài nguyên có thể được quản lý tập trung thông qua Active Directory.

#### 4.5. Hệ thống File Server tập trung

FS01 cung cấp dịch vụ chia sẻ dữ liệu nội bộ thông qua SMB.

Các tài nguyên được tổ chức theo phòng ban:

- Accounting.
- HR.
- IT.
- Public.

Quyền truy cập được quản lý thông qua Security Groups, Share Permissions và NTFS Permissions.

Ngoài ra, dự án nghiên cứu và thực hành các chức năng quản trị lưu trữ như FSRM và Shadow Copies.

**Kết quả chức năng:** Người dùng được phép truy cập tài nguyên phù hợp với nhóm quyền của mình, đồng thời bị hạn chế truy cập các tài nguyên không được cấp quyền.

#### 4.6. Hạ tầng dịch vụ Linux

LINUX01 được triển khai với Ubuntu Server 22.04.5 LTS, bổ sung môi trường quản trị Linux vào hệ thống.

Các nhóm chức năng được đưa vào phạm vi dự án gồm:

- Quản trị hệ điều hành Ubuntu.
- Quản trị từ xa qua SSH.
- Dịch vụ Web với Nginx.
- Chia sẻ dữ liệu bằng Samba.
- Quản trị cơ sở dữ liệu PostgreSQL.
- Quản lý lưu trữ bằng LVM.
- Giám sát hạ tầng với Zabbix.
- Kiểm tra dịch vụ và xử lý sự cố Linux.

**Kết quả chức năng:** Hệ thống có môi trường Linux để triển khai và quản trị các dịch vụ bổ trợ. Trạng thái hoạt động cụ thể của từng dịch vụ cần được xác nhận bằng kết quả kiểm thử thực tế.

---

### 5. Khả năng tích hợp giữa các thành phần

Giá trị kỹ thuật quan trọng của dự án nằm ở sự phối hợp giữa nhiều dịch vụ hạ tầng.

Ví dụ, khi nhân viên sử dụng máy trạm tại vùng USER để truy cập File Server:

1. Máy trạm nhận cấu hình mạng thông qua DHCP.
2. pfSense cung cấp khả năng định tuyến đến vùng SERVER.
3. DNS phân giải tên máy chủ và các dịch vụ Domain.
4. DC01 thực hiện xác thực tài khoản Domain.
5. Group Policy áp dụng các cấu hình phù hợp.
6. FS01 kiểm tra quyền truy cập và cung cấp tài nguyên được phép sử dụng.

Quy trình này cho thấy các thành phần Networking, Active Directory, DNS, DHCP, Group Policy và File Server không hoạt động độc lập mà có quan hệ phụ thuộc với nhau.

---

### 6. Kết quả kiểm thử

Các chức năng đã được kiểm tra trong quá trình thực hành gồm:

| Thành phần | Nội dung kiểm thử |
|---|---|
| Networking | Kết nối và định tuyến giữa các vùng mạng liên quan |
| Active Directory | Domain Controller Discovery, Domain Client |
| DNS | Phân giải tên Domain và máy chủ |
| DHCP | Cấp phát địa chỉ IP cho Client |
| Group Policy | Kiểm tra áp dụng chính sách và ánh xạ ổ đĩa |
| File Server | Truy cập, đọc/ghi và giới hạn quyền theo phòng ban |
| Linux Services | Kiểm tra hoạt động các dịch vụ riêng lẻ |
| Backup & Recovery | Thiết kế và kiểm tra khả năng phục hồi |

Các bằng chứng kiểm thử sẽ được tổ chức trong thư mục `evidence/` và tổng hợp tại `docs/07-integration-validation.md`.

---

### 7. Những vấn đề kỹ thuật cần giải quyết

Trong quá trình triển khai, dự án tập trung xử lý các vấn đề thường gặp trong quản trị hệ thống:

**Kết nối mạng:** Kiểm tra IP Addressing, Routing, Firewall Rules và khả năng truy cập dịch vụ giữa các subnet.

**Domain Services:** Xác minh DNS, Domain Controller Discovery, Domain Join và quá trình xử lý Group Policy.

**Quản lý quyền truy cập:** Phân biệt Network Connectivity, Share Permissions và NTFS Permissions khi xử lý lỗi truy cập File Server.

**Quản trị Linux:** Kiểm tra Network Configuration, System Services, File Permissions và Application Logs.

**Giới hạn tài nguyên:** Phân bổ RAM hợp lý và lựa chọn máy ảo cần chạy trong từng tình huống kiểm thử.

Các tình huống sự cố thực tế, nguyên nhân và cách xử lý được tổng hợp riêng trong tài liệu `08-engineering-challenges.md`.

---

### 8. Phạm vi và giới hạn

Dự án được xây dựng trong môi trường lab phục vụ học tập và phát triển kỹ năng thực hành, không phải hệ thống Production.

Các giới hạn hiện tại gồm:

- Máy Host có 16 GB RAM, giới hạn số lượng VM chạy đồng thời.
- Chưa triển khai High Availability.
- Chưa thực hiện Performance Testing ở quy mô doanh nghiệp thực tế.
- Chưa xác nhận hoàn chỉnh quy trình Disaster Recovery.
- Một số chức năng Linux cần bổ sung bằng chứng kiểm thử.
- Monitoring Coverage cần được xác minh theo trạng thái triển khai thực tế.

Những giới hạn này được ghi nhận nhằm phân biệt rõ giữa chức năng đã kiểm thử, chức năng đang hoàn thiện và định hướng mở rộng.

---

### 9. Kết quả đạt được và giá trị dự án

Project 1 thể hiện khả năng xây dựng và quản trị một môi trường hạ tầng IT doanh nghiệp mô phỏng với các thành phần chính:

- Hệ thống mạng phân vùng và định tuyến tập trung.
- Active Directory Domain.
- Dịch vụ DNS và DHCP.
- Máy trạm tham gia Domain.
- Quản lý cấu hình bằng Group Policy.
- File Server và phân quyền theo phòng ban.
- Môi trường quản trị Linux và các dịch vụ bổ trợ.
- Kiểm thử tích hợp và xử lý sự cố.

Thông qua dự án, các kiến thức về Networking, Windows Server và Linux được áp dụng trong một hệ thống thống nhất, giúp phát triển tư duy thiết kế kiến trúc, quản trị dịch vụ và phân tích sự cố hạ tầng.

---

### 10. Kết luận

**Enterprise IT Infrastructure & System Administration Lab** là dự án thực hành triển khai hệ thống hạ tầng IT doanh nghiệp trên nền tảng ảo hóa VMware Workstation.

Trọng tâm của dự án không chỉ là cài đặt các công nghệ riêng lẻ mà còn là thiết kế cách chúng phối hợp, đáp ứng các yêu cầu quản trị và cung cấp dịch vụ nội bộ.

Dự án được xây dựng theo định hướng **thiết kế kiến trúc → triển khai chức năng → tích hợp hệ thống → kiểm thử → đánh giá kết quả**, làm cơ sở thể hiện năng lực thực hành trong lĩnh vực System Administration và IT Infrastructure Engineering.