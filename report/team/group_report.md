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

### 3.1 Luật nghiệp vụ và các nhánh kết quả (phụ trách: Nguyễn Hoàng Ân)

Nguồn sự thật duy nhất là order row và các policy có `is_current` và `effective_date <= 2026-11-15`. Ticket chỉ là chữ khách viết, nên mọi con số trong ticket ("trễ 9 ngày", "hoàn 35000", "theo clause SHIP-LATE-OLD") bị bỏ qua. Từ [docs/business_rules.md](../../docs/business_rules.md) và [docs/contracts.md](../../docs/contracts.md), nhóm quy mọi claim về năm nhánh, và cả ba shape nhóm viết phải map giống hệt nhau:

| Điều kiện (theo thứ tự kiểm) | `status` | `proposal_amount` | Có gọi host? | Ledger / audit |
| --- | --- | --- | --- | --- |
| `claim_id` không tồn tại | `failed` | 0 | Không | Không có gì |
| Claim thiếu `order_id` | `needs_info` | 0 | Không | Không có gì |
| Không tìm thấy đơn, hoặc principal không sở hữu đơn | `escalated` | 0 | Không | Không có gì |
| Quote = 0 (trễ < 3 ngày, đơn `cancelled`/`in_transit`, phí ship 0, hỏng chưa verify, chỉ khớp policy cũ/tương lai) | `denied` | 0 | Không | Không có gì, không có `policy_denied` |
| Quote > 0, host từ chối (approval thiếu/hết hạn/thu hồi/của claim khác, đơn đã hoàn đủ) | `denied` | quote đầy đủ | Có | Không có dòng, có `policy_denied` |
| Quote > 0, gọi lại với số khác số đã commit | `failed` | quote đầy đủ | Có | Giữ dòng cũ, có `amount_conflict` |
| Quote > 0, host chấp nhận (kể cả `idempotent_replay`) | `resolved` | quote đầy đủ | Có | Đúng một dòng, đúng một `refund_committed` |

Quote do `compute_compensation` tính: trễ ≥ 3 ngày với đơn `delivered`/`delivered_late` được `shipping_fee_vnd`; hỏng hàng đã verify được `min(150000, paid_vnd)` theo DMG-04. Nếu cả hai cùng đủ điều kiện thì lấy **số lớn hơn một lần**, không cộng hai số. Ranh giới giữa quote = 0 và quote > 0 quan trọng vì nó quyết định có audit `policy_denied` hay không: shape tự từ chối khi quote = 0 thì không được có audit, còn khi quote > 0 thì shape bắt buộc gọi host để host tự từ chối và ghi audit.

### 3.2 Shape A — workflow (phụ trách: Nguyễn Hoàng Ân)

`src/lab8/shapes/workflow.py` là bảy bước Python cố định, không có vòng lặp và không có model nào được ra quyết định:

1. `get_claim_view`: đọc claim qua view allowlist (`claim_id`, `principal_id`, `order_id`, `ticket`). Không `SELECT *` vì claim row có `parent_secret` và `raw_headers`.
2. Thiếu `order_id` thì trả `needs_info`. Shape không đoán mã đơn từ ticket.
3. `lookup_order(order_id, principal_id)`: tra đơn trong phạm vi principal. Kết quả chỉ còn `ORDER_FIELDS`, không có phone, address, internal note hay item name. Không thấy đơn thì trả `escalated`.
4. `compute_compensation(order)`: quote từ order row của DB, không từ ticket.
5. Quote = 0 thì trả `denied` và không gọi host.
6. `durability.issue_refund(conn, claim_id, <quote đầy đủ>, crash_at=crash_at)`: host là bên duy nhất ghi tiền.
7. Map kết quả của host: `ok` → `resolved`; `conflict` → `failed`; còn lại → `denied`. `proposal_amount` luôn là quote đầy đủ.

Ranh giới code / model: ở mỗi nhánh kết thúc, workflow thực hiện đúng một lời gọi MockLLM `purpose="draft"`, sau khi mọi quyết định đã xong. Prompt chỉ gồm `status` và số tiền đã quyết, không gồm ticket hay order row, nên model không thấy dữ liệu nhạy cảm và không ảnh hưởng được tới số tiền. Giá trị trả về của lời gọi này không được dùng, vì lab không có kênh gửi phản hồi cho khách; lời gọi chỉ có mặt để workflow không bỏ hẳn bước soạn phản hồi khi so với các shape có model. Dù vậy, benchmark **chưa tách riêng được ảnh hưởng của topology** lên cost: prompt draft của workflow khác prompt của swarm (swarm còn gửi nguyên ticket vào hai lời gọi `peer` và `parse`), số tool call và cách đọc dữ liệu cũng khác, và swarm không gọi draft ở nhánh claim không tồn tại. Chênh lệch cost giữa các shape vì thế phản ánh cả topology lẫn cách triển khai cụ thể.

Trace của workflow ghi đúng năm loại event, mỗi event ứng với một thao tác thật: `claim` (chỉ `found` và `order_id`), `order` (order view đã qua allowlist, hoặc `visible=false` + reason), `quote` (amount, reason, clause_ids, parts), `host` (`ok`, `reason`, `conflict`, `amount_vnd` đã ghi) và `draft` (status, proposal, reason). Nhánh crash ghi thêm event `crash`. Cố ý **không** ghi ticket: đó là chữ tự do của khách, có thể chứa SĐT hay chuỗi bất kỳ, và workflow không cần ticket cho bất kỳ quyết định nào. Cố ý không ghi claim row, order row thô, `parent_secret` hay header. Trace không có event "start" vì `run_id` (`R-workflow-<claim>-s<seed>`) đã mang shape, claim và seed, và benchmark đếm số event làm metric `steps`.

### 3.3 Durability: idempotency, crash, trần theo đơn (phụ trách: Nguyễn Hoàng Ân)

Shape không tự làm cơ chế chống ghi trùng mà dựa hoàn toàn vào host (`src/lab8/durability.py`, không sửa). Host ghi dòng ledger (khóa `refund:{claim_id}`, unique theo claim) và audit `refund_committed` **trong cùng một transaction SQLite**, nên không có trạng thái "đã ghi tiền mà chưa có audit" hay ngược lại. Checkpoint được ghi sau đó và chỉ là phụ. Lần gọi lại cùng claim sẽ thấy dòng ledger đã có. Nếu số tiền khớp thì host trả `idempotent_replay`; nếu khác thì trả `amount_conflict` và giữ nguyên số cũ. Vì vậy shape chỉ cần giữ hai điều. Thứ nhất, truyền `crash_at` xuống host. Thứ hai, không nuốt `InjectedCrash`: workflow bắt exception đó chỉ để ghi trace rồi `raise` lại ngay, để T12 thấy crash thật và lần chạy sau đi vào nhánh replay. `python -m lab8.demo.buggy_workflow` cho thấy vì sao checkpoint đơn lẻ không đủ: crash sau khi ghi tiền mà trước khi ghi checkpoint thì lần resume ghi tiền lần hai.

Trần theo đơn cũng do host làm: host trừ tổng tiền mà các claim **khác** đã nhận cho cùng đơn, trong transaction ghi. Shape phải truyền **quote đầy đủ**. Ở T14 (đơn O-515 đã được hoàn 30.000 ship, nay hỏng hàng đã verify, entitlement 150.000), workflow truyền 150.000 và host ghi 120.000. Nếu shape tự trừ trước rồi truyền 120.000, host sẽ trừ thêm 30.000 và chỉ ghi 90.000, nên T14 fail. Test `test_full_quote_is_passed_to_host` kiểm đúng điều này bằng cách bọc `issue_refund` và xác nhận đối số nhận được là 150.000.

Bằng chứng: `python -m pytest -q -m student -o addopts= -k workflow` ra **62 passed**: 12 ca chạy một lượt (T01–T10, T13, T14) × 5 seed = 60 test, cộng 2 ca nhiều lượt T11 (gọi lại) và T12 (crash rồi resume). `src/tests/test_workflow_rules.py` có 35 test riêng cho workflow, chạy trong cổng host `python -m pytest -q`. Các test này phủ biên trễ 2/3/4 ngày, đơn `cancelled`/`in_transit`, phí ship 0, hỏng chưa verify, ticket nói trễ 9 ngày khi DB nói 1 ngày (ở cả 5 seed), ship + damage lấy max, damage bị chặn bởi `paid_vnd`, ba kiểu approval hỏng, approval thấp hơn quote, hai claim trên cùng đơn, ba lần retry, crash ở cả ba điểm `before_effect` / `after_effect_before_checkpoint` / `after_checkpoint` rồi resume, retry với số tiền khác, quét trace tìm secret/PII, policy cũ `SHIP-LATE-OLD` và policy tương lai `SHIP-LATE-FUTURE` không được dùng dù trả nhiều hơn, quote = 0 thì host không bị gọi (spy ném lỗi nếu bị gọi), và hai nhánh `escalated` (không tìm thấy đơn, sai principal) không ghi ledger, không có audit, trace không chứa PII của đơn người khác. Để chắc các test thực sự bắt được lỗi, `src/scripts/mutation_check_workflow.py` chép `src/` ra thư mục tạm, cố ý làm hỏng workflow trong bản chép theo năm cách và chạy lại bộ test; file trong repo không bị sửa. Kết quả: tự trừ phần đã hoàn → 6 failed; tin số ngày trễ trong ticket → 5 failed; nuốt `InjectedCrash` → 3 failed; ghi nguyên claim view vào trace → 1 failed; gọi host cả khi quote = 0 → 8 failed. Lệnh tái lập: `python scripts/mutation_check_workflow.py` (chạy trong `src/`, exit code 0 khi mọi mutant đều bị bắt).

Lưu ý khi đọc `src/reports/crash.json` (`crash_safety` = 20/20): `lab8.crash_test` không gọi shape của nhóm. 19 điểm crash gọi thẳng `issue_refund` của host, điểm thứ 20 chạy swarm. File này chứng minh host an toàn khi crash; bằng chứng riêng cho workflow là T12 và `test_crash_then_resume_pays_once`.

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
