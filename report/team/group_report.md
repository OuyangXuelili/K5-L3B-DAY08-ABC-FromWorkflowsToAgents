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

...

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

...

## 8. Tài liệu tham khảo

...
