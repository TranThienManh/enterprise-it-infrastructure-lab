# 03 — Triển khai hạ tầng mạng doanh nghiệp

## 1. Tổng quan

Hạ tầng mạng là thành phần nền tảng của dự án **Enterprise IT Infrastructure & System Administration Lab**, đảm bảo các máy chủ, máy trạm và dịch vụ nội bộ có thể giao tiếp với nhau.

Hệ thống được triển khai trên VMware Workstation, sử dụng **pfSense CE 2.5.2** làm Router và Firewall trung tâm.

Kiến trúc mạng được phân chia thành ba vùng chức năng:

- **SERVER:** Chứa các máy chủ cung cấp dịch vụ.
- **USER:** Chứa máy trạm của người dùng.
- **MGMT:** Chứa máy trạm phục vụ quản trị.

Việc phân chia này nhằm mô phỏng cách tổ chức mạng trong doanh nghiệp, đồng thời tạo môi trường thực hành Routing, Firewall Access Control, DHCP Relay và Troubleshooting.

---

## 2. Yêu cầu và mục tiêu triển khai

Hệ thống mạng cần đáp ứng các yêu cầu sau:

| Yêu cầu | Mục tiêu triển khai |
|---|---|
| Phân vùng mạng | Tách máy chủ, máy trạm và thiết bị quản trị thành các subnet |
| Kết nối liên mạng | Cho phép các subnet giao tiếp qua Router |
| Kiểm soát truy cập | Quản lý lưu lượng bằng Firewall Rules |
| Cấp phát IP tập trung | Cho phép Client nhận IP từ DHCP Server tại DC01 |
| Truy cập dịch vụ nội bộ | Hỗ trợ DNS, Domain Authentication, Group Policy và SMB |
| Quản trị hệ thống | Cung cấp kết nối từ MGMT Network đến các dịch vụ quản trị được phép |
| Xử lý sự cố | Xác định lỗi tại các lớp IP, Routing, Firewall và Application |

**Mục tiêu cuối cùng:** Các máy trạm có thể sử dụng dịch vụ hạ tầng tại SERVER Network mà không cần đặt tất cả thiết bị trong cùng một subnet.

---

## 3. Kiến trúc mạng

### 3.1. Sơ đồ logic

```text
                         External Network
                                |
                                |
                         WAN (Bridged)
                                |
                         +-------------+
                         |   pfSense   |
                         | CE 2.5.2    |
                         +-------------+
                                |
             +------------------+------------------+
             |                  |                  |
             |                  |                  |
         SERVER               USER                MGMT
      10.10.10.0/24       10.10.20.0/24       10.10.30.0/24
             |                  |                  |
      GW 10.10.10.1      GW 10.10.20.1      GW 10.10.30.1
             |                  |                  |
       +-----+-----+            |                  |
       |     |     |            |                  |
      DC01  FS01 LINUX01      USER01             MGMT01
       |     |     |            |                  |
      .10   .20   .30          .102                .10
```

*Hình 3.1 — Sơ đồ logic hạ tầng mạng Project 1.*

### 3.2. Phân hoạch địa chỉ IP

| Vùng mạng | Network Address | Gateway | Thiết bị |
|---|---|---|---|
| SERVER | `10.10.10.0/24` | `10.10.10.1` | DC01, FS01, LINUX01 |
| USER | `10.10.20.0/24` | `10.10.20.1` | USER01 |
| MGMT | `10.10.30.0/24` | `10.10.30.1` | MGMT01 |

### 3.3. Địa chỉ các máy chủ và máy trạm

| Hostname | IP Address | Network | Vai trò |
|---|---|---|---|
| DC01 | `10.10.10.10` | SERVER | AD DS, DNS, DHCP |
| FS01 | `10.10.10.20` | SERVER | File Server |
| LINUX01 | `10.10.10.30` | SERVER | Linux Services |
| USER01 | `10.10.20.102` | USER | Domain Client |
| MGMT01 | `10.10.30.10` | MGMT | Administration Client |

Các địa chỉ IP máy chủ được thiết kế ổn định để thuận tiện cho việc cung cấp dịch vụ và phân giải tên.

---

## 4. pfSense — Router và Firewall trung tâm

### 4.1. Vai trò

pfSense được sử dụng để kết nối và kiểm soát lưu lượng giữa các subnet.

Các chức năng chính trong dự án:

- Default Gateway.
- Inter-Subnet Routing.
- Firewall Rules.
- DHCP Relay.
- Kết nối ra mạng bên ngoài thông qua WAN.

### 4.2. Cấu hình giao diện mạng

| Interface | IP Address | Vai trò |
|---|---|---|
| WAN | DHCP từ mạng ngoài | External Connectivity |
| `em2` | `10.10.10.1/24` | SERVER Gateway |
| `em3` | `10.10.20.1/24` | USER Gateway |
| `em4` | `10.10.30.1/24` | MGMT Gateway |

WAN sử dụng chế độ Bridged trong VMware Workstation.

Ba interface nội bộ được gắn với các mạng ảo tương ứng, giúp phân tách lưu lượng giữa SERVER, USER và MGMT.

### 4.3. Lý do sử dụng pfSense

pfSense được lựa chọn vì cung cấp các chức năng định tuyến, Firewall và DHCP Relay trong cùng một hệ thống.

Trong môi trường lab, điều này giúp:

- Giảm nhu cầu triển khai nhiều thiết bị mạng riêng biệt.
- Quản lý các subnet từ một điểm tập trung.
- Kiểm thử Firewall Rules và Network Connectivity.
- Mô phỏng cách tổ chức mạng có phân vùng chức năng.

**Giới hạn thiết kế:** pfSense là Router/Firewall duy nhất của hệ thống, chưa có High Availability hoặc Failover.

---

## 5. Network Segmentation

### 5.1. SERVER Network

SERVER Network sử dụng dải `10.10.10.0/24`.

Đây là vùng mạng chứa các dịch vụ cốt lõi của hệ thống.

| Server | Chức năng |
|---|---|
| DC01 | Domain Authentication, DNS, DHCP |
| FS01 | SMB File Sharing |
| LINUX01 | Linux Infrastructure Services |

Các dịch vụ trong SERVER Network được truy cập từ những vùng mạng khác thông qua pfSense.

### 5.2. USER Network

USER Network sử dụng dải `10.10.20.0/24`.

USER01 đại diện cho máy trạm nhân viên, sử dụng các dịch vụ tại SERVER Network.

Các nhu cầu truy cập chính:

- DNS Resolution.
- Domain Authentication.
- Group Policy Processing.
- SMB File Sharing.
- DHCP Address Assignment.

### 5.3. MGMT Network

MGMT Network sử dụng dải `10.10.30.0/24`.

MGMT01 đại diện cho máy trạm quản trị, sử dụng RSAT và các công cụ quản trị hệ thống.

Mục đích của vùng mạng này là phân tách hoạt động quản trị khỏi lưu lượng sử dụng dịch vụ thông thường.

Việc phân tách mạng không thay thế cơ chế xác thực, phân quyền hoặc Firewall Rules.

### 5.4. Phân biệt Network Segmentation và VLAN

Dự án sử dụng ba subnet và các giao diện mạng ảo riêng trên pfSense.

Đây là hình thức phân vùng mạng ở mức IP và mạng ảo.

Không mặc định hệ thống đã triển khai VLAN Tagging theo chuẩn IEEE 802.1Q trên Switch vật lý.

Trong tài liệu, thuật ngữ **Inter-Subnet Routing** được sử dụng để mô tả chính xác chức năng đã triển khai.

---

## 6. Inter-Subnet Routing

### 6.1. Mục đích

Các máy trong những subnet khác nhau không thể giao tiếp trực tiếp ở Layer 2 nếu không có cơ chế định tuyến phù hợp.

pfSense cung cấp khả năng định tuyến giữa các subnet được kết nối trực tiếp.

Ví dụ:

- USER01 truy cập DC01.
- USER01 truy cập FS01.
- LINUX01 giao tiếp với các dịch vụ cần thiết trong SERVER Network.
- MGMT01 truy cập các máy chủ được cho phép.

### 6.2. Luồng kết nối USER01 đến FS01

```text
USER01
10.10.20.102
      |
      v
Default Gateway
10.10.20.1
      |
      v
pfSense Routing
      |
      v
SERVER Network
10.10.10.0/24
      |
      v
FS01
10.10.10.20
      |
      v
SMB Service
TCP 445
```

Khi USER01 truy cập FS01, lưu lượng được gửi đến Default Gateway tại `10.10.20.1`.

pfSense kiểm tra Firewall Rules và định tuyến gói tin đến SERVER Network nếu lưu lượng được cho phép.

FS01 tiếp nhận yêu cầu tại dịch vụ tương ứng.

Việc kết nối TCP thành công không đồng nghĩa người dùng đã được cấp quyền truy cập tài nguyên SMB.

### 6.3. Kết quả kiểm thử

Trong quá trình thực hành, các chức năng sau đã được kiểm tra:

- Kết nối giữa USER Network và SERVER Network.
- Domain Controller Discovery.
- Truy cập SYSVOL và NETLOGON.
- Truy cập File Server.
- Kết nối từ LINUX01 đến DC01 và FS01.

Các kết quả này chứng minh khả năng kết nối của những luồng đã kiểm thử, không đồng nghĩa mọi giao thức đều được phép giữa tất cả các subnet.

---

## 7. DHCP Relay và cấp phát địa chỉ IP

### 7.1. Bài toán triển khai

DHCP Server được đặt trên DC01 tại địa chỉ `10.10.10.10`.

Trong khi đó, USER01 thuộc USER Network `10.10.20.0/24`.

DHCP Discovery ban đầu được truyền bằng Broadcast trong mạng cục bộ và thông thường không được Router chuyển tiếp sang subnet khác.

Do đó, hệ thống cần DHCP Relay để chuyển tiếp yêu cầu đến DHCP Server.

### 7.2. Giải pháp

pfSense đảm nhiệm chức năng DHCP Relay từ mạng USER đến DC01.

```text
USER01
10.10.20.0/24
      |
      | DHCP Request
      v
pfSense
DHCP Relay
      |
      | Forward Request
      v
DC01
10.10.10.10
DHCP Server
      |
      | DHCP Response
      v
pfSense
      |
      v
USER01
```

### 7.3. Kết quả chức năng

Máy trạm tại USER Network có thể nhận cấu hình IP từ DHCP Server tập trung trên DC01.

Cấu hình này cho phép quản trị DHCP tập trung mà không cần đặt một DHCP Server riêng tại từng subnet.

Việc xác minh được thực hiện thông qua DHCP Lease, thông tin IP trên Client và khả năng kết nối sau khi nhận địa chỉ.

---

## 8. Firewall Access Control

### 8.1. Mục đích

Việc có Routing giữa các subnet không đồng nghĩa mọi lưu lượng phải được cho phép.

Firewall Rules được sử dụng để kiểm soát những kết nối có thể đi qua pfSense.

Các nhóm lưu lượng cần xem xét gồm:

| Nguồn | Đích | Nhu cầu truy cập |
|---|---|---|
| USER | DC01 | DNS, Authentication, Group Policy |
| USER | FS01 | SMB File Sharing |
| MGMT | Windows Servers | Remote Administration |
| MGMT | LINUX01 | SSH Administration |
| SERVER | External Network | Updates và dịch vụ bên ngoài khi cần |

Bảng này thể hiện nhu cầu thiết kế, không phải danh sách Firewall Rules đã được xác minh đang hoạt động.

### 8.2. Nguyên tắc kiểm soát truy cập

Firewall Policy được thiết kế theo nguyên tắc chỉ cho phép các luồng phục vụ chức năng cần thiết.

Tuy nhiên, trong môi trường lab, một số Rules có thể được mở rộng tạm thời để kiểm thử hoặc xử lý sự cố.

Vì vậy, cần phân biệt:

- **Required Traffic:** Lưu lượng cần thiết để dịch vụ hoạt động.
- **Configured Rules:** Các Rules đang được cấu hình thực tế.
- **Validated Traffic:** Các luồng đã kiểm thử thành công.

Ba nhóm này không mặc định tương đương nhau.

### 8.3. Firewall và quyền truy cập ứng dụng

Firewall kiểm soát khả năng kết nối mạng.

Quyền truy cập tài nguyên được kiểm tra bởi hệ điều hành và dịch vụ đích.

Ví dụ, USER01 có thể kết nối đến TCP 445 của FS01 nhưng vẫn bị từ chối truy cập thư mục HR nếu tài khoản không có quyền.

Điều này thể hiện sự khác biệt giữa:

**Network Access Control** và **Resource Authorization**.

---

## 9. Tích hợp mạng với Active Directory

Hệ thống mạng hỗ trợ các chức năng quan trọng của Active Directory.

### 9.1. DNS Resolution

Domain Client sử dụng DNS để tìm các dịch vụ liên quan đến `corp.local`.

Nếu DNS không được cấu hình đúng, máy trạm có thể không tìm thấy Domain Controller mặc dù vẫn kết nối được đến địa chỉ IP của máy chủ.

### 9.2. Domain Authentication

Quá trình xác thực Domain cần kết nối đến DC01 thông qua các giao thức và cổng dịch vụ phù hợp.

Firewall hoặc Routing không chính xác có thể ảnh hưởng đến đăng nhập và sử dụng dịch vụ Domain.

### 9.3. Group Policy

Group Policy phụ thuộc vào khả năng truy cập Domain Controller, DNS và các tài nguyên chính sách như SYSVOL.

Trong quá trình thực hành, USER01 đã được kiểm tra khả năng truy cập SYSVOL và NETLOGON.

### 9.4. File Server Access

USER01 truy cập FS01 thông qua SMB.

Khả năng truy cập thành công phụ thuộc vào:

1. Network Connectivity.
2. DNS Resolution.
3. Firewall Policy.
4. SMB Service.
5. Domain Authentication.
6. Share Permissions.
7. NTFS Permissions.

Vì vậy, lỗi truy cập File Server cần được phân tích ở cả lớp mạng và lớp phân quyền.

---

## 10. Các tình huống kỹ thuật đã xử lý

Trong quá trình triển khai, một số sự cố thực tế đã được ghi nhận.

### 10.1. DNS trỏ về địa chỉ cũ

**Hiện tượng:** Tên Domain hoặc máy chủ được phân giải về địa chỉ không còn phù hợp với kiến trúc mạng mới.

**Nguyên nhân liên quan:** Thông tin DNS cũ được lưu trong bộ nhớ đệm hoặc cấu hình phân giải chưa được cập nhật.

**Hướng xử lý:** Kiểm tra DNS Server, bản ghi DNS và xóa DNS Cache khi phù hợp.

**Bài học:** Kiểm tra khả năng phân giải tên trước khi kết luận lỗi Routing hoặc Firewall.

### 10.2. ICMP bị chặn

**Hiện tượng:** Một máy chủ không phản hồi Ping từ máy khác mặc dù các dịch vụ cần thiết vẫn có thể hoạt động.

**Nguyên nhân liên quan:** Windows Firewall hoặc Firewall Policy không cho phép ICMP.

**Bài học:** Ping thất bại không đủ để kết luận toàn bộ kết nối mạng bị gián đoạn.

### 10.3. ARP Cache không còn phù hợp

**Hiện tượng:** LINUX01 gặp lỗi kết nối đến Gateway trong quá trình điều chỉnh cấu hình mạng.

**Hướng xử lý:** Kiểm tra Interface, Gateway, Neighbor Table và làm mới ARP/Neighbor Cache khi cần.

**Bài học:** Sau khi thay đổi topology hoặc địa chỉ mạng, cần kiểm tra cả thông tin Layer 2 và Layer 3.

### 10.4. Firewall Rule ảnh hưởng đến File Sharing

**Hiện tượng:** Máy trạm có thể sử dụng một số dịch vụ Domain nhưng không truy cập được File Share.

**Hướng phân tích:** Kiểm tra TCP 445, Firewall Rules, SMB Service và Permissions.

**Bài học:** Cần xác định lỗi xảy ra ở lớp mạng hay lớp phân quyền trước khi thay đổi cấu hình.

---

## 11. Đánh giá kết quả triển khai

| Chức năng | Kết quả |
|---|---|
| Phân chia SERVER/USER/MGMT | Ba subnet được sử dụng trong lab |
| pfSense Routing | Kết nối giữa các mạng liên quan |
| DHCP Relay | USER Client nhận IP từ DHCP trên DC01 |
| DNS Connectivity | Client phân giải và tìm Domain Controller |
| Domain Services | SYSVOL/NETLOGON truy cập được |
| SMB Connectivity | Các luồng truy cập File Server cần thiết hoạt động |
| Firewall Least Privilege | Đối chiếu toàn bộ Rules với nhu cầu truy cập |
| MGMT Access Control | Xác nhận chính sách quản trị theo nguồn/đích |

## 12. Giới hạn của kiến trúc mạng

### Single Point of Failure

pfSense là Router/Firewall duy nhất. Nếu máy ảo pfSense ngừng hoạt động, các luồng cần định tuyến qua nó sẽ bị ảnh hưởng.

### Virtualized Networking

Hệ thống được triển khai trên mạng ảo VMware Workstation, chưa kiểm thử với Switch và Router vật lý.

### Firewall Policy

Chưa có kết quả đối chiếu đầy đủ giữa toàn bộ Firewall Rules đang hoạt động và nguyên tắc Least Privilege.

### Network Performance

Dự án chưa thực hiện các bài kiểm thử chuyên sâu về Throughput, Latency hoặc tải mạng ở quy mô doanh nghiệp.

---

## 13. Kết luận

Hạ tầng mạng Project 1 được xây dựng theo mô hình phân vùng chức năng, sử dụng pfSense làm trung tâm định tuyến và kiểm soát lưu lượng.

Ba vùng SERVER, USER và MGMT giúp tổ chức các thành phần theo vai trò và tạo nền tảng cho quản trị truy cập.

Các chức năng Routing, DHCP Relay và kết nối đến dịch vụ nội bộ đã được thực hành và kiểm thử trong môi trường VMware.

Quan trọng hơn, hệ thống mạng hỗ trợ trực tiếp các chức năng Active Directory, DNS, Group Policy và File Server, thể hiện khả năng tích hợp giữa Network Infrastructure và System Administration.

Đây là nền tảng kết nối cho toàn bộ các dịch vụ được triển khai trong Project 1.