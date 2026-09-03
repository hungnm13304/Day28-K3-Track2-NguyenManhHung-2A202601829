# Bài nộp Day 28 Track 2 — Phần trả lời

**Sinh viên:** Nguyễn Mạnh Hưng  
**MSSV:** 2A202601829  
**Hình thức:** Cá nhân

## Đánh đổi kỹ thuật

- Kafka tiếp nhận bất đồng bộ và dùng khóa idempotency làm định danh replay; `202 Accepted` nghĩa là đã xếp hàng, chưa phải đã ghi Delta.
- Delta MERGE giữ bản ghi mới nhất theo `(occurred_at, event_id)`, không phụ thuộc thứ tự Kafka.
- Feast phục vụ online, Spark tính toán offline và materialize; độ trễ thấp hơn nhưng có độ trễ làm tươi dữ liệu.
- Qdrant dùng UUID xác định từ document ID nên replay cập nhật đúng một vector.
- MLflow dùng alias `champion` để promotion/rollback nhanh và giữ provenance.
- Readiness phân biệt lỗi bắt buộc (`not_ready`) và tùy chọn (`degraded`).

## Khoảng trống production

Lab dùng broker, volume và credential local; production cần broker dự phòng, storage bền vững, secret management, backup/restore, cluster thật, registry, policy, autoscaling và canary rollback.

GPU profile phục vụ `Qwen/Qwen3-1.7B` bằng vLLM 0.28.0 trên RTX 5060 Ti; cần VRAM utilization 0.80. IP07 và GPU golden-path đã đạt. LangSmith đã xác minh với project `day22-lab`; key chỉ nằm trong file local.

## Đóng góp cá nhân

Nguyễn Mạnh Hưng thực hiện toàn bộ: logic Kafka/idempotency/trace, Delta replay-safe, Feast, readiness vLLM/gateway, integration journeys, evidence, manifest validation, kiến trúc và reflection.

## Sự cố và khôi phục

- DAG đầu tiên báo `polled=0`, chưa tạo Delta. Kafka vẫn khỏe; nguyên nhân là race khi consumer khởi động, không mất dữ liệu.
- DAG remediation sau đó drain backlog, tạo Delta, materialize Feast và index Qdrant. Replay/golden-path pass; IP02/IP03 chứng minh không mất dữ liệu.
- Burst qua Envoy tạo HTTP 429 sau giới hạn 10 RPS. Dấu hiệu là 429 và `x-request-id`; xử lý bằng backoff/retry phía client. Chi tiết ở IP08.

## Kiểm thử và evidence

- Test nền tảng: `87 passed`.
- Integration non-GPU: `56 passed`; GPU vLLM: `3 passed`; LangSmith: `1 passed` với `day22-lab`.
- Integration matrix: `245 checks passed`; Ruff, portability và Kubernetes/GitOps manifest đều đạt.
- Evidence JSON đã commit trong `evidence/`, gồm IP01–IP10 và báo cáo tổng hợp.
- Load profile 200 request qua Envoy: 8 worker P50/P95/P99 = 0.76/307.57/360.01 ms; 16 worker = 3.12/5.20/215.96 ms. Burst thể hiện rate-limit đúng cấu hình.

## Reflection

Khó nhất là giữ idempotency xuyên Kafka–Delta–Feast–Qdrant và phân biệt lỗi hạ tầng với lỗi dữ liệu. Lựa chọn chính là MERGE theo bản ghi mới nhất, alias champion và readiness nhiều mức. Cải tiến tiếp theo là broker/storage dự phòng, remediation tự động và tải đại diện production.
