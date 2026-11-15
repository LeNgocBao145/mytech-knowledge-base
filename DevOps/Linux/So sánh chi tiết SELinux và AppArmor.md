SELinux và AppArmor đều là các giải pháp bảo mật cấp Kernel dựa trên cơ chế **Mandatory Access Control (MAC)** trong Linux, dùng để giới hạn quyền hạn của tiến trình kể cả khi tiến trình đó chạy dưới quyền `root`. Tuy nhiên, hai hệ thống này khác nhau căn bản về **triết lý định danh (Label vs Path)**, **độ phức tạp** và **hệ sinh thái mặc định**.

## 1. Bảng so sánh tổng quan

|**Tiêu chí**|**SELinux (Security-Enhanced Linux)**|**AppArmor (Application Armor)**|
|---|---|---|
|**Triết lý kiểm soát**|**Label-based** (Dựa trên nhãn gán cho đối tượng/tiến trình)|**Path-based** (Dựa trên đường dẫn tuyệt đối của file thực thi)|
|**Độ phức tạp**|Rất cao, học tốn thời gian, cấu hình ma trận quyền chi tiết|Thấp đến trung bình, cú pháp thân thiện, dễ đọc/viết|
|**Nơi lưu cấu hình**|Nhãn nằm trong **Extended Attributes (`xattr`)** của File System|File cấu hình dạng văn bản trong `/etc/apparmor.d/`|
|**Chế độ hoạt động**|`Enforcing`, `Permissive`, `Disabled`|`Enforce`, `Complain`, `Unconfined`|
|**Khả năng kiểm soát Mạng**|Chi tiết đến từng IP, Port, Network Interface, Socket Type|Giới hạn theo Family/Type (TCP, UDP, Raw Socket); không chặn theo IP/Port|
|**Hệ phân phối mặc định**|RHEL, CentOS, Fedora, Rocky Linux, AlmaLinux, Android|Ubuntu, Debian, SUSE / openSUSE|
|**Tự động hóa Container**|Tự động sinh nhãn động MCS cách ly container (Podman/Docker)|Dùng các Profile tĩnh hoặc nạp qua Kubernetes Operator/DaemonSet|

## 2. Phân tích các điểm khác biệt trọng tâm

### Cơ chế định danh: Label-based vs Path-based

- **SELinux (Label-based):**
    
    - Mỗi tiến trình và file/socket/port đều được gán một **Security Context** dạng `user:role:type:level` (ví dụ: `system_u:object_r:httpd_sys_content_t:s0`).
        
    - SELinux không quan tâm file nằm ở đâu hay tên gì. Chỉ cần file có đúng nhãn `httpd_sys_content_t`, web server sẽ đọc được.
        
    - _Ưu điểm:_ Cực kỳ linh hoạt. Dù hacker đổi tên hay di chuyển file nhạy cảm sang thư mục khác, quy tắc bảo mật vẫn giữ nguyên.
        
    - _Nhược điểm:_ Phải quản lý và "dán nhãn" đúng cho mọi file trên đĩa (dùng `chcon`, `restorecon`).
        
- **AppArmor (Path-based):**
    
    - Ràng buộc trực tiếp đường dẫn thực thi, ví dụ: `/usr/sbin/nginx { ... }`.
        
    - _Ưu điểm:_ Trực quan, dễ viết. Nhìn vào profile là biết ngay ứng dụng nào được đọc/ghi thư mục nào.
        
    - _Nhược điểm:_ Nhạy cảm với vị trí file. Nếu ứng dụng di chuyển từ `/usr/bin/myapp` sang `/opt/myapp`, profile cũ mất tác dụng (phải sửa lại path hoặc dùng wildcard).
        

### Cú pháp & Ngôn ngữ chính sách (Policy)

- **SELinux:**
    
    - Sử dụng Type Enforcement (TE), Role-Based Access Control (RBAC) và Multi-Level Security (MLS).
        
    - Chính sách được viết bằng ngôn ngữ riêng (`.te`), sau đó phải biên dịch qua nhiều bước (`checkmodule` $\rightarrow$ `semodule_package` $\rightarrow$ `semodule -i`).
        
- **AppArmor:**
    
    - File cấu hình dạng text dễ đọc. Hỗ trợ cơ chế gom nhóm quyền (`#include <abstractions/base>`), wildcard (`/var/log/*`, `/data/**`), và các cờ thực thi (`px`, `ix`).
        
    - Nạp trực tiếp qua `apparmor_parser -r` mà không cần quy trình biên dịch phức tạp.
        

### Kiểm soát Mạng (Network Fine-Grained Control)

- **SELinux:** Cho phép tạo quy tắc kiểm soát sâu: gán nhãn cho từng Port (`http_port_t` = 80, 443), kiểm soát tiến trình nào được phép bind vào Port nào, hoặc thiết lập bảo mật IPsec (labeled IPsec).
    
- **AppArmor:** Chỉ kiểm soát ở mức độ khởi tạo Socket (`network inet stream`, `network raw`). Không thể cấm ứng dụng kết nối tới một Port hoặc IP cụ thể trực tiếp trong profile.
    

### Hành vi khi khôi phục / Nhân bản (Clone & Backup)

- **SELinux:** Khi copy hoặc rsync file từ máy này sang máy khác, nếu không giữ thuộc tính mở rộng (`xattr`), file mới sẽ bị mất nhãn và tiến trình có thể bị từ chối truy cập ngay lập tức.
    
- **AppArmor:** Vì không ghi dữ liệu gì lên ổ đĩa của file, việc backup/restore hay copy file không ảnh hưởng tới profile AppArmor hiện có.
    

## 3. Khi nào nên dùng giải pháp nào?

- **Nên dùng AppArmor khi:**
    
    - Đang vận hành môi trường dựa trên **Ubuntu / Debian / SUSE**.
        
    - Muốn tự viết profile bảo mật cho các ứng dụng nội bộ một cách nhanh chóng, dễ bảo trì.
        
    - Ưu tiên sự đơn giản, muốn dễ dàng debug bằng chế độ `complain` mà không mất nhiều thời gian học syntax phức tạp.
        
- **Nên dùng SELinux khi:**
    
    - Đang chạy hệ thống trên **RHEL / Rocky Linux / CentOS / Fedora**.
        
    - Yêu cầu tiêu chuẩn bảo mật khắt khe (Chính phủ, Tài chính, Y tế - tuân thủ HIPAA, PCI-DSS, STIG).
        
    - Cần cơ chế cách ly container đa tầng động (Multi-Category Security) mạnh mẽ với Podman/CRI-O.