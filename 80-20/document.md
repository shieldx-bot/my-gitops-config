
# Chia traffic 80/20 (Canary) với Istio (VirtualService + DestinationRule)

Tài liệu này mô tả kỹ thuật **chia traffic theo tỉ lệ** (ví dụ 80% về phiên bản ổn định *v1*, 20% về phiên bản mới *v2*) bằng Istio.

Trong repo này, manifest minh hoạ nằm ở: `80-20/canary.yaml`.

---

## 1) Bài toán & mục tiêu

Khi triển khai phiên bản mới, thay vì chuyển 100% người dùng sang *v2* ngay lập tức, ta sẽ:

- Deploy *v2* chạy song song với *v1*
- Chia traffic theo tỉ lệ (80/20)
- Theo dõi lỗi/latency/log…
- Nếu ổn: tăng dần weight của *v2* lên 50/50 → 90/10 → 100/0
- Nếu lỗi: rollback nhanh bằng cách đưa weight *v2* về 0

---

## 2) Kiến trúc Istio liên quan

Istio thực hiện traffic splitting thông qua các CRD sau:

- **Gateway**: điểm vào (Ingress) của traffic HTTP/HTTPS (nếu bạn expose dịch vụ ra ngoài cluster).
- **VirtualService (VS)**: định tuyến HTTP (route) và chia traffic theo **weight**.
- **DestinationRule (DR)**: định nghĩa **subsets** (nhóm endpoint) dựa trên **label** của Pod, ví dụ subset `v1` / `v2`.

Luồng hoạt động (rút gọn):

1. Request đi vào Ingress (Istio IngressGateway)
2. VS khớp `hosts/gateways/path` → chọn route
3. Route chia weight đến `subset: v1` và `subset: v2`
4. DR map `subset` ↔ `labels` → Envoy chọn Pod phù hợp

---

## 3) Điều kiện tiên quyết (quan trọng)

### 3.1 Namespace và sidecar

Ví dụ trong manifest dùng namespace: `sre-mart`.

Istio chỉ điều phối traffic theo subset khi Pod có sidecar Envoy.

Các cách phổ biến để inject sidecar:

- Bật auto-injection ở namespace:
	- `kubectl label ns sre-mart istio-injection=enabled`
- Hoặc set annotation per-pod/per-deployment:
	- `sidecar.istio.io/inject: "true"` (đã có trong `canary.yaml`)

### 3.2 Service **không** được selector theo version

Service `frontend` (hoặc service bạn định chia traffic) phải trỏ tới **cả v1 và v2**.

Ví dụ selector **đúng**:

- `app: frontend`

Ví dụ selector **sai** (sẽ làm v2 không bao giờ nhận traffic):

- `app: frontend`
- `version: v1`

Kiểm tra selector:

- `kubectl -n sre-mart get svc frontend -o yaml`

### 3.3 Pod labels phải khớp subset

DestinationRule dùng labels để định nghĩa subset:

- subset `v1` → `version: v1`
- subset `v2` → `version: v2`

Vì vậy các Deployment phải có labels tương ứng trên Pod template:

- `app: frontend`
- `version: v1|v2`

---

## 4) Nội dung manifest và ý nghĩa

File `80-20/canary.yaml` gồm 3 phần:

### 4.1 Deployment `frontend-v2`

Tạo phiên bản mới chạy song song:

- `metadata.labels` và `spec.template.metadata.labels` có `version: v2`
- Có annotation `sidecar.istio.io/inject: "true"` để chắc chắn được inject Envoy

Lưu ý: tài liệu này giả định **frontend v1 đã tồn tại** (Deployment/Pods có `version: v1`).

### 4.2 DestinationRule `frontend-dr`

Định nghĩa 2 subset dựa trên label `version`.

Trong `canary.yaml` host được dùng dạng FQDN để tránh nhầm namespace:

- `host: frontend.sre-mart.svc.cluster.local`

Subsets:

- `v1` ↔ `version: v1`
- `v2` ↔ `version: v2`

### 4.3 VirtualService `frontend-vs`

Chia traffic theo weight:

- 80% → subset `v1`
- 20% → subset `v2`

Điểm cần nhất quán:

- `destination.host` trong VS phải **khớp** với `spec.host` trong DR.

---

## 5) Gateway: cần có trước (nếu bạn route từ bên ngoài)

Trong `canary.yaml`, VirtualService tham chiếu gateway: `frontend-gateway`.

Repo hiện **không** kèm manifest Gateway, vì vậy bạn cần:

- Tạo Gateway cùng namespace `sre-mart`, hoặc
- Sửa VirtualService để trỏ đến Gateway mà bạn đang dùng.

Ví dụ Gateway tối thiểu (tham khảo):

- `kind: Gateway`
- `metadata.name: frontend-gateway`
- `spec.selector` trỏ về Istio ingress gateway (thường là `istio: ingressgateway`)
- `servers` mở port 80 (hoặc 443 nếu TLS)

Nếu bạn chỉ test **nội bộ trong cluster** (không qua ingress), có thể bỏ `gateways:` hoặc dùng mesh gateway (`mesh`).

---

## 6) Cách triển khai

Áp dụng manifest:

- `kubectl apply -f 80-20/canary.yaml`

Kiểm tra resource đã tạo:

- `kubectl -n sre-mart get deploy,po,svc`
- `kubectl -n sre-mart get destinationrule,virtualservice`

---

## 7) Cách xác minh traffic split

### 7.1 Kiểm tra subset endpoints có tồn tại

- `kubectl -n sre-mart get po -l app=frontend -L version`

Bạn phải thấy cả Pod `version=v1` và `version=v2` đang Ready.

### 7.2 Dùng Istio config dump (mức cơ bản)

Nếu có `istioctl`, bạn có thể phân tích nhanh:

- `istioctl analyze -n sre-mart`

### 7.3 Kiểm chứng bằng log / metrics

Tuỳ ứng dụng, bạn có thể:

- xem access log của ingress gateway
- xem metrics (Prometheus) theo destination workload (v1/v2)
- hoặc bật tracing

Mẹo: nếu frontend không “lộ” version ra response, cách đơn giản để demo là dùng *v2 image* có thay đổi UI/banner để nhìn thấy khác biệt.

---

## 8) Rollback & tăng dần canary

### Rollback nhanh (v2 = 0%)

Sửa VirtualService:

- `weight: v1 = 100`
- `weight: v2 = 0`

Apply lại YAML.

### Tăng dần (progressive delivery)

Ví dụ lộ trình:

- 95/5 → 90/10 → 80/20 → 50/50 → 0/100

Mỗi lần thay đổi chỉ cần chỉnh `weight` trong VirtualService và apply lại.

---

## 9) Lỗi thường gặp & cách xử lý

### 9.1 Subset không match (v2 không nhận traffic)

Nguyên nhân thường gặp:

- Pod thiếu label `version: v2`
- Service selector có `version: v1` (lọc mất v2)
- Không inject sidecar nên Envoy không điều phối theo VS/DR

### 9.2 Host trong VS/DR không khớp

Triệu chứng:

- VS tạo nhưng traffic không chia đúng

Khắc phục:

- Dùng cùng một `host` (khuyến nghị dùng FQDN):
	- `frontend.sre-mart.svc.cluster.local`

### 9.3 Gateway không tồn tại

Triệu chứng:

- Route từ bên ngoài không đi vào app

Khắc phục:

- Tạo Gateway `frontend-gateway` hoặc đổi VS trỏ đúng gateway hiện có.

---

## 10) Tài liệu liên quan

- Istio: VirtualService / DestinationRule (traffic management)
- Progressive delivery (canary, blue/green)s

