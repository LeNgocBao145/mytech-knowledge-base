**Helm** là trình quản lý gói (Package Manager) dành riêng cho Kubernetes — đóng vai trò tương tự như `apt`/`yum` trên Linux hay `npm` trên Node.js.

Nếu Kubernetes là hệ điều hành, thì **Helm là công cụ giúp bạn cài đặt, cập nhật và quản lý các ứng dụng** chạy trên hệ điều hành đó.

### 1. Vấn đề Helm giải quyết

Để triển khai một ứng dụng phức tạp (như WordPress + MySQL) lên Kubernetes bằng cách thông thường, bạn phải tự viết và quản lý rất nhiều file YAML riêng lẻ: `Deployment`, `Service`, `ConfigMap`, `Secret`, `Ingress`, `PersistentVolumeClaim`,...

- **Không có Helm:** Bạn phải dùng lệnh `kubectl apply -f` cho hàng chục file YAML, tự sửa IP/Port thủ công và rất khó quản lý khi muốn rollback (quay lại phiên bản cũ).
    
- **Có Helm:** Tất cả các file YAML trên được đóng gói chung thành một package gọi là **Helm Chart**. Bạn chỉ cần chạy **một lệnh duy nhất** để cài đặt toàn bộ ứng dụng.

### 2. Các khái niệm cốt lõi trong Helm

Helm hoạt động dựa trên 3 thành phần chính:

- **Helm Chart:** Là gói chứa đầy đủ tất cả các bản khai báo (manifests) YAML cần thiết của một ứng dụng, đi kèm với các biến tùy chỉnh (`values.yaml`).
    
- **Helm Release:** Là một **thể hiện (instance)** của Chart sau khi được cài vào Kubernetes cluster. Bạn có thể cài 1 Chart nhiều lần tạo ra các Release khác nhau (VD: `my-db-dev`, `my-db-prod`).
    
- **Helm Repository:** Kho lưu trữ chứa các Helm Chart công khai hoặc nội bộ (tương tự Docker Hub hay Artifact Hub).

### 3. Kiến trúc và tính năng nổi bật

- **Chạy hoàn toàn ở Client-side (Helm v3):** Helm tương tác trực tiếp với Kubernetes API Server thông qua file `kubeconfig`, không cần cài đặt thành phần quản lý Server-side (đã loại bỏ Tiller từ bản v3 để đảm bảo bảo mật).
    
- **Templating Engine:** Helm cho phép biến file YAML tĩnh thành file dynamic (động) bằng cách dùng biến thay thế.
    
- **Rollback dễ dàng:** Helm lưu lại lịch sử các lần thay đổi (revision). Nếu phiên bản mới bị lỗi, bạn quay lại phiên bản trước chỉ bằng 1 dòng lệnh.

### 4. Các lệnh Helm thông dụng


```
# Tìm kiếm một ứng dụng (ví dụ NGINX) trên Artifact Hub
helm search hub nginx

# Cài đặt ứng dụng từ Chart vào Cluster
helm install my-nginx bitnami/nginx

# Nâng cấp cấu hình hoặc phiên bản ứng dụng
helm upgrade my-nginx bitnami/nginx -f my-values.yaml

# Xem danh sách các ứng dụng đang chạy do Helm quản lý
helm list

# Rollback về phiên bản trước đó nếu gặp lỗi
helm rollback my-nginx 1

# Gỡ bỏ toàn bộ tài nguyên của ứng dụng
helm uninstall my-nginx
```

## Khi deploy bằng helm trong k8s thì helm sẽ tìm đến file nào để deploy và những file bắt buộc phải có để helm có thể hoạt động?

Khi thực thi lệnh `helm install` hoặc `helm upgrade`, Helm sẽ quét thư mục Chart (hoặc file nén `.tgz`) và xử lý dữ liệu qua các bước:

### 1. Helm tìm và xử lý những file nào khi deploy?

Helm chạy theo quy trình quét và tổng hợp dữ liệu từ các file:

1. **`Chart.yaml`**: Helm đọc file này đầu tiên để kiểm tra metadata, định danh Chart, phiên bản (`version`), kiểm tra tính tương thích của API (`apiVersion: v2`).
    
2. **`values.yaml`**: Helm đọc file này để lấy các cấu hình/biến mặc định. Nếu bạn truyền thêm file cấu hình riêng qua cờ `-f custom-values.yaml` hoặc đè biến bằng `--set key=value`, Helm sẽ hợp nhất (merge) các giá trị này vào dữ liệu cấu hình chung.
    
3. **Thư mục `templates/`**: Helm quét toàn bộ các file mẫu (`.yaml`, `.tpl`) bên trong thư mục này, dùng engine **Go Template** ghép với các biến đã thu thập ở bước 2 để biên dịch (render) thành các file Kubernetes Manifest nguyên bản (`Deployment`, `Service`, `Ingress`...) rồi gửi đến Kubernetes API Server.
    

### 2. Những file BẮT BUỘC phải có để Helm hoạt động

Để một thư mục được Helm công nhận là một Chart hợp lệ và có thể deploy được, bắt buộc phải đáp ứng **2 thành phần tối thiểu**:

1. **`Chart.yaml` (Bắt buộc 100%):** Thiếu file này, Helm sẽ lập tức báo lỗi `Error: Chart.yaml file is missing` và dừng thực thi._Cấu hình tối thiểu chuẩn Helm v3:_
    
    YAML
    
    ```
    apiVersion: v2
    name: my-app
    version: 0.1.0
    ```
    
2. **Thư mục `templates/` chứa ít nhất 1 file manifest (Bắt buộc):** Thư mục này phải tồn tại và chứa tối thiểu một file template (ví dụ `deployment.yaml` hoặc `pod.yaml`). Nếu thư mục rỗng hoặc không tồn tại, Helm sẽ không thể tạo ra bất kỳ tài nguyên nào trên Kubernetes.
    

### Cấu trúc một Helm Chart tiêu chuẩn

Plaintext

```
my-chart/
├── Chart.yaml          # [BẮT BUỘC] Khai báo tên, phiên bản và metadata
├── templates/          # [BẮT BUỘC] Thư mục chứa các tệp mẫu K8s manifest
│   ├── deployment.yaml # Template tạo Deployment
│   ├── service.yaml    # Template tạo Service
│   ├── _helpers.tpl    # (Tùy chọn) Định nghĩa các hàm template phụ trợ
│   └── NOTES.txt       # (Tùy chọn) In thông báo hướng dẫn ra terminal sau khi deploy
├── values.yaml         # [Khuyên dùng] Chứa các giá trị biến mặc định cho templates
└── .helmignore         # (Tùy chọn) Khai báo các file Helm cần bỏ qua khi đóng gói
```

_Lưu ý:_ File `values.yaml` về mặt kỹ thuật không bắt buộc nếu trong `templates/` bạn điền cứng giá trị mà không dùng biến `{{ .Values }}`. Tuy nhiên, trong thực tế phát triển, `values.yaml` là file không thể thiếu để đảm bảo tính linh hoạt của Helm.