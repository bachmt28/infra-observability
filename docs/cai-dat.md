# Cài đặt từng cấu phần

Lệnh khớp với cách đã cài trên cluster thật (`helm repo list` trên box Jenkins xác nhận nguồn
chart). Chạy tay hoặc qua script — không tự động qua agent trên môi trường live.

Quy ước Helm: `helm install <release-name> <repo-alias>/<chart-name>`. `<repo-alias>` là tên đặt
tay cho 1 URL Helm repo qua `helm repo add <alias> <url>` — không phải tên cố định của hãng, chỉ
là nhãn cục bộ trỏ tới kho chart thật. Mỗi mục dưới đây ghi rõ URL thật đứng sau alias và ai
maintain chart đó.

## Loki

Chart `loki` do chính **Grafana Labs** maintain, alias `grafana` trỏ
`https://grafana.github.io/helm-charts` (kho chart chính thức của Grafana Labs, không chỉ có
Loki — còn Tempo, Mimir, Grafana...). Chart hỗ trợ 3 mode triển khai
(`SingleBinary`/`SimpleScalable`/`Distributed`); `helm-loki-values.yaml` cấu hình mode
**SimpleScalable** — tách Loki thành 3 role `backend`/`read`/`write` chạy riêng process (đọc/ghi
độc lập, scale riêng) đứng sau 1 `gateway` (nginx) làm entrypoint duy nhất, thay vì 1 binary gánh
hết như mode `SingleBinary`.

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

Chart do cộng đồng **prometheus-community** (org độc lập trên GitHub, không phải Prometheus/Grafana
Labs chính chủ) maintain, alias `prometheus-community` trỏ
`https://prometheus-community.github.io/helm-charts`. Đây là bản đóng gói phổ biến nhất của dự án
[kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) — 1 chart cài gộp nhiều
thành phần độc lập qua subchart: **Prometheus Operator** (controller quản lý Prometheus/
Alertmanager bằng CRD thay vì StatefulSet tay), **Prometheus** server, **Alertmanager**,
**Grafana** (subchart riêng của chính Grafana Labs, được nhúng vào đây), **kube-state-metrics**
(expose trạng thái object K8s thành metric) và **node-exporter** (metric phần cứng/OS mỗi node).

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 84.5.0 \
  -n app-kube-pg-stack --create-namespace \
  -f kube-prometheus-stack/values.yaml
```

`values.yaml` trong repo là bản đầy đủ (không phải file override riêng) — sửa trực tiếp file này
khi cần đổi cấu hình rồi `helm upgrade` cùng câu lệnh trên (thay `install` bằng `upgrade`).

Ns `app-kube-pg-stack` còn 1 release riêng `pushgateway` (chart `prometheus-pushgateway` cùng repo
`prometheus-community` — cho phép job chạy ngắn/batch đẩy metric vào thay vì để Prometheus tự kéo)
— không thuộc phạm vi repo này, cài độc lập nếu cần:
```sh
helm install pushgateway prometheus-community/prometheus-pushgateway -n app-kube-pg-stack
```

Verify: `kubectl -n app-kube-pg-stack get pods` — Grafana/Prometheus/Alertmanager/kube-state-metrics/
operator/node-exporter Running. Sau khi cài xong, làm tiếp `docs/loki-grafana-setup.md` để trỏ
Grafana vào Loki + dựng dashboard.

## OpenTelemetry Collector

Dự án chính thức **OpenTelemetry** (Operator quản lý vòng đời Collector qua CRD
`OpenTelemetryCollector`). Có sẵn Helm chart (`open-telemetry/opentelemetry-operator`) nhưng
cluster này KHÔNG dùng — cài trực tiếp bằng manifest raw tải từ GitHub Release của dự án
(`otel/opentelemetry-operator.yaml`, 18k dòng: CRD + Deployment của Operator).

Sau khi Operator chạy, `OpenTelemetryCollector` là 1 CRD riêng — mỗi CR mô tả 1 collector độc lập
(receiver/processor/exporter + mode chạy `deployment`/`daemonset`), Operator tự dựng
Deployment/DaemonSet tương ứng. 2 CR đang dùng:
- `otel-deploy-gateway` (mode `deployment`, 1 replica) — điểm hội tụ trung tâm, nhận dữ liệu rồi
  phân phối ra Loki/Kafka/Prometheus/Tempo.
- `otel-dms-agent` (mode `daemonset`, 1 pod/node) — chạy trên từng node để đọc log container tại
  chỗ, forward về gateway ở trên.

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

Chart `tempo-distributed` cũng của **Grafana Labs** (alias `grafana`, cùng repo với Loki ở trên).
Tempo có 2 chart khác nhau trong cùng repo: `tempo` (monolith, 1 process gộp hết) và
`tempo-distributed` (tách distributor/ingester/querier/compactor chạy riêng, scale độc lập —
giống tinh thần SimpleScalable của Loki). Cluster này dùng `tempo-distributed`, xác nhận qua HPA
còn sót lại tên `tempo-ingester`.

File `tempo/tempo-operator.yaml` là manifest của **Tempo Operator** (dự án riêng, quản lý Tempo
qua CRD `TempoStack` — cách tổ chức khác hẳn Helm chart) — có trong repo nhưng **không phải cách
đã cài thật**, cluster hiện không có CRD `TempoStack` nào cả, chỉ để tham khảo hướng thay thế nếu
sau này muốn đổi cách quản lý.

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

Chart chính thức của **InfluxData** (alias `influxdata`), triển khai Telegraf — agent thu thập
metric/log dạng plugin (input/output pluggable), 1 sản phẩm trong hệ sinh thái TICK Stack của
InfluxData. Đã bỏ, thay bằng `otel-dms-agent` DaemonSet làm agent thu thập. Không có gì cần cài;
nếu muốn dựng lại để so sánh:

```sh
helm repo add influxdata https://helm.influxdata.com/
helm repo update
helm install telegraf influxdata/telegraf --version 1.8.70 \
  -n app-otel \
  -f telegraf/values.yaml
```
