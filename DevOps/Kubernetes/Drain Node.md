CO**Drain Node** (`kubectl drain <node-name>`) là hành động **di dời an toàn toàn bộ Pod** ra khỏi một Node cụ thể để chuẩn bị bảo trì, nâng cấp phần cứng, nâng cấp phiên bản Kubernetes hoặc xóa Node đó khỏi cluster.

Khi bạn thực hiện lệnh `kubectl drain`:

1. **Cordoning (Đánh dấu không tiếp nhận):** Node sẽ tự động bị đánh dấu là `SchedulingDisabled` (không cho phép tạo Pod mới trên Node này nữa).
    
2. **Eviction (Bảo vệ và di dời):** K8s sẽ xóa (evict) các Pod hiện tại trên Node đó một cách êm đẹp (Gracefully). Các Pod thuộc các Controller như `Deployment` hay `ReplicaSet` sẽ tự động được hạ tầng dựng lại trên các Node khác còn trống.
    
3. **Hoàn tất:** Khi Node hoàn toàn sạch Pod, bạn có thể an toàn tắt máy hoặc can thiệp kỹ thuật.
    

> **Lưu ý:** `kubectl drain` mặc định sẽ bỏ qua các Pod dạng `DaemonSet` (vì DaemonSet chạy trên mọi Node) và yêu cầu cờ `--ignore-daemonsets`.