Trong Istio hiện tại có 2 kiến trúc (deployment models) chính:

### 1. Mô hình Sidecar (Truyền thống)

- **Cách hoạt động:** Istio sẽ tiêm (inject) một container proxy nhỏ (Envoy Proxy) chạy **kèm side-by-side** ngay bên trong từng Pod ứng dụng.
    
- **Đặc điểm:**
    
    - Mỗi Pod có 1 container app + 1 container Envoy proxy.
        
    - Mọi traffic đi vào/ra khỏi Pod đều bắt buộc phải đi qua sidecar proxy này để xử lý mã hóa mTLS, routing, tracing...
        
- **Nhược điểm:** Tốn thêm tài nguyên CPU/RAM cho từng Pod, tăng thời gian boot Pod và phức tạp khi nâng cấp (phải khởi động lại Pod ứng dụng để cập nhật sidecar).
    

### 2. Mô hình Ambient Mesh (Mô hình Sidecarless / "Appliance-like")

Mô hình này loại bỏ hoàn toàn các sidecar proxy chạy bên trong Pod và chuyển sang quản lý mạng ở cấp độ Node (tương tự như một thiết bị ảo - Virtual Appliance / Node Agent).

Ambient Mesh chia công việc làm 2 tầng rõ rệt:

1. **Tầng L4 (Transport Layer - Layer 4): Dùng `ztunnel` (Zero-Trust Tunnel)**
    
    - Mỗi Node trong Kubernetes chỉ chạy đúng **1 Pod `ztunnel` duy nhất** (dưới dạng DaemonSet).
        
    - Tất cả Pod trên Node đó sẽ chia sẻ chung `ztunnel` này để thực hiện mã hóa mTLS và định tuyến L4.
        
    - Cực kỳ nhẹ, tiết kiệm tài nguyên CPU/RAM và không cần inject bất kỳ sidecar nào vào Pod ứng dụng.
        
2. **Tầng L7 (Application Layer - Layer 7): Dùng `Waypoint Proxy`**
    
    - Nếu ứng dụng cần các tính năng nâng cao ở Layer 7 (như HTTP routing, rate limiting, header-based routing, L7 authorization policies...), Istio mới triển khai một proxy riêng gọi là **Waypoint Proxy**.
        
    - Waypoint Proxy chạy độc lập ngoài Pod (thường là cấp Namespace hoặc Service), hoạt động như một gateway trung gian chứ không dính chặt vào Pod app.
        

### Bảng so sánh nhanh

|**Tiêu chí**|**Sidecar Mode**|**Ambient Mesh (Sidecarless)**|
|---|---|---|
|**Vị trí Proxy**|1 Envoy Proxy nằm bên trong **mỗi Pod**|1 `ztunnel` nằm trên **mỗi Node** (+ Waypoint Proxy nếu dùng L7)|
|**Yêu cầu sửa/Reboot Pod**|Có (phải inject sidecar container)|**Không** (Pod ứng dụng giữ nguyên 100%)|
|**Tiêu tốn CPU / RAM**|Cao (nhân số lượng Pod lên)|**Rất thấp** (chia sẻ dùng chung cấp Node)|
|**Độ trễ (Latency)**|Nhỉnh hơn (do đi qua 2 sidecar ở 2 đầu)|Thấp hơn nhiều ở Layer 4|
|**Độ linh hoạt**|Bắt buộc bật đủ cả L4 & L7|Tách biệt rõ L4 (mTLS đơn giản) và L7 (chỉ bật khi cần)|

## `CRDs` and `cluster-wide resources` của Istio base components là gì?

Khi bạn cài đặt component **`istio-base`** (thường qua Helm chart `base` hoặc Istio Operator), đây là bước tiên quyết để khởi tạo toàn bộ "bộ khung" kiến trúc cho Istio Service Mesh trên Kubernetes.

Mục đích duy nhất của `istio-base` là đăng ký các **Custom Resource Definitions (CRDs)** và các tài nguyên dùng chung trên toàn cụm (**Cluster-wide resources**). Nó chưa chạy bất kỳ pod/proxy hay xử lý luồng mạng (traffic) nào.

### 1. Các CRDs (Custom Resource Definitions) do `istio-base` tạo ra

CRD cho phép Kubernetes API Server hiểu và xử lý các loại file cấu hình riêng của Istio. Khi cài `istio-base`, toàn bộ các định nghĩa API thuộc nhóm `*.istio.io` sẽ được đăng ký vào cụm:

#### **Nhóm Traffic Management (Quản lý luồng mạng - `networking.istio.io`)**

- **`VirtualService`**: Định nghĩa các quy tắc routing, phân tách traffic (A/B testing, Canary deployment), retry, timeout, fault injection.
    
- **`DestinationRule`**: Cấu hình các policy áp dụng cho traffic SAU KHÔNG KHI routing (như cấu hình load balancing algorithm, TLS mode, Circuit Breaking/Outlier Detection).
    
- **`Gateway`**: Quản lý các điểm Ingress/Egress cho phép traffic từ bên ngoài đi vào hoặc đi ra khỏi Service Mesh.
    
- **`ServiceEntry`**: Khai báo các dịch vụ bên ngoài (External Services - như Database cloud, API bên thứ 3) vào làm một phần của Service Mesh.
    
- **`Sidecar`**: Cấu hình chi tiết xem Envoy Proxy của một Pod cụ thể được phép giao tiếp với những Service nào (dùng để giảm tải dung lượng memory cho Envoy).
    
- **`WorkloadEntry` & `WorkloadGroup`**: Tích hợp các máy chủ ảo (VM) hoặc bare-metal server bên ngoài Kubernetes vào trong Service Mesh.
    

#### **Nhóm Security & Authorization (Bảo mật - `security.istio.io`)**

- **`PeerAuthentication`**: Cấu hình chế độ mã hóa **mTLS** giữa các Pod (STRICT, PERMISSIVE, hoặc DISABLE).
    
- **`AuthorizationPolicy`**: Phân quyền truy cập giữa các microservices (cho phép/chặn request dựa trên IP, Namespace, ServiceAccount, HTTP Header, JWT Claim).
    
- **`RequestAuthentication`**: Xác thực token người dùng (JWT Validation) ngay tại Envoy Proxy trước khi chuyển tới ứng dụng.
    

#### **Nhóm Telemetry & Extensions (Giám sát & Mở rộng - `telemetry.istio.io` / `extensions.istio.io`)**

- **`Telemetry`**: Cấu hình việc ghi log (Access Logs), thu thập metrics (Prometheus), và tạo distributed tracing (Jaeger/Zipkin).
    
- **`WasmPlugin`**: Cho phép nạp thêm các module mở rộng viết bằng WebAssembly (Wasm) trực tiếp vào Envoy Proxy.
    

### 2. Các Cluster-wide Resources (Tài nguyên phạm vi toàn cụm)

Bên cạnh CRDs, `istio-base` tạo ra các tài nguyên không thuộc về một Namespace cố định nào (Non-namespaced resources) để quản lý phân quyền và kiểm soát hệ thống:

- **`ClusterRole` & `ClusterRoleBinding`**:
    
    - Định nghĩa các quyền tối thiểu trên toàn cụm cho điều khiển viên Istiod (ví dụ: quyền đọc thông tin Pod, Service, Endpoints, Secret của tất cả các Namespace).
        
    - Ví dụ: `istiod-clusterrole-istio-system`, `istio-reader-clusterrole-istio-system`.
        
- **`ValidatingWebhookConfiguration` (`istiod-default-validator`)**:
    
    - Đăng ký một Webhook với Kubernetes API Server. Mỗi khi bạn dùng lệnh `kubectl apply` một file YAML của Istio (như `VirtualService` hay `Gateway`), Webhook này sẽ tự động kiểm tra xem syntax của file có hợp lệ hay không trước khi lưu vào `etcd`.
        
- **`MutatingWebhookConfiguration` (`istio-sidecar-injector`)**:
    
    - Đăng ký Webhook tự động chèn (inject) Envoy Proxy container vào bất kỳ Pod nào mới khởi tạo trong các Namespace có dán nhãn `istio-injection=enabled`.
        

### Tóm tắt thứ tự cài đặt chuẩn bằng Helm

Bash

```
# 1. Cài đặt CRDs & Cluster-wide resources trước
helm install istio-base istio/base -n istio-system --create-namespace

# 2. Sau đó mới cài Control Plane (Istiod)
helm install istiod istio/istiod -n istio-system --wait
```

## `DestinationRule`

Trong hệ sinh thái **Istio Service Mesh**, `DestinationRule` là một CRD (Custom Resource Definition) được sử dụng để **cấu hình các chính sách (policies) áp dụng cho luồng traffic SAU KHÔNG KHI luồng traffic đó đã được định tuyến (routing)** bởi `VirtualService`.

Nếu `VirtualService` đóng vai trò là "biển chỉ đường" (Traffic Routing - quyết định request sẽ đi đến đâu), thì `DestinationRule` đóng vai trò là "quy tắc xử lý tại điểm đích" (Traffic Policies - quyết định request được chuyển giao tới các instance cụ thể như thế nào).

### Các tính năng chính của DestinationRule

1. **Phân chia Subset (Subsets / Versions):**
    
    - Cho phép bạn nhóm các Pod của cùng một Service thành các tập hợp nhỏ hơn dựa trên `labels` (ví dụ: chia theo phiên bản `v1`, `v2`, hoặc môi trường `canary`, `stable`).
        
    - `VirtualService` sẽ dựa vào các `subset` này để điều hướng traffic (ví dụ: 90% vào `v1`, 10% vào `v2`).
        
2. **Thuật toán Cân bằng tải (Load Balancing Algorithms):**
    
    - Định nghĩa cách Envoy Proxy phân phối request đến từng Pod trong một subset/service:
        
        - `ROUND_ROBIN` (Mặc định)
            
        - `RANDOM`
            
        - `LEAST_REQUEST` (Gửi đến Pod đang có ít request nhất)
            
        - `CONSISTENT_HASH` (Dựa trên HTTP Header, Cookie, hoặc Source IP - phục vụ Sticky Session).
            
3. **Cấu hình mã hóa mTLS (TLS Settings):**
    
    - Quản lý chế độ TLS/mTLS từ Envoy Proxy client tới Pod đích: `DISABLE`, `SIMPLE`, `MUTUAL` (mTLS), hoặc `ISTIO_MUTUAL` (dùng chứng chỉ tự động do Istiod cấp).
        
4. **Quản lý kết nối & Ngắt mạch (Circuit Breaking / Connection Pool):**
    
    - **Connection Pool:** Giới hạn số lượng kết nối TCP tối đa, số lượng pending requests tối đa tới service để tránh quá tải.
        
    - **Outlier Detection (Ejection):** Tự động phát hiện và ngắt (loại bỏ tạm thời) các Pod liên tục trả về lỗi HTTP 5xx ra khỏi danh sách Load Balancer trong một khoảng thời gian.
        

### Cấu trúc một file `DestinationRule` mẫu

YAML

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-service-destination-rule
  namespace: default
spec:
  host: my-service.default.svc.cluster.local # Tên Service đích trong K8s
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
    tls:
      mode: ISTIO_MUTUAL # Sử dụng mTLS quản lý bởi Istio
    connectionPool:
      tcp:
        maxConnections: 100 # Giới hạn tối đa 100 kết nối TCP
    outlierDetection: # Cấu hình Circuit Breaker
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1.0.0
    - name: v2
      labels:
        version: v2.0.0
```

### Tóm tắt sự phối hợp: `VirtualService` vs `DestinationRule`

Plaintext

```
[ Incoming Request ]
        │
        ▼
[ VirtualService ]  ──(Quyết định)──>  "Gửi 10% traffic sang subset v2"
        │
        ▼
[ DestinationRule ] ──(Quyết định)──>  "Lựa chọn Pod thuộc v2 bằng thuật toán LEAST_REQUEST,
                                       mã hóa bằng ISTIO_MUTUAL và giới hạn max 100 conns"
        │
        ▼
   [ Target Pod ]
```

## DestinationRule chỉ có trong Istio chứ không có trong k8s ? virtual service giống virtual host trong nginx còn destination rule giống các rule để proxy trong nginx ?

Chính xác 100%! Bạn đã nắm trọn vẹn bản chất và vị trí của hai khái niệm này.

### 1. `DestinationRule` là của Istio hay Kubernetes?

`DestinationRule` (và cả `VirtualService`) **chỉ có trong Istio**, hoàn toàn không thuộc về Kubernetes gốc (K8s core).

Kubernetes gốc chỉ có các tài nguyên cơ bản như `Service`, `Ingress`, `Endpoints`. Istio mở rộng Kubernetes bằng cách cài thêm các Custom Resource Definitions (CRDs) này để phục vụ quản lý luồng mạng nâng cao (Service Mesh).

### 2. So sánh tư duy: Istio vs Nginx

Cách bạn ví von với Nginx rất chuẩn xác và trực quan:

|Khái niệm|Trong Istio|Tương đương trong Nginx|Vai trò chính|
|---|---|---|---|
|**Lớp chỉ đường (Routing)**|`VirtualService`|`server { ... }` block & `location /`|**Match & Route:** Khai báo domain/path nào sẽ đón request và hướng traffic đi đâu (ví dụ: match URI `/api` thì chuyển sang upstream backend).|
|**Lớp quy tắc xử lý (Proxy Rules)**|`DestinationRule`|`upstream { ... }` block & các chỉ thị `proxy_*`|**Traffic Policy:** Định nghĩa tập hợp server đích (subsets), thuật toán load balancing (`round_robin`, `least_conn`), mã hóa mTLS, timeout, retry, ngắt mạch (circuit breaker).|

### Ví dụ đối chiếu cấu hình thực tế

#### Trong Nginx (`nginx.conf`):

Nginx

```
# Tương đương DestinationRule: Định nghĩa danh sách server đích & thuật toán
upstream backend_v1 {
    least_conn; # Thuật toán Load Balancing
    server 10.0.0.1:8080 max_conns=100; # Max connection
}

# Tương đương VirtualService: Định nghĩa Routing rule
server {
    listen 80;
    server_name my-app.com;

    location / {
        proxy_pass http://backend_v1; # Chuyển request tới upstream
    }
}
```

#### Trong Istio (YAML):

YAML

```
# 1. VirtualService (Tương đương server & location block)
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app-vs
spec:
  hosts:
    - "my-app.com"
  http:
    - route:
        - destination:
            host: my-app-service
            subset: v1 # Hướng traffic tới subset v1

---
# 2. DestinationRule (Tương đương upstream block & proxy parameters)
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-app-dr
spec:
  host: my-app-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST # Thuật toán Load Balancing
    connectionPool:
      tcp:
        maxConnections: 100
  subsets:
    - name: v1
      labels:
        version: v1.0.0
```

