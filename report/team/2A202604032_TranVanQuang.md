# Báo cáo cá nhân — 2A202604032_TranVanQuang

## 1. Thông tin thành viên

- Họ tên: Trần Văn Quang
- MSSV: 2A202604032
- Email: không công khai.
- GitHub dùng để commit phần này: OuyangXuelili.

## 2. Phần việc phụ trách

Phần 2 — Single Agent/ReAct trong [single.py](../../src/lab8/shapes/single.py), test bổ sung trong [test_team_single.py](../../src/tests/test_team_single.py). Phần này có thể chạy riêng với host, MockLLM và dữ liệu public có sẵn, không phụ thuộc hai shape workflow/orchestrator.

Phạm vi chính là triển khai và kiểm thử Single Agent. Sau khi đồng bộ workflow và orchestrator, nhóm đã chạy benchmark bốn shape, audit và hoàn thiện ADR chung; bằng chứng chốt tại `c68da30`.

## 3. Đóng góp cụ thể

- Thay stub `not_implemented` bằng vòng `run_react` giới hạn 8 bước. `decide` chọn các tool đọc `get_claim_view`, `lookup_order`, `compute_compensation` theo observation đã có. Dữ liệu được giữ trong state vì runtime chỉ chuyển observation cuối sang bước tiếp theo.
- Dùng MockLLM ở các bước plan, parse ticket và draft. Mọi lời gọi đi qua `complete` để cập nhật Budget; tool đọc được runtime tính phí. MockLLM của starter trả văn bản mô phỏng, không phải model production tự sinh native tool call.
- Không dùng số tiền, principal, tool hoặc approval từ văn bản model. Quote lấy từ tool code đọc order có cấu trúc và policy hiện hành. Quyết định cuối của model không được thay số tiền quote.
- Sau vòng đọc, gọi `durability.issue_refund` với quote đầy đủ. Host xử lý approval, trần theo đơn, phần đã hoàn, idempotency và audit. Không tự trừ khoản hoàn cũ ở shape.
- Xử lý thiếu order bằng `needs_info`, order không nhìn thấy bằng `escalated`, quote không đủ điều kiện bằng `denied`, thành công bằng `resolved`, step cap/lỗi/conflict bằng `failed`.
- Chuyển nguyên `crash_at` tới host và để `InjectedCrash` truyền lên; connection luôn đóng bằng `finally`. Ghi trace yêu cầu trước khi gọi host để crash không bị trình bày thành refund thành công.
- Trả kết quả qua `finish()` để lấy effect và audit count từ ledger thực tế. Không tự dựng các field tiền đã ghi.
- Trace chỉ ghi bước, tên tool, cờ found/visible, số tiền quote, kết quả host dạng boolean và trạng thái. Không ghi ticket, carrier, raw tool payload, model output, thông báo exception, header hay parent context. Lỗi chỉ ghi tên loại exception.

Test bổ sung có 24 trường hợp: model trả tool ghi/số tiền giả; chạm 8 bước; lỗi model; trace mới ở nhánh sớm; dữ liệu tainted ở ticket/carrier/quote reason/host reason; approval mượn từ claim khác; approval cap; quote đầy đủ ở T13/T14; ba cửa sổ crash; quote thay đổi khi retry; budget thực; final của model không được quyết định số tiền.

Commit code và test phần 2: `c4805f3ae9612d574cf3dc7a0acbb738a50c5bfd` — `Single agent`.

## 4. Bằng chứng

Chạy ngày 17/11/2026 trên Windows, Python 3.12.13, MockLLM offline. Mọi lệnh dưới chạy trong `src/` bằng Python của venv.

| Kiểm tra | Kết quả thực tế |
| --- | --- |
| Ma trận student chỉ cho Single Agent | 62 passed |
| Cổng host trước khi tích hợp hai phần còn lại | 78 passed, 248 deselected: 54 test gốc + 24 trường hợp mới |
| Cổng host sau khi đồng bộ workflow/orchestrator | 137 passed, 248 deselected |
| Toàn bộ ma trận student sau tích hợp | 248 passed, 137 deselected |
| Benchmark Single Agent sau tích hợp | 70/70 public trial, pass^5 14/14, variance 0, cost/success 30,5314 VND mô phỏng |
| Audit Single Agent sau tích hợp | 70/70 trace hợp lệ và mới, 0 leak trial theo index public |

Lệnh ma trận riêng đã chạy:

```powershell
.venv/Scripts/python.exe -B -X utf8 -m pytest -q tests/test_team_single.py tests/test_public_matrix.py -m student -o addopts= -k 'single and not workflow and not orchestrator and not swarm' --tb=short
```

Kết quả: `62 passed, 210 deselected`. Marker student loại test nhóm trong lệnh này; chúng được chạy ở cổng host bên dưới.

```powershell
.venv/Scripts/python.exe -B -X utf8 -m pytest -q --tb=short
```

Kết quả trước tích hợp: `78 passed, 248 deselected`; sau khi đồng bộ hai phần còn lại từ `origin/main`: `137 passed, 248 deselected`.

Lệnh ma trận student toàn bộ sau tích hợp:

```powershell
.venv/Scripts/python.exe -B -X utf8 -m pytest -q -m student -o addopts= --tb=short
```

Kết quả: `248 passed, 137 deselected`. Có thể chạy riêng các trường hợp bổ sung bằng:

```powershell
.venv/Scripts/python.exe -B -X utf8 -m pytest -q tests/test_team_single.py
```

Lưu ý chọn test: `-k single` đơn lẻ còn khớp tên `test_public_single_run_cases` của các shape khác. Lần chạy ban đầu vì vậy gồm 120 fail của hai stub workflow/orchestrator, 122 pass và 60 deselected. Bộ lọc đầy đủ ở trên xác nhận riêng 62 test Single Agent đều pass; không sửa test gốc để loại lỗi.

Run id dùng định dạng gốc `R-single-<claim_id>-s<seed>`, ví dụ `R-single-C-001-s11`. Test bổ sung ghi trace trong thư mục tạm của pytest; trace chạy public matrix ở `src/traces/`, được Git bỏ qua. Không commit DB hoặc trace thô.

## 5. Công cụ AI đã dùng

ChatGPT/Codex hỗ trợ đọc contract, triển khai Single Agent, viết test và đối chiếu kết quả chạy. Các số test ở mục 4 lấy từ lệnh đã thực thi, không phải số dự kiến. Người nộp cần đọc lại mã, xác nhận báo cáo và tự giải thích được luồng ReAct, ranh giới đọc/ghi, quote đầy đủ và chính sách trace trước khi nộp.

Tham chiếu: [business rules](../../docs/business_rules.md), [output/trace contract](../../docs/contracts.md), [metrics](../../docs/metrics.md), [rules của lab](../../RULES.md).

## 6. Tự đánh giá

Đánh giá kỹ thuật hiện tại: Single Agent qua ma trận public riêng và các trường hợp bổ sung nêu trên. Các file host, policy, durability, metrics, config và public data được giữ nguyên. Không sử dụng API key hoặc gọi model thật.

Giới hạn: chưa được chấm bằng hidden grader; public không chứng minh đã đạt ngưỡng hidden. MockLLM là mô phỏng theo seed, chi phí là VND mô phỏng và latency unavailable. Sau tích hợp, Single Agent đạt cùng độ đúng public với workflow nhưng có cost/success cao hơn (30,53 so với 28,87 VND); nhóm chọn workflow trong ADR.
