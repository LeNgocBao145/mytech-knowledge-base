Bộ tiêu chuẩn GitOps do nhóm công tác **OpenGitOps (thuộc CNCF)** định nghĩa gồm **4 nguyên tắc cốt lõi**, quy định cách thức mô tả, lưu trữ và vận hành trạng thái của hệ thống từ mã nguồn tới hạ tầng:

1. **Declarative (Tính khai báo)**
    
    - **Nội dung:** Toàn bộ trạng thái mong muốn (_Desired State_) của hệ thống phải được biểu diễn dưới dạng khai báo tĩnh (như file YAML của Kubernetes, file `values.yaml` của Helm, hoặc Kustomize) thay vì các kịch bản mệnh lệnh (_Imperative_ như chạy chuỗi lệnh `kubectl create`, `kubectl scale`, hay script bash từng bước).
        
    - **Giá trị kỹ thuật:** File khai báo mô tả _kết quả cuối cùng_ hệ thống phải đạt được, giúp công cụ triển khai tự tính toán các bước cần thực hiện bất kể trạng thái hiện tại của cụm đang ở đâu.
        
2. **Versioned and Immutable (Được quản lý phiên bản và bất biến)**
    
    - **Nội dung:** Trạng thái mong muốn được lưu trữ trong một hệ thống đảm bảo tính bất biến, đánh phiên bản rõ ràng và lưu giữ toàn bộ lịch sử thay đổi — phổ biến nhất là **Git** (đóng vai trò _Single Source of Truth_), kết hợp với **OCI Registry** (khóa phiên bản bằng mã băm SHA256 digest).
        
    - **Giá trị kỹ thuật:** Mọi thay đổi trên môi trường đều gắn với một commit cụ thể (ai sửa, sửa gì, lúc nào). Khi xảy ra sự cố, việc khôi phục (_rollback_) chỉ đơn giản là chạy `git revert` về commit ổn định trước đó thay vì phải tìm lại pipeline cũ để chạy lại.
        
3. **Pulled Automatically (Được kéo về tự động)**
    
    - **Nội dung:** Các agent chạy ngay bên trong hạ tầng đích (như **Argo CD** hoặc **Flux**) tự động kéo (_pull_) cấu hình từ kho lưu trữ về, thay vì để hệ thống CI bên ngoài (GitHub Actions, Jenkins) chủ động đẩy (_push_) lệnh `kubectl apply` vào cụm.
        
    - **Giá trị kỹ thuật:** Cụm Kubernetes không cần mở cổng API ra Internet hay cấp quyền `cluster-admin` (file `kubeconfig`) cho máy chủ CI bên ngoài, thu hẹp đáng kể bề mặt tấn công bảo mật.
        
4. **Continuously Reconciled (Được đối chiếu và tự phục hồi liên tục)**
    
    - **Nội dung:** Các agent liên tục quan sát trạng thái thực tế đang chạy trên cụm (_Live State_), đối chiếu với trạng thái khai báo trong Git (_Desired State_), và tự động điều chỉnh nếu phát hiện sai lệch.
        
    - **Giá trị kỹ thuật:** Khác với CI pipeline chỉ chạy một lần lúc build rồi dừng, vòng lặp đối chiếu (_reconciliation loop_) hoạt động 24/7. Nếu có người can thiệp thủ công bằng `kubectl edit` hoặc xóa nhầm tài nguyên trên cụm (gây hiện tượng _Configuration Drift_), Argo CD sẽ lập tức phát hiện trạng thái `OutOfSync` và kéo hệ thống quay về đúng chuẩn đã ghi trong Git.
        

### Ánh xạ 4 nguyên tắc vào thực tế vận hành

|**Nguyên tắc**|**Cách làm truyền thống (CI Push)**|**Chuẩn mực GitOps (Pull & Reconcile)**|
|---|---|---|
|**Declarative**|Chạy `helm upgrade --set image.tag=v2` trong script CI|Sửa `tag: v2` trong file `values.yaml` trên Git|
|**Versioned & Immutable**|Trạng thái cụm nằm rải rác qua các lần chạy pipeline|Toàn bộ lịch sử nằm trọn trong `git log` và `Chart.lock`|
|**Pulled Automatically**|Jenkins/GitHub Actions giữ secret `KUBECONFIG` để đẩy vào cụm|Pod `argocd-application-controller` trong cụm tự kéo từ Git|
|**Continuously Reconciled**|Sửa tay bằng `kubectl edit` làm cụm lệch khỏi code mà không ai biết|Argo CD tự phát hiện lệch cấu hình (_Drift_) và tự đồng bộ lại|

## Tại sao việc dùng yaml để tạo thẳng application dùng helm lại vi phạm tư duy tiêu chuẩn gitops so với việc tạo helm chart rồi application dùng nó thông qua git thôi?

Việc khai báo trực tiếp upstream Helm chart cùng tham số cấu hình ngay bên trong file `Application` của Argo CD đi ngược lại chuẩn mực GitOps vì nó **trộn lẫn cấu hình ứng dụng (Application Configuration) với chỉ thị điều phối (CD Routing)** và làm mất tính độc lập của **Desired State (Trạng thái mong muốn)** trên Git.

Trong kiến trúc GitOps tiêu chuẩn (theo đặc tả OpenGitOps), file `Application` chỉ đóng vai trò là "cầu nối" chỉ định _lấy mã nguồn ở đâu và đồng bộ vào cụm nào_, còn toàn bộ định nghĩa về ứng dụng phải nằm trọn vẹn và tự chủ trong Git. Cách làm nhồi trực tiếp [[Helm]] vào `Application` phá vỡ 4 nguyên tắc cốt lõi sau:

### 1. Vi phạm nguyên tắc tách bạch trách nhiệm (Separation of Concerns)

- **Vai trò đúng của `Application`:** Thuộc về tầng hạ tầng/Platform (thường do DevOps quản lý thông qua mô hình _App-of-Apps_ hoặc _ApplicationSet_), quy định cluster đích, namespace, và chính sách `syncPolicy`.
    
- **Hệ quả khi nhồi tham số Helm vào `Application`:** Bạn đang ép cấu hình vận hành của ứng dụng (biến môi trường, image tag, tài nguyên CPU/RAM) nằm chung một file với cấu hình của công cụ CD. Khi developer cần chỉnh một thông số ứng dụng, họ buộc phải sửa vào tài nguyên quản trị của Argo CD, làm rối phân quyền (RBAC) và làm bẩn lịch sử commit của hạ tầng.
    

### 2. Trạng thái mong muốn (Desired State) bị khóa chặt vào Argo CD

- **Yêu cầu của GitOps:** Thư mục trên Git phải là một đơn vị cấu hình chuẩn mực mà bất kỳ công cụ Kubernetes tiêu chuẩn nào cũng đọc và kiểm thử được mà không cần biết đến sự tồn tại của Argo CD.
    
- **Hệ quả khi dùng `helm.parameters` / `helm.values` trong `Application`:** Toàn bộ cấu hình của bạn bị nhốt bên trong schema đặc thù của CRD `argoproj.io/v1alpha1`. Bạn không thể đứng tại repo Git để chạy trực tiếp `helm template` hay dùng các công cụ kiểm tra bảo mật/cú pháp trong CI pipeline (như `kubeconform`, `trivy config`, `kyverno`) trước khi merge PR, trừ khi phải viết thêm script bóc tách dữ liệu từ file `Application`.
    
- **Lợi thế khi dùng Helm Chart trên Git:** Mọi pipeline CI hoặc lập trình viên đều có thể chạy `helm dependency build && helm template .` để render ra toàn bộ YAML thuần và kiểm duyệt ngay trên Pull Request.
    

### 3. Thiếu tính bất biến tuyệt đối (Immutability) và khóa mã băm

- Khi trỏ `Application` trực tiếp ra một Helm repository hoặc OCI registry bên ngoài qua `targetRevision: 1.2.0`, Git của bạn chỉ lưu một **con trỏ phiên bản** chứ không lưu bằng chứng toàn vẹn của gói chart đó. Nếu bên cung cấp chart đẩy đè (overwrite) nội dung khác lên cùng tag `1.2.0`, trạng thái cụm Kubernetes của bạn sẽ bị thay đổi mà không hề có một commit nào xuất hiện trong Git.
    
- Khi đóng gói qua Helm Chart trên Git, lệnh `helm dependency update` sẽ sinh ra file **`Chart.lock`** lưu cứng mã băm **SHA256 digest** của chart phụ thuộc. Bất kỳ sự sai lệch nào ở upstream đều bị phát hiện ngay lập tức, đảm bảo nguyên tắc _mọi thay đổi trên cụm đều phải bắt nguồn từ một commit có chữ ký/lịch sử rõ ràng trên Git_.
    

### 4. Phá vỡ tính nguyên tử (Atomicity) của một đơn vị triển khai

- Một ứng dụng thực tế hiếm khi chỉ gồm mỗi Helm chart gốc; nó luôn cần đi kèm các tài nguyên đặc thù của hệ thống như `ExternalSecret` (để kéo mật khẩu từ AWS Secrets Manager), `NetworkPolicy`, hoặc `Ingress` tùy chỉnh.
    
- Nếu dùng `Application` trỏ thẳng upstream chart, bạn buộc phải tách các tài nguyên phụ trợ này sang một `Application` khác, khiến việc triển khai hoặc `git revert` khi có sự cố bị phân mảnh. Khi đưa về một Helm chart riêng trên Git, chart gốc và các template bổ sung nằm chung một thư mục, được đồng bộ hoặc rollback cùng nhau trong duy nhất một commit.
    

Để giải quyết sự bất tiện khi phải tạo quá nhiều Wrapper Chart rườm rà mà vẫn không vi phạm tư duy GitOps, từ phiên bản **Argo CD 2.6+**, cộng đồng đã bổ sung tính năng **Multi-Source Application** — cho phép một `Application` lấy mã nguồn chart từ OCI registry ở nguồn số 1 và lấy file `values.yaml` thuần từ một repo Git ở nguồn số 2, giữ cho file `Application` hoàn toàn sạch bóng các tham số cấu hình nội tuyến.
