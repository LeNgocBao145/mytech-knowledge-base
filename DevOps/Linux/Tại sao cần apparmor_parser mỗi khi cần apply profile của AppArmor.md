Lý do bạn cần chạy lệnh **`apparmor_parser`** và sự khác biệt giữa cơ chế của AppArmor với SELinux được giải thích chi tiết dưới đây:

### 1. Tại sao cần `apparmor_parser` khi đã có file trong `/etc/apparmor.d`?

Có 2 lý do chính: **Biên dịch định dạng** và **Hiệu lực tức thì**.

- **File trong `/etc/apparmor.d` chỉ là mã nguồn dạng văn bản (Text file):**
    
    Linux Kernel không thể đọc và phân tích file văn bản mỗi khi một tiến trình chạy (vì như vậy sẽ cực kỳ chậm). `apparmor_parser` đóng vai trò là một trình biên dịch: nó đọc file văn bản đó, dịch thành dạng **nhị phân (bytecode/binary policy)** mà Kernel hiểu được, sau đó nạp trực tiếp vào bộ nhớ RAM của Kernel (`/sys/kernel/security/apparmor/.load`).
    
- **Hiệu lực ngay mà không cần reboot:**
    
    Việc bạn chép hay sửa một file trong `/etc/apparmor.d` chỉ mới lưu thay đổi lên **ổ đĩa**. Kernel đang chạy hoàn toàn chưa biết gì về thay đổi này. Lệnh `apparmor_parser -r` ép Kernel cập nhật lại bộ quy tắc trong RAM ngay lập tức.
    

### 2. Khi Restart máy thì sao? (Vấn đề lưu trữ RAM vs Ổ đĩa)

**Suy luận của bạn hoàn toàn chính xác:**

Tất cả các profile AppArmor đang thực thi đều nằm trên **RAM của Kernel**. Khi restart máy, RAM bị xóa hoàn toàn, đồng nghĩa với việc toàn bộ profile trong Kernel cũng bị mất.

**Tuy nhiên, nó không bị mất cấu hình vì:**

Khi hệ điều hành khởi động lại, dịch vụ hệ thống `apparmor.service` (do `systemd` quản lý) sẽ tự động quét toàn bộ thư mục `/etc/apparmor.d/` và tự động gọi `apparmor_parser` để nạp lại tất cả các file đó vào RAM cho Kernel.

> **Tóm lại:**
> 
> - Luôn để file trong `/etc/apparmor.d/` $\rightarrow$ Để **giữ cấu hình vĩnh viễn** qua các lần reboot.
>     
> - Chạy `apparmor_parser` $\rightarrow$ Để **nạp ngay vào RAM** cho Kernel dùng liền mà không phải khởi động lại máy.
>     

### 3. So sánh với SELinux: Tại sao SELinux có vẻ "chỉ cần gắn label"?

SELinux và AppArmor dùng 2 triết lý quản lý truy cập (MAC) khác nhau:

|**Tiêu chí**|**AppArmor (Path-based)**|**SELinux (Label-based)**|
|---|---|---|
|**Cách quản lý**|Dựa trên **Đường dẫn file** (`/usr/bin/nginx`).|Dựa trên **Nhãn nhạy cảm (Security Label)** gắn trên file (`system_u:object_r:httpd_exec_t:s0`).|
|**Lưu trữ nhãn**|Không lưu nhãn nào lên file system.|Lưu nhãn trực tiếp vào thuộc tính mở rộng của ổ đĩa (Extended Attributes - `xattr`).|
|**Cơ chế nạp Policy**|Nạp bộ Profile biên dịch vào RAM qua `apparmor_parser`.|Nạp bộ Policy biên dịch vào RAM qua `semodule` / lệnh lúc boot.|

**Bản chất thực sự của SELinux:**

1. Khi bạn dùng lệnh đặt label trong SELinux (như `restorecon` hay `chcon`), bạn đang ghi thuộc tính nhãn xuống **ổ đĩa**. Nhãn này sẽ nằm cố định trên filesystem.
    
2. **Tuy nhiên, bộ quy tắc (SELinux Policy)** — thứ định nghĩa _"Label A có quyền đọc Label B hay không"_ — **VẪN PHẢI NẠP VÀO RAM** khi hệ thống khởi động (tương tự như AppArmor).
    
3. Lý do bạn ít khi phải chạy lệnh nạp policy thủ công bên SELinux vì hầu hết các label bạn dùng đã nằm sẵn trong bộ Policy mặc định (`targeted policy`) đã được nạp sẵn vào RAM từ lúc hệ thống boot up. Nếu bạn tự viết một SELinux Policy hoàn toàn mới (`.te`), bạn vẫn bắt buộc phải biên dịch (`checkmodule`) và nạp vào RAM (`semodule -i`) giống hệt như `apparmor_parser`.