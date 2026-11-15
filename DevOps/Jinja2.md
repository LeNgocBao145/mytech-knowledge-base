**Jinja2** là một thư viện **template engine** (động cơ mẫu) phổ biến dành cho ngôn ngữ lập trình Python, được phát triển bởi dự án Pallets (đội ngũ tạo ra Flask framework). Jinja2 cho phép tách biệt logic xử lý dữ liệu (Python) khỏi định dạng hiển thị hoặc cấu hình (HTML, YAML, JSON, file văn bản).

### Cú pháp cốt lõi

Jinja2 sử dụng 3 loại thẻ cú pháp chính để nhúng dữ liệu vào văn bản:

- **`{{ ... }}` (Expressions):** In giá trị của biến hoặc biểu thức ra output.
    
    Đoạn mã
    
    ```
    Hello {{ user.name }}, bạn có {{ items|length }} mục trong giỏ hàng.
    ```
    
- **`{% ... %}` (Statements):** Thực thi các câu lệnh điều khiển như vòng lặp, điều kiện, hoặc kế thừa template.
    
    Đoạn mã
    
    ```
    {% if is_admin %}
      <p>Chào mừng Admin!</p>
    {% else %}
      <p>Chào mừng người dùng!</p>
    {% endif %}
    ```
    
- **`{# ... #}` (Comments):** Ghi chú trong template, nội dung này sẽ bị loại bỏ hoàn toàn khi render.
    

### Các tính năng nổi bật

1. **Template Inheritance (Kế thừa mẫu):** Cho phép định nghĩa một trang khung (base template) chứa cấu hình chung và các trang con chỉ cần ghi đè các khối (`{% block %}`) cụ thể.
    
2. **Filters & Macros:**
    
    - **Filter:** Biến đổi dữ liệu trực tiếp bằng toán tử pipe `|` (ví dụ: `{{ name | upper }}` hoặc `{{ list | join(', ') }}`).
        
    - **Macro:** Định nghĩa các đoạn template tái sử dụng được, tương tự như hàm trong lập trình.
        
3. **Auto-escaping (Tự động mã hóa HTML):** Tự động chuyển đổi các ký tự đặc biệt trong dữ liệu đầu vào thành HTML entities (ví dụ `<` thành `<`) để bảo vệ ứng dụng khỏi lỗ hổng Cross-Site Scripting (XSS).
    
4. **Môi trường Sandbox (Sandbox Execution):** Cung cấp chế độ thực thi an toàn, ngăn chặn template chạy các đoạn mã Python nguy hiểm nếu template đó do người dùng cuối tải lên.
    

### Ứng dụng thực tế

- **Lập trình Web (Python Web Development):** Là template engine mặc định của **Flask** và thường xuyên được tích hợp trong **FastAPI** để render HTML tĩnh/động từ phía server (SSR).
    
- **Quản lý cấu hình & DevOps (Configuration Management):** Được sử dụng làm công cụ xử lý file template mặc định trong **Ansible** (các file `.j2`), **SaltStack**, giúp sinh ra các file cấu hình hệ thống (`/etc/nginx/nginx.conf`, file Kubernetes Manifest YAML, v.v.) dựa trên biến môi trường.