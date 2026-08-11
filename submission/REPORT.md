# Báo cáo Day 13 Observability

## 1. Thông tin nhóm

- Tên nhóm: [Flash]
- Repository URL: [https://github.com/lengockhanh-code/Day13-K3-Observability-Flash]
- Commit SHA cuối: `8352785b8ce37386264bcd81a9555d403ac3b8a0`
- Thành viên và vai trò:
  - Nguyễn Tuấn Anh - 2A202601775 - Dashboard, SLO & Alert
  - Lê Mạnh Cương - 2A202601137 - Tracing & Prompt Version
  - Lê Ngọc Khánh - 2A202601487 - Incident, Report & Demo
  - Vũ Ngọc Thiện - 2A202601793 - Logging & PII

## 2. Kết quả kỹ thuật

- Điểm `validate_logs.py`: 50/100
- Tổng số traces: 22
- Số PII leak còn lại: 0
- Link/đường dẫn dashboard: [Điền link dashboard hoặc ảnh evidence]

## 3. Logging và tracing

- Evidence correlation ID: Log hiện có `correlation_id` và cho phép nối các dòng thuộc cùng một request. Ví dụ challenge có các correlation ID: `req-c3144354`, `req-247b7289`, `req-be702446`, `req-33f8e10c`, `req-d8cf6798`.
- Evidence PII redaction: Log đã redact PII, ví dụ dữ liệu nhạy cảm được thay bằng token dạng `[REDACTED_...]`. Kết quả `validate_logs.py` báo `Potential PII leaks detected: 0`.
- Evidence trace waterfall: Mở Langfuse, lọc theo `feature=refund` hoặc `session_id` trong khung thời gian `2026-08-11 03:45Z`, sau đó mở trace để xem waterfall.
- Giải thích một span đáng chú ý: Span retrieve/RAG là span đáng chú ý nhất vì incident `rag_slow` xảy ra trước bước generation, làm toàn bộ request tăng latency nhưng không sinh lỗi 500.

**Câu hỏi phản biện (Checkpoint 1):**
- *Sự khác biệt lớn nhất giữa log baseline (CP0) và log CP1:* Log CP0 thiếu `correlation_id` nên không thể gom nhóm các sự kiện của cùng 1 request, thiếu metadata ngữ cảnh và dễ để lộ dữ liệu nhạy cảm. Log CP1 gắn `correlation_id`, thêm enrichment như `user_id_hash`, `session_id`, `feature`, `model`, đồng thời đã redact PII nên dễ truy vết và an toàn hơn.
- *Tại sao phải gọi `clear_contextvars()` ở đầu middleware?* Vì FastAPI/Uvicorn xử lý bất đồng bộ nên context có thể bị giữ lại giữa các request. Nếu không xóa context cũ thì metadata hoặc dữ liệu nhạy cảm của request trước có thể bị ghi nhầm sang request sau.

## 4. Prompt versioning

- Prompt name: `day13-chat`
- Version/label baseline: `production`
- Version/label candidate: [Điền version/label đã tạo trên Langfuse]
- Trace ID của mỗi version: [Điền trace ID thực tế]
- Bằng chứng đổi label hoặc rollback: [Đính kèm ảnh hoặc trace evidence trên Langfuse]

## 5. Dashboard, SLO và alerts

- Kết quả `validate_dashboard.py`: `HỢP LỆ: 6/6 panel có trong dashboard contract.`
- Evidence dashboard: Dashboard đã có đủ 6 nhóm panel theo contract trong `config/dashboard.yaml`. [Đính kèm ảnh dashboard runtime]
- SLO đã chọn và lý do: Chọn SLO latency theo p95 cho endpoint chat, đặc biệt theo feature `refund`, với ngưỡng dưới 2000ms vì challenge quy định `latency_threshold_ms=2000`. Đây là chỉ số nhạy nhất để phát hiện incident `rag_slow`.
- Alert rules và runbook: Alert khi p95 latency vượt ngưỡng 2000ms trong một khoảng thời gian ngắn. Runbook điều tra theo thứ tự metrics → trace waterfall → correlation ID → logs để xác định vị trí và nguyên nhân gốc.

## 6. Điều tra challenge

- Challenge ID: `day13-k3-observability-v1` (`incident=rag_slow`, `affected_feature=refund`, `latency_threshold_ms=2000`)
- Evidence metrics/dashboard: `submission/evidence/checkpoint3_role3_dashboard-incident.png`, `submission/evidence/challenge-metrics.txt`, `submission/evidence/challenge-log-evidence.txt`.
- Triệu chứng từ metrics: Sau khi chạy `python scripts/load_test.py --challenge --concurrency 5` trong lượt điều tra ngày `2026-08-11`, cả 5 request đều `200 OK` nhưng latency tăng vọt. Snapshot từ `/metrics` khi đó là: `traffic=5`, `latency_p50=3380ms`, `latency_p95=3397ms`, `latency_p99=3397ms`, `error_breakdown={}`. So với baseline trước incident khoảng `0.8-0.9s/request` trong log, đây là latency spike chứ không phải error spike.
- Evidence dashboard role 3 ghi nhận cùng loại triệu chứng: baseline `latency_p95=1143ms`; sau challenge `latency_p95=3594ms`, `latency_p99=3664ms`, `error_breakdown={}`, `quality_avg=0.8778`. Dashboard incident hiển thị latency P95 vượt SLO `3000ms`, còn traffic/error/cost/tokens/quality không vi phạm.
- Trace ID liên quan: Trên Langfuse, lọc traces theo `feature=refund` hoặc `session_id` trong khung `2026-08-11 03:45Z`. Các session của challenge: `k3-challenge-s01` → `req-c3144354`, `k3-challenge-s02` → `req-247b7289`, `k3-challenge-s03` → `req-be702446`, `k3-challenge-s04` → `req-33f8e10c`, `k3-challenge-s05` → `req-d8cf6798`. Trace waterfall kỳ vọng span chậm nằm ở bước retrieve/RAG, không phải generation.
- Log line/correlation ID liên quan: `req-be702446` (`2026-08-11T03:45:26Z` → `03:45:30Z`, `latency_ms=3388`), `req-c3144354` (`latency_ms=3380`), `req-d8cf6798` (`latency_ms=3397`), `req-33f8e10c` (`latency_ms=3354`), `req-247b7289` (`latency_ms=3361`). Tất cả đều cùng `feature=refund`, model `claude-sonnet-4-5`, env `dev`, và xuất hiện sau log `incident_enabled` với payload `rag_slow`.
<<<<<<< HEAD
- Root cause: incident `rag_slow` được bơm vào layer retrieval. Bằng chứng ở `app/mock_rag.py`: khi `STATE["rag_slow"] = True`, hàm `retrieve()` chủ động `time.sleep(2.5)`. Vì `app/agent.py` gọi `retrieve(message)` trước LLM generate, toàn bộ request bị đội latency thêm khoảng 2.5 giây.
- Fix action: tắt incident bằng `python scripts/inject_incident.py --disable` hoặc `POST /incidents/rag_slow/disable`, sau đó chạy lại load test để xác nhận latency quay về baseline. Nếu đây là production thật, cách fix kỹ thuật là timeout retrieval, cache hot queries `refund`, và fallback gracefully khi vector store chậm.
- Preventive measure: đặt alert cho p95 latency theo feature `refund` vượt `2000ms`; thêm sub-component tracing cho span `retrieve`; ghi metadata nguồn docs/cache-hit; thêm circuit breaker/timeout cho RAG; và chuẩn hóa runbook `metrics → trace waterfall → correlation ID → logs`.
- Trace ID Role 2 sau khi bổ sung child spans:
  - `k3-challenge-s01`: `f6e6142c21b9a611c7cc1217e717a18f`
  - `k3-challenge-s02`: `6b989dcde63d896fb132698d94070a73`
  - `k3-challenge-s03`: `a512d07b66b260df864235cc3b3c3191`
  - `k3-challenge-s04`: `a1ae0be6f1404c1c22a59552ca9d365c`
  - `k3-challenge-s05`: `a5c5edd50f6f3c757299b95f9de28011`
- Correlation ID Role 2 tương ứng: `req-21e3d8dc`, `req-71209877`, `req-f7a0dae0`, `req-9bdb656b`, `req-719c1e28`.
- Span bất thường trong trace waterfall Role 2: `rag.retrieve` mất khoảng `2.501-2.505s`, trong khi `llm.generate` chỉ khoảng `0.151-0.153s`.
- Evidence Role 2: `submission/evidence/role2_checkpoint3_trace_summary.md`.
=======
- Root cause: Incident `rag_slow` được bơm vào layer retrieval. Bằng chứng ở `app/mock_rag.py`: khi `STATE["rag_slow"] = True`, hàm `retrieve()` chủ động `time.sleep(2.5)`. Vì `app/agent.py` gọi `retrieve(message)` trước LLM generate, toàn bộ request bị đội latency thêm khoảng 2.5 giây.
- Fix action: Tắt incident bằng `python scripts/inject_incident.py --disable` hoặc `POST /incidents/rag_slow/disable`, sau đó chạy lại load test để xác nhận latency quay về baseline. Nếu đây là production thật, cách fix kỹ thuật là timeout retrieval, cache hot queries `refund`, và fallback gracefully khi vector store chậm.
- Preventive measure: Đặt alert cho p95 latency theo feature `refund` vượt `2000ms`; thêm sub-component tracing cho span `retrieve`; ghi metadata nguồn docs/cache-hit; thêm circuit breaker/timeout cho RAG; và chuẩn hóa runbook `metrics → trace waterfall → correlation ID → logs`.
>>>>>>> 579813c (docs: update group report)

## 7. Đóng góp cá nhân

| Thành viên | Phần việc | Commit/PR | Điều đã học |
|---|---|---|---|
<<<<<<< HEAD
| Nguyễn Tuấn Anh - 2A202601775 | Dashboard, SLO & Alert: dựng dashboard, cấu hình SLO và alert rule | `f1a02e5` | Biết cách chọn chỉ số quan trọng và biến chúng thành dashboard/alert hữu ích |
| Lê Mạnh Cương - 2A202601137 | Tracing & Prompt Version: cấu hình trace, theo dõi prompt version, bổ sung child spans `rag.retrieve`, `prompt.resolve`, `llm.generate` và dùng waterfall để khoanh vùng checkpoint 3 | `f1a02e5`, commit Role 2 checkpoint 3 | Hiểu cách trace giúp khoanh vùng bottleneck, phân biệt nghẽn retrieval với LLM và liên kết prompt version với request |
| Lê Ngọc Khánh - 2A202601487 | Incident, Report & Demo: chạy challenge, nối metrics → traces → logs, xác định root cause `rag_slow`, hoàn thiện báo cáo và demo | `8352785b8ce37386264bcd81a9555d403ac3b8a0` | Biết cách điều tra incident theo flow metrics → traces → logs và dùng correlation ID để chứng minh root cause |
| Vũ Ngọc Thiện - 2A202601793 | Logging & PII: chuẩn hóa JSON log, correlation ID, enrichment và PII redaction | `7a57bfb` | Hiểu vai trò của structured logging và cách giảm rủi ro lộ dữ liệu nhạy cảm |

## 8. File evidence đã có trong repo

- `submission/evidence/validate-logs.txt`
- `submission/evidence/validate-dashboard.txt`
- `submission/evidence/README.md`
- `submission/evidence/role2_checkpoint3_trace_summary.md`

## 9. Việc còn lại để nộp đầy đủ

- Bổ sung ảnh danh sách traces trên Langfuse với tối thiểu `10 traces`.
- Bổ sung ảnh trace waterfall.
- Bổ sung ảnh hai prompt version và trace tương ứng cho `baseline` và `candidate`.
- Bổ sung ảnh thao tác promote/rollback label `production`.
- Điền link dashboard runtime và các trace ID thực tế vào các mục còn trống ở trên.
=======
| Nguyễn Tuấn Anh - 2A202601775 | Dashboard, SLO & Alert: dựng dashboard, cấu hình SLO và alert rule | [Điền commit/PR] | Biết cách chọn chỉ số quan trọng và biến chúng thành dashboard/alert hữu ích |
| Lê Mạnh Cương - 2A202601137 | Tracing & Prompt Version: cấu hình trace, theo dõi prompt version và evidence trên Langfuse | [Điền commit/PR] | Hiểu cách trace giúp khoanh vùng bottleneck và liên kết prompt version với request |
| Lê Ngọc Khánh - 2A202601487 | Incident, Report & Demo: chạy challenge, nối metrics → traces → logs, xác định root cause `rag_slow`, hoàn thiện báo cáo và demo | `8352785b8ce37386264bcd81a9555d403ac3b8a0` | Biết cách điều tra incident theo flow metrics → traces → logs và dùng correlation ID để chứng minh root cause |
| Vũ Ngọc Thiện - 2A202601793 | Logging & PII: chuẩn hóa JSON log, correlation ID, enrichment và PII redaction | [Điền commit/PR] | Hiểu vai trò của structured logging và cách giảm rủi ro lộ dữ liệu nhạy cảm |
>>>>>>> 579813c (docs: update group report)
