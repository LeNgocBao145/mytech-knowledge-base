**Keycloak** là một giải pháp quản lý định danh và truy cập mã nguồn mở (**IAM** – _Identity and Access Management_), hiện là dự án thuộc hệ sinh thái CNCF (do Red Hat khởi xướng). Nó đóng vai trò như một trạm xác thực trung tâm, cung cấp tính năng **Đăng nhập một lần (SSO)**, xác thực (_Authentication_) và phân quyền (_Authorization_) cho các ứng dụng và hệ thống microservices mà không cần phải tự viết lại logic đăng nhập cho từng dịch vụ.

### Các tính năng cốt lõi

- **Single Sign-On (SSO) & Single Sign-Out:** Người dùng chỉ cần đăng nhập một lần tại giao diện của Keycloak để truy cập đồng thời nhiều ứng dụng khác nhau, và đăng xuất khỏi toàn bộ hệ thống chỉ với một thao tác.
    
- **Hỗ trợ đầy đủ các giao thức chuẩn:** Tích hợp sẵn **OpenID Connect (OIDC)**, **OAuth 2.0** và **SAML 2.0**, cho phép kết nối trực tiếp với hầu hết các framework web/mobile và công cụ hạ tầng hiện đại.
    
- **Xác thực đa yếu tố (MFA/2FA) & Bảo mật tài khoản:** Bật sẵn xác thực hai lớp qua ứng dụng OTP (Google Authenticator, FreeOTP), WebAuthn/Passkeys, quản lý phiên đăng nhập, chống brute-force và quy tắc độ mạnh mật khẩu.
    
- **Kết nối nguồn định danh bên ngoài:** Hỗ trợ đồng bộ tài khoản trực tiếp từ hệ thống **LDAP / Active Directory** sẵn có của doanh nghiệp (_User Federation_) hoặc cho phép đăng nhập qua tài khoản mạng xã hội như GitHub, Google, Microsoft (_Identity Brokering_).
    
- **Cấp phát JWT cho Microservices:** Thay vì mỗi dịch vụ backend phải tự truy vấn database để kiểm tra session, Keycloak cấp phát **JSON Web Token (JWT)** có chữ ký số chứa sẵn danh tính và quyền hạn (`roles`, `groups`), giúp các microservice xác thực phi trạng thái (_stateless_).
    

### 3 khái niệm nền tảng cần nắm

|**Khái niệm**|**Vai trò trong Keycloak**|**Ví dụ thực tế**|
|---|---|---|
|**Realm**|Không gian quản lý độc lập chứa tập hợp người dùng, cấu hình bảo mật và các ứng dụng riêng biệt (tương đương khái niệm _Tenant_).|`master` (quản trị Keycloak), `internal-tools`, `customer-app`|
|**Client**|Một ứng dụng hoặc dịch vụ đăng ký với Keycloak để ủy quyền việc xác thực người dùng.|`argocd`, `grafana`, `web-portal`, `payment-api`|
|**Role & Group**|Tập hợp các quyền hạn gắn cho người dùng hoặc nhóm, được đính kèm trực tiếp vào token trả về cho ứng dụng.|`cluster-admin`, `devops-viewer`, `standard-user`|

Trong thực tế triển khai Kubernetes và microservices, Keycloak thường được dùng theo hai hướng song song: làm cổng xác thực tập trung (qua OIDC) cho toàn bộ công cụ vận hành nội bộ như Argo CD, Grafana, Harbor, Kubernetes API; và đứng sau API Gateway để quản lý đăng nhập, cấp phát token cho các dịch vụ ứng dụng hướng tới người dùng cuối.