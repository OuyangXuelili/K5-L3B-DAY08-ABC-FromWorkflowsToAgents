# Báo cáo nhóm — Lab 8 LogiX-VN

Tối đa 5 trang ngoài phụ lục. Số liệu lấy từ file trong `src/reports/`, không chép tay từ terminal.

## 1. Thông tin nhóm

- Tên nhóm:
- Thành viên: xem `TEAM.md`
- Commit chốt (trùng `report/team/decision_adr.md`):

## 2. Mục tiêu và cách tiếp cận

Nhóm hiểu bài toán thế nào, chia việc ra sao, thứ tự làm bốn shape.

...

## 3. Giải pháp

Mô tả ba shape nhóm viết (workflow, single, orchestrator): mỗi shape đọc gì, quyết định gì, gọi host ra sao. Chỉ rõ ranh giới code / model (tham khảo [docs/architecture.md](../../docs/architecture.md)), cách handoff được sanitize và trace ghi những event nào.

Phần workflow/single do chủ sở hữu tương ứng điền; dưới đây là phần C do Phong phụ trách.

### C — Orchestrator và investigator chỉ đọc

`src/lab8/shapes/orchestrator.py` chạy theo năm bước cố định:

1. Đọc claim qua `get_claim_view` (không `SELECT *`). Claim không tồn tại → `failed`; thiếu `order_id` → `needs_info`; cả hai không ghi ledger.
2. Gọi MockLLM một lần để plan với prompt không chứa ticket. Model không chọn số tiền.
3. Tạo `handoff.v1` bằng `make_handoff`: chỉ `claim_id`, `task`, `facts` đã lọc theo `FACT_KEYS`, `allowed_tools` read-only. Không chuyển `ticket`, `parent_secret`, `raw_headers`, history hay `principal_id`.
4. `run_readonly_investigators` chạy `lookup_order`, `lookup_policy`, `lookup_refund_history`, `compute_compensation` trong principal scope của claim. Order ngoài scope hoặc không tồn tại → `escalated`. Investigator trả order nhưng thiếu quote → orchestrator tự gọi `compute_compensation` trên order view đã sanitize (fallback có giới hạn) và ghi event `host_recompute_quote`; không có order thì fail-closed, không ghi tiền.
5. Chỉ orchestrator gọi `durability.issue_refund`, và truyền **quote đầy đủ**. Host kiểm approval, trần theo đơn và idempotency trong một transaction. `proposal_amount` giữ quote: T13 denied 35.000 VND; T14 resolved 150.000 VND nhưng ledger chỉ 120.000 VND.

Ranh giới code/model: model chỉ plan/draft chữ qua MockLLM (2 model call/trial); tiền, quyền ghi, principal scope, approval, cap và retry do code/host quyết. Investigator không có registry ghi.

### Handoff, trace và bảo mật dữ liệu

- Allowlist nằm trong code (`lab8/handoff.py`), không nằm trong prompt. `sanitize` bỏ key lạ và từ chối canary còn sót; `make_handoff` từ chối mọi tool trong `WRITE_TOOLS` (`issue_refund`, `append_audit`).
- Trace chỉ ghi `plan`, `handoff` đã sanitize, `investigator` (order view đã qua `ORDER_FIELDS`, quote, refund history count), `quote`, `decision`, `refund`, `crash`. Không ghi ticket, secret, header, SĐT, địa chỉ, ghi chú nội bộ hay tên hàng.
- Lệnh rà: `python -m lab8.trace_audit --shapes workflow,single,orchestrator --seeds 11,22,33,44,55 --out reports/trace_audit.json`. Audit kiểm tra run id, file mới trong từng lần chạy, và quét result + mọi file `<run_id>*` theo index secret/PII dựng từ world/claims. Report chỉ ghi fingerprint, không ghi giá trị nhạy cảm. Kết quả tại commit này: 210/210 trial có trace hợp lệ và mới, 0 leak trial; orchestrator 70/70 pass. Workflow/single mới đạt phần trace (70/70 trace sạch, 0 pass) vì shape còn stub khi rà — sẽ chạy lại sau khi hai owner hoàn thành.
- Test: `tests/test_orchestrator_handoff.py` (19 test) và `tests/test_trace_audit.py` (5 test) phủ allowlist, quyền tool, principal scope, thiếu đơn/claim, investigator thiếu dữ liệu, crash, trace freshness và leak.

## 4. Kết quả

Bảng so sánh 4 shape trên bộ public (từ `src/reports/benchmark.json`): pass rate, pass^5, seed min–max, variance, cost/success, mean/p95 steps. Kèm crash safety (`src/reports/crash.json`) và, nếu có, kết quả tự kiểm trên `student_drop/`.

...

Quyết định kiến trúc và lý do: xem `report/team/decision_adr.md`. Phiếu nghiên cứu và ablation: xem `report/team/research_card.md`.

## 5. Khó khăn và cách xử lý

Lỗi gặp phải (kèm lệnh, output hoặc ảnh chụp) và cách nhóm xử lý.

...

## 6. Tổng hợp đóng góp

Chi tiết đóng góp của từng thành viên nằm trong `TEAM.md` (bảng đóng góp) và báo cáo cá nhân `report/team/MSSV_HoTen.md`.

## 7. Hạn chế và hướng cải thiện

MockLLM không phải model sản xuất; latency là `unavailable`. Nêu thêm giới hạn của nhóm và điều kiện làm kết luận thay đổi.

- Orchestrator tốn thêm một model call plan và một draft. Ablation thêm investigator cho thấy cost/success tăng 32,5% (57,17 → 75,74 VND mô phỏng) mà pass rate, pass^5 và variance không đổi. Vì vậy chỉ thêm investigator khi có quyết định thực sự phụ thuộc vào kết quả của nó; với claim flow tuần tự này, một investigator là đủ.
- Audit trace chứng minh không lộ dữ liệu theo index public (secret, header, SĐT, địa chỉ, ghi chú, tên hàng, ticket). Bộ hidden có thể có PII ở dạng khác, nên phải chạy lại `lab8.trace_audit` trên `student_drop/` và vẫn cần review code; scanner không thay thế kiểm tra ranh giới.
- Kết luận scaling chỉ trong phạm vi 14 claim public × 5 seed, task tuần tự, MockLLM. Nếu task có nhánh song song thật hoặc investigator thứ hai cung cấp kiểm chứng độc lập mà nhánh quyết định dùng, kết quả có thể đảo.

...

## 8. Tài liệu tham khảo

...
