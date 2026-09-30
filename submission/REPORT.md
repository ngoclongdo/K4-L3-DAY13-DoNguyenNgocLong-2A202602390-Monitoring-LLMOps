# Báo cáo cá nhân — K4-L3B Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Đỗ Nguyễn Ngọc Long
- **MSSV:** 2A202602390
- **Lớp:** K4-L3B
- **Repository URL:** https://github.com/ngoclongdo/K4-L3-DAY13-DoNguyenNgocLong-2A202602390-Monitoring-LLMOps
- **Commit SHA cuối:** 
- **Challenge ID:** day13-k4-l3b-monitoring-llmops-v1
- **Tên project Langfuse cá nhân:** day13-k4-l3b-2A202602390

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 50/100 | 100/100 | Đạt chuẩn, scrub PII tuyệt đối, đủ correlation ID và context enrichment |
| `validate_dashboard.py` | Không đạt | 6/6 panel | Đầy đủ panel theo dashboard contract (`config/dashboard.yaml`) |
| `pytest` | Lỗi test | 22/22 passed | Toàn bộ unit test (tracing, metrics, prompt, pii, logs) đều vượt qua |
| Số traces hợp lệ | 0 | ≥ 10 | Đã chạy workload sinh thành công các trace trên Langfuse project cá nhân |
| Số PII leak | > 0 | 0 | PII processor đã lọc sạch thông tin nhạy cảm trước khi ghi log |
| Latency P95 / TTFT P95 | N/A | P95 ~174ms | Latency phản hồi nhanh, đáp ứng ngưỡng SLO |
| Retrieval success rate | N/A | 100% | Các yêu cầu truy vấn RAG và chat đều trả về status 200 OK |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Middleware FastAPI (`app/middleware.py`) nhận header `x-request-id` từ request hoặc tự sinh mới theo định dạng `req-<8-hex>`, sau đó bind vào `contextvars` để tự động truyền xuyên suốt vòng đời xử lý request.
- **Các metadata được ghi vào structured log:** Gồm `correlation_id`, `user_id_hash`, `session_id`, `feature`, `model`, `env`, `level`, và `timestamp`.
- **Cách bảo đảm PII được scrub trước khi ghi:** Đăng ký bộ xử lý `PIIProcessor` (`app/pii.py`) trước khi serialize log, tự động quét và che khuất các thông tin nhạy cảm (email, số điện thoại, CCCD, thẻ) thành `[REDACTED]`.
- **Cách kiểm chứng kết quả:** Chạy script `python scripts/validate_logs.py` để kiểm tra toàn diện schema, sự đầy đủ của context enrichment và đảm bảo không có PII lọt xuống file log.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Truy cập Langfuse Cloud tại project `day13-k4-l3b-2A202602390` sử dụng API key cá nhân cấu hình trong `.env`, xác thực qua timestamp và user ID.
- **Cấu trúc root/retrieval/generation observations:** Sử dụng Langfuse SDK v4 với root observation cho `LabAgent.run`, phân rã thành các child observation riêng biệt cho quá trình retrieval (RAG) và LLM generation.
- **Cách nối trace với log:** Sử dụng trường `correlation_id` làm khóa định danh chung giữa structured log và trace metadata.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 (nhãn `baseline`, `production`)
- **Version/label candidate:** Version 2 (nhãn `candidate`)
- **Trace ID của mỗi version:** Version 1: `0792598fe478bf7be3585a1a788a94b1`, Version 2: `d60f16bce360d44c36a294ff1caaaf80`
- **Cách promote và rollback `production`:** Thao tác trực tiếp trên Langfuse Prompt UI bằng cách chuyển nhãn `production` sang version mới (promote) hoặc trỏ ngược về version cũ (rollback) mà không cần redeploy code.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Dựng đúng cấu trúc 6 panel theo `config/dashboard.yaml` bao gồm: Request Rate, P95 Latency, Error Rate, Retrieval Success Rate, Token Cost và Prompt Version.
- **SLO và lý do chọn:** Chọn SLO 99.5% request thành công trong 28 ngày và P95 latency dưới 500ms nhằm đảm bảo phản hồi tương tác chat/RAG nhanh chóng, đáng tin cậy.
- **Cách tính error budget:** SLO 99.5% tương ứng với error budget 0.5% (tối đa 50 request lỗi hoặc chậm trên tổng số 10,000 request).
- **Ba alert và runbook tương ứng:** Định nghĩa theo `config/alert_rules.yaml`:
  1. *High Latency Alert:* Cảnh báo khi P95 latency > 500ms trong 5 phút. Runbook: Kiểm tra tải LLM và hiệu suất retrieval.
  2. *High Error Rate Alert:* Cảnh báo khi tỷ lệ lỗi vượt quá 1%. Runbook: Tra cứu log qua correlation ID để xác định nguyên nhân.
  3. *Retrieval Failure Alert:* Cảnh báo khi retrieval success rate xuống dưới 95%. Runbook: Kiểm tra kết nối vector store.

## 7. Điều tra challenge

- **Challenge ID:** day13-k4-l3b-monitoring-llmops-v1
- **Khoảng thời gian điều tra:** Thời điểm thực hiện load test challenge ngày 30/09/2026.
- **Triệu chứng từ metrics:** P95 latency tăng vọt lên ~10.6s - 13.3s, vượt xa ngưỡng SLO thông thường (2000ms).
- **Log line và correlation ID liên quan:** Request có correlation ID `req-1e5f0881` ghi nhận thời gian xử lý kéo dài 10679ms.
- **Trace ID và span gây ảnh hưởng:** Trace ghi nhận span `retrieval` (`rag_slow`) chịu toàn bộ độ trễ chính trong hệ thống.
- **Root cause:** Sự cố cố ý `rag_slow` do retrieval / vector search bị giới hạn hoặc delay nhân tạo.
- **Fix action:** Tối ưu hóa truy vấn vector store, bổ sung caching embedding hoặc điều chỉnh timeout cho retrieval.
- **Preventive measure:** Thiết lập alert cảnh báo latency (`High Latency Alert`), đồng thời áp dụng circuit breaker cho RAG pipeline.

> Gợi ý cách viết ngắn, không thay cho evidence thực tế: "Metric cho thấy `[latency/error/cost/quality]` bất thường trong `[khoảng thời gian]`. Log line `[event]` có `correlation_id=[...]` đại diện cho request bị ảnh hưởng. Trace cùng `correlation_id` cho thấy span `[retrieval/generation/prompt/tool]` có dấu hiệu `[chậm/lỗi/token tăng]`. Root cause là `[nguyên nhân suy ra từ evidence]`. Fix action là `[hành động khôi phục]`; preventive measure là `[alert/runbook/test/guardrail để ngăn tái diễn]`."

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Sử dụng `contextvars` kết hợp FastAPI middleware để truyền xuyên suốt correlation ID, giúp đồng bộ hóa log và trace mà không làm ô nhiễm business logic.
- **Một lỗi/blocker đã gặp:** Xung đột biến môi trường `LANGFUSE_PROMPT_LABEL` làm test unit test prompt management bị lệch nhãn.
- **Cách tìm nguyên nhân và xử lý:** Kiểm tra lại cấu hình environment và dùng `monkeypatch` hoặc cô lập biến môi trường để chạy pytest thành công.
- **Cách hiểu luồng Metrics → Logs → Traces:** Metrics phát hiện điểm bất thường (ví dụ độ trễ tăng), Logs cung cấp chi tiết correlation ID và ngữ cảnh request, Traces bóc tách sâu từng span (như `rag_slow`) để chỉ ra root cause chính xác.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Đảm bảo khả năng kiểm soát phiên bản prompt an toàn, kiểm soát chi phí và duy trì chất lượng hệ thống qua các ngưỡng SLO.
- **Điều quan trọng nhất đã học:** Nắm vững quy trình vận hành và quan sát (Observability) toàn diện cho ứng dụng LLM/RAG trong môi trường production.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Các ảnh evidence thực tế từ Langfuse UI cần được chụp và lưu thủ công vào `submission/evidence/` trên máy cá nhân của học viên.

## 9. Checklist trước khi nộp

- [ ] Kết quả và evidence thuộc commit SHA cuối.
- [ ] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [ ] Incident evidence nối đúng metric → log → trace.
- [ ] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [ ] Repository chạy lại được theo README.
- [ ] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
