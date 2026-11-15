**OCI chart** là một **Helm Chart** tiêu chuẩn được đóng gói và lưu trữ trực tiếp trên các **Container Registry hỗ trợ chuẩn OCI** (Open Container Initiative) như GitHub Container Registry (GHCR), AWS ECR, Docker Hub, Harbor hay Quay — dùng chung hạ tầng lưu trữ và cơ chế xác thực với container image thay vì phải dựng một [[Helm]] repository truyền thống qua giao thức HTTP.

Trước đây, để chia sẻ Helm chart, bạn phải host các file nén `.tgz` cùng một file mục lục **`index.yaml`** trên web server (như GitHub Pages, S3 hoặc ChartMuseum). Từ phiên bản **Helm v3.8+**, OCI trở thành chuẩn chính thức, cho phép đẩy và kéo Helm chart qua tiền tố **`oci://`** dưới dạng một OCI Artifact có version tag và mã băm SHA256 (digest) giống hệt Docker image.

### So sánh Helm Repo truyền thống và OCI Chart

|Đặc điểm|Helm Repo truyền thống (`https://`)|OCI Chart (`oci://`)|
|---|---|---|
|**Nơi lưu trữ**|Web server tĩnh (GitHub Pages, S3, ChartMuseum)|Container Registry (GHCR, AWS ECR, Docker Hub, Harbor)|
|**Cơ chế tra cứu phiên bản**|Đọc qua một file **`index.yaml`** chứa danh sách toàn bộ chart và version|Truy vấn trực tiếp theo **Tag / Digest** của từng artifact|
|**Xác thực & Phân quyền**|Quản lý tài khoản/mật khẩu riêng cho từng Helm repo|Dùng chung tài khoản, IAM Role hoặc token với nơi lưu Docker image|
|**Hiệu năng (đặc biệt với Argo CD)**|Phải tải toàn bộ file `index.yaml` (các repo lớn như Bitnami nặng hàng chục MB, dễ gây ngốn RAM)|Chỉ tải đúng metadata của phiên bản chart được chỉ định|
|**Ký số bảo mật (Signing)**|Dùng GPG key (`helm package --sign`)|Hỗ trợ các công cụ ký container hiện đại như **Cosign (Sigstore)**, Notary|

### Cách sử dụng thực tế

#### 1. Đẩy và cài đặt bằng Helm CLI

Bash

```
# 1. Đóng gói thư mục chart thành file .tgz
helm package ./my-chart

# 2. Đăng nhập vào Container Registry (ví dụ GHCR)
helm registry login ghcr.io -u <username>

# 3. Đẩy chart lên registry qua giao thức oci://
helm push my-chart-0.1.0.tgz oci://ghcr.io/my-org/charts

# 4. Cài đặt trực tiếp từ OCI registry
helm install my-release oci://ghcr.io/my-org/charts/my-chart --version 0.1.0
```

#### 2. Khai báo OCI Chart trong Argo CD (`Application`)

Khi trỏ Argo CD tới một OCI chart, trường `repoURL` chỉ ghi đường dẫn registry (**không** thêm tiền tố `https://` hay `oci://`):

YAML

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  source:
    repoURL: ghcr.io/my-org/charts
    chart: my-chart
    targetRevision: 0.1.0
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

_(Nếu OCI registry ở chế độ private, bạn chỉ cần khai báo thêm `enableOCI: "true"` trong Secret cấu hình repository của Argo CD bên cạnh `username` và `password`)._

## Docker registry tôi tưởng chỉ lưu image thôi mà?

Ban đầu đúng là như vậy, nhưng từ khi chuẩn **OCI (Open Container Initiative)** mở rộng đặc tả lưu trữ (**OCI Distribution Spec**), Docker Registry (hay đúng hơn là Container Registry) đã chuyển mình thành một **kho lưu trữ tổng quát (Universal Artifact Store)** chứ không còn gói gọn ở container image nữa.

### Tại sao Registry lại lưu được các thứ khác?

Về mặt kỹ thuật, một Docker image khi đẩy lên registry thực chất chỉ bao gồm 2 thành phần:

- **Blobs (Layers):** Các file nén `.tar.gz` chứa dữ liệu thực tế.
    
- **Manifest:** Một file JSON đóng vai trò mục lục, liệt kê danh sách các blob và khai báo định dạng của gói dữ liệu thông qua trường **`mediaType`** (hoặc `artifactType`).
    

Cộng đồng OCI nhận ra cấu trúc _"1 file JSON mô tả + N file nén"_ này có thể áp dụng cho bất kỳ loại gói phần mềm nào. Bằng cách mở rộng danh sách `mediaType` hợp lệ, registry không cần quan tâm bên trong file nén là hệ điều hành Linux hay một thư mục Helm Chart — nó chỉ việc lưu trữ các cục dữ liệu (blobs) và trả về đúng metadata khi client gọi tới.

### Những gì đang được lưu trên Docker/OCI Registry hiện nay

Nhờ cơ chế OCI Artifacts, các registry hiện đại (như GitHub Container Registry, AWS ECR, Harbor, Docker Hub) hiện lưu trữ song song nhiều loại tài nguyên khác nhau bên cạnh Docker image:

- **Helm Charts:** Đóng gói file `.tgz` của chart và gắn tag phiên bản (`oci://...`).
    
- **Chữ ký số & SBOM (Software Bill of Materials):** Các công cụ như **Cosign** hay **Trivy** đẩy chữ ký xác thực và file danh mục thành phần bảo mật (SPDX/CycloneDX) lên registry, liên kết trực tiếp với mã băm (digest) của image gốc.
    
- **Policy & Config Bundles:** Các bộ luật kiểm soát bảo mật của Open Policy Agent (OPA), Kyverno, hoặc các module của Terraform.
    
- **WebAssembly (Wasm) modules & AI Model weights:** Đóng gói binary Wasm hoặc trọng số mô hình AI để tận dụng tốc độ tải phân lớp và hạ tầng CDN sẵn có của registry.
    

Lợi ích lớn nhất của hướng đi này là hợp nhất hạ tầng: thay vì phải dựng riêng một server lưu Docker image, một server lưu Helm chart (như ChartMuseum) và một server lưu file cấu hình, bạn chỉ cần vận hành duy nhất một Container Registry với chung một cơ chế xác thực và phân quyền.