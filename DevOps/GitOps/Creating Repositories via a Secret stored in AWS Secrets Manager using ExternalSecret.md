`ExternalSecret` kết nối với AWS **chạy được trên mọi cụm Kubernetes** (Minikube, K3s, cụm tự cài bằng `kubeadm` trên VM, hay cả cụm trên GCP/Azure), không bắt buộc phải là Amazon EKS.

Sự khác biệt duy nhất giữa EKS và các cụm K8s khác chỉ nằm ở **cách xác thực (Authentication)** trong `ClusterSecretStore`:

- **Trên Amazon EKS:** Dùng xác thực động qua **IRSA / EKS Pod Identity** (không cần lưu Access Key tĩnh).
    
- **Trên cụm K8s ngoài EKS (Minikube, On-prem, VM):** Dùng cặp khóa tĩnh **`AWS_ACCESS_KEY_ID` và `AWS_SECRET_ACCESS_KEY`** lưu trong một Kubernetes `Secret` để ESO gọi API ra AWS.
    

### Hướng dẫn Demo `ExternalSecret` với LocalStack + Minikube + Argo CD

Khi chạy **LocalStack** trên máy host (Docker/WSL2) và **Minikube** làm cụm K8s, lưu ý quan trọng nhất là **mạng**: bên trong Pod của ESO, `localhost:4566` là chính bản thân Pod đó chứ không phải máy host. Bạn phải dùng tên miền nội bộ **`[http://host.minikube.internal:4566](http://host.minikube.internal:4566)`** để Minikube gọi ngược ra LocalStack trên máy host.

#### Bước 1: Tạo Secret trên LocalStack Secrets Manager

Đảm bảo LocalStack đang chạy và lắng nghe ở cổng `4566`, sau đó tạo secret chứa thông tin Private Repo:

Bash

```
aws --endpoint-url=http://localhost:4566 secretsmanager create-secret \
  --name argocd/private-repo \
  --region us-east-1 \
  --secret-string '{
    "url": "https://github.com/LeNgocBao145/argocd-tutorial.git",
    "username": "git",
    "password": "<YOUR_GITHUB_PAT>"
  }'
```

_Xác nhận:_ Lệnh trả về JSON chứa `"ARN": "arn:aws:secretsmanager:us-east-1:000000000000:secret:argocd/private-repo-..."`.

#### Bước 2: Cài External Secrets Operator (ESO) và tạo Credentials giả lập trên Minikube

1. Cài đặt ESO bằng [[Helm]]:
    
    Bash
    
    ```
    helm repo add external-secrets https://charts.external-secrets.io
    helm repo update
    helm install external-secrets external-secrets/external-secrets \
      -n external-secrets --create-namespace
    ```
    
    _Xác nhận:_ Chạy `kubectl get pods -n external-secrets` thấy đủ 3 pod (`external-secrets`, `webhook`, `cert-controller`) ở trạng thái `Running`.
    
2. Tạo `Secret` chứa cặp khóa AWS giả lập (LocalStack chấp nhận mọi chuỗi bất kỳ như `test`):
    
    Bash
    
    ```
    kubectl create secret generic aws-localstack-creds \
      -n external-secrets \
      --from-literal=access-key=test \
      --from-literal=secret-access-key=test
    ```
    

#### Bước 3: Tạo `ClusterSecretStore` trỏ về LocalStack

Sử dụng trường **`additionalEndpoints`** để ép ESO chuyển hướng yêu cầu gọi AWS Secrets Manager về cổng `4566` của LocalStack thay vì AWS thật. Tạo file `cluster-secret-store.yaml`:

YAML

```
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      additionalEndpoints:
        SecretsManager: http://host.minikube.internal:4566
        STS: http://host.minikube.internal:4566
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-localstack-creds
            namespace: external-secrets
            key: access-key
          secretAccessKeySecretRef:
            name: aws-localstack-creds
            namespace: external-secrets
            key: secret-access-key
```

_(Lưu ý: Nếu bạn cài LocalStack chạy dưới dạng Pod bên trong Minikube thay vì chạy ngoài Docker host, hãy đổi URL thành `http://<tên-service-localstack>.<namespace>.svc.cluster.local:4566`)._

Áp dụng và kiểm tra trạng thái kết nối:

Bash

```
kubectl apply -f cluster-secret-store.yaml
kubectl get clustersecretstore aws
```

_Xác nhận:_ Cột `STATUS` hiển thị `Valid` và `READY` là `True`.

#### Bước 4: Tạo `ExternalSecret` để đồng bộ sang Argo CD

Tạo file `external-secret-repo.yaml` ánh xạ 3 trường `url`, `username`, `password` từ LocalStack vào `Secret` của Argo CD:

YAML

```
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: private-repo
  namespace: argocd
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: aws
    kind: ClusterSecretStore
  target:
    name: private-repo
    creationPolicy: Owner
    template:
      type: Opaque
      metadata:
        labels:
          argocd.argoproj.io/secret-type: repository
  data:
    - secretKey: url
      remoteRef:
        key: argocd/private-repo
        property: url
    - secretKey: username
      remoteRef:
        key: argocd/private-repo
        property: username
    - secretKey: password
      remoteRef:
        key: argocd/private-repo
        property: password
```

Áp dụng và kiểm tra:

Bash

```
kubectl apply -f external-secret-repo.yaml
kubectl get externalsecret private-repo -n argocd
kubectl get secret private-repo -n argocd --show-labels
```

_Xác nhận:_ `ExternalSecret` báo trạng thái `SecretSynced`, `Secret` tên `private-repo` tự động xuất hiện trong namespace `argocd` kèm nhãn `argocd.argoproj.io/secret-type=repository`, và trên Argo CD Web UI (**Settings $\rightarrow$ Repositories**) tự động nhận diện được repo của bạn.

**Lấy IP của `localstack-aws` trên mạng Docker `minikube` và cập nhật lại biến môi trường cho ESO:**

Bash

```
LOCALSTACK_IP=$(docker inspect -f '{{.NetworkSettings.Networks.minikube.IPAddress}}' localstack-aws)
echo "LocalStack IP: $LOCALSTACK_IP"
kubectl set env deployment/external-secrets -n external-secrets AWS_ENDPOINT_URL="http://${LOCALSTACK_IP}:4566"
kubectl rollout status deployment/external-secrets -n external-secrets
```

_Xác nhận:_ `echo` in ra một địa chỉ IP thuộc dải mạng Minikube (ví dụ `192.168.49.3`) và `kubectl rollout status` báo `deployment "external-secrets" successfully rolled out`.