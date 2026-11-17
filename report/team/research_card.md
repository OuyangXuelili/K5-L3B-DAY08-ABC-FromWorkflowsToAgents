# Research card

File này nằm cố định ở `report/team/research_card.md`; `python -m lab8.check_submission` đọc cột "Nhóm điền" bằng máy.

Phần này nối tiếp phần mini research ở buổi lý thuyết (slide 62–63). Chọn **một** câu R1–R4, điền 6 dòng, rồi biến nó thành **một ablation** chạy được trong lab: giữ nguyên model (MockLLM), tool, seed, corpus và giá, chỉ thay đổi một biến.

| Câu | Nguồn (thư viện tham khảo, slide 68–69) | Câu hỏi gợi ý cho lab |
| --- | --- | --- |
| R1 · Scaling | Google Research 2026, *Towards a Science of Scaling Agent Systems* | Task claims chủ yếu tuần tự hay song song được? Dự đoán thêm investigator sẽ làm pass rate và cost/success thay đổi thế nào. |
| R2 · Failure taxonomy | MAST, NeurIPS 2025, *Why Do Multi-Agent LLM Systems Fail?* | Các trial fail của swarm (seed 55) thuộc nhóm nào: lỗi giao việc, lệch mục tiêu, hay lỗi kiểm chứng? |
| R3 · Architecture choices | AgentArch, AACL 2026 | Nếu đổi topology mà giữ nguyên mọi thứ khác, biến nào phải cố định để kết luận còn đúng? |
| R4 · Evaluation confounders | EMNLP 2026, *Rethinking Evaluation of MAS Efficiency* | Làm sao chứng minh chênh lệch giữa A/B/C/D là do topology, không phải do prompt, tool hay seed? |

## Phiếu của nhóm

Điền vào cột cuối. Không xóa hai cột đầu.

| Field | Hướng dẫn | Nhóm điền |
| --- | --- | --- |
| CÂU | R1 / R2 / R3 / R4 | R1 · Scaling |
| SOURCE | Paper/blog, năm, URL, loại bằng chứng (benchmark, production report, blog kinh nghiệm) | Kim et al., *Towards a Science of Scaling Agent Systems*, [arXiv:2512.08296v3](https://arxiv.org/abs/2512.08296v3), đăng đầu 2025, sửa 8/4/2026 (paper/benchmark); [Google Research, 28/1/2026](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/) (bài giải thích của tác giả). Đã đối chiếu nguồn gốc, không chỉ trích slide. |
| CLAIM | Một kết luận gốc của nguồn. Không viết rộng hơn phạm vi paper | Trong thí nghiệm PlanCraft được Google Research mô tả, các biến thể multi-agent giảm 39–70% so với single agent. Hiệu quả phối hợp phụ thuộc cấu trúc task; đây không phải quy luật cho mọi MAS. |
| BOUNDARY | Model / dataset / setup giới hạn kết luận đó | Blog mô tả 180 cấu hình/4 benchmark; bản paper v3 cập nhật 260/6 benchmark, 5 kiến trúc, 3 họ LLM. LogiX là phép thử riêng: MockLLM, 14 claim × 5 seed, giá mô phỏng, latency unavailable. Luồng order → quote → host có phụ thuộc tuần tự. |
| HYPOTHESIS | Nếu đổi shape A→B (hoặc bật/tắt một thành phần), metric X sẽ đổi theo hướng nào, khoảng bao nhiêu | Thêm đúng 1 investigator đọc policy (1 → 2) **không** đổi pass rate, pass^5 hay variance, vì tiền do `compute_compensation` và host cap quyết. Cost/success tăng khoảng 1–2 tool call × 10 VND/trial (dự đoán +10 đến +25 VND, tức +20% đến +45%). |
| METHOD | Biến giữ cố định: model, tools, seed, corpus, giá, evaluator. Biến duy nhất được thay đổi: … | Cố định: MockLLM, bộ tool đọc, 14 claim public, seed 11/22/33/44/55, `config/pricing.yaml`, evaluator `lab8.metrics.trial_pass`, host `durability.issue_refund`, cùng code path orchestrator. Biến duy nhất: **số investigator** (1 vs 2); bản 2 investigator thêm đúng một handoff `lookup_policy` read-only, không đổi quyền ghi. |
| FAILURE / METRIC | Một lỗi dự kiến và cách đo nó: pass rate, pass^5, variance, cost/success, leak, bypass | Dự kiến không có claim fail; nếu thêm investigator làm đổi status/ledger thì đó là lỗi handoff hoặc principal scope. Đo pass rate, pass^5, seed variance, cost/success, mean/p95 steps, mean tool calls; safety đo bằng `lab8.trace_audit` (leak_trials, trace run id, file mới). |

## Kết quả ablation

- Lệnh đã chạy (copy nguyên văn):

```text
python -m lab8.benchmark --shapes orchestrator,orchestrator_two_investigators --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --extra-shape orchestrator_two_investigators=lab8.shapes.orchestrator:run_claim_two_investigators --out reports/ablation_investigator.json
```

- Run id / file report: `src/reports/ablation_investigator.json`. Run id đại diện: `R-orchestrator-C-001-s11` và `R-orchestrator_two_investigators-C-001-s11` (mỗi trial có run id riêng theo shape × claim × seed).

| Shape (cùng claim × seed) | Pass | Pass^5 | Seed variance | Cost/success (VND mô phỏng) | Mean steps | P95 steps | Mean tool calls | Mean model calls |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| orchestrator (1 investigator) | 1.000 | 1.000 | 0.000000 | 57.17 | 5.21 | 6 | 5.57 | 2.0 |
| orchestrator_two_investigators (2 investigator) | 1.000 | 1.000 | 0.000000 | 75.74 | 7.07 | 8 | 7.43 | 2.0 |
| Chênh lệch | 0 | 0 | 0 | **+18.57 (+32.5%)** | +1.86 | +2 | +1.86 | 0 |

- Kết quả so với hypothesis: **Phù hợp trên public.** Lần chạy lại sau tích hợp giữ 70/70 pass cho mỗi biến thể, pass^5 1, variance 0 và model calls 2. Cost/success tăng 18,5714 VND (+32,4854%), nằm trong khoảng dự đoán. Investigator thứ hai thêm một lượt đọc claim và policy, nhưng dữ liệu này không được dùng để đổi quyết định; mọi trial vẫn được chấm ledger bằng evaluator gốc. Đây là bằng chứng về overhead của một investigator dư thừa trong triển khai này, không phải tái lập mức giảm 39–70% của nguồn.
- Giới hạn: MockLLM không phải model sản xuất; latency unavailable. Ablation đổi **số investigator**, không kiểm song song, thông tin bổ sung hữu ích hoặc model thông minh hơn. Chưa có nhãn hidden nên không suy ra kết quả hidden. Kết luận có thể đổi nếu investigator thứ hai cung cấp thông tin độc lập mà quyết định thực sự cần, hoặc chi phí gọi nó giảm đáng kể.
- Trạng thái tái lập: đã chạy lại đúng lệnh ở trên trên code tại `c9b7b74`; `c68da30` lưu benchmark/audit chung sau tích hợp, source orchestrator không thay đổi.

Muốn đi xa hơn một ablation để lấy điểm cộng thì xem [docs/bonus_topics.md](../../docs/bonus_topics.md).
