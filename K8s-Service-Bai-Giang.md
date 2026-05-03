# 🎓 BÀI GIẢNG: KUBERNETES SERVICE TỪ A → Z

> **Giáo viên:** Chuyên gia Kubernetes
> **Đối tượng:** Người đã biết khái niệm Pod / Deployment, muốn hiểu sâu Service
> **Thời lượng đọc:** ~30 phút + tương tác với 6 animation HTML

---

## 📂 Cách dùng tài liệu này

Tài liệu gồm 2 phần song song:

1. **File lý thuyết** (chính là file này) — đọc tuần tự từ trên xuống.
2. **6 file HTML animation** — mở trong trình duyệt khi gặp mục `▶ Mở animation: ...`. Mỗi animation tương tác được, click chạy lại được.

👉 **Bắt đầu nhanh:** mở [index.html](index.html) — trang mục lục dẫn đến mọi animation.

| # | File animation | Nội dung |
|---|----------------|----------|
| 1 | [01-pod-ephemeral.html](01-pod-ephemeral.html) | Vấn đề: Pod IP thay đổi |
| 2 | [02-service-loadbalancing.html](02-service-loadbalancing.html) | Service phân phối traffic |
| 3 | [03-service-types.html](03-service-types.html) | 4 loại Service |
| 4 | [04-kube-proxy.html](04-kube-proxy.html) | kube-proxy & iptables |
| 5 | [05-selector-matching.html](05-selector-matching.html) | Cơ chế selector/label |
| 6 | [06-headless-vs-clusterip.html](06-headless-vs-clusterip.html) | Headless Service |

---

## 📌 PHẦN 1: TẠI SAO CHÚNG TA CẦN SERVICE?

### Vấn đề cốt lõi: Pod là **ephemeral** (tạm thời)

Trong Kubernetes, mỗi Pod có 1 IP riêng. Nhưng IP này **KHÔNG ổn định**:

- Pod crash → ReplicaSet tạo Pod mới → **IP MỚI hoàn toàn**.
- Scale up/down → Pod đến và đi liên tục.
- Rolling update → toàn bộ Pod được thay thế.

> ▶ **Mở animation:** [`01-pod-ephemeral.html`](01-pod-ephemeral.html)
> Xem Pod B chết, được tái sinh với IP mới — và client lưu IP cũ → kết nối FAIL.

```
        ┌──────────────────────────────────────────┐
        │           KUBERNETES CLUSTER             │
        │   ┌─────────┐  ┌─────────┐  ┌─────────┐ │
        │   │  Pod A  │  │  Pod B  │  │  Pod C  │ │
        │   │10.1.1.5 │  │10.1.1.6 │  │10.1.1.7 │ │
        │   └─────────┘  └─────────┘  └─────────┘ │
        └──────────────────────────────────────────┘
                            │
                            │ Pod B chết 💀
                            ▼
        ┌──────────────────────────────────────────┐
        │   ┌─────────┐  ┌─────────┐  ┌─────────┐ │
        │   │  Pod A  │  │  Pod B' │  │  Pod C  │ │
        │   │10.1.1.5 │  │10.1.1.42│  │10.1.1.7 │ │  ← IP MỚI!
        │   └─────────┘  └─────────┘  └─────────┘ │
        └──────────────────────────────────────────┘
```

👉 **Hậu quả:** nếu client lưu cứng IP `10.1.1.6`, sau khi Pod B chết, client gọi → timeout.

> 💡 **Service ra đời để giải quyết:** cung cấp 1 IP/DNS **CỐ ĐỊNH** đứng trước nhóm Pod.

---

## 📌 PHẦN 2: SERVICE LÀ GÌ?

**Service** = lớp trừu tượng cung cấp:
- Một **địa chỉ mạng ổn định** (IP ảo + DNS) đứng trước 1 nhóm Pod.
- **Load balancing** giữa các Pod đó.

> ▶ **Mở animation:** [`02-service-loadbalancing.html`](02-service-loadbalancing.html)
> Click "Gửi 10 requests" để xem Service chia đều traffic theo round-robin.

```
              ┌──────────────────────┐
              │   CLIENT (Frontend)  │
              └──────────┬───────────┘
                         │
                         │ gọi http://my-service:80
                         ▼
              ┌──────────────────────┐
              │      SERVICE         │ ◄── IP cố định: 10.96.0.10
              │   "my-service"       │     DNS: my-service.default.svc
              │   selector:          │
              │     app: backend     │
              └──────────┬───────────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
       ┌────────┐   ┌────────┐   ┌────────┐
       │ Pod 1  │   │ Pod 2  │   │ Pod 3  │
       │app:    │   │app:    │   │app:    │
       │backend │   │backend │   │backend │
       └────────┘   └────────┘   └────────┘
       10.1.1.5     10.1.1.6     10.1.1.7
```

### 🔑 Cơ chế Selector — Service tìm Pod bằng cách nào?

Service **KHÔNG** lưu danh sách Pod cụ thể. Nó dùng **labels** (key-value) để match động:

```yaml
# Service
selector:
  app: backend

# Pod
metadata:
  labels:
    app: backend     # ✅ MATCH → Service "thấy" Pod này
    tier: api
```

> ▶ **Mở animation:** [`05-selector-matching.html`](05-selector-matching.html)
> Click các button để thay đổi selector và xem Pod nào match / không match. Thử button "label sai!" để thấy Endpoints rỗng.

⚠️ **Điều quan trọng:**
- Selector sai 1 ký tự → 0 Pod match → **Endpoints rỗng** → traffic FAIL (503).
- Đây là **lỗi #1** khi debug Service. Luôn `kubectl get endpoints <svc>` đầu tiên!

---

## 📌 PHẦN 3: 4 LOẠI SERVICE — TỪNG LOẠI GIẢI QUYẾT GÌ?

> ▶ **Mở animation:** [`03-service-types.html`](03-service-types.html)
> Click 4 tab để xem luồng traffic của từng loại.

### 1️⃣ ClusterIP (mặc định) — chỉ truy cập **TRONG cluster**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP        # mặc định, có thể bỏ
  selector:
    app: backend
  ports:
    - port: 80           # port của Service
      targetPort: 8080   # port của Pod
```

**Đặc điểm:**
- Service được gán 1 IP **ảo** trong dải `clusterIP CIDR` (vd: 10.96.0.0/12).
- IP này chỉ định tuyến được trong cluster — **Internet KHÔNG vào được**.
- DNS: `<svc-name>.<namespace>.svc.cluster.local`.

**Use case:** giao tiếp service-to-service nội bộ (frontend → backend, backend → DB).

---

### 2️⃣ NodePort — mở 1 port trên **MỌI Node**

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080    # tùy chọn, range mặc định: 30000–32767
```

**Đặc điểm:**
- K8s mở port (vd: `30080`) trên TẤT CẢ các Node trong cluster.
- Truy cập `<bất kỳ NodeIP nào>:<NodePort>` → kube-proxy forward → Pod.
- Tự động tạo kèm 1 ClusterIP bên trong.

**Use case:** dev/test, demo nhanh. **Không khuyên dùng production** vì:
- Phải biết IP của Node (Node có thể thay đổi).
- Không có TLS/SSL termination.
- Port range bị giới hạn.

---

### 3️⃣ LoadBalancer — phổ biến nhất cho **production trên cloud**

```yaml
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

**Đặc điểm:**
- K8s **gọi API của cloud provider** (AWS/GCP/Azure) tự động tạo 1 Load Balancer thật, gán IP public.
- Traffic flow: `User → Cloud LB → Service → Pod`.
- Tự tạo kèm NodePort + ClusterIP bên dưới.

**Use case:** expose ứng dụng ra Internet trên môi trường cloud.

⚠️ **Hạn chế:** mỗi LoadBalancer = 1 IP public = **TỐN TIỀN** (~$18/tháng trên AWS).
👉 **Giải pháp:** dùng **Ingress** (1 LoadBalancer duy nhất, route nhiều service theo path/host).

---

### 4️⃣ ExternalName — trỏ DNS đến **dịch vụ ngoài cluster**

```yaml
spec:
  type: ExternalName
  externalName: db.example.com
```

**Đặc điểm:**
- KHÔNG có ClusterIP, KHÔNG có Pod selector.
- Chỉ là 1 **CNAME DNS record**: khi Pod query `my-db.default.svc` → CoreDNS trả về `db.example.com`.

**Use case:** DB của bạn là AWS RDS / Cloud SQL bên ngoài, nhưng Pod muốn gọi qua DNS nội bộ ổn định:

```
App Pod → "postgres.prod.svc" → CNAME → "myapp-prod.abc.rds.amazonaws.com"
```

Lợi ích: nếu sau này migrate DB sang trong cluster, chỉ cần đổi Service — code không phải sửa.

---

## 📌 PHẦN 4: SERVICE HOẠT ĐỘNG NHƯ THẾ NÀO BÊN TRONG?

Đây là phần **nâng cao** mà ít tài liệu giải thích kỹ. Hiểu phần này = bạn debug Service ở mức master.

> ▶ **Mở animation:** [`04-kube-proxy.html`](04-kube-proxy.html)
> Click "Mô phỏng request" để xem packet đi qua từng chain iptables.

### Kiến trúc tổng quan

```
   ┌────────────────────────────────────────────────────┐
   │                  CONTROL PLANE                     │
   │   ┌─────────────┐         ┌──────────────────┐    │
   │   │  API Server│◄────────│ Endpoints        │    │
   │   │   (etcd)    │         │ Controller       │    │
   │   └─────┬───────┘         └──────────────────┘    │
   └─────────┼──────────────────────────────────────────┘
             │ watch
             ▼
   ┌────────────────────────────────────────────────────┐
   │                   WORKER NODE                      │
   │   ┌─────────────┐                                  │
   │   │ kube-proxy  │  ◄── chạy trên MỌI node          │
   │   └──────┬──────┘                                  │
   │          │ ghi rules                               │
   │          ▼                                         │
   │   ┌──────────────────────────────────────────┐    │
   │   │  iptables / IPVS (kernel)                │    │
   │   │  IF dest = 10.96.0.10:80                 │    │
   │   │  THEN DNAT → 10.1.1.5:8080 (random)      │    │
   │   │           OR → 10.1.1.6:8080             │    │
   │   │           OR → 10.1.1.7:8080             │    │
   │   └──────────────────────────────────────────┘    │
   └────────────────────────────────────────────────────┘
```

### Quy trình hoàn chỉnh

| Bước | Thành phần | Công việc |
|------|-----------|-----------|
| 1 | User | `kubectl apply -f service.yaml` |
| 2 | API Server | Lưu Service object vào **etcd** |
| 3 | **Endpoints Controller** | Quét Pod nào match selector → tạo object **Endpoints** chứa list `IP:port` |
| 4 | **kube-proxy** | "Watch" thay đổi từ API Server, cập nhật **iptables/IPVS rules** trên Node |
| 5 | Client | Gửi packet đến `10.96.0.10:80` |
| 6 | Kernel (iptables) | DNAT packet → đổi destination thành `10.1.1.x:8080` (chọn ngẫu nhiên) |
| 7 | Pod | Nhận packet, xử lý, trả response |

### 💡 Insight quan trọng

> **Service KHÔNG phải 1 process đang chạy!**
> Nó chỉ là **rule trong iptables** của Linux kernel.
> Vì xử lý ở kernel space (không qua user space proxy) → cực nhẹ và cực nhanh.

### iptables vs IPVS mode

kube-proxy có 2 mode chính:

| Tiêu chí | iptables (mặc định) | IPVS |
|----------|---------------------|------|
| Performance | O(n) — chậm dần khi nhiều Service | O(1) — hash table |
| Phù hợp | < 1000 Services | > 1000 Services, cluster lớn |
| Thuật toán LB | Random (probability) | Round-robin, least conn, hash, ... |
| Kích hoạt | Mặc định | `--proxy-mode=ipvs` |

---

## 📌 PHẦN 5: HEADLESS SERVICE — TRƯỜNG HỢP ĐẶC BIỆT

> ▶ **Mở animation:** [`06-headless-vs-clusterip.html`](06-headless-vs-clusterip.html)
> So sánh kết quả `nslookup` giữa 2 loại Service.

```yaml
spec:
  clusterIP: None    # ← điểm KHÁC BIỆT duy nhất!
  selector:
    app: db
```

### Khác biệt mấu chốt

| | ClusterIP thường | Headless |
|---|---|---|
| `clusterIP` | có IP ảo (10.96.x.x) | `None` |
| iptables rules | có (DNAT) | không có |
| DNS trả về | 1 IP (Service IP) | **danh sách IP của tất cả Pod** |
| Load balancing | kernel làm hộ | **client tự làm** |

### Khi nào dùng Headless?

- **StatefulSet** (Kafka, MongoDB replica set, Cassandra, Elasticsearch) — mỗi Pod có vai trò khác nhau (master/replica), client phải kết nối đúng Pod.
- **Service discovery** trong code (Spring Cloud, gRPC client-side LB).
- gRPC long-lived connections — không muốn iptables xen vào.

---

## 📌 PHẦN 6: BÀI TẬP THỰC HÀNH

### Bài 1: Tạo ClusterIP Service đơn giản

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

### Câu lệnh kiểm tra

```bash
# Apply
kubectl apply -f deployment.yaml -f service.yaml

# Kiểm tra
kubectl get svc nginx-svc
kubectl get endpoints nginx-svc      # ← LUÔN check khi debug!
kubectl describe svc nginx-svc

# Test từ trong cluster
kubectl run test --rm -it --image=busybox -- sh
# trong shell:
wget -qO- http://nginx-svc
nslookup nginx-svc
```

### Bài 2: Debug khi Service không hoạt động

Checklist 4 bước khi traffic không tới Pod:

```bash
# 1. Service có tồn tại không?
kubectl get svc <name>

# 2. Endpoints có Pod IP không?
kubectl get endpoints <name>
# → nếu RỖNG → selector sai label

# 3. Pod có đúng label không?
kubectl get pods --show-labels

# 4. Pod có healthy không? Container có lắng nghe đúng port không?
kubectl exec -it <pod> -- netstat -tlnp
```

---

## 🎯 TÓM TẮT — BẢN ĐỒ TƯ DUY

```
                      SERVICE
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   "Tại sao?"      "Loại nào?"     "Hoạt động?"
        │                │                │
   Pod IP đổi       ┌────┴─────┐    kube-proxy
   liên tục →       │          │    + iptables
   cần IP           ClusterIP  ExternalName
   cố định          NodePort   (kernel space)
                    LoadBalancer
                         │
                  Headless (clusterIP: None)
                  → cho StatefulSet
```

| Loại | Phạm vi | Khi nào dùng |
|------|---------|--------------|
| **ClusterIP** | Trong cluster | Service nội bộ (mặc định) |
| **NodePort** | NodeIP:port | Dev/test |
| **LoadBalancer** | Public IP | Production trên cloud |
| **ExternalName** | DNS CNAME | Trỏ ra DB ngoài |
| **Headless** | DNS trả nhiều IP | StatefulSet |

---

## 💡 LỜI KHUYÊN TỪ "GIÁO VIÊN"

1. **90% trường hợp** bạn dùng **ClusterIP** + **Ingress** (không phải LoadBalancer cho từng service).
2. Luôn `kubectl get endpoints <service>` khi debug — nếu rỗng → selector sai label.
3. Service **không phải proxy chạy ở user space** — nó là rule kernel → cực nhanh.
4. Đừng nhầm `port` (Service) và `targetPort` (Pod) — đây là lỗi #1 của người mới.
5. Production trên cloud: dùng **Ingress Controller** (nginx-ingress, Traefik) thay vì N LoadBalancer.

---

## 📚 BƯỚC TIẾP THEO

Sau khi nắm chắc Service, hãy học:

1. **Ingress** — lớp HTTP routing trên Service (path-based, host-based)
2. **NetworkPolicy** — firewall cho Pod (giới hạn ai gọi được ai)
3. **Service Mesh** (Istio, Linkerd) — quản lý traffic nâng cao (canary, retry, mTLS)
4. **CoreDNS deep-dive** — cách DNS resolution hoạt động trong K8s

---

## 🗂 CẤU TRÚC FILE

```
/Users/vinh/Desktop/service/
├── index.html                          ← MỤC LỤC (mở file này trước)
├── K8s-Service-Bai-Giang.md            ← Lý thuyết (file này)
├── 01-pod-ephemeral.html               ← Animation: Pod IP đổi
├── 02-service-loadbalancing.html       ← Animation: load balance
├── 03-service-types.html               ← Animation: 4 loại Service
├── 04-kube-proxy.html                  ← Animation: kube-proxy/iptables
├── 05-selector-matching.html           ← Animation: selector
└── 06-headless-vs-clusterip.html       ← Animation: Headless
```

> 💡 **Mở [index.html](index.html) trong trình duyệt** để bắt đầu trải nghiệm tương tác.

---

*Chúc bạn học vui và áp dụng tốt vào production! 🚀*
