Cả [[Helm]] và Kustomize đều giải quyết chung một bài toán là tùy biến cấu hình Kubernetes cho nhiều môi trường (`dev`, `staging`, `prod`) mà không phải copy-paste file YAML, nhưng **cách tiếp cận của chúng trái ngược nhau**: [[Helm]] là một **Package Manager dùng cơ chế Templating**, còn Kustomize là công cụ **Overlay/Patching trên YAML thuần** (hoàn toàn không dùng biến hay cú pháp template).

|Tiêu chí|Helm|Kustomize|
|---|---|---|
|**Cơ chế cốt lõi**|**Templating:** Chèn biến và logic (`if`, `range`) vào YAML qua Go Template, sau đó bơm dữ liệu từ file `values.yaml`.|**Patching (Ghi đè):** Giữ nguyên file YAML gốc hợp lệ, sau đó áp các bản vá (patch) để sửa/thêm trường theo từng môi trường.|
|**File gốc (Base)**|Không phải YAML hợp lệ (chứa đầy cú pháp `{{ .Values... }}`), không thể chạy trực tiếp bằng `kubectl apply`.|Là **YAML Kubernetes thuần 100%**, có thể `kubectl apply` trực tiếp bất cứ lúc nào.|
|**Quản lý gói (Package)**|**Có.** Đóng gói thành Chart, quản lý phiên bản (Release), rollback, cài đặt dễ dàng các app bên thứ 3 (như bạn vừa làm với `external-secrets`).|**Không.** Chỉ làm nhiệm vụ biến đổi các file YAML, không quản lý vòng đời cài đặt hay kho lưu trữ gói.|
|**Cài đặt CLI**|Phải cài đặt binary `helm` riêng.|**Tích hợp sẵn** bên trong `kubectl` từ bản 1.14 (`kubectl apply -k`).|
|**Độ dễ đọc & Debug**|Khó đọc khi chart phức tạp, lồng nhiều vòng lặp/điều kiện; phải chạy `helm template` mới thấy YAML thật.|Dễ đọc và dễ review trên Git vì cấu trúc YAML được giữ nguyên bản.|

### Minh họa sự khác biệt khi muốn đổi `replicas: 3` cho môi trường Prod

**Với Helm (phải đặt chỗ trước bằng biến trong template):**

YAML

```
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
```

Sau đó truyền giá trị qua file `values-prod.yaml` (`replicaCount: 3`). Nếu người viết chart quên đặt biến `{{ .Values... }}` ở một trường nào đó, bạn không thể tự ý sửa trường đó từ bên ngoài.

**Với Kustomize (sửa trực tiếp trên YAML chuẩn thông qua lớp phủ):**

YAML

```
# base/deployment.yaml (YAML thuần, mặc định chạy 1 replica)
spec:
  replicas: 1
```

Ở thư mục `overlays/prod/kustomization.yaml`, bạn chỉ cần khai báo đoạn patch đè đúng trường cần đổi mà không cần đụng vào file gốc:

YAML

```
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
    target:
      kind: Deployment
      name: my-app
```

### Cách chọn khi làm GitOps với Argo CD

- **Dùng Helm** khi triển khai các **hệ thống/công cụ bên thứ 3** (Argo CD, External Secrets, Prometheus, Ingress NGINX...) vì cộng đồng đã đóng gói sẵn Chart chuẩn, hoặc khi cần đóng gói ứng dụng của bạn để phân phối cho nhiều bên khác nhau cài đặt.
    
- **Dùng Kustomize** khi quản lý **manifest của chính dịch vụ nội bộ** qua các môi trường (`dev` → `staging` → `prod`), nơi cấu trúc deployment giống nhau đến 90% và bạn chỉ cần thay đổi image tag, số lượng replica, hoặc biến môi trường một cách rõ ràng, dễ kiểm soát trên Git.
    
- **Kết hợp cả hai:** Trong thực tế, Argo CD hỗ trợ dùng Helm để tải và render Chart gốc của bên thứ 3, sau đó chuyển kết quả qua Kustomize (cơ chế _Helm Post-Renderer_) để vá thêm các trường đặc thù mà Helm Chart gốc không hỗ trợ.