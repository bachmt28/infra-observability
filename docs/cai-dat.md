# Cài đặt từng cấu phần

Lệnh khớp với cách đã cài trên cluster thật (`helm repo list` trên box Jenkins xác nhận nguồn
chart). Chạy tay hoặc qua script — không tự động qua agent trên môi trường live.

## Loki

```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install loki grafana/loki --version 7.0.0 \
  -n app-otel --create-namespace \
  -f loki/helm-loki-values.yaml
```

Verify: `kubectl -n app-otel get pods -l app.kubernetes.io/name=loki` — đủ `backend`/`read`/`write`/
`gateway`/2 memcached cache Running.

## kube-prometheus-stack (Prometheus + Grafana + Alertmanager)

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 84.5.0 \
  -n app-kube-pg-stack --create-namespace \
  -f kube-prometheus-stack/values.yaml
```

`values.yaml` trong repo là bản đầy đủ (không phải file override riêng) — sửa trực tiếp file này
khi cần đổi cấu hình rồi `helm upgrade` cùng câu lệnh trên (thay `install` bằng `upgrade`).

Ns `app-kube-pg-stack` còn 1 release riêng `pushgateway` (chart `prometheus-pushgateway`,
`prometheus-community`) — không thuộc phạm vi repo này, cài độc lập nếu cần:
```sh
helm install pushgateway prometheus-community/prometheus-pushgateway -n app-kube-pg-stack
```

Verify: `kubectl -n app-kube-pg-stack get pods` — Grafana/Prometheus/Alertmanager/kube-state-metrics/
operator/node-exporter Running. Sau khi cài xong, làm tiếp `docs/loki-grafana-setup.md` để trỏ
Grafana vào Loki + dựng dashboard.

## OpenTelemetry Collector

3 bước: cài Operator → CR gateway → CR agent. Operator quản lý CR qua CRD riêng (không phải Helm).

```sh
kubectl apply -f otel/opentelemetry-operator.yaml
kubectl apply -f otel/rbac-extra.yaml
```

```sh
kubectl apply -f otel/otelcol-deploy-gateway.yaml
kubectl apply -f otel/otelcol-dms-agent.yaml
```

Verify: `kubectl -n app-otel get opentelemetrycollectors` — cả 2 `READY` đủ replica
(`otel-deploy-gateway` 1/1, `otel-dms-agent` bằng số node).

## Tempo (hiện chưa cài — xem README mục "Vấn đề đã biết")

Cách cài lại bằng Helm chart phân tán (đây là cách THẬT đã dùng trước — file
`tempo/tempo-operator.yaml` là manifest Operator dạng CRD `TempoStack`, có trong repo nhưng
**không phải cách đã cài** — cluster hiện không có CRD Tempo nào cả):

```sh
helm repo add grafana https://grafana.github.io/helm-charts   # bỏ qua nếu đã add ở bước Loki
helm repo update
helm install tempo grafana/tempo-distributed --version 1.61.3 \
  -n app-otel \
  -f tempo/values.yaml
```

**Trước khi cài:** điền `storage.trace.s3.access_key`/`secret_key` trong `tempo/values.yaml`
(bucket `dtu-otel-traces`, endpoint `s3.example.com`) trước khi `helm install`.

Cài xong không cần sửa gì ở `otel-deploy-gateway` — pipeline traces đã sẵn cấu hình trỏ
`tempo-gateway.app-otel.svc.cluster.local:4317`, chỉ cần service đó tồn tại là tự thông ngay.

## Telegraf (không dùng — giữ lại tham khảo)

Đã bỏ, thay bằng `otel-dms-agent` DaemonSet làm agent thu thập. Không có gì cần cài; nếu muốn
dựng lại để so sánh:

```sh
helm repo add influxdata https://helm.influxdata.com/
helm repo update
helm install telegraf influxdata/telegraf --version 1.8.70 \
  -n app-otel \
  -f telegraf/values.yaml
```
