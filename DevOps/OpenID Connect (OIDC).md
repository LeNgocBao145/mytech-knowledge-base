**OpenID Connect (OIDC)** là một lớp xác thực danh tính (**Authentication**) được xây dựng trực tiếp bên trên giao thức phân quyền **OAuth 2.0**. Nó cho phép ứng dụng (Client) xác minh danh tính người dùng thông qua một máy chủ định danh tập trung (**Identity Provider** như [[Keycloak]], Google, Okta) và nhận về thông tin hồ sơ người dùng dưới định dạng chuẩn **JWT (JSON Web Token)**.

### Sự khác biệt giữa OAuth 2.0 và OpenID Connect

OAuth 2.0 ban đầu được thiết kế chỉ để **cấp quyền truy cập tài nguyên** (ví dụ: cho phép một ứng dụng đọc Google Drive của bạn) chứ không có chuẩn chung để biết _người đang đăng nhập thực sự là ai_. OIDC ra đời để bổ sung đúng mảnh ghép định danh còn thiếu này:

|Tiêu chí|OAuth 2.0|OpenID Connect (OIDC)|
|---|---|---|
|**Mục đích chính**|**Authorization** (Phân quyền — _Bạn được phép làm gì?_)|**Authentication** (Xác thực — _Bạn là ai?_)|
|**Loại token đặc trưng**|**Access Token** (dùng để gọi API tài nguyên)|**ID Token** (luôn ở định dạng JWT, chứa thông tin người dùng)|
|**Scope bắt buộc**|Tùy ứng dụng (`read:repo`, `drive.readonly`...)|Luôn phải có scope **`openid`** (kèm `profile`, `email`...)|
|**Lấy thông tin user**|Mỗi nhà cung cấp có một API riêng (GitHub khác Google)|Chuẩn hóa thống nhất qua **ID Token** và **`/userinfo` endpoint**|

### Các thành phần cốt lõi của OIDC

- **ID Token:** Một chuỗi **JWT** được ký số bởi Identity Provider (như Keycloak), chứa các thông tin định danh (_claims_) như `sub` (ID duy nhất của user), `email`, `name`, `groups`, và thời gian hết hạn `exp`. Ứng dụng dùng public key của Keycloak để xác minh chữ ký của token này mà không cần hỏi lại database.
    
- **UserInfo Endpoint:** Một REST API tiêu chuẩn (thường là `/protocol/openid-connect/userinfo` trên Keycloak) cho phép ứng dụng gửi `Access Token` lên để lấy thêm các thuộc tính chi tiết của người dùng nếu chưa có đủ trong `ID Token`.
    
- **OIDC Discovery (`/.well-known/openid-configuration`):** Một đường dẫn JSON công khai tự động liệt kê toàn bộ thông số kỹ thuật của máy chủ OIDC (URL trang đăng nhập, URL đổi token, địa chỉ lấy public key `jwks_uri`, các `scopes` hỗ trợ). Nhờ chuẩn này, khi tích hợp Keycloak vào **Argo CD** hay **Grafana**, bạn chỉ cần khai báo duy nhất URL gốc của Realm mà không phải nhập tay từng endpoint.
    

### Luồng hoạt động thực tế (Authorization Code Flow)

Khi bạn cấu hình đăng nhập **Argo CD** qua **Keycloak** bằng OIDC, quá trình diễn ra theo 4 bước:

1. **Chuyển hướng đăng nhập:** Bạn truy cập giao diện Argo CD và bấm _Login via Keycloak_. Argo CD chuyển hướng trình duyệt sang trang đăng nhập của Keycloak kèm tham số `scope=openid profile email`.
    
2. **Xác thực tại IdP:** Bạn nhập tài khoản/mật khẩu (và mã 2FA nếu có) trực tiếp trên giao diện của Keycloak — Argo CD hoàn toàn không chạm vào mật khẩu của bạn.
    
3. **Trả về mã ủy quyền (Authorization Code):** Xác thực thành công, Keycloak đẩy trình duyệt quay lại `redirect_uri` của Argo CD kèm một mã `code` dùng một lần.
    
4. **Đổi Token & Ánh xạ quyền:** Backend của Argo CD dùng `code` này (kèm `client_secret`) gọi ngầm sang Keycloak để đổi lấy **`ID Token`** và **`Access Token`**. Argo CD giải mã `ID Token`, đọc trường `groups` bên trong JWT và tự động gán quyền RBAC tương ứng trên cụm Kubernetes.