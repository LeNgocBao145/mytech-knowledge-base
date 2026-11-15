Mô hình **Wrapper Chart (hoặc Umbrella Chart)**: bạn tạo một [[Helm]] chart vỏ bọc trên Git, khai báo chart gốc từ OCI vào mục `dependencies` trong `Chart.yaml` và quản lý toàn bộ cấu hình bằng các file `values.yaml` riêng biệt, thay vì trỏ thẳng `Application` tới OCI rồi nhồi tham số vào `spec.source.helm.parameters`.

Cách tiếp cận này giải quyết triệt để các giới hạn về khả năng bảo trì khi số lượng cấu hình tăng lên:

|**Tiêu chí**|**Wrapper Chart (OCI dependency + values.yaml)**|**Khai báo trực tiếp (Application + helm.parameters)**|
|---|---|---|
|**Định dạng cấu hình**|File `values.yaml` chuẩn, có highlight cú pháp và gợi ý schema từ IDE.|Danh sách phẳng `name: key`, `value: val` (tương đương cờ `--set` của Helm CLI).|
|**Dữ liệu phức tạp (List, Map, Multiline)**|Khai báo tự nhiên, rõ ràng theo cấu trúc cây YAML.|Rất rối khi đánh chỉ mục mảng (`items[0].name`) hoặc chèn chuỗi nhiều dòng.|
|**Đính kèm tài nguyên K8s bổ sung**|Chỉ cần thêm file vào thư mục `templates/` của wrapper chart để deploy cùng lúc.|Không hỗ trợ trực tiếp; phải tạo thêm một `Application` riêng lẻ khác.|
|**Quản lý nhiều môi trường**|Dùng chung một `Chart.yaml`, chỉ cần tách `values-dev.yaml`, `values-prod.yaml`.|Phải nhân bản toàn bộ khối `parameters` dài dòng sang từng file `Application`.|
|**Tính sạch sẽ của `Application` CRD**|File `Application` rất gọn, chỉ làm đúng nhiệm vụ định tuyến triển khai.|File `Application` phình to, trộn lẫn metadata của Argo CD với cấu hình app.|

### 4 lý do thực tế khiến Wrapper Chart + `values.yaml` dễ kiểm soát hơn

1. **Tránh lỗi ép kiểu ngầm (Type Coercion) của `helm.parameters`** Cơ chế `parameters` hoạt động giống hệt cờ `--set` trên dòng lệnh: các giá trị như `"true"`, `"null"` hoặc chuỗi số `"01234"` thường bị Helm tự động ép sang kiểu `boolean` hoặc `integer`, gây lỗi schema khi pod khởi chạy. Dùng `values.yaml` giữ nguyên chính xác kiểu dữ liệu gốc.
    
2. **Đóng gói kèm các tài nguyên phụ trợ không có sẵn trong OCI Chart** Khi triển khai một chart từ bên thứ 3 (như Prometheus, Ingress NGINX hay ứng dụng nội bộ), bạn thường cần tạo thêm `ExternalSecret` (để kéo mật khẩu từ AWS Secrets Manager), `Certificate` hoặc `NetworkPolicy`. Nhờ có thư mục `templates/` trong Wrapper Chart, toàn bộ các manifest phụ trợ này được quản lý và đồng bộ chung một vòng đời với chart chính.
    
3. **Tách bạch trách nhiệm và làm sạch lịch sử Git (Git Diff)** Người vận hành hạ tầng chỉ cần quan tâm đến file `Application` (deploy vào cluster/namespace nào, bật auto-sync ra sao), còn lập trình viên chỉ thao tác trên `values.yaml` (đổi image tag, chỉnh biến môi trường, tăng replica). Khi review Pull Request, lịch sử thay đổi tách bạch rõ ràng thay vì sửa chung một file YAML khổng lồ.
    
4. **Khóa tính toàn vẹn phiên bản qua `Chart.lock`** Khi khai báo OCI chart trong `Chart.yaml`, file `Chart.lock` đi kèm sẽ lưu lại chính xác mã băm `digest` (SHA256) của artifact đó, ngăn chặn rủi ro một tag phiên bản trên registry bị đẩy đè nội dung mới.
    

### Điểm đánh đổi và một lựa chọn hiện đại khác trên Argo CD

- **Nhược điểm duy nhất của Wrapper Chart:** Vì OCI chart lúc này đóng vai trò là _sub-chart_ (dependency), toàn bộ cấu hình trong `values.yaml` của bạn phải thụt lề vào một cấp dưới tên của dependency đó (ví dụ phải bọc toàn bộ cấu hình bên dưới key `my-oci-chart:`), thay vì copy-paste thẳng file `values.yaml` mặc định của upstream.
    
- **Giải pháp thay thế (Argo CD v2.6+):** Nếu không muốn tạo thư mục `Chart.yaml` vỏ bọc và không muốn bị thụt lề key trong `values.yaml`, Argo CD hỗ trợ tính năng **Multi-Source Application** — cho phép bạn khai báo đồng thời 2 nguồn trong cùng một file `Application`: nguồn 1 trỏ trực tiếp tới OCI chart, và nguồn 2 trỏ tới repo Git chỉ để đọc file `values.yaml` thông qua biến tham chiếu `$values`.