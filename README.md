# infra-observability

Cấu hình monitoring/logging/tracing của cluster `k8s-cluster-example`, gom về một chỗ từ box Jenkins
(`/root/git/infra-loki`, `/root/git/infra-kube-prometheus-stack`, `/root/git/infra-tempo`,
`/root/git/infra-telegraf`, `/root/opentelemetry-operator.yaml`) và từ cấu hình sống trên cluster.

Chưa đồng bộ GitLab — repo hiện chỉ tồn tại local, xem mục "Việc còn lại".

## Kiến trúc pipeline

```
AIX/CORE (syslog) ──┐
                    ├──► otel-dms-agent (DaemonSet, 52 node, ns app-otel)
Container logs ─────┘         filelog + k8s_attributes + mask OTP
App OTLP ───────────────────►      │
                                    ▼
                    otel-deploy-gateway (Deployment, ns app-otel)
                    nhận: otlp:4317/4318, syslog tcp/udp:54527
                                    │
        ┌──────────┬──────────┬────┴──────┬─────────────┬──────────────┐
        ▼          ▼          ▼           ▼             ▼              ▼
      Loki      Kafka     Prometheus   Tempo (chết)  logmon (ngoài) netdata (ngoài)
```

## Cấu phần

| Thư mục | Thành phần | Trạng thái | Ghi chú |
|---|---|---|---|
| `loki/` | Loki (log storage) | Đang chạy | Helm release `loki` rev 10, ns `app-otel`, chart `loki-7.0.0`, mode SimpleScalable |
| `kube-prometheus-stack/` | Prometheus + Grafana + Alertmanager | Đang chạy | Helm release `kube-prometheus-stack`, ns `app-kube-pg-stack`. Source chart đầy đủ, không phải riêng file values |
| `otel/` | OpenTelemetry Collector | Đang chạy | Operator cài raw bằng `kubectl apply` (không qua Helm), ns `opentelemetry-operator-system`. Chi tiết pipeline bên dưới |
| `tempo/` | Tempo (tracing backend) | Đã gỡ, còn rác | Xem "Vấn đề đã biết" |
| `telegraf/` | Telegraf (metrics agent) | Không dùng | Hướng thử trước khi chốt dùng `otel-dms-agent` làm agent |
| `grafana-dashboard/` | Dashboard Grafana | — | Xem `docs/loki-grafana-setup.md` |

### OTel Collector — 2 CR trong ns `app-otel`

- `otel-deploy-gateway` (mode `deployment`) — nhận OTLP + syslog tcp/udp `:54527` (nguồn AIX/CORE
  qua 3 topic Kafka `OTEL_CORE_AIX_LOGS/METRICS/TRACES`, broker `dc1d-kkbro01/02/03:9094`). Xuất ra
  Loki, Kafka, Prometheus remote-write, Tempo, và 2 endpoint ngoài (`logmon` 10.10.0.21:14317,
  `netdata` 10.10.0.21:4317). File: `otel/otelcol-deploy-gateway.yaml`.
- `otel-dms-agent` (mode `daemonset`, 52 node) — thu container log `/var/log/containers/*_app-*_*.log`
  (loại trừ `kube-system` + `app-otel`), nhận OTLP app gửi thẳng, mask OTP trong log body
  (`<otp>123</otp>` → `<otp>***</otp>`), enrich `k8s_attributes`, load-balance (DNS) sang gateway.
  File: `otel/otelcol-dms-agent.yaml`.
- `otel/rbac-extra.yaml` — ClusterRole/Binding phụ cho SA `otel-dms-agent-collector` đọc metadata
  pod/namespace/node, apply tay riêng ngoài chart/CR.

Cả 2 file CR trong `otel/` lấy trực tiếp từ cluster (`kubectl get opentelemetrycollector -o yaml`)
— trước đó không có bản backup nào khác.

## Vấn đề đã biết

- **Traces rớt vào endpoint chết.** `otel-deploy-gateway` vẫn xuất traces sang
  `tempo-gateway.app-otel.svc.cluster.local:4317` — service này không tồn tại (Tempo đã gỡ). Dữ
  liệu trace hiện mất hoàn toàn cho tới khi dựng lại Tempo.
- **Rác từ lần gỡ Tempo:** HPA `tempo-ingester` + 4 PVC 110GB (3×30GB ingester + 1×50GB
  `storage-tempo-0`) vẫn Bound trên ns `app-otel`, chưa dọn.

## Việc còn lại

- Đồng bộ GitLab nội bộ: git init + push, box pull về `/root/git/infra-observability` thay vì
  rải rác 4 thư mục như trước.
- Dựng lại Tempo, dùng `tempo/values.yaml` làm nền.
