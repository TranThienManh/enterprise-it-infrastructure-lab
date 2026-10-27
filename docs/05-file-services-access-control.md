# 05 — Triển khai File Server và quản lý quyền truy cập

## 1. Tổng quan

Trong môi trường doanh nghiệp, các phòng ban cần một hệ thống lưu trữ tập trung để chia sẻ tài liệu, phối hợp công việc và kiểm soát quyền truy cập dữ liệu.

Nếu tài liệu chỉ được lưu trên máy tính cá nhân hoặc chia sẻ không có cơ chế phân quyền rõ ràng, doanh nghiệp có thể gặp các vấn đề:

- Dữ liệu phân tán, khó quản lý.
- Khó kiểm soát người dùng được phép truy cập tài nguyên.
- Quyền truy cập không nhất quán giữa các nhân viên.
- Thiếu cơ chế quản lý dung lượng và loại tệp.
- Khó khôi phục dữ liệu khi người dùng vô tình thay đổi hoặc xóa tệp.

Để giải quyết các yêu cầu này, Project 1 triển khai **Windows File Server trên FS01**, tích hợp với Active Directory Domain `corp.local`.

Hệ thống sử dụng SMB để chia sẻ dữ liệu, kết hợp Active Directory Security Groups, Share Permissions và NTFS Permissions để kiểm soát quyền truy cập.

Ngoài ra, dự án đưa FSRM và Shadow Copies vào phạm vi quản trị lưu trữ.

---

## 2. Mục tiêu triển khai

| Yêu cầu | Mục tiêu | Giải pháp |
|---|---|---|
| Lưu trữ tập trung | Cung cấp tài nguyên dùng chung | Windows File Server |
| Chia sẻ dữ liệu | Cho phép truy cập qua mạng nội bộ | SMB |
| Phân quyền theo phòng ban | Kiểm soát quyền đọc, ghi và sửa dữ liệu | AD Security Groups, NTFS |
| Quản lý truy cập mạng | Kiểm soát quyền ở cấp Share | SMB Share Permissions |
| Truy cập thuận tiện | Cung cấp ổ đĩa mạng cho người dùng | Group Policy Drive Mapping |
| Quản lý lưu trữ | Hỗ trợ kiểm soát dung lượng và loại tệp | FSRM |
| Khôi phục phiên bản tệp | Hỗ trợ phục hồi dữ liệu từ snapshot | Shadow Copies |

Mục tiêu quan trọng nhất là xây dựng cơ chế **Role-Based Access Control (RBAC)** đối với tài nguyên lưu trữ nội bộ.

---

## 3. Kiến trúc File Server

### 3.1. Thông tin triển khai

| Thuộc tính | Giá trị |
|---|---|
| Hostname | FS01 |
| Operating System | Windows Server 2022 |
| IP Address | `10.10.10.20` |
| Network | SERVER — `10.10.10.0/24` |
| Domain | `corp.local` |
| Chức năng | File Sharing và Storage Administration |
| Giao thức chia sẻ | SMB |

FS01 được triển khai riêng biệt với DC01 để phân tách chức năng quản lý danh tính và lưu trữ dữ liệu.

### 3.2. Sơ đồ kiến trúc

```text
                  Active Directory
                       DC01
                  10.10.10.10
                        |
                 Domain Identity
                 Security Groups
                        |
                        v
USER01 ---- pfSense ---- FS01
                       10.10.10.20
                            |
                    SMB File Sharing
                            |
               +------------+------------+
               |            |            |
               v            v            v
           Accounting       HR           IT
               |
               +---------- Public

                 Storage Management
                    FSRM / VSS
```

**Hình 5.1 — Kiến trúc tích hợp Active Directory và Windows File Server.**

DC01 quản lý tài khoản và nhóm bảo mật, trong khi FS01 thực hiện kiểm tra quyền truy cập tài nguyên.

### 3.3. Lý do tách FS01 khỏi DC01

Việc sử dụng máy chủ riêng cho File Services mang lại những lợi ích:

- Phân định rõ trách nhiệm của Domain Controller và File Server.
- Quản lý tài nguyên lưu trữ độc lập với dịch vụ xác thực.
- Thuận tiện cấu hình Share Permissions và NTFS Permissions.
- Giảm nhu cầu lưu trữ dữ liệu người dùng trên Domain Controller.
- Hỗ trợ xử lý sự cố File Sharing mà không cần thay đổi trực tiếp các dịch vụ AD DS.

Thiết kế này phản ánh nguyên tắc phân tách vai trò máy chủ trong quản trị hạ tầng.

---

## 4. Thiết kế tài nguyên lưu trữ

### 4.1. Cấu trúc dữ liệu

Hệ thống tổ chức dữ liệu theo phòng ban, sử dụng ổ lưu trữ `E:` trên FS01.

Cấu trúc logic:

```text
E:\
└── Shares\
    ├── Accounting\
    ├── HR\
    ├── IT\
    └── Public\
```

Đây là cách biểu diễn cấu trúc tài nguyên theo chức năng. Đường dẫn vật lý cần được đối chiếu với cấu hình thực tế trên FS01 trước khi công bố như một bản kiểm kê.

### 4.2. Chức năng từng thư mục

| Tài nguyên | Mục đích | Đối tượng sử dụng |
|---|---|---|
| Accounting | Lưu trữ dữ liệu kế toán | Nhóm Accounting |
| HR | Lưu trữ tài liệu nhân sự | Nhóm HR |
| IT | Lưu trữ tài liệu kỹ thuật | Nhóm IT |
| Public | Chia sẻ tài liệu chung | Các nhóm được cấp quyền |

Việc phân chia tài nguyên theo phòng ban giúp đơn giản hóa quản lý và kiểm soát truy cập.

### 4.3. Tài nguyên chia sẻ qua SMB

FS01 cung cấp tài nguyên chia sẻ qua giao thức Server Message Block (SMB).

Trong quá trình kiểm thử, hệ thống sử dụng đường dẫn mạng:

`\\fs01.corp.local\CORP-SHARE`

Đường dẫn này đại diện cho điểm truy cập tài nguyên chia sẻ đã được xác minh trong lab.

Các thư mục phòng ban được quản lý thông qua cấu trúc dữ liệu và cơ chế phân quyền tương ứng.

---

## 5. Thiết kế Role-Based Access Control

### 5.1. Mục đích

Quyền truy cập dữ liệu được quản lý theo vai trò và phòng ban.

Thay vì cấp quyền trực tiếp cho từng tài khoản người dùng, hệ thống sử dụng Active Directory Security Groups để đại diện cho các nhóm được phép sử dụng tài nguyên.

Ví dụ:

```text
Domain User
     |
     v
Active Directory
Security Group
     |
     v
FS01
SMB Share
     |
     v
NTFS Permissions
     |
     v
Authorized Resource
```

Cách thiết kế này hỗ trợ quản trị quyền truy cập tập trung và giảm việc cấu hình lặp lại.

### 5.2. Mô hình quyền truy cập

| Nhóm người dùng | Accounting | HR | IT | Public |
|---|---|---|---|---|
| Accounting | Được cấp quyền | Hạn chế | Hạn chế | Được cấp quyền |
| HR | Hạn chế | Được cấp quyền | Hạn chế | Được cấp quyền |
| IT | Theo quyền được cấp | Theo quyền được cấp | Được cấp quyền | Được cấp quyền |

Bảng thể hiện mục tiêu phân quyền theo phòng ban.

Quyền thực tế của từng nhóm phải được xác minh bằng Share Permissions, NTFS ACLs và kết quả kiểm thử bằng tài khoản tương ứng.

### 5.3. Security Group Membership

Một trường hợp thực tế trong dự án là tài khoản:

`corp\hr.kimuyen`

Tài khoản này được chủ đích đặt trong cơ cấu Accounting và thuộc nhóm:

`GG_Accounting`

Mặc dù tên tài khoản có tiền tố `hr.`, quyền truy cập được xác định theo Security Group Membership.

Điều này thể hiện nguyên tắc:

**Quyền truy cập phải dựa trên danh tính và nhóm bảo mật đã được cấp quyền, không dựa vào tên tài khoản.**

---

## 6. Share Permissions và NTFS Permissions

### 6.1. Share Permissions

Share Permissions kiểm soát quyền truy cập tài nguyên thông qua SMB.

Các mức quyền phổ biến:

| Quyền | Chức năng |
|---|---|
| Read | Đọc nội dung được phép |
| Change | Đọc, tạo, sửa và xóa trong phạm vi được cấp |
| Full Control | Toàn quyền ở cấp Share |

Share Permissions chỉ áp dụng khi truy cập qua đường chia sẻ mạng.

### 6.2. NTFS Permissions

NTFS Permissions kiểm soát quyền đối với thư mục và tệp trên hệ thống tệp NTFS.

Các quyền thường sử dụng:

- Read.
- Read & Execute.
- List Folder Contents.
- Write.
- Modify.
- Full Control.

NTFS Permissions có thể áp dụng cho cả truy cập cục bộ và truy cập qua mạng.

### 6.3. Kết hợp hai lớp quyền

Khi truy cập tài nguyên SMB, quyền hiệu lực chịu sự chi phối của cả Share Permissions và NTFS Permissions.

Ví dụ:

| Share Permission | NTFS Permission | Kết quả thông thường |
|---|---|---|
| Read | Modify | Chỉ đọc qua SMB |
| Change | Read | Chỉ đọc qua SMB |
| Change | Modify | Có thể sửa trong phạm vi được cấp |
| Full Control | Read | Chỉ đọc qua SMB |

Bảng minh họa trường hợp Allow Permissions cơ bản. Quyền hiệu lực thực tế còn phụ thuộc vào Group Membership, Deny ACEs, Inheritance và các ACL khác.

### 6.4. Nguyên tắc thiết kế

Dự án áp dụng cách tiếp cận:

- Sử dụng Security Groups để quản lý đối tượng được cấp quyền.
- Kiểm soát quyền truy cập tại Share và NTFS.
- Phân biệt quyền truy cập mạng với quyền truy cập dữ liệu.
- Kiểm thử bằng tài khoản Domain thực tế.

Đây là nền tảng của mô hình phân quyền theo vai trò đối với File Server.

---

## 7. Tích hợp File Server với Group Policy

### 7.1. Mục đích

Người dùng doanh nghiệp cần truy cập tài nguyên chia sẻ một cách thuận tiện.

Thay vì phải nhập đường dẫn UNC mỗi lần sử dụng, hệ thống có thể ánh xạ tài nguyên mạng thành các ổ đĩa trên Windows.

Project 1 sử dụng Group Policy Drive Mapping để thực hiện chức năng này.

### 7.2. Kiến trúc hoạt động

```text
Domain User Login
        |
        v
Group Policy Processing
        |
        v
Drive Mapping
        |
        v
Windows File Explorer
        |
        v
Mapped Network Drives
        |
        v
FS01 SMB Resources
```

Group Policy xác định cấu hình ánh xạ, còn quyền truy cập thực tế vẫn do FS01 kiểm soát.

### 7.3. Kết quả thực tế

Trong bài kiểm thử với tài khoản Accounting:

| Drive Letter | Tài nguyên | Kết quả |
|---|---|---|
| `A:` | Accounting | Truy cập thành công |
| `P:` | Public | Truy cập thành công |

Người dùng có thể truy cập các tài nguyên được cấp quyền từ File Explorer.

**Giá trị kỹ thuật:** Kết hợp quản lý cấu hình tập trung và phân quyền truy cập tài nguyên.

---

## 8. Access-Based Enumeration

### 8.1. Mục đích

Access-Based Enumeration (ABE) là chức năng cho phép File Server ẩn các thư mục hoặc tệp mà người dùng không có quyền truy cập phù hợp khi liệt kê nội dung Share.

Ví dụ, một người dùng thuộc Accounting không cần nhìn thấy các thư mục nội bộ dành riêng cho HR hoặc IT.

### 8.2. Vai trò trong kiến trúc

ABE giúp:

- Giảm hiển thị các tài nguyên không liên quan.
- Đơn giản hóa trải nghiệm truy cập của người dùng.
- Hạn chế việc lộ tên thư mục không cần thiết.

Tuy nhiên, ABE không thay thế NTFS Permissions.

**ABE kiểm soát khả năng hiển thị khi liệt kê tài nguyên; NTFS Permissions kiểm soát quyền truy cập thực tế.**

### 8.3. Trạng thái kiểm chứng

ABE nằm trong phạm vi thiết kế và tài liệu File Server của dự án.

Cần bổ sung ảnh cấu hình ABE và kiểm thử bằng các tài khoản có quyền khác nhau trước khi xác nhận đầy đủ chức năng ẩn thư mục đã hoạt động.

---

## 9. File Server Resource Manager

### 9.1. Mục đích

File Server Resource Manager (FSRM) hỗ trợ quản lý việc sử dụng tài nguyên lưu trữ trên Windows Server.

Các chức năng liên quan đến dự án gồm:

- File Screening.
- Storage Reports.
- Quản lý và đánh giá tình trạng sử dụng dung lượng.

### 9.2. File Screening

File Screening được sử dụng để kiểm soát các loại tệp được lưu vào những thư mục được quản lý.

Ví dụ về nhu cầu doanh nghiệp:

- Hạn chế một số loại tệp không phục vụ công việc.
- Giảm nguy cơ sử dụng dung lượng lưu trữ không phù hợp.
- Áp dụng chính sách lưu trữ theo từng nhóm thư mục.

Đây là chức năng quản lý dữ liệu, không phải cơ chế phòng chống mã độc thay thế Antivirus.

### 9.3. Storage Reports

Storage Reports hỗ trợ thống kê và đánh giá việc sử dụng tài nguyên lưu trữ.

Thông tin báo cáo có thể được sử dụng để:

- Xác định thư mục tiêu thụ nhiều dung lượng.
- Phân tích loại tệp đang được lưu trữ.
- Hỗ trợ lập kế hoạch quản lý dung lượng.

### 9.4. Trạng thái triển khai

FSRM đã được đưa vào phạm vi thực hành trên FS01, bao gồm File Screening và Storage Reports.

Cần đính kèm cấu hình, báo cáo đầu ra và kết quả thử nghiệm File Screening để xác minh đầy đủ hiệu lực của từng chính sách.

---

## 10. Shadow Copies và khả năng phục hồi phiên bản tệp

### 10.1. Bài toán

Trong quá trình sử dụng File Server, người dùng có thể:

- Vô tình xóa dữ liệu.
- Ghi đè nội dung tệp.
- Thay đổi tài liệu không mong muốn.

Do đó, hệ thống cần có cơ chế hỗ trợ khôi phục phiên bản dữ liệu trước đó.

### 10.2. Giải pháp

Project 1 sử dụng Shadow Copies trên phân vùng lưu trữ của FS01.

Cấu hình đã ghi nhận trong quá trình thực hành:

| Thuộc tính | Giá trị |
|---|---|
| Công nghệ | Volume Shadow Copy Service |
| Dung lượng tối đa dành cho Shadow Copies | 2 GB |
| Tỷ lệ cấu hình | Khoảng 10% dung lượng volume tại thời điểm thiết lập |
| Mục đích | Khôi phục phiên bản tệp hoặc thư mục |

### 10.3. Nguyên lý hoạt động

```text
Original File
      |
      v
Shadow Copy Snapshot
      |
      v
File Modified / Deleted
      |
      v
Previous Versions
      |
      v
Restore
```

Shadow Copies cung cấp các phiên bản dữ liệu tại những thời điểm snapshot được tạo.

Khi cần, người quản trị hoặc người dùng có quyền phù hợp có thể sử dụng Previous Versions để truy xuất phiên bản trước đó.

### 10.4. Giới hạn

Shadow Copies không tương đương với một hệ thống Backup độc lập.

Nếu volume hoặc hệ thống lưu trữ chứa cả dữ liệu và Shadow Copies gặp sự cố nghiêm trọng, khả năng khôi phục có thể bị ảnh hưởng.

Vì vậy, Shadow Copies chỉ được xem là một lớp hỗ trợ phục hồi phiên bản dữ liệu, không thay thế chiến lược Backup và Disaster Recovery.

---

## 11. Kiến trúc Backup và Recovery

### 11.1. Mục tiêu

Một hệ thống File Server cần có khả năng bảo vệ dữ liệu trước các sự cố như:

- Hỏng ổ lưu trữ.
- Xóa dữ liệu ngoài ý muốn.
- Lỗi hệ điều hành.
- Hỏng máy chủ.
- Thay đổi cấu hình không mong muốn.

### 11.2. Phân biệt Shadow Copies và Backup

| Tiêu chí | Shadow Copies | Backup |
|---|---|---|
| Mục đích chính | Phục hồi phiên bản dữ liệu | Bảo vệ và khôi phục dữ liệu |
| Phụ thuộc volume nguồn | Thường có | Không bắt buộc |
| Khôi phục khi mất toàn bộ volume | Không bảo đảm | Có thể, nếu bản sao lưu độc lập và hợp lệ |
| Phục hồi tệp riêng lẻ | Có thể | Tùy giải pháp |
| Disaster Recovery | Không thay thế | Là một thành phần quan trọng |

### 11.3. Trạng thái dự án

Backup Architecture đã được đưa vào phạm vi thiết kế.

Tuy nhiên, chưa có đủ bằng chứng để xác nhận một quy trình Backup và Restore hoàn chỉnh cho FS01.

Vì vậy, dự án không tuyên bố đã triển khai thành công Disaster Recovery.

---

## 12. Kiểm thử tích hợp và phân quyền

### 12.1. Môi trường kiểm thử

| Thành phần | Vai trò |
|---|---|
| DC01 | Domain Authentication và Security Groups |
| FS01 | SMB File Server |
| USER01 | Máy trạm kiểm thử |
| pfSense | Kết nối USER và SERVER Networks |

Tài khoản kiểm thử:

`corp\hr.kimuyen`

Security Group:

`GG_Accounting`

### 12.2. Các trường hợp kiểm thử

| Test ID | Tình huống |
|---|---|---|
| FS-01 | USER01 kết nối đến FS01 |
| FS-02 | Truy cập Accounting Share |
| FS-03 | Đọc dữ liệu Accounting |
| FS-04 | Ghi dữ liệu Accounting |
| FS-05 | Truy cập Public Share |
| FS-06 | Truy cập tài nguyên HR không được cấp quyền |
| FS-07 | Truy cập tài nguyên IT không được cấp quyền |
| FS-08 | Ánh xạ ổ `A:` |
| FS-09 | Ánh xạ ổ `P:` |
| FS-10 | Khôi phục dữ liệu bằng Shadow Copies |

Các trường hợp kiểm thử phản ánh những kết quả đã ghi nhận trong quá trình triển khai. Ảnh chụp và kết quả lệnh tương ứng cần được bổ sung vào repository để người đọc có thể đối chiếu.

### 12.3. Kết quả chức năng

Các bài kiểm thử thể hiện ba khả năng quan trọng:

**Network Connectivity:** Máy trạm tại USER Network có thể kết nối đến FS01 tại SERVER Network.

**Authentication:** Tài khoản Domain được sử dụng để truy cập tài nguyên mạng.

**Authorization:** Quyền đọc/ghi và hạn chế truy cập được thực thi dựa trên cấu hình quyền của File Server.

---

## 13. Những vấn đề kỹ thuật đã xử lý

### 13.1. Có kết nối mạng nhưng không truy cập được Share

Một lỗi truy cập File Server có thể xuất phát từ nhiều nguyên nhân:

- DNS phân giải sai địa chỉ.
- Firewall chặn TCP 445.
- SMB Service không khả dụng.
- Share Permissions không phù hợp.
- NTFS Permissions không phù hợp.
- Tài khoản chưa thuộc Security Group được cấp quyền.

Do đó, việc xử lý sự cố cần phân biệt rõ giữa lỗi kết nối mạng và lỗi phân quyền.

### 13.2. Quyền truy cập không tương ứng với tên tài khoản

Trường hợp `corp\hr.kimuyen` cho thấy không thể suy ra quyền truy cập chỉ dựa vào tiền tố trong username.

Quyền thực tế được xác định bởi Group Membership và ACLs.

### 13.3. Group Policy Drive Mapping

Một ổ đĩa mạng không xuất hiện có thể liên quan đến:

- Group Policy chưa được áp dụng.
- Item-Level Targeting hoặc Security Filtering.
- Sai đường dẫn UNC.
- DNS hoặc Network Connectivity.
- Quyền truy cập tài nguyên.

Vì vậy, cần kiểm tra cả Group Policy Processing và trạng thái SMB Access.

---

## 14. Đánh giá kết quả triển khai

| Chức năng | Kết quả |
|---|---|---|
| Windows File Server | FS01 tham gia Domain |
| SMB File Sharing | Truy cập Share từ USER01 |
| Departmental Resources | Accounting, HR, IT, Public |
| Active Directory Integration | Sử dụng tài khoản và Security Group |
| Share/NTFS Permissions | Phân quyền theo tài nguyên |
| Drive Mapping | Ổ `A:` và `P:` |
| Access-Based Enumeration | Ẩn tài nguyên không được cấp quyền |
| FSRM | File Screening và Storage Reports | Đã thực hành; cần Evidence kết quả |
| Shadow Copies | Snapshot và Restore |
| Backup & Recovery | Phục hồi từ bản sao lưu độc lập |
| High Availability | File Server dự phòng |

---

## 16. Giới hạn và hướng mở rộng

### Giới hạn hiện tại

- FS01 là File Server duy nhất.
- Chưa có File Server Failover Cluster.
- Chưa kiểm thử khôi phục dữ liệu từ bản Backup độc lập.
- Chưa có kết quả đánh giá hiệu năng SMB ở quy mô lớn.
- Một số chức năng quản trị lưu trữ cần bổ sung Evidence.
- Chưa có kết quả kiểm toán toàn diện đối với mọi ACL và quyền kế thừa.

### Hướng mở rộng

Các chức năng có thể phát triển trong tương lai:

- Tự động hóa kiểm tra Share và NTFS Permissions.
- Thiết kế Backup Policy và thực hiện Restore Testing.
- Giám sát dung lượng và tình trạng File Server.
- Chuẩn hóa Security Groups theo mô hình AGDLP.
- Xây dựng quy trình Access Review định kỳ.

Các nội dung này là hướng mở rộng, không được tính là chức năng đã hoàn thành.

---

## 17. Kết luận

Project 1 triển khai Windows File Server trên FS01 nhằm cung cấp hệ thống lưu trữ và chia sẻ dữ liệu tập trung cho môi trường doanh nghiệp mô phỏng.

Giải pháp kết hợp SMB, Active Directory Security Groups, Share Permissions và NTFS Permissions để kiểm soát quyền truy cập theo vai trò.

Việc tích hợp Group Policy Drive Mapping giúp người dùng truy cập tài nguyên thuận tiện hơn, trong khi FSRM và Shadow Copies mở rộng khả năng quản trị lưu trữ và phục hồi phiên bản dữ liệu.

Các bài kiểm thử thực tế cho thấy hệ thống có khả năng phân biệt quyền truy cập giữa các phòng ban và cho phép người dùng thực hiện thao tác phù hợp với quyền được cấp.

Dự án thể hiện năng lực **triển khai File Services, quản lý quyền truy cập, tích hợp Active Directory và xử lý sự cố tài nguyên mạng** trong một kiến trúc hạ tầng IT thống nhất.