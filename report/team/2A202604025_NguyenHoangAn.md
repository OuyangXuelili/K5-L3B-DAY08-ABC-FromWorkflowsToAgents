# Báo cáo cá nhân — 2A202604025_NguyenHoangAn

## 1. Thông tin thành viên

- Họ tên: Nguyễn Hoàng Ân
- MSSV: 2A202604025
- Email: hoangan061196@gmail.com
- GitHub: hoangan06

## 2. Phần việc phụ trách

Tôi phụ trách shape A `workflow` (`src/lab8/shapes/workflow.py`) và độ đúng nghiệp vụ của luồng xử lý claim. Phần việc gồm: chuyển luật hoàn tiền thành các nhánh kết quả (`resolved`, `denied`, `needs_info`, `escalated`, `failed`); viết luồng đọc claim → kiểm tra đơn và chủ sở hữu → tính quote → gọi host → trả kết quả; kiểm tra retry và crash; viết test riêng cho workflow; viết mục 3.1–3.3 của báo cáo nhóm (nghiệp vụ, workflow, durability).

## 3. Đóng góp cụ thể

Commit chính: `<điền hash sau khi commit>`.

Code: `src/lab8/shapes/workflow.py` thay stub `not_implemented` bằng bảy bước cố định. Đó là `get_claim_view` → kiểm `order_id` → `lookup_order` theo principal → `compute_compensation` → quote = 0 thì `denied` → `durability.issue_refund(conn, claim_id, <quote đầy đủ>, crash_at=crash_at)` → map kết quả của host. Mọi nhánh return đi qua cùng một hàm kết thúc dùng `common.finish()`, nên trace luôn được ghi và `refund_effects` / `committed_audit_count` luôn lấy từ ledger.

Test: `src/tests/test_workflow_rules.py` có 35 test, chạy trong cổng host. Các test chia theo năm nhóm:
- Biên điều kiện: trễ 2/3/4 ngày; đơn `cancelled`/`in_transit`; phí ship 0; hỏng hàng chưa verify; ticket nói trễ 9 ngày khi DB nói 1 ngày, ở cả 5 seed; ship + damage lấy max chứ không cộng; damage bị chặn bởi `paid_vnd`; thiếu `order_id`; claim không tồn tại.
- Approval: thu hồi, hết hạn, cấp cho claim khác (kiểm lý do trong audit `policy_denied`), approval thấp hơn quote.
- Trần theo đơn: spy trên `issue_refund` xác nhận T14 truyền 150.000 chứ không phải 120.000; hai claim trên cùng một đơn thì claim sau bị `order_already_refunded` và tổng ledger của đơn không vượt 35.000.
- Các nhánh không được chạm host: quote = 0 (trễ 2 ngày, đơn hủy, phí ship 0, hỏng chưa verify) và hai nhánh `escalated` (không tìm thấy đơn, sai principal), kiểm bằng spy ném lỗi nếu `issue_refund` bị gọi; policy cũ `SHIP-LATE-OLD` và policy tương lai `SHIP-LATE-FUTURE` không được dùng dù trả nhiều hơn.
- Durability và trace: ba lần retry vẫn một dòng; crash ở cả ba điểm rồi resume vẫn một dòng và một audit; retry với số tiền khác thì `failed`/`amount_conflict` và giữ số cũ; quét trace tìm secret, header, SĐT, địa chỉ, internal note, tên hàng và SĐT viết trong ticket.

Quyết định kỹ thuật:

1. Truyền quote đầy đủ cho host và không gọi `lookup_refund_history` trong workflow. Host đã tự trừ phần các claim khác đã nhận, nên shape tự trừ sẽ thành trừ hai lần (T14 fail). Workflow cũng không cần biết lịch sử hoàn để ra quyết định, nên bỏ tool này đi bớt được một tool call mỗi claim.
2. Bắt `InjectedCrash` chỉ để ghi trace rồi `raise` lại ngay. Nếu nuốt exception thì T12 không thấy crash, và lần chạy sau không chứng minh được gì.
3. Không ghi ticket vào trace. Ticket là chữ tự do của khách, có thể chứa SĐT hay chuỗi bất kỳ, và workflow không dùng ticket cho quyết định nào.
4. Mỗi event trong trace ứng với một thao tác thật. Bản đầu có thêm event `start` và `decision`. Khi đọc `lab8/benchmark.py`, tôi thấy metric `steps` chính là số event trong trace, nên hai event chỉ để ghi chép đó làm workflow bị đếm thừa bước (mean steps 5.43 thay vì 4.43 trên bộ public). Tôi bỏ `start` và gộp quyết định vào event `draft`.
5. Ở mỗi nhánh kết thúc, workflow thực hiện một lời gọi MockLLM `draft`, với prompt chỉ gồm status và số tiền; giá trị trả về không được dùng. Mục đích là không bỏ hẳn bước soạn phản hồi khi so với các shape có model. Lời gọi này không làm cost so sánh được hoàn toàn: prompt của workflow khác prompt của swarm, số tool call khác, và swarm không gọi draft ở nhánh claim không tồn tại. Vì vậy benchmark chưa tách riêng được ảnh hưởng của topology.

## 4. Bằng chứng

- `python -m pytest -q -m student -o addopts= -k workflow` → `62 passed`.
- `python -m pytest -q` (cổng host, gồm 35 test mới) → `89 passed`.
- `python -m pytest -q tests/test_workflow_rules.py` → `35 passed`.
- `python -m lab8.crash_test --count 20 --out reports/crash.json` → `crash_safety 20/20`. File này kiểm host và swarm, không gọi workflow; bằng chứng crash riêng của workflow là T12 và `test_crash_then_resume_pays_once` (3 điểm crash).
- Kiểm tra test có bắt được lỗi không: `python scripts/mutation_check_workflow.py` (trong `src/`) chép `src/` ra thư mục tạm, làm hỏng workflow trong bản chép theo năm cách rồi chạy `test_workflow_rules.py`; file trong repo không bị sửa. Kết quả: tự trừ phần đã hoàn → 6 failed; tin số ngày trễ trong ticket → 5 failed; nuốt `InjectedCrash` → 3 failed; ghi nguyên claim view vào trace → 1 failed; gọi host cả khi quote = 0 → 8 failed. Script trả exit code 0 khi mọi mutant đều bị bắt.
- Trace mẫu: `src/traces/R-workflow-C-014-s11.json` (không commit, sinh lại bằng lệnh test) có các event `claim`, `order`, `quote` (parts SHIP-LATE-03 = 30.000, DMG-04 = 150.000), `host` (`amount_vnd` 120.000), `draft` (`proposal_amount` 150.000).
- Số liệu benchmark của workflow: xem `src/reports/benchmark.json` và `report/team/decision_adr.md` (run id do nhóm chốt).

## 5. Công cụ AI đã dùng

Tôi dùng Claude để đọc code host (`tools.py`, `durability.py`, `policy.py`, `compensation.py`, `metrics.py`, `benchmark.py`), viết code workflow, viết test và soạn bản nháp báo cáo này. Tôi tự kiểm tra theo ba cách. Thứ nhất, chạy ma trận student, cổng host và crash suite. Thứ hai, làm hỏng code có chủ đích bằng script mutation (năm trường hợp ở mục 4) để chắc test không pass một cách vô nghĩa. Thứ ba, tự đọc trace sinh ra để xác nhận không có field nhạy cảm. Tôi giải thích được ranh giới code / model của workflow, lý do truyền quote đầy đủ và những field trace cố ý không ghi.

## 6. Tự đánh giá

Workflow đạt 62/62 test student và qua 35 test biên do tôi viết; tiền, trạng thái, ledger và audit khớp contract trên bộ public. Giới hạn: tôi chưa kiểm trên bộ hidden (`student_drop/`), và mọi số đo đều chạy trên MockLLM chứ không phải model thật.
