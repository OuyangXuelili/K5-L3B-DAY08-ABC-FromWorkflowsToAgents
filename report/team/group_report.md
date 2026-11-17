# Báo cáo nhóm — Day 8 LogiX-VN

## 1. Thông tin nhóm

- Nhóm: ABC · Lớp: L3B (theo xác nhận của nhóm; giữ tên repo hiện tại).
- Thành viên: Nguyễn Hoàng Ân — 2A202604025; Trần Văn Quang — 2A202604032; Trương Văn Phong — 2A202604091.
- Repo: [K5-L3B-DAY08-ABC-FromWorkflowsToAgents](https://github.com/OuyangXuelili/K5-L3B-DAY08-ABC-FromWorkflowsToAgents), nhánh main.
- Commit code và bằng chứng: c68da30d448de039686252a9c519cd39c316d524. Commit tài liệu sau đó không đổi phần chạy.
- Môi trường: Windows, Python 3.12.13; MockLLM offline, không dùng API key; as_of giữ 2026-11-15 theo cấu hình đề.

## 2. Mục tiêu và cách tiếp cận

Nhóm xử lý cùng claim hoàn tiền vận chuyển bằng workflow, single agent và orchestrator; giữ swarm tham chiếu để đối chứng. Nguồn quyết định là order có cấu trúc và policy hiện hành. Ticket là dữ liệu không tin cậy. Tiền do compute_compensation tính; host là bên duy nhất kiểm approval, áp trần theo đơn và ghi ledger.

Ân triển khai workflow/nghiệp vụ/durability; Quang triển khai Single Agent và test giới hạn; Phong triển khai orchestrator/handoff, audit trace và ablation. Tích hợp ba phần rồi so trên cùng 14 claim × 5 seed 11/22/33/44/55. Mỗi trial dùng SQLite mới; các test retry/crash chủ động dùng lại DB để kiểm hành vi nhiều lượt.

## 3. Giải pháp

**Luật chung.** Thiếu order_id trả needs_info; không có đơn hoặc sai principal trả escalated; quote bằng 0 trả denied mà không gọi ghi. Quote dương luôn truyền đầy đủ tới durability.issue_refund: host từ chối trả denied có audit policy_denied, chấp nhận trả resolved, xung đột khoản đã ghi trả failed. Nếu ship và damage cùng đủ thì lấy max một lần. Ở T14 quote 150.000 VND, host trừ 30.000 đã hoàn và ghi 120.000; shape không tự trừ trước. Kết quả dùng finish() để đối chiếu ledger/audit thực.

**A — Workflow (Ân).** Luồng Python cố định: get_claim_view → kiểm mã đơn → lookup_order có principal scope → compute_compensation → host → finish. MockLLM chỉ draft sau quyết định bằng prompt status/amount. Không dùng số ngày hoặc clause do ticket tự khai. Mỗi nhánh ghi trace mới, connection luôn đóng. Truyền crash_at xuống host và phát lại InjectedCrash; checkpoint không thay được ledger.

**B — Single Agent (Quang).** run_react giới hạn 8 vòng. decide chọn tool đọc theo observation và giữ state claim/order/quote; MockLLM plan/parse/draft được tính Budget. Kết quả parse ticket hoặc final của model không có quyền đặt tiền, principal, approval hay tool argument. Sau vòng đọc, code lấy quote của tool và gọi host. Trace chỉ ghi action, cờ found/visible, amount và kết quả host, không dump raw payload/model output/thông báo exception.

**C — Orchestrator (Phong).** Orchestrator tạo handoff.v1 bằng make_handoff; investigator chỉ có tool đọc trong allowlist. Không chuyển ticket, parent secret, header hoặc context cha. Khi investigator trả order nhưng thiếu quote, code có fallback compute_compensation có giới hạn; thiếu order thì dừng an toàn. Chỉ orchestrator gọi host với quote đầy đủ; không có write registry trong investigator.

**Durability và an toàn.** Host ghi refund và audit committed trong một transaction; retry cùng claim chỉ còn một effect, đổi amount thì conflict. Không sửa host. Cả ba shape qua T11/T12; test nhóm kiểm thêm các cửa sổ crash và approval. Công cụ crash 20/20 chủ yếu kiểm host (19 trường hợp) và swarm (1 trường hợp), vì vậy không dùng riêng report này để thay bằng chứng crash của ba shape. Audit kiểm result cùng các file run_id tương ứng, freshness và index secret/PII; không đưa giá trị nhạy cảm vào report.

## 4. Kết quả

Nguồn số: [benchmark.json](../../src/reports/benchmark.json), đủ 280 trial và cùng giá mô phỏng. Steps là số event trace.

| Shape | Pass n/N | Pass rate | Pass^5 | Seed min–max | Variance | VND/success | Mean/p95 steps |
|---|---:|---:|---:|---|---:|---:|---:|
| workflow | 70/70 | 100.000% | 100.000% | 1.000–1.000 | 0.000000 | 28.87 | 4.43/5 |
| single | 70/70 | 100.000% | 100.000% | 1.000–1.000 | 0.000000 | 30.53 | 8.86/10 |
| orchestrator | 70/70 | 100.000% | 100.000% | 1.000–1.000 | 0.000000 | 57.17 | 5.21/6 |
| swarm | 69/70 | 98.571% | 92.857% | 0.929–1.000 | 0.000816 | 49.57 | 2.80/3 |

Workflow/single/orchestrator có 0 lỗi nghiệp vụ trên public; swarm sai một trial R-swarm-C-002-s55: DB trễ 2 ngày nhưng tin ticket 5 ngày, trả resolved thay vì denied. Giữ nguyên điểm yếu tham chiếu. Bộ test student vẫn xanh vì chủ động xác nhận trial swarm này phải fail khi chấm nghiệp vụ.

- Cổng host và test bổ sung: **137 passed** = 54 gốc + 35 workflow + 24 single + 19 orchestrator + 5 trace audit.
- Ma trận student: **248 passed**, gồm 62 test/shape, có T11 retry và T12 crash/resume.
- [Crash report](../../src/reports/crash.json): **20/20**.
- [Audit trace](../../src/reports/trace_audit.json): **280/280** trace hợp lệ và mới, **0 leak trial** trên bốn shape theo index public. Các ca approval sai/thiếu và audit tương ứng đều qua test; không quan sát approval bypass ở dữ liệu đã kiểm.
- [Research ablation](../../src/reports/ablation_investigator.json): tăng 1 → 2 investigator giữ 70/70 pass, pass^5 1 và variance 0, nhưng cost/success **57,17 → 75,74 VND (+32,5%)**, mean tool calls **5,57 → 7,43**, model calls vẫn 2.

Nhóm chọn **workflow** vì đạt cùng độ đúng với single/orchestrator, ít tự chủ và rẻ nhất. [ADR](decision_adr.md) giải thích điều kiện đổi quyết định; [research card](research_card.md) ghi phép thử một biến. Chưa có dữ liệu/nhãn hidden của giảng viên, không suy điểm hidden hoặc bonus từ 70/70 public.

### Lệnh tái lập

Chạy trong src/ bằng Python của venv. Trên Windows có thể thay python bằng .venv/Scripts/python.exe.

    python -m pytest -q
    python -m pytest -q -m student -o addopts=
    python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json
    python -m lab8.crash_test --count 20 --out reports/crash.json
    python -m lab8.trace_audit --shapes workflow,single,orchestrator,swarm --seeds 11,22,33,44,55 --out reports/trace_audit.json
    python -m lab8.check_submission

Lệnh ablation đầy đủ nằm trong research card. Các JSON được công cụ sinh, không sửa tay. Trace/SQLite là dữ liệu chạy local được bỏ qua bởi Git; chạy lại các lệnh sẽ tái tạo.

## 5. Khó khăn và cách xử lý

Single Agent ban đầu được chọn bằng -k single, nhưng cụm này cũng khớp tên test_public_single_run_cases của các shape khác. Khi hai shape còn stub, kết quả lẫn 120 fail của chúng với 62 test single đã qua. Dùng bộ lọc single and not workflow and not orchestrator and not swarm để xác nhận phần riêng; sau tích hợp chạy toàn bộ 248 student. Không sửa test gốc để làm xanh.

Trace phải đủ để audit mà không chứa raw data. Nhóm giữ field cần thiết, bỏ ticket/context cha và thử dữ liệu canary trong test. Script mutation của Ân và chi tiết kiểm thử nằm trong báo cáo cá nhân, không thay đổi file đo chính thức. Khác biệt số event giữa shape được ghi như giới hạn, không diễn giải steps thành latency.

## 6. Tổng hợp đóng góp

[TEAM.md](../../TEAM.md) giữ ba commit kỹ thuật: Ân 8e2e45b (workflow + 35 test), Quang c4805f3 (single + 24 trường hợp), Phong 64a747d (orchestrator/handoff/audit + 19/5 test). Mỗi người có báo cáo cá nhân và khai báo công cụ/tự kiểm theo Rules. Không nhận công thay cho commit của thành viên khác.

## 7. Hạn chế và hướng cải thiện

Public chỉ có 14 claim, không chứng minh đã đạt cổng hidden 40 × 5. Nhóm chưa có student_drop nên không tạo hidden_selfcheck hoặc tự bịa nhãn/điểm. MockLLM không kiểm năng lực model thực; chi phí là VND mô phỏng, latency unavailable. Bảng chính khác cả số call và prompt, nên không cô lập riêng topology; ablation chỉ tách thêm investigator đọc policy, chưa thử song song hay thông tin độc lập hữu ích. Không khai nhận bonus. Chỉ đổi kiến trúc khi task/benchmark mới chứng minh cần thêm tự chủ hoặc kiểm chứng.

## 8. Tài liệu tham khảo

[Luật nghiệp vụ](../../docs/business_rules.md), [contracts](../../docs/contracts.md), [metrics](../../docs/metrics.md), [rubric](../../RUBRIC.md), [rules gốc](../../RULES.md). Nguồn nghiên cứu R1: [Kim et al., arXiv:2512.08296v3](https://arxiv.org/abs/2512.08296v3), và [Google Research, 28/1/2026](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/). Phạm vi nguồn và liên hệ với phép thử được nêu trong research card.
