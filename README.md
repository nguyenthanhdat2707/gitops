# GitOps Canary Auto-Abort Lab

## 1. Mục tiêu

Lab này triển khai GitOps với ArgoCD, monitoring bằng Prometheus và canary deployment bằng Argo Rollouts. Mục tiêu chính là chứng minh hệ thống có thể tự động abort bản canary khi error rate vượt ngưỡng.

## 2. Thành phần sử dụng

* ArgoCD App of Apps
* kube-prometheus-stack
* Argo Rollouts
* Flask API demo
* ServiceMonitor
* PrometheusRule
* AnalysisTemplate

## 3. Kiến trúc

GitHub là nguồn desired state. ArgoCD root Application theo dõi thư mục `argocd/apps` và tạo các child Application.

Các child Application chính:

* `web-demo`: app demo ban đầu
* `kube-prometheus-stack`: cài Prometheus/Grafana/Alertmanager
* `argo-rollouts`: cài Argo Rollouts controller và CRD
* `api`: deploy Flask API bằng Rollout

Flow tổng quát:

```text
GitHub main
→ ArgoCD root Application
→ child Applications
→ Kubernetes cluster
→ Prometheus scrape metric
→ Argo Rollouts chạy canary analysis
→ auto-abort nếu metric lỗi vượt ngưỡng
```

## 4. API demo

API là Flask app dùng để giả lập workload và lỗi HTTP 500.

Endpoint:

* `/`: trả response theo version hiện tại
* `/healthz`: readiness probe
* `/metrics`: Prometheus scrape metric

Biến môi trường:

```yaml
VERSION: v1 | v2
ERROR_RATE: 0 | 0.5 | 1
```

Trong demo auto-abort, bản v2 được cấu hình `ERROR_RATE=1` để tạo lỗi 500 và kích hoạt rollback.

## 5. Monitoring

Prometheus scrape metric từ Flask API thông qua `ServiceMonitor`.

Metric chính:

```promql
flask_http_request_total
```

Query kiểm tra error rate:

```promql
(
  sum(rate(flask_http_request_total{namespace="demo",service="api",status=~"5.."}[1m]))
  /
  clamp_min(sum(rate(flask_http_request_total{namespace="demo",service="api"}[1m])), 0.001)
)
```

Ý nghĩa:

```text
HTTP 5xx request rate / total request rate
```

## 6. SLO / Alert

Lab có định nghĩa `PrometheusRule` tên `api-slo-alerts`.

Alert chính:

```text
APIHigh5xxRate
```

Điều kiện:

```promql
(
  sum(rate(flask_http_request_total{namespace="demo",service="api",status=~"5.."}[1m]))
  /
  clamp_min(sum(rate(flask_http_request_total{namespace="demo",service="api"}[1m])), 0.001)
) > 0.10
```

Ngưỡng được chọn là 10% 5xx error rate trong 1 phút. Đây là ngưỡng phục vụ mục tiêu demo để dễ quan sát lỗi và rollback.

## 7. AnalysisTemplate

`AnalysisTemplate` tên `api-error-rate` được dùng để Argo Rollouts tự query Prometheus trong quá trình canary.

Điều kiện:

```yaml
successCondition: result[0] < 0.10
failureCondition: result[0] >= 0.10
failureLimit: 0
```

Ý nghĩa:

* Nếu error rate nhỏ hơn 10% thì analysis pass.
* Nếu error rate lớn hơn hoặc bằng 10% thì analysis fail.
* `failureLimit: 0` nghĩa là chỉ cần một lần measurement fail thì AnalysisRun fail ngay.

## 8. Canary strategy

Rollout chạy theo flow:

```text
1. Scale canary lên 25%
2. Pause 60 giây để Prometheus thu metric
3. Chạy AnalysisTemplate
4. Nếu analysis pass thì rollout tiếp
5. Nếu analysis fail thì tự abort
6. Canary ReplicaSet bị scale down
7. Stable ReplicaSet quay lại phục vụ traffic
```

## 9. Kết quả demo

Khi release v2 lỗi, Argo Rollouts tạo revision 6 làm canary.

AnalysisRun fail:

```text
api-6b64f778fc-6-2   Failed
```

Rollout tự abort:

```text
RolloutAborted: Rollout aborted update to revision 6:
Metric "api-5xx-error-rate" assessed Failed due to failed (1) > failureLimit (0)
```

Kết quả rollback:

```text
revision 6: ScaledDown canary
revision 5: Healthy stable
Available: 4
Updated: 0
```

Điều này chứng minh canary auto-abort hoạt động dựa trên metric Prometheus, không cần chạy lệnh abort thủ công.

## 10. Evidence

Ảnh minh chứng được lưu trong thư mục `docs/images`.

### AnalysisRun failed

<img width="713" height="192" alt="image" src="https://github.com/user-attachments/assets/6eb0acc1-9f6e-4ef1-94e5-ba01009f9a01" />

### Rollout auto-aborted

<img width="1638" height="987" alt="image" src="https://github.com/user-attachments/assets/43bdd8cb-3e7c-42a4-a2cd-06e5b7e7042b" />

### API 5xx errors

<img width="1911" height="553" alt="image" src="https://github.com/user-attachments/assets/f560c955-46ee-4232-9182-dc42abd9b578" />

## 11. Email notification

Email notification chưa được triển khai trong phạm vi lab này vì cụm lab chưa có SMTP relay hoặc mail provider được cấu hình sẵn. Nếu cấu hình email cần thêm SMTP credential vào Alertmanager thông qua Kubernetes Secret, đồng thời phải kiểm soát cách lưu secret để tránh đưa thông tin nhạy cảm lên Git repo.

Trong lab này em đã hoàn thành phần PrometheusRule để phát hiện lỗi API 5xx. Phần còn lại để gửi email sẽ được mở rộng bằng cách cấu hình Alertmanager receiver, SMTP Secret và route alert `APIHigh5xxRate` tới receiver đó.

## 12. Các lệnh kiểm tra

```bash
kubectl -n argocd get applications
kubectl -n demo get analysistemplate,prometheusrule,rollout
kubectl -n demo get analysisrun
kubectl argo rollouts get rollout api -n demo
kubectl -n demo logs load --tail=30
```

## 13. Kết luận

Lab đã hoàn thành các phần chính:

* GitOps bằng ArgoCD
* App of Apps pattern
* Monitoring bằng Prometheus
* API metric bằng Flask exporter
* Canary deployment bằng Argo Rollouts
* Auto-abort bằng AnalysisTemplate
* Rollback về stable ReplicaSet khi error rate vượt ngưỡng
