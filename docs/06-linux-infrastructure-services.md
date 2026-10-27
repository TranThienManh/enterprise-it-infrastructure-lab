# 06 — Triển khai và quản trị hạ tầng dịch vụ Linux

## 1. Tổng quan

Trong môi trường doanh nghiệp, hệ thống Windows Server thường được sử dụng để quản lý danh tính, máy trạm và tài nguyên nội bộ. Bên cạnh đó, Linux Server đóng vai trò quan trọng trong việc vận hành Web Server, Database Server, File Services, Monitoring và các ứng dụng nội bộ.

Project triển khai máy chủ **LINUX01 sử dụng Ubuntu Server 22.04.5 LTS**, hoạt động trong cùng hạ tầng mạng với Domain Controller và Windows File Server.

Mục tiêu của việc bổ sung LINUX01 không chỉ là thực hành quản trị hệ điều hành Linux mà còn xây dựng một môi trường hạ tầng hỗn hợp **Windows–Linux**, trong đó các dịch vụ có thể được truy cập, quản lý và kiểm thử thông qua hệ thống mạng tập trung.

Các công nghệ thuộc phạm vi triển khai gồm:

- Ubuntu Server Administration.
- SSH Remote Administration.
- Nginx Web Server.
- Samba File Sharing.
- PostgreSQL Database Server.
- Logical Volume Manager (LVM).
- Zabbix Monitoring.
- Linux Security Hardening.

---

## 2. Mục tiêu triển khai

| Nhóm chức năng | Mục tiêu | Công nghệ |
|---|---|---|
| Quản trị hệ điều hành | Quản lý máy chủ, tài khoản và dịch vụ | Ubuntu Server, systemd |
| Quản trị từ xa | Truy cập và quản lý máy chủ qua mạng | OpenSSH |
| Web Services | Cung cấp dịch vụ HTTP nội bộ | Nginx |
| File Sharing | Chia sẻ dữ liệu với các thiết bị trong mạng | Samba |
| Database Services | Cung cấp hệ quản trị cơ sở dữ liệu quan hệ | PostgreSQL |
| Storage Management | Quản lý dung lượng lưu trữ linh hoạt | LVM |
| Infrastructure Monitoring | Theo dõi trạng thái hệ thống và dịch vụ | Zabbix |
| System Security | Kiểm soát truy cập và giảm rủi ro cấu hình | Linux Permissions, Firewall, SSH Security |

Các chức năng này được lựa chọn nhằm mô phỏng những nhiệm vụ thường gặp của System Administrator và IT Infrastructure Engineer.

---

## 3. Kiến trúc Linux Infrastructure

### 3.1. Thông tin máy chủ

| Thuộc tính | Giá trị |
|---|---|
| Hostname | LINUX01 |
| Operating System | Ubuntu Server 22.04.5 LTS |
| IP Address | `10.10.10.30` |
| Network | SERVER |
| Subnet | `10.10.10.0/24` |
| Default Gateway | `10.10.10.1` |
| Network Infrastructure | pfSense |
| Vai trò | Linux Infrastructure Services |

LINUX01 được đặt trong SERVER Network cùng với DC01 và FS01.

Điều này cho phép máy chủ Linux sử dụng hạ tầng mạng hiện có và giao tiếp với các thành phần khác theo chính sách Firewall.

### 3.2. Sơ đồ kiến trúc

```text
                       External Network
                              |
                           pfSense
                              |
                    SERVER NETWORK
                    10.10.10.0/24
                              |
          +-------------------+-------------------+
          |                   |                   |
          v                   v                   v
         DC01                FS01               LINUX01
    10.10.10.10         10.10.10.20         10.10.10.30
          |                   |                   |
       AD DS/DNS          Windows SMB          Ubuntu
                                              |
                    +-------------------------+------------------+
                    |           |             |                  |
                    v           v             v                  v
                  Nginx       Samba       PostgreSQL           Zabbix
                    |
               Web Services

                    Supporting Infrastructure
                    SSH | LVM | Security | Logs
```

**Hình 6.1 — Vị trí của LINUX01 trong hạ tầng Project.**

Sơ đồ thể hiện phạm vi chức năng của LINUX01. Nó không khẳng định tất cả các dịch vụ đang hoạt động đồng thời hoặc đã được tích hợp với nhau.

### 3.3. Lý do sử dụng một Linux Server

Do giới hạn tài nguyên của máy Host, các chức năng Linux được tổ chức trên một máy ảo thay vì triển khai mỗi dịch vụ trên một máy riêng.

Ưu điểm:

- Giảm mức sử dụng RAM và CPU.
- Thuận tiện quản trị và thực hành.
- Cho phép nghiên cứu nhiều loại dịch vụ Linux.
- Đơn giản hóa kết nối trong môi trường lab.

Đánh đổi:

- Các dịch vụ chia sẻ tài nguyên trên cùng một hệ điều hành.
- Sự cố của LINUX01 có thể ảnh hưởng đến nhiều dịch vụ.
- Khó mô phỏng đầy đủ kiến trúc phân tách ứng dụng và cơ sở dữ liệu.
- Chưa có cơ chế dự phòng.

Đây là lựa chọn phù hợp với mục tiêu thực hành, nhưng không được xem là kiến trúc Production tối ưu cho mọi trường hợp.

---

## 4. Quản trị hệ điều hành Ubuntu Server

### 4.1. Mục đích

LINUX01 cung cấp môi trường để thực hành các hoạt động quản trị Linux Server, bao gồm:

- Quản lý người dùng và nhóm.
- Quản lý quyền truy cập tệp.
- Cài đặt và cập nhật phần mềm.
- Quản lý tiến trình và dịch vụ.
- Kiểm tra trạng thái tài nguyên.
- Quản lý cấu hình mạng.
- Phân tích nhật ký hệ thống.

### 4.2. Quản lý dịch vụ

Ubuntu sử dụng systemd để quản lý nhiều dịch vụ hệ thống.

Các dịch vụ như SSH, Nginx và PostgreSQL có thể được kiểm tra thông qua systemd và các công cụ quản trị tương ứng.

Trong môi trường doanh nghiệp, việc quản lý dịch vụ cần xác định:

- Dịch vụ có được cài đặt hay không.
- Dịch vụ có đang chạy hay không.
- Dịch vụ có được cấu hình tự khởi động hay không.
- Dịch vụ có đang lắng nghe trên cổng mạng phù hợp hay không.
- Dịch vụ có hoạt động đúng từ góc nhìn Client hay không.

Một dịch vụ ở trạng thái `active` không đồng nghĩa toàn bộ chức năng ứng dụng đã hoạt động đúng.

### 4.3. Kết quả và phạm vi xác minh

LINUX01 đã được tích hợp vào SERVER Network và kiểm tra kết nối với các thành phần hạ tầng liên quan.

Các kết quả kiểm tra riêng về CPU, RAM, Disk, System Services và Logs cần được tổng hợp thành Evidence để chứng minh trạng thái vận hành cụ thể.

---

## 5. SSH — Quản trị Linux từ xa

### 5.1. Bài toán

Người quản trị cần có khả năng truy cập Linux Server mà không phải thao tác trực tiếp trên giao diện Console của máy ảo.

SSH được sử dụng để cung cấp kênh quản trị từ xa có mã hóa.

### 5.2. Vai trò trong hệ thống

OpenSSH hỗ trợ:

- Remote Shell.
- Thực hiện tác vụ quản trị.
- Truyền tệp an toàn thông qua các công cụ phù hợp.
- Kiểm tra cấu hình và trạng thái dịch vụ.
- Quản lý máy chủ từ thiết bị quản trị.

### 5.3. Kiến trúc truy cập

```text
MGMT01
10.10.30.10
     |
     v
pfSense Firewall
     |
     v
LINUX01
10.10.10.30
     |
     v
SSH Service
     |
     v
Linux User Authentication
     |
     v
Authorized Shell Session
```

### 5.4. Nguyên tắc bảo mật

Các biện pháp cần xem xét:

- Hạn chế tài khoản được phép SSH.
- Sử dụng SSH Key Authentication khi phù hợp.
- Hạn chế đăng nhập trực tiếp bằng root.
- Giới hạn nguồn truy cập qua Firewall.
- Kiểm tra Authentication Logs.
- Cấp quyền sudo theo nhu cầu thực tế.

Các biện pháp này là tiêu chí thiết kế bảo mật. Không mặc định tất cả đã được áp dụng nếu chưa có cấu hình xác minh.

---

## 6. Nginx — Web Server nội bộ

### 6.1. Mục đích triển khai

Nginx được đưa vào hệ thống nhằm cung cấp môi trường thực hành triển khai và quản trị Web Services trên Linux.

Trong môi trường doanh nghiệp, Web Server có thể được sử dụng để:

- Cung cấp website nội bộ.
- Phục vụ nội dung tĩnh.
- Làm Reverse Proxy cho ứng dụng.
- Quản lý Virtual Hosts.
- Ghi nhận Access Logs và Error Logs.

### 6.2. Vai trò trong Project

LINUX01 đóng vai trò máy chủ lưu trữ dịch vụ Nginx.

Luồng truy cập cơ bản:

```text
Internal Client
      |
      v
pfSense Routing
      |
      v
LINUX01
10.10.10.30
      |
      v
Nginx
      |
      v
HTTP Response
```

### 6.3. Thiết kế chức năng

Các chức năng cần đánh giá:

| Chức năng | Tiêu chí kiểm thử |
|---|---|
| Web Server Availability | Nginx đang hoạt động |
| HTTP Connectivity | Client kết nối được đến dịch vụ |
| Content Delivery | Nội dung được trả về chính xác |
| Virtual Host | Hostname được ánh xạ đúng nếu có cấu hình |
| Logging | Access/Error Logs ghi nhận hoạt động |
| Firewall | Chỉ những nguồn được phép mới truy cập được |

### 6.4. Trạng thái kiểm chứng

Nginx nằm trong phạm vi thực hành Linux Infrastructure.

Để xác nhận dịch vụ đã triển khai thành công, cần bổ sung:

- Trạng thái dịch vụ.
- Cổng lắng nghe.
- Cấu hình Server Block.
- Kết quả HTTP Request từ Client.
- Log tương ứng.

Không mặc định hệ thống đã triển khai HTTPS, Reverse Proxy hoặc Public Web Hosting nếu chưa có bằng chứng.

---

## 7. Samba — Chia sẻ dữ liệu giữa các hệ điều hành

### 7.1. Mục đích

Samba cung cấp khả năng chia sẻ tài nguyên sử dụng giao thức SMB trên Linux.

Việc triển khai Samba cho phép thực hành File Sharing trong môi trường có cả Windows và Linux.

### 7.2. Phân biệt Samba và Windows File Server

| Tiêu chí | FS01 | LINUX01 |
|---|---|---|
| Hệ điều hành | Windows Server | Ubuntu Server |
| Công nghệ chia sẻ | Windows SMB Server | Samba |
| Mục đích chính | File Services theo phòng ban | Thực hành SMB trên Linux |
| Quản lý quyền | Share/NTFS Permissions | Linux Permissions, Samba Configuration |
| Domain Integration | Thành viên `corp.local` | Chưa xác nhận Domain Join |

FS01 vẫn là File Server chính phục vụ tài nguyên phòng ban của dự án.

Samba trên LINUX01 là một thành phần thực hành riêng, không thay thế FS01.

### 7.3. Kiến trúc chức năng

```text
Windows / Linux Client
          |
          v
      SMB Protocol
          |
          v
       LINUX01
          |
          v
      Samba Server
          |
          v
  Linux Filesystem Permissions
          |
          v
      Shared Directory
```

### 7.4. Các yêu cầu kiểm thử

- Samba Service hoạt động.
- Share được khai báo hợp lệ.
- Client có thể kết nối qua mạng.
- Người dùng được xác thực theo cơ chế đã cấu hình.
- Quyền đọc/ghi được thực thi chính xác.
- Truy cập không được cấp quyền bị từ chối.

### 7.5. Trạng thái kiểm chứng

Cần bổ sung cấu hình Share, tài khoản truy cập và kết quả kiểm thử từ Client trước khi xác nhận đầy đủ chức năng Samba.

Chưa có cơ sở để khẳng định Samba đang sử dụng Active Directory Authentication.

---

## 8. PostgreSQL — Database Services

### 8.1. Mục đích triển khai

PostgreSQL được đưa vào phạm vi Linux Infrastructure để thực hành quản trị một hệ quản trị cơ sở dữ liệu quan hệ trên máy chủ Linux.

Trong hệ thống doanh nghiệp, Database Server cung cấp khả năng lưu trữ và truy vấn dữ liệu cho các ứng dụng.

### 8.2. Vai trò trong kiến trúc

Các chức năng quản trị liên quan:

- Quản lý PostgreSQL Service.
- Tạo và quản lý Database.
- Quản lý Database Roles.
- Kiểm soát Authentication.
- Cấu hình kết nối.
- Kiểm tra trạng thái và nhật ký dịch vụ.

### 8.3. Kiến trúc dịch vụ

```text
Application / Database Client
             |
             v
       Network Access
             |
             v
          LINUX01
             |
             v
         PostgreSQL
             |
             v
       Database Storage
```

Sơ đồ thể hiện kiến trúc logic của Database Service, không khẳng định hiện đã có ứng dụng cụ thể kết nối với PostgreSQL.

### 8.4. Kiểm soát truy cập

Các cơ chế cần xem xét:

- Database Roles và Privileges.
- Client Authentication.
- Network Listening Configuration.
- `pg_hba.conf`.
- Firewall Rules.
- Giới hạn quyền truy cập theo nhu cầu.

Việc cài đặt PostgreSQL không đồng nghĩa Database được phép truy cập từ mọi subnet.

### 8.5. Trạng thái kiểm chứng

Để xác nhận PostgreSQL đã triển khai thành công, cần bổ sung bằng chứng:

- PostgreSQL Service đang chạy.
- Kết nối Database thành công.
- Database và Role được tạo.
- Truy vấn SQL được thực hiện.
- Quyền truy cập được kiểm tra.

Chưa xác nhận Database Replication, Automated Backup hoặc High Availability.

---

## 9. LVM — Quản lý lưu trữ Linux

### 9.1. Bài toán

Trong môi trường máy chủ, nhu cầu lưu trữ có thể thay đổi theo thời gian.

Việc quản lý phân vùng theo cách cố định có thể gây khó khăn khi cần mở rộng dung lượng.

Logical Volume Manager (LVM) cung cấp cơ chế quản lý lưu trữ linh hoạt thông qua các lớp logic.

### 9.2. Kiến trúc LVM

```text
Physical Disk
      |
      v
Physical Volume (PV)
      |
      v
Volume Group (VG)
      |
      v
Logical Volume (LV)
      |
      v
Filesystem
      |
      v
Mount Point
```

### 9.3. Vai trò từng thành phần

| Thành phần | Chức năng |
|---|---|
| Physical Volume | Thiết bị hoặc phân vùng được quản lý bởi LVM |
| Volume Group | Tập hợp dung lượng từ các Physical Volumes |
| Logical Volume | Đơn vị lưu trữ logic được cấp từ Volume Group |
| Filesystem | Hệ thống tệp được tạo trên Logical Volume |
| Mount Point | Vị trí truy cập dữ liệu trong Linux |

### 9.4. Mục tiêu thực hành

- Xác định các thiết bị lưu trữ.
- Quản lý Physical Volumes.
- Tổ chức Volume Groups.
- Tạo Logical Volumes.
- Định dạng và Mount Filesystems.
- Kiểm tra khả năng mở rộng dung lượng.

### 9.5. Giá trị kỹ thuật

LVM giúp tách cấu trúc lưu trữ logic khỏi bố cục phân vùng vật lý, hỗ trợ quản lý dung lượng linh hoạt hơn.

Tuy nhiên, LVM không tự động cung cấp RAID, Backup hoặc khả năng chống mất dữ liệu khi ổ đĩa hỏng.

### 9.6. Trạng thái kiểm chứng

Cần bổ sung kết quả kiểm tra PV, VG, LV, Filesystem và Mount Point để xác nhận cấu trúc LVM thực tế trên LINUX01.

---

## 10. Zabbix — Giám sát hạ tầng

### 10.1. Mục đích

Giám sát là một phần quan trọng trong quản trị hạ tầng vì người quản trị cần biết trạng thái hoạt động của máy chủ và dịch vụ.

Zabbix được đưa vào phạm vi Project nhằm nghiên cứu và thực hành giám sát hạ tầng.

### 10.2. Các chỉ số giám sát

Những nhóm thông tin phù hợp gồm:

| Nhóm | Chỉ số |
|---|---|
| CPU | CPU Utilization, Load |
| Memory | RAM Usage |
| Storage | Disk Usage, Free Space |
| Network | Interface Traffic, Connectivity |
| Services | Trạng thái dịch vụ |
| Availability | Host Reachability |

### 10.3. Kiến trúc giám sát

```text
Monitored Host / Service
          |
          v
    Metrics Collection
          |
          v
        Zabbix
          |
          v
   Monitoring Dashboard
          |
          v
   Alerts / Investigation
```

Đây là kiến trúc chức năng tham chiếu. Mô hình Zabbix Server, Agent, Database và các Host được giám sát cần được xác nhận từ cấu hình thực tế.

### 10.4. Giá trị triển khai

Một hệ thống Monitoring phù hợp có thể giúp:

- Phát hiện tình trạng sử dụng tài nguyên bất thường.
- Theo dõi khả năng hoạt động của dịch vụ.
- Hỗ trợ phân tích nguyên nhân sự cố.
- Cung cấp dữ liệu phục vụ quản trị dung lượng.

### 10.5. Trạng thái kiểm chứng

Zabbix thuộc phạm vi thực hành Linux Infrastructure.

Chưa có đủ bằng chứng để khẳng định:

- Zabbix Server đã vận hành hoàn chỉnh.
- Các Host Windows/Linux đều được giám sát.
- Dashboard đã hiển thị đầy đủ Metrics.
- Trigger và Alert đã được kiểm thử.

Các chức năng này chỉ được đánh dấu hoàn thành sau khi có Evidence tương ứng.

---

## 11. Linux Security và quản lý quyền truy cập

### 11.1. Mục tiêu

Máy chủ Linux cần được cấu hình để giảm các rủi ro liên quan đến truy cập trái phép và cấu hình dịch vụ không phù hợp.

Các nội dung thuộc phạm vi bảo mật:

- Linux Users và Groups.
- File Ownership.
- File Permissions.
- sudo Privileges.
- SSH Access Control.
- Host Firewall.
- Service Exposure.
- System Updates.
- System Logs.

### 11.2. Phân quyền tệp

Linux sử dụng các quyền cơ bản:

| Quyền | Ký hiệu | Ý nghĩa |
|---|---|---|
| Read | `r` | Đọc nội dung |
| Write | `w` | Ghi hoặc thay đổi nội dung |
| Execute | `x` | Thực thi hoặc truy cập thư mục theo ngữ cảnh |

Quyền được đánh giá đối với Owner, Group và Others, đồng thời có thể chịu ảnh hưởng của ACLs và các cơ chế bảo mật bổ sung.

### 11.3. Kiểm soát dịch vụ mạng

Một dịch vụ chỉ nên lắng nghe trên những interface và cổng cần thiết.

Các lớp kiểm soát gồm:

1. Cấu hình ứng dụng.
2. Linux Host Firewall.
3. pfSense Firewall.
4. Authentication và Authorization.

Các lớp này có mục đích khác nhau và cần được kiểm tra độc lập.

### 11.4. Giới hạn

Chưa có bằng chứng về một cuộc Security Audit toàn diện hoặc việc tuân thủ đầy đủ tiêu chuẩn hardening cụ thể như CIS Benchmark.

Vì vậy, tài liệu chỉ mô tả phạm vi quản trị bảo mật, không khẳng định hệ thống đã đạt chuẩn Production Security.

---

## 12. Tích hợp Linux với hạ tầng Windows

### 12.1. Mục tiêu tích hợp

LINUX01 hoạt động trong cùng SERVER Network với DC01 và FS01.

Mục tiêu là xây dựng một môi trường hỗn hợp, trong đó Windows và Linux có thể sử dụng chung hạ tầng mạng và các cơ chế kết nối.

### 12.2. Các luồng kết nối

| Nguồn | Đích | Mục đích |
|---|---|---|
| LINUX01 | pfSense | Default Gateway |
| LINUX01 | DC01 | Network Connectivity và dịch vụ liên quan |
| LINUX01 | FS01 | Network Connectivity |
| MGMT01 | LINUX01 | SSH Administration, nếu được cấu hình |
| USER01 | LINUX01 | Truy cập Web/SMB, nếu dịch vụ được cung cấp |

Các luồng ở trạng thái thiết kế cần được phân biệt với kết quả kiểm thử thực tế.

### 12.3. Kết quả đã ghi nhận

Trong quá trình triển khai mạng, LINUX01 đã được kiểm tra khả năng kết nối đến DC01 và FS01.

Một sự cố liên quan đến ARP/Neighbor Cache đã được xử lý trong quá trình điều chỉnh cấu hình mạng.

Điều này chứng minh LINUX01 đã tham gia hạ tầng mạng nội bộ và có khả năng giao tiếp với các máy chủ liên quan.

Tuy nhiên, kết nối IP không chứng minh rằng toàn bộ các dịch vụ ứng dụng trên LINUX01 đã hoạt động.

---

## 13. Kiểm thử và đánh giá chức năng

### 13.1. Ma trận kiểm thử

| Test ID | Thành phần | Tiêu chí |
|---|---|---|
| LNX-01 | Network | Kết nối đến Gateway |
| LNX-02 | Network | Kết nối đến DC01 |
| LNX-03 | Network | Kết nối đến FS01 |
| LNX-04 | SSH | Đăng nhập từ máy quản trị |
| LNX-05 | Nginx | HTTP Response thành công |
| LNX-06 | Samba | Client truy cập Share |
| LNX-07 | PostgreSQL | Kết nối và thực hiện truy vấn |
| LNX-08 | LVM | Xác minh PV/VG/LV và Filesystem |
| LNX-09 | Zabbix | Thu thập và hiển thị Metrics |
| LNX-10 | Security | Xác minh Permissions và Firewall |

### 13.2. Phương pháp đánh giá

Mỗi dịch vụ cần được kiểm tra theo ba cấp độ:

**Cấp độ 1 — Deployment**

Xác nhận phần mềm hoặc thành phần hệ thống đã được cài đặt và cấu hình.

**Cấp độ 2 — Service Availability**

Xác nhận dịch vụ đang hoạt động, có thể tiếp nhận yêu cầu và không phát sinh lỗi cấu hình nghiêm trọng.

**Cấp độ 3 — Functional Validation**

Xác nhận người dùng hoặc ứng dụng thực hiện được chức năng mong muốn từ một Client phù hợp.

Một dịch vụ chỉ được xem là hoàn thành kiểm thử chức năng khi đạt cấp độ 3 với bằng chứng cụ thể.

---

## 14. Những vấn đề kỹ thuật trong quá trình triển khai

### 14.1. Kết nối đến Gateway

**Hiện tượng:** LINUX01 từng gặp lỗi kết nối đến Gateway sau khi điều chỉnh môi trường mạng.

**Hướng phân tích:** Kiểm tra Interface, IP Address, Routing Table và ARP/Neighbor Cache.

**Kết quả:** Kết nối được khôi phục sau khi xử lý thông tin Neighbor Cache không còn phù hợp.

**Bài học:** Cần kiểm tra đồng thời các thành phần Layer 2 và Layer 3 khi xử lý sự cố mạng Linux.

### 14.2. Phân biệt Network Connectivity và Service Availability

Một máy chủ phản hồi Ping không đồng nghĩa Nginx, PostgreSQL hoặc Samba đang hoạt động.

Việc đánh giá cần kiểm tra riêng:

- Host Reachability.
- TCP Connectivity.
- Service Status.
- Application Response.
- Authentication và Authorization.

### 14.3. Phân quyền trên Linux

Quyền truy cập tài nguyên có thể bị ảnh hưởng bởi Linux File Permissions, Ownership, ACLs và cấu hình ứng dụng.

Khi xử lý lỗi Samba hoặc Web Server, cần kiểm tra quyền hệ thống tệp cùng với quyền được khai báo trong dịch vụ.

---

## 16. Đánh giá kiến trúc Linux

### 16.1. Ưu điểm

**Đa nền tảng:** Bổ sung môi trường Linux vào hạ tầng Windows Server.

**Tích hợp mạng:** LINUX01 sử dụng hệ thống Routing và Firewall hiện có.

**Đa dạng dịch vụ:** Hỗ trợ thực hành Web, File Sharing, Database, Storage và Monitoring.

**Tối ưu tài nguyên:** Nhiều nhóm chức năng được nghiên cứu trên một máy ảo.

**Khả năng mở rộng kiến thức:** Tạo nền tảng để nghiên cứu Automation, Service Monitoring và Linux Security.

### 16.2. Giới hạn

- Một Linux Server đảm nhiệm nhiều nhóm chức năng.
- Chưa có High Availability.
- Chưa xác nhận đầy đủ tình trạng hoạt động của từng dịch vụ.
- Chưa có kết quả Performance Testing.
- Chưa có bằng chứng đầy đủ về Backup và Recovery.
- Chưa xác nhận tích hợp Linux Authentication với Active Directory.
- Chưa có đánh giá Security Hardening toàn diện.

---

## 17. Kết luận

Project bổ sung LINUX01 nhằm xây dựng môi trường hạ tầng hỗn hợp Windows–Linux và mở rộng phạm vi thực hành quản trị hệ thống.

LINUX01 được bố trí trong SERVER Network và đã tham gia các bài kiểm thử kết nối với những máy chủ nội bộ.

Các công nghệ SSH, Nginx, Samba, PostgreSQL, LVM và Zabbix tạo thành phạm vi nghiên cứu và triển khai các dịch vụ Linux quan trọng trong môi trường doanh nghiệp.

Giá trị của thành phần này nằm ở khả năng **quản trị Linux Server, tổ chức dịch vụ, kiểm soát truy cập, đánh giá trạng thái hoạt động và xử lý sự cố trong một hạ tầng mạng thống nhất**.

Các chức năng chưa có kết quả kiểm thử đầy đủ sẽ được tiếp tục xác minh và bổ sung Evidence, bảo đảm tài liệu phản ánh chính xác mức độ hoàn thành của dự án.