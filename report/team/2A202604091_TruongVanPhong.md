# Báo cáo cá nhân — 2A202604091_TruongVanPhong

## 1. Thông tin thành viên

- Họ tên: Trương Văn Phong
- MSSV: 2A202604091
- Email: phongtruongvan2005@gmail.com
- GitHub: vanphong205

## 2. Phần việc phụ trách

- Shape C — orchestrator + investigator chỉ đọc (`src/lab8/shapes/orchestrator.py`).
- Handoff allowlist và ranh giới dữ liệu: chỉ chuyển field trong `FACT_KEYS`, chỉ cấp tool đọc, không chuyển parent context/secret/header.
- Test cho handoff, quyền tool, principal scope, các nhánh thiếu đơn/thiếu claim/investigator thiếu dữ liệu, crash và nguy cơ rò dữ liệu.
- Công cụ rà trace ba kiến trúc (`src/lab8/trace_audit.py`) và chạy rà trên bộ public.
- Research card: thiết kế ablation chỉ đổi số investigator, ghi hypothesis và đối chiếu kết quả thực tế.
- Viết phần orchestrator, handoff, bảo mật và giới hạn trong báo cáo nhóm.

## 3. Đóng góp cụ thể

- `src/lab8/shapes/orchestrator.py`: orchestrator 5 bước; handoff sanitize; investigator read-only; chỉ orchestrator gọi `durability.issue_refund`; truyền quote đầy đủ; xử lý unknown claim, thiếu `order_id`, order not found/ngoài principal scope, quote 0, host denial, per-order cap, crash; fallback `host_recompute_quote` khi investigator thiếu quote.
- `src/lab8/shapes/orchestrator.py::run_claim_two_investigators`: biến thể ablation (thêm đúng một investigator policy), không phải shape chính thức.
- `src/lab8/trace_audit.py`: audit trace contract v1 + leak scan; kiểm tra run id, file mới, quét result và mọi file `<run_id>*`; report chỉ ghi fingerprint, không ghi giá trị nhạy cảm.
- `src/tests/test_orchestrator_handoff.py`: 19 test cho allowlist, write-tool denial, investigator không ghi ledger, principal scope, missing order/claim, ticket injection, incomplete investigation, crash, trace sạch, ablation.
- `src/tests/test_trace_audit.py`: 5 test cho index secret/PII, fingerprint-only report, stale trace và leak detection.
- `src/tests/__init__.py`: sửa lỗi import `tests.conftest` khi máy có package `tests` trong site-packages.

Kết quả chính:

- 62/62 test student của orchestrator pass.
- 78 test host pass (`python -m pytest -q`), gồm 24 test mới.
- Trace audit 3 shape × 14 claim × 5 seed = 210 trial: 210/210 trace hợp lệ và mới, 0 leak trial. Tại thời điểm rà, workflow/single còn stub nên chỉ đạt phần trace; orchestrator 70/70 pass.
- Ablation 1 → 2 investigator: pass 1.000 → 1.000, pass^5 1.000 → 1.000, variance 0 → 0; cost/success 57,17 → 75,74 VND (+32,5%); model calls giữ 2,0; tool calls 5,57 → 7,43.

Commit chính: `64a747d` (orchestrator + test + trace audit); commit báo cáo: xem `git log -- report/team/2A202604091_TruongVanPhong.md`.

## 4. Bằng chứng

- `python -m pytest -q -m student -o addopts= -k orchestrator` → `62 passed, 264 deselected`.
- `python -m pytest -q` → `78 passed, 248 deselected`.
- `python -m lab8.trace_audit --shapes workflow,single,orchestrator --seeds 11,22,33,44,55 --out reports/trace_audit.json` → 210 trial, 0 leak; file `src/reports/trace_audit.json`.
- `python -m lab8.benchmark --shapes orchestrator,orchestrator_two_investigators --seeds 11,22,33,44,55 --extra-shape orchestrator_two_investigators=lab8.shapes.orchestrator:run_claim_two_investigators --out reports/ablation_investigator.json` → file `src/reports/ablation_investigator.json`.
- Run id đại diện: `R-orchestrator-C-001-s11`, `R-orchestrator-C-005-s11`, `R-orchestrator-C-010-s11`, `R-orchestrator_two_investigators-C-001-s11`.
- Phần báo cáo nhóm: mục 3 (C — Orchestrator; Handoff/trace/bảo mật) và mục 7 (giới hạn).
- Research card: `report/team/research_card.md` (R1, ablation số investigator).

## 5. Công cụ AI đã dùng

- OpenCode (model DeepSeek V4.1 Flash Free) để đọc repo, gợi ý cấu trúc orchestrator/test và soạn thảo báo cáo.
- Tự kiểm tra: chạy lại toàn bộ pytest host và 62 test student của orchestrator; đọc trực tiếp trace JSON để đối chiếu contract v1; đối chiếu từng nhánh với `docs/contracts.md`, `docs/business_rules.md`; kiểm tra `artifact_blob` không chứa canary/PII; tự chạy benchmark và đọc số từ file JSON, không chép tay từ terminal.

## 6. Tự đánh giá

- Hoàn thành phần orchestrator, handoff, test bảo mật, trace audit và research card.
- Hạn chế tại thời điểm bàn giao phần C: workflow/single còn stub nên audit ban đầu chỉ xác nhận trace của chúng. Sau tích hợp đã chạy lại đủ bốn shape: 280/280 trace hợp lệ và mới, 0 leak trial; orchestrator đạt 70/70 public trial. Kết luận vẫn giới hạn ở MockLLM và 14 claim × 5 seed, chưa có kết quả hidden.
- Tự đánh giá lúc bàn giao phần C: 9/10; đây là nhận xét của thành viên tại mốc đó. Các kết quả tích hợp mới nằm trong báo cáo nhóm và ADR, không tự thay điểm tự nhận xét cá nhân.
