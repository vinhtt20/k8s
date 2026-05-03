# 🔬 KUBERNETES NETWORKING — DEEP DIVE: ĐƯỜNG ĐI THẬT CỦA PACKET

> **Bài giảng cấp chuyên gia** — dành cho người đã hiểu cơ bản về Service và muốn đi đến tận **kernel-level**.
> Sau bài này bạn sẽ đọc được output `iptables-save | grep KUBE`, debug được mọi vấn đề networking, và hiểu vì sao Cilium/eBPF lại nhanh hơn iptables mode.

> ▶ **Mở animation tương tác:** [`07-packet-journey-deep.html`](07-packet-journey-deep.html) — 4 scene, mỗi scene 8-10 bước, có thể click "Bước tiếp" theo nhịp đọc.

---

## 📚 MỤC LỤC

1. [Các thành phần kernel cần biết](#part-1)
2. [Scene A — Pod ↔ Pod cùng node (không Service)](#part-2)
3. [Scene B — Pod → Service → Pod cùng node](#part-3)
4. [Scene C — Cross-node traffic & VXLAN](#part-4)
5. [Scene D — External → NodePort → Pod (vì sao mất IP gốc?)](#part-5)
6. [conntrack — kẻ vô danh đứng sau mọi NAT](#part-6)
7. [iptables vs IPVS vs eBPF (Cilium)](#part-7)
8. [Debug toolkit thực chiến](#part-8)

---

<a name="part-1"></a>
## 🔧 PHẦN 1: CÁC THÀNH PHẦN KERNEL CẦN BIẾT

Trước khi đọc đường đi packet, bạn phải nắm được "diễn viên":

### 1.1. Network Namespace (netns)

Mỗi Pod có **1 network namespace riêng** — như 1 "máy ảo mạng" độc lập:
- Có interface, routing table, iptables rules **độc lập** với host.
- 2 Pod khác nhau cùng có interface tên `eth0` mà không xung đột.

```bash
# Kiểm tra netns của 1 Pod
$ kubectl exec my-pod -- ip addr show
1: lo: ...
3: eth0@if12: <BROADCAST,MULTICAST,UP> ...
   inet 10.244.1.5/24 ...
```

### 1.2. veth pair — "đường hầm" giữa 2 namespace

`veth` (virtual Ethernet) luôn đi theo cặp 2 đầu. Packet bỏ vào đầu này → bay ra đầu kia. CNI plugin tạo veth pair khi Pod sinh ra:
- 1 đầu nằm trong **Pod's netns** (đặt tên `eth0`).
- 1 đầu nằm trong **host's netns** (tên kiểu `vethXXXXX@if3`).

```bash
# Trên host
$ ip link
12: vethbf3a2c@if3: <BROADCAST,MULTICAST,UP> master cni0 ...
                                              ^^^^^^^^^^
                                              gắn vào bridge cni0
```

### 1.3. Linux bridge (`cni0` / `docker0`)

Là 1 **switch ảo Layer 2** trong kernel. Mọi `veth` đầu host được "cắm" vào bridge này. Bridge nhìn MAC address để forward giữa các Pod cùng node.

```bash
$ brctl show cni0
bridge name   bridge id           interfaces
cni0          8000.0a580a040001   vethbf3a2c
                                  veth8d44f1
                                  veth4eaa2b
```

### 1.4. iptables / netfilter

iptables là **giao diện user-space**. Engine thật là **netfilter** trong kernel — gồm các "hook" mà packet đi qua:

```
                 ┌─────────────┐
       ┌────────►│ PREROUTING  │ ◄── packet vào host
       │         └──────┬──────┘
       │                │ routing decision
       │       ┌────────┴────────┐
       │       ▼                 ▼
       │  ┌─────────┐      ┌─────────┐
       │  │  INPUT  │      │ FORWARD │
       │  └────┬────┘      └────┬────┘
       │       │                │
       │  (đến local proc)      │
       │                        ▼
       │                   ┌─────────────┐
       │                   │ POSTROUTING │ ◄── packet ra interface
       │                   └─────────────┘
```

**Các table quan trọng cho K8s:**

| Table | Chain bạn cần biết | Mục đích |
|-------|-------------------|----------|
| `nat` | PREROUTING, POSTROUTING | DNAT/SNAT — kube-proxy chủ yếu sống ở đây |
| `filter` | INPUT, FORWARD | Cho phép/chặn packet (NetworkPolicy dùng) |

### 1.5. Các chain mà kube-proxy tạo ra

Khi bạn có Service, kube-proxy tự động tạo các chain với prefix `KUBE-`:

```
KUBE-SERVICES         ← entry point (jump từ PREROUTING/OUTPUT)
KUBE-NODEPORTS        ← cho NodePort traffic
KUBE-EXT-<svc>        ← cho external traffic của LoadBalancer
KUBE-SVC-<svc>        ← chain riêng mỗi Service (chứa load balancing rules)
KUBE-SEP-<endpoint>   ← chain riêng mỗi endpoint Pod (làm DNAT)
KUBE-MARK-MASQ        ← đánh dấu packet cần SNAT sau này
KUBE-POSTROUTING      ← thực thi MASQUERADE dựa trên mark
```

Tên chain được hash từ tên Service nên trông kỳ kỳ:
```
KUBE-SVC-NPX46M4PTMTKRN6Y       # đây là service "kubernetes" :)
KUBE-SEP-RT3F6ULLHVTESBOE
```

---

<a name="part-2"></a>
## 🚶 PHẦN 2: SCENE A — POD ↔ POD CÙNG NODE (KHÔNG QUA SERVICE)

> ▶ Animation: [`07-packet-journey-deep.html`](07-packet-journey-deep.html) — chọn **Scene A**

Đây là trường hợp **đơn giản nhất**: Pod A gọi trực tiếp IP của Pod B (cả 2 cùng Node). KHÔNG có iptables KUBE-* nào can thiệp.

### Đường đi từng bước

```
[Pod A app]
    ↓ syscall send()
[Pod A netns: eth0]                  src=10.244.1.5  dst=10.244.1.7
    ↓ veth pair (kernel teleport)
[Host netns: vethaaaa]
    ↓ bridge L2 forward
[cni0 bridge]                        nhìn ARP table → port veth-bbbb
    ↓
[Host netns: vethbbbb]
    ↓ veth pair
[Pod B netns: eth0]                  packet đến app port 8080
[Pod B app]
```

### Vì sao không có iptables can thiệp?

- Packet đi qua bridge `cni0` ở **Layer 2**. 
- Module `br_netfilter` vẫn cho bridge gọi netfilter, NHƯNG vì dst IP (`10.244.1.7`) là 1 Pod IP có thật trên ARP table → không match rule nào trong KUBE-SERVICES → packet đi thẳng.

### Lệnh kiểm chứng

```bash
# Trên host của Node, xem cni0 và veth
ip link | grep cni0
ip link | grep veth

# Vào netns của Pod xem routing
nsenter -t $(docker inspect -f '{{.State.Pid}}' <container>) -n ip route
# default via 10.244.1.1 dev eth0
# 10.244.1.0/24 dev eth0 scope link
```

---

<a name="part-3"></a>
## 🎯 PHẦN 3: SCENE B — POD → CLUSTERIP SERVICE → POD CÙNG NODE

> ▶ Animation: chọn **Scene B**

Đây là case **phổ biến nhất** trong production. Hãy đi từng bước.

### Bước 1: App gọi Service DNS

```python
requests.get("http://backend-svc")
```

CoreDNS resolve `backend-svc.default.svc.cluster.local` → `10.96.0.10`.
**Quan trọng:** `10.96.0.10` là **IP ảo** — không tồn tại trên bất kỳ interface nào trong cluster. Nó chỉ là 1 con số dùng để match rule.

### Bước 2: Packet ra Pod's eth0

```
src=10.244.1.5 (Pod A IP)
dst=10.96.0.10 (Service ClusterIP)
dport=80
```

### Bước 3: Qua veth → cni0 → trigger netfilter

`cni0` bridge có `bridge-nf-call-iptables=1` → kernel chạy chain **PREROUTING (table nat)**.

```bash
# Kiểm tra
sysctl net.bridge.bridge-nf-call-iptables
# → 1
```

### Bước 4: PREROUTING jump KUBE-SERVICES

```
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
```

### Bước 5: KUBE-SERVICES match Service IP

```
-A KUBE-SERVICES -d 10.96.0.10/32 -p tcp -m tcp --dport 80 \
    -j KUBE-SVC-BACKEND
```

### Bước 6: KUBE-SVC-BACKEND — load balancing với probability

Nếu Service có 3 endpoints, kube-proxy tạo 3 rule:

```
-A KUBE-SVC-BACKEND -m statistic --mode random --probability 0.33333333349 \
    -j KUBE-SEP-POD1
-A KUBE-SVC-BACKEND -m statistic --mode random --probability 0.50000000000 \
    -j KUBE-SEP-POD2
-A KUBE-SVC-BACKEND -j KUBE-SEP-POD3
```

**Toán đằng sau:**
- Pod 1: chọn với xs = **1/3 ≈ 0.333**
- Pod 2: trong số 2/3 còn lại, chọn với xs = **1/2** → tổng = (2/3) × (1/2) = 1/3
- Pod 3: fallback = (2/3) × (1/2) = 1/3

→ Phân phối đều **xấp xỉ** round-robin.

### Bước 7: KUBE-SEP-POD1 thực hiện DNAT

```
-A KUBE-SEP-POD1 -p tcp -m tcp -j DNAT --to-destination 10.244.1.8:8080
```

Kernel sửa header packet:
```
TRƯỚC:  src=10.244.1.5  dst=10.96.0.10:80
SAU:    src=10.244.1.5  dst=10.244.1.8:8080  ⚡ DNAT
```

**Đồng thời** kernel tạo entry trong **conntrack table** để response biết đường về:
```
tcp 10.244.1.5:54321 → 10.244.1.8:8080  (sẽ un-NAT thành 10.96.0.10 khi response)
```

### Bước 8: Routing decision sau DNAT

Kernel check routing table:
```
10.244.1.0/24 dev cni0
```
→ Packet đi ngược lại vào `cni0` → forward đến `vethbbbb` → vào Pod B.

### Bước 9: Pod B nhận packet

Pod B thấy:
```
src=10.244.1.5
dst=10.244.1.8 (chính nó)
```
App xử lý request, gửi response. Conntrack tự động **un-DNAT** response: src=10.244.1.8 được sửa lại thành 10.96.0.10 → client thấy đúng IP Service.

> 💡 **Insight:** đây là lý do bạn dùng `tcpdump` trên Pod target sẽ thấy src là Pod IP thật (10.244.1.5), KHÔNG phải Service IP. Nhiều người mới bị confuse chỗ này.

---

<a name="part-4"></a>
## 🌐 PHẦN 4: SCENE C — CROSS-NODE TRAFFIC & VXLAN

> ▶ Animation: chọn **Scene C**

Khi DNAT chọn Pod ở **Node khác**, mọi thứ phức tạp hơn vì cần đi qua mạng vật lý.

### Vấn đề

- Mạng underlay (router/switch giữa các Node) **không biết** về Pod CIDR (10.244.0.0/16).
- Bạn không thể bắt sysadmin add static route cho 10.244.x.x lên router.

### Giải pháp: Overlay network (VXLAN)

CNI plugin (Flannel, Calico+VXLAN, Weave) tạo overlay bằng cách **bọc packet Pod trong UDP**:

```
┌──────────────────────────────────────────────┐
│ OUTER L2: ethernet                           │
├──────────────────────────────────────────────┤
│ OUTER L3: src=192.168.1.10  dst=192.168.1.20 │  ← Node IP thật
├──────────────────────────────────────────────┤
│ OUTER L4: UDP port 8472                      │  ← VXLAN port
├──────────────────────────────────────────────┤
│ VXLAN header: VNI=1 (virtual network ID)     │
├──────────────────────────────────────────────┤
│ INNER L2: ethernet                           │
├──────────────────────────────────────────────┤
│ INNER L3: src=10.244.1.5  dst=10.244.2.7     │  ← Pod IP
├──────────────────────────────────────────────┤
│ INNER L4 + payload (HTTP request)            │
└──────────────────────────────────────────────┘
```

### Đường đi đầy đủ

```
Pod A (10.244.1.5)
    ↓
veth → cni0 → iptables DNAT → 10.244.2.7
    ↓
routing: "10.244.2.0/24 via flannel.1"
    ↓
flannel.1 (VTEP - VXLAN Tunnel Endpoint)
    ↓ ENCAPSULATE: bọc trong UDP
host eth0 (192.168.1.10)
    ↓
mạng vật lý (router/switch — chỉ thấy 2 host nói chuyện UDP)
    ↓
Node 2 eth0 (192.168.1.20)
    ↓
flannel.1 (Node 2)
    ↓ DECAPSULATE: tháo UDP wrapper
cni0 (Node 2) → veth → Pod B (10.244.2.7)
```

### Tradeoff

| Tiêu chí | VXLAN (Flannel) | Native routing (Calico BGP) |
|----------|-----------------|------------------------------|
| Setup | Đơn giản, plug-and-play | Cần config BGP với router |
| Performance | Chậm hơn ~5-10% (encap overhead) | Native — full speed |
| Visibility | Khó debug (encap hide IP gốc) | tcpdump thấy Pod IP trực tiếp |
| Network constraint | Hoạt động trên mọi underlay | Cần router hỗ trợ BGP |

### MTU: cái bẫy kinh điển

VXLAN thêm 50 bytes overhead. Nếu MTU mạng thật là 1500, MTU của Pod chỉ có thể là **1450**. Sai MTU → packet lớn bị fragment hoặc drop → app "chậm/treo bí ẩn". 

Kiểm tra:
```bash
kubectl exec mypod -- cat /sys/class/net/eth0/mtu
# Nếu là 1500 mà bạn dùng VXLAN → BUG! Phải sửa CNI config.
```

---

<a name="part-5"></a>
## 🌍 PHẦN 5: SCENE D — EXTERNAL TRAFFIC, VÌ SAO MẤT IP GỐC?

> ▶ Animation: chọn **Scene D**

Đây là scenario rất nhiều người **fail interview**: vì sao app log ra IP của Node thay vì IP user thật?

### Đường đi

```
User (203.0.113.50)
    ↓
Cloud LB (54.10.1.5)
    ↓
Node:30080 (NodePort)
    ↓
iptables PREROUTING → KUBE-SERVICES → KUBE-NODEPORTS
    ↓
KUBE-EXT-WEB → KUBE-MARK-MASQ ⚡   ← đánh dấu mark 0x4000
                       ↓
            KUBE-SVC-WEB → KUBE-SEP-X → DNAT → 10.244.1.8:8080
                                        ↓
                              POSTROUTING → match mark → MASQUERADE ⚡
                                        ↓ (src đổi thành Node IP)
                              cni0 → veth → Pod
```

### Vì sao cần SNAT (MASQUERADE)?

Giả sử KHÔNG có SNAT:

```
Pod nhận: src=203.0.113.50, dst=10.244.1.8
Pod response: src=10.244.1.8, dst=203.0.113.50
   ↓
Routing: 0.0.0.0/0 via gateway → ra eth0 host → ra Internet
   ↓
User nhận response từ IP host của Node (192.168.1.10)
   ↓
NHƯNG user gửi request đến 54.10.1.5 (LB) → response từ IP khác → 
   TCP stack của user reject! Connection broken.
```

→ Phải SNAT để **response đi cùng đường với request** (qua LB).

### Hệ quả: Pod log thấy IP của Node, không phải user

```python
# Trong app
request.remote_addr  # → 192.168.1.10 (Node IP), không phải 203.0.113.50
```

### Cách giữ IP gốc

**1. `externalTrafficPolicy: Local`**

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local   # ← thêm dòng này
```

Hành vi:
- Chỉ route traffic đến **Pod cùng Node** với entry point.
- Không SNAT → giữ IP user gốc.
- **Tradeoff:** Node không có Pod sẽ healthcheck FAIL, LB ngừng gửi traffic đến đó. Có thể mất cân bằng tải.

**2. `X-Forwarded-For` header (nếu LB là L7)**

Cloud LB ở chế độ HTTP có thể inject header `X-Forwarded-For: 203.0.113.50`. App đọc header này thay vì TCP src.

**3. Proxy Protocol**

Cho L4 LB (TCP/UDP). LB prepend 1 binary header chứa IP gốc trước payload. Nginx-ingress/HAProxy hỗ trợ parse.

---

<a name="part-6"></a>
## 🧠 PHẦN 6: CONNTRACK — KẺ VÔ DANH ĐỨNG SAU MỌI NAT

`conntrack` (connection tracking) là module kernel ghi lại **state của mọi connection**. Khi DNAT, kernel KHÔNG phải DNAT từng packet — nó chỉ DNAT packet đầu, rồi conntrack auto-apply cho mọi packet sau cùng connection.

### Xem conntrack table

```bash
# Trên Node
$ conntrack -L
tcp 6 86399 ESTABLISHED 
    src=10.244.1.5 dst=10.96.0.10 sport=54321 dport=80
    src=10.244.1.8 dst=10.244.1.5 sport=8080 dport=54321
    [ASSURED]
```

Đọc:
- Dòng 1: connection từ góc nhìn "outgoing" (request).
- Dòng 2: connection ở góc nhìn "incoming reply" (response sau DNAT). 

Khi response đến, kernel match dòng 2 → tự động un-DNAT.

### Conntrack table FULL — sự cố production thường gặp

```
nf_conntrack: table full, dropping packet
```

Mỗi connection chiếm 1 slot. Default `nf_conntrack_max` thường là 65k. Cluster traffic cao → đầy → drop random packets → app báo timeout không rõ nguyên nhân.

**Fix:**
```bash
sysctl -w net.netfilter.nf_conntrack_max=1048576
# Persist trong /etc/sysctl.d/99-k8s.conf
```

---

<a name="part-7"></a>
## ⚡ PHẦN 7: IPTABLES vs IPVS vs eBPF (CILIUM)

### iptables mode (mặc định)

- Mỗi Service thêm vài rule. **Linear scan** mỗi packet.
- Cluster có 5000 Service × 10 endpoints = 50k+ rules → packet đi qua tốn ms-level.
- Update rules đắt: `iptables-restore` block kernel networking trong khi load.

### IPVS mode

- Dùng IPVS (IP Virtual Server) — load balancer kernel built-in cho LVS.
- Lookup **hash table O(1)** thay vì linear.
- Hỗ trợ nhiều thuật toán: `rr`, `wrr`, `lc` (least conn), `sh` (source hash).
- Bật:
  ```yaml
  # kube-proxy config
  mode: ipvs
  ipvs:
    scheduler: rr
  ```
- Vẫn dùng iptables cho 1 vài chain phụ (KUBE-MARK-MASQ).

### eBPF (Cilium)

- **Bỏ hoàn toàn iptables/IPVS**. Dùng eBPF programs gắn trực tiếp vào tc/XDP.
- Load balancing diễn ra ở mức **rất sớm** trong network stack — thậm chí trước cả netfilter.
- Hỗ trợ replace conntrack bằng eBPF map → nhanh hơn, scale tốt hơn.
- Trade-off: cần kernel ≥ 4.19 (recommended ≥ 5.4), debug khó hơn.

**Benchmark thực tế** (cluster 1000 services):
| Mode | p99 latency |
|------|-------------|
| iptables | ~2.5ms |
| IPVS | ~0.8ms |
| Cilium eBPF | ~0.3ms |

---

<a name="part-8"></a>
## 🛠 PHẦN 8: DEBUG TOOLKIT THỰC CHIẾN

### Checklist khi Service không hoạt động

```bash
# 1. Service tồn tại?
kubectl get svc <name> -o wide

# 2. Endpoints có Pod IP không? (lỗi #1: selector sai)
kubectl get endpoints <name>
kubectl get endpointslices -l kubernetes.io/service-name=<name>

# 3. Pod đúng label?
kubectl get pods --show-labels | grep <selector>

# 4. Pod listen đúng port?
kubectl exec <pod> -- ss -tlnp
# (hoặc netstat -tlnp nếu có)

# 5. Test trực tiếp Pod IP từ trong cluster
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash
$ curl http://<pod-ip>:<port>     # nếu fail → Pod sập
$ curl http://<svc-ip>:<port>     # nếu fail → iptables/kube-proxy
$ nslookup <svc-name>             # nếu fail → CoreDNS
```

### Xem iptables rules thật của Service

```bash
# Trên Node
iptables-save -t nat | grep <service-name>

# Hoặc dùng tool đẹp hơn
iptables -t nat -L KUBE-SERVICES -n -v --line-numbers
```

### Xem conntrack cho 1 connection

```bash
conntrack -L | grep <pod-ip>
conntrack -E    # event stream realtime
```

### tcpdump trên Pod target

```bash
# Vào netns của Pod
PID=$(crictl inspect <container-id> | jq .info.pid)
nsenter -t $PID -n tcpdump -i eth0 -nn -v port 8080
```

### kubeshark — Wireshark cho K8s

Tool open-source capture mọi traffic giữa Pod, decode HTTP/gRPC/Kafka/Redis. **Cực kỳ hữu ích** khi debug microservice.

---

## 🎯 TỔNG KẾT — BẢN ĐỒ HOÀN CHỈNH

```
                          KUBERNETES NETWORKING
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
   "Kernel components"        "iptables chains"           "Overlay"
        │                           │                           │
   netns                      KUBE-SERVICES                  VXLAN
   veth pair                  KUBE-SVC-*                    (Flannel)
   cni0 bridge                KUBE-SEP-*                    BGP routing
   conntrack                  KUBE-NODEPORTS                (Calico)
                              KUBE-MARK-MASQ                eBPF
                              KUBE-POSTROUTING              (Cilium)
```

### 5 điều quan trọng nhất phải nhớ

1. **Service IP là IP ẢO** — không tồn tại trên interface nào. Chỉ là số để match iptables rule.
2. **kube-proxy không xử lý packet** — nó chỉ ghi iptables rules. Kernel làm hết.
3. **DNAT chỉ apply cho packet đầu** của connection. Conntrack handle phần còn lại.
4. **Cross-node traffic dùng overlay (VXLAN)** — bọc packet trong UDP. MTU phải nhỏ hơn underlay 50 bytes.
5. **NodePort/LoadBalancer SNAT mất IP gốc**. Dùng `externalTrafficPolicy: Local` hoặc Proxy Protocol để giữ.

---

## 📖 ĐỌC TIẾP

Sau khi nắm chắc bài này, bạn nên học:

1. **NetworkPolicy** — firewall L3/L4 cho Pod (dùng iptables filter table)
2. **Cilium Hubble** — observability cho mọi packet trong cluster
3. **Service Mesh sidecar** (Istio/Linkerd) — thêm 1 layer L7 với Envoy proxy
4. **eBPF deep dive** — tự viết XDP program

> 🎓 *"Hiểu networking K8s = hiểu Linux kernel networking. Không có shortcut."*

---

> ▶ **Quay lại animation tương tác:** [`07-packet-journey-deep.html`](07-packet-journey-deep.html)
> ▶ **Mục lục:** [`index.html`](index.html)
