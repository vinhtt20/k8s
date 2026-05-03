# 🎓 BÀI GIẢNG: KUBERNETES SERVICE TỪ A → Z

> **Giáo viên:** Chuyên gia Kubernetes
> **Đối tượng:** Người đã biết khái niệm Pod / Deployment, muốn hiểu sâu Service
> **Thời lượng đọc:** ~45 phút + tương tác với nhiều animation HTML

---

## 📂 Cách dùng tài liệu này

Tài liệu gồm 2 phần song song:

1. **File lý thuyết** (chính là file này) — đọc tuần tự từ trên xuống.
2. **Các file HTML animation** — mở trong trình duyệt khi gặp mục `▶ Mở animation: ...`. Mỗi animation tương tác được, click chạy lại được.

👉 **Bắt đầu nhanh:** mở [index.html](index.html) — trang mục lục dẫn đến mọi animation.

| # | File animation | Nội dung |
|---|----------------|----------|
| 1 | [01-pod-ephemeral.html](01-pod-ephemeral.html) | Vấn đề: Pod IP thay đổi |
| 2 | [02-service-loadbalancing.html](02-service-loadbalancing.html) | Service phân phối traffic |
| 3 | [03-service-types.html](03-service-types.html) | 4 loại Service |
| 4 | [04-kube-proxy.html](04-kube-proxy.html) | kube-proxy & iptables |
| 5 | [05-selector-matching.html](05-selector-matching.html) | Cơ chế selector/label |
| 6 | [06-headless-vs-clusterip.html](06-headless-vs-clusterip.html) | Headless Service |
| 7 | [07-packet-journey-deep.html](07-packet-journey-deep.html) | Đường đi thật của packet |
| 9 | [09-service-types-detailed.html](09-service-types-detailed.html) | 5 loại Service bằng ẩn dụ + kỹ thuật |
| 10 | [10-service-comparison-animated.html](10-service-comparison-animated.html) | So sánh đường đi của 5 loại Service |
| 11 | [11-ingress-clusterip-vs-loadbalancer.html](11-ingress-clusterip-vs-loadbalancer.html) | Ingress + ClusterIP vs LoadBalancer |

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

## 📌 PHẦN 6: INGRESS + CLUSTERIP KHÁC LOADBALANCER NHƯ THẾ NÀO?

> ▶ **Mở animation:** [`11-ingress-clusterip-vs-loadbalancer.html`](11-ingress-clusterip-vs-loadbalancer.html)
> Bấm từng bước để xem request đi từ Internet vào Ingress Controller, rồi qua ClusterIP Service đến Pod.

### Hãy tưởng tượng Kubernetes là một tòa nhà

- **LoadBalancer trực tiếp** = mỗi phòng mở một cửa riêng ra đường lớn.
- **Ingress + ClusterIP** = tòa nhà có một sảnh lễ tân chung. Khách đi vào một cửa, lễ tân nhìn biển hiệu rồi dẫn đến đúng phòng.

### Đường đi khi dùng Ingress + ClusterIP

```
User
  │ gõ https://shop.example.com/api
  ▼
DNS
  │ trả về IP public của Load Balancer đứng trước Ingress Controller
  ▼
Cloud Load Balancer
  │ nếu dùng HTTPS: TLS handshake / kiểm tra certificate
  ▼
Ingress Controller Pod
  │ đọc Host/Path: shop.example.com + /api
  ▼
Ingress Rule
  │ quyết định chuyển đến api-svc
  ▼
ClusterIP Service
  │ chọn 1 backend Pod khỏe mạnh
  ▼
API Pod
```

Điểm cực kỳ quan trọng:

- **Ingress** chỉ là luật route HTTP/HTTPS.
- **Ingress Controller** mới là thành phần nhận traffic thật và thực thi luật đó.
- Backend phía sau Ingress thường là **ClusterIP Service**, vì backend không cần mở public trực tiếp ra Internet.

### Bóc tách cực chi tiết: 13 bước từ Browser đến Pod

Ví dụ người dùng mở:

```text
https://shop.example.com/api/orders
```

Hãy tách URL này thành 3 phần:

- `https` = dùng HTTPS, nên sẽ có TLS/SSL.
- `shop.example.com` = host/domain, dùng cho DNS và Ingress host rule.
- `/api/orders` = path, dùng cho Ingress path rule.

Quy trình đầy đủ:

| Bước | Chuyện xảy ra | Ai quyết định? |
|------|---------------|----------------|
| 1 | Browser nhìn URL và thấy cần kết nối HTTPS tới `shop.example.com`. | Browser |
| 2 | Browser hỏi DNS: `shop.example.com` là IP nào? | DNS resolver |
| 3 | DNS trả về A/AAAA record hoặc CNAME tới hostname của Load Balancer. | DNS |
| 4 | Browser mở TCP connection tới IP public, port `443`. | Browser + network |
| 5 | Browser gửi TLS ClientHello, kèm **SNI** = `shop.example.com`. | Browser |
| 6 | Load Balancer hoặc Ingress Controller trả certificate phù hợp. | TLS termination point |
| 7 | Browser kiểm tra certificate đúng domain, còn hạn, CA tin cậy. | Browser |
| 8 | Browser gửi HTTP request bên trong HTTPS: `GET /api/orders`, `Host: shop.example.com`. | Browser |
| 9 | Public Load Balancer chuyển traffic vào target khỏe trong cluster. | Cloud Load Balancer |
| 10 | Ingress Controller đọc `Host` + `Path`, match Ingress rule. | Ingress Controller |
| 11 | Controller chuyển request đến `api-svc:80`, tức ClusterIP Service. | Ingress Controller |
| 12 | kube-proxy/IPVS/iptables chọn một Endpoint Pod, ví dụ `10.1.2.8:8080`. | Kubernetes networking |
| 13 | API Pod xử lý request và response quay ngược về Browser theo kết nối cũ. | App + network stack |

Ẩn dụ đời thường:

- **DNS** giống hỏi bản đồ: “tòa nhà này nằm ở địa chỉ nào?”.
- **TCP** giống mở đường dây điện thoại tới tòa nhà.
- **TLS/SSL** giống kiểm tra căn cước của tòa nhà rồi tạo đường dây riêng không ai nghe lén được.
- **Load Balancer** giống cửa bảo vệ chọn thang máy/cửa đang hoạt động.
- **Ingress Controller** giống lễ tân đọc “phòng API” hay “phòng Web”.
- **Service ClusterIP** giống số máy nội bộ ổn định.
- **EndpointSlice/Pod** là nhân viên thật đang xử lý công việc.

### DNS hoạt động như thế nào trong luồng này?

DNS là bước đầu tiên để biến tên dễ nhớ thành địa chỉ mạng:

```
shop.example.com  →  IP public hoặc hostname của Load Balancer
```

Các record hay gặp:

- **A record**: domain trỏ trực tiếp tới IPv4 public.
- **AAAA record**: domain trỏ tới IPv6 public.
- **CNAME record**: domain trỏ tới hostname khác, ví dụ hostname của AWS ALB/GCP Load Balancer.
- **TTL**: thời gian cache DNS. TTL cao thì client giữ kết quả lâu; TTL thấp thì đổi record nhanh có hiệu lực hơn.

Điểm cần nhớ: **DNS chỉ đưa người dùng tới cửa public**. DNS không biết Pod nào xử lý request, cũng không route `/api` hay `/web`. Việc route theo host/path là việc của **Ingress Controller**.

### Khi có SSL/TLS thì chuyện gì xảy ra?

Với `https://shop.example.com`, sau khi DNS trả IP, trình duyệt mở kết nối tới IP đó và làm **TLS handshake**:

1. Trình duyệt nói domain muốn truy cập, thường qua **SNI**.
2. Load Balancer hoặc Ingress Controller trả về certificate cho `shop.example.com`.
3. Trình duyệt kiểm tra certificate có đúng domain, còn hạn, và được CA tin cậy không.
4. Nếu hợp lệ, hai bên tạo kênh mã hóa HTTPS.
5. Sau khi HTTPS được giải mã ở điểm **TLS termination**, Ingress Controller mới đọc được HTTP Host/Path để route.

TLS termination có thể đặt ở:

- **Cloud Load Balancer**: LB giải mã HTTPS rồi gửi HTTP/HTTPS vào cluster.
- **Ingress Controller**: LB chuyển TCP/HTTPS vào cluster, Controller giữ certificate và giải mã.
- **Trong Pod**: ít phổ biến hơn cho web app thông thường, nhưng có thể dùng nếu app tự quản TLS.

Ví dụ Ingress có TLS:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
spec:
  tls:
    - hosts:
        - shop.example.com
      secretName: shop-tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
```

`shop-tls` là Kubernetes Secret chứa certificate và private key. Trong production, nhiều team dùng **cert-manager** để tự xin và tự gia hạn certificate từ Let's Encrypt hoặc CA nội bộ.

### Đường đi khi dùng Service type LoadBalancer trực tiếp

```
User
  │ gọi IP/domain public của Service
  ▼
DNS
  │ trả về IP public/hostname của Load Balancer riêng
  ▼
Cloud Load Balancer
  │ có thể terminate TLS tại đây, tùy cloud/config
  ▼
Service type LoadBalancer
  │ bên dưới vẫn có NodePort + ClusterIP
  ▼
Pod
```

Mô hình này đơn giản: một cửa public đi vào một Service. Nhưng nếu bạn có 10 app public, rất có thể bạn tạo 10 Load Balancer riêng, vừa tốn tiền vừa khó quản lý TLS/domain.

### Khi nào chọn cái nào?

| Tình huống | Nên chọn |
|------------|----------|
| Website/API HTTP(S), nhiều path như `/`, `/api`, `/admin` | **Ingress + ClusterIP** |
| Nhiều service backend chỉ gọi nội bộ | **ClusterIP** |
| Một app cần public nhanh trên cloud | **LoadBalancer** |
| Traffic không phải HTTP/HTTPS, ví dụ TCP đặc biệt | **LoadBalancer** hoặc giải pháp L4 riêng |

👉 Câu nhớ nhanh: **Ingress là một cửa thông minh cho nhiều Service; LoadBalancer Service là một cửa riêng cho một Service.**

---

## 📌 PHẦN 7: BÀI TẬP THỰC HÀNH

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
                    Ingress + ClusterIP
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
| **Ingress + ClusterIP** | Public HTTP(S) qua một cửa chung | Route nhiều app theo host/path |

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
├── 06-headless-vs-clusterip.html       ← Animation: Headless
├── 07-packet-journey-deep.html         ← Animation: packet deep-dive
├── 08-K8s-Networking-Deep-Dive.md      ← Lý thuyết deep-dive
├── 09-service-types-detailed.html      ← Bài giảng ẩn dụ + kỹ thuật
├── 10-service-comparison-animated.html ← So sánh đường đi mạng
└── 11-ingress-clusterip-vs-loadbalancer.html ← Ingress vs LoadBalancer
```

> 💡 **Mở [index.html](index.html) trong trình duyệt** để bắt đầu trải nghiệm tương tác.

---

*Chúc bạn học vui và áp dụng tốt vào production! 🚀*
