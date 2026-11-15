**Pod Disruption Budget (PDB)** là một đối tượng (object) trong Kubernetes dùng để giới hạn số lượng Pod của ứng dụng bị dừng cùng một lúc trong quá trình **Voluntary Disruption** (gián đoạn có kế hoạch).

PDB đảm bảo ứng dụng luôn giữ đủ số lượng Pod tối thiểu để duy trì tính sẵn sàng (High Availability), ngăn chặn rủi ro downtime khi quản trị viên thực hiện bảo trì cluster.

### Phân biệt Gián đoạn trong Kubernetes

|**Loại Gián đoạn**|**Nguyên nhân**|**PDB có bảo vệ được không?**|
|---|---|---|
|**Voluntary Disruption** _(Có kế hoạch)_|Dùng `kubectl drain` để bảo trì node, nâng cấp kernel, autoscaling hạ số lượng node|**Có** — PDB sẽ chặn hành động drain nếu việc xóa Pod vi phạm ngưỡng an toàn.|
|**Involuntary Disruption** _(Không kế hoạch)_|Phần cứng node hỏng, sập nguồn, hư ổ đĩa, Out of Memory (OOM)|**Không** — Sự cố vật lý xảy ra tức thì, Kubernetes không thể ngăn cản.|

### Cách cấu hình PDB

Bạn cấu hình PDB bằng cách thiết lập **`minAvailable`** (số Pod tối thiểu phải chạy) hoặc **`maxUnavailable`** (số Pod tối đa có thể bị gián đoạn cùng lúc).

#### 1. Sử dụng `minAvailable`

Đảm bảo luôn có **ít nhất 2 Pod** chạy ổn định:

```
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2 # Có thể dùng phần trăm, ví dụ: 80%
  selector:
    matchLabels:
      app: my-app
```

#### 2. Sử dụng `maxUnavailable`

Chỉ cho phép xóa **tối đa 1 Pod** tại một thời điểm:

```
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  maxUnavailable: 1 # Hoặc dùng phần trăm, ví dụ: 20%
  selector:
    matchLabels:
      app: my-app
```

### Nguyên lý hoạt động thực tế

Giả sử bạn có ứng dụng chạy **4 replicas** và PDB cấu hình `minAvailable: 3`:

1. Quản trị viên chạy lệnh `kubectl drain node-1` để nâng cấp node.
    
2. Kubernetes kiểm tra PDB: Xóa 1 Pod trên `node-1` làm tổng số Pod giảm xuống `3` (vẫn thỏa mãn `minAvailable: 3`). Hành động xóa được chấp thuận.
    
3. Nếu quản trị viên tiếp tục drain `node-2` ngay lập tức, PDB thấy số Pod hiện tại là `3`, nếu xóa tiếp sẽ còn `2` (vi phạm PDB).
    
4. Lệnh `kubectl drain` bị **hoãn lại (block)** cho đến khi Pod mới trên `node-1` được schedule và đạt trạng thái `Ready`.