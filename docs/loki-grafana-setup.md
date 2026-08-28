# Trỏ Grafana vào Loki + dựng dashboard APM - Loki Log

Áp dụng khi Grafana bị cài lại / mất datasource + dashboard (helm uninstall, sự cố, dựng instance
mới). Trên **dev** có thể chạy script/API trực tiếp. Trên **live/prod** làm tay qua UI, hoặc chạy
script PowerShell ở mỗi bước (không chạy tự động qua agent).

## Chuẩn bị

- URL Grafana: `https://grafana.example.com`
- Token service account: Administration → Service accounts → tạo mới (hoặc dùng token còn hạn) →
  Add token. Role tối thiểu: Editor (tạo được datasource + dashboard).
- File dashboard: `grafana-dashboard/dashboard-apm-loki-log.json` trong repo này.

## Bước 1 — Tạo datasource Loki

**Qua UI:** Connections → Data sources → Add data source → chọn **Loki** →
- Name: `Loki`
- URL: `http://loki-gateway.app-otel.svc.cluster.local`
- Các mục khác để mặc định → Save & test.

**Qua script (bash + curl):**
```sh
TOKEN='<TOKEN>'
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Loki","type":"loki","access":"proxy","url":"http://loki-gateway.app-otel.svc.cluster.local","uid":"loki"}' \
  "https://grafana.example.com/api/datasources"
```

Dashboard trong repo này tham chiếu datasource bằng `uid: "loki"` — nếu tạo qua UI, sau khi lưu
vào sửa uid của datasource trong Grafana cho khớp `loki` (Data sources → Loki → Settings → UID),
hoặc sửa lại toàn bộ `"uid": "loki"` trong file dashboard cho khớp uid UI tự sinh.

**Verify:** Explore → chọn datasource Loki → chạy `{k8s_namespace_name=~".+"}` trong khoảng vài
phút gần nhất → phải ra log thật.

## Bước 2 — Import dashboard APM - Loki Log

**Qua UI:** Dashboards → New → Import → Upload dashboard JSON file → chọn
`grafana-dashboard/dashboard-apm-loki-log.json` → ở bước chọn datasource, chọn `Loki` cho các
panel Loki và `Prometheus` cho biến `workload` → Import.

**Qua script (bash + curl):**
```sh
TOKEN='<TOKEN>'
F='<path>/grafana-dashboard/dashboard-apm-loki-log.json'
printf '{"dashboard":%s,"overwrite":true}' "$(cat "$F")" \
  | curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
      "https://grafana.example.com/api/dashboards/db" --data-binary @-
```

File dashboard đã có sẵn `"id": null` ở top-level nên không cần chỉnh gì thêm trước khi gửi —
nếu copy JSON từ 1 instance Grafana khác (còn giữ `id` số thật) thì phải xoá/null field đó trước,
không thì bị conflict với dashboard đang có id đó trên instance đích.

**Verify:** mở `/d/loki-log-explorer/apm-loki-log` → chọn 1 namespace thật → dropdown Workload
phải ra đủ cả Deployment/StatefulSet/DaemonSet (không chỉ Deployment) → chọn 1 workload → log
hiện đúng, không lẫn workload khác trong cùng namespace.

## Cơ chế lọc (để hiểu khi cần sửa)

Dashboard dùng 3 biến: `namespace` (match chính xác `=`, không phải regex), `workload` (lấy từ
recording rule Prometheus `namespace_workload_pod:kube_pod_owner:relabel` — gộp sẵn tên
Deployment/StatefulSet/DaemonSet/Job/CronJob thành 1 label `workload` duy nhất), `pod`.

Query log OR cả 5 label owner-kind của Loki vì Loki không có 1 label "workload" gộp sẵn như
Prometheus:
```
{k8s_namespace_name="$namespace"}
  | (k8s_deployment_name=~"$workload" or k8s_statefulset_name=~"$workload"
     or k8s_daemonset_name=~"$workload" or k8s_cronjob_name=~"$workload"
     or k8s_job_name=~"$workload")
  and k8s_pod_name=~"$pod"
  |~ "(?i)$search"
  | line_format "[{{.k8s_pod_name}}] {{__line__}}"
```

Dropdown Pod lấy list theo namespace (không lọc thêm theo workload) — do API
`/loki/api/v1/label/<name>/values` của Loki chỉ nhận pure label matcher trong `{}`, không nhận
pipe/label-filter. Tên pod luôn có prefix tên workload nên vẫn chọn đúng dễ dàng dù list rộng hơn.
