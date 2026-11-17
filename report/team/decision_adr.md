# Decision record

- Shape chọn: workflow
- Commit hash: c68da30d448de039686252a9c519cd39c316d524
- Run id dẫn chứng: R-workflow-C-001-s11; toàn bộ 70 run id trong src/reports/benchmark.json
- Pass rate / pass^5 / seed min-max / variance: public 70/70 = 1.000 / 14/14 = 1.000 / 1.000–1.000 / 0.000000
- Cost per success (mô phỏng, VND): 28.8714285714; tổng 2021.00 VND mô phỏng cho 70 trial
- Crash safety: 20/20 ở src/reports/crash.json; T11/T12 của workflow và ba cửa sổ crash trong test_workflow_rules.py đều pass
- Secret leak / approval bypass: 0/70 leak trial của workflow theo src/reports/trace_audit.json; 0 bypass quan sát trên public và các test approval. Chưa có kết quả hidden của giảng viên.

## Vì sao shape này

Workflow là kiến trúc ít tự chủ nhất trong bốn phương án, đồng thời đạt 70/70 public trial, pass^5 14/14 và variance bằng 0. Single và orchestrator đạt cùng độ đúng nhưng cost/success lần lượt 30,53 và 57,17 VND, cao hơn workflow 28,87 VND. Swarm đạt 69/70 và cost/success 49,57 VND; lỗi duy nhất là C-002/seed 55 tin số ngày trong ticket. Chọn workflow cho luồng claim có luật rõ, tiền và quyền ghi do code kiểm soát. Đây là lựa chọn để đem chấm hidden, chưa phải tuyên bố đã qua ngưỡng hidden.

## Có đáng đổi lấy multi-agent không

Trong lần đo này, thêm agent chưa đem lại độ đúng tốt hơn. Ablation trên cùng orchestrator chỉ thêm một investigator đọc policy làm cost/success tăng từ 57,17 lên 75,74 VND (+32,5%), trong khi cả hai vẫn pass 70/70, pass^5 1 và variance 0. Nhóm sẽ xem xét đổi kiến trúc nếu xuất hiện dữ liệu phi cấu trúc cần kiểm chứng độc lập, công việc đọc thực sự song song được hoặc benchmark mới chứng minh lợi ích đủ bù chi phí/phức tạp. Không suy rộng rằng mọi multi-agent đều kém.

## Trace và bằng chứng

Workflow ghi claim metadata, order view đã lọc, quote, host outcome và draft; crash có event riêng trước khi truyền lỗi lên. Không ghi ticket, claim/order row thô, parent context, secret, header, SĐT, địa chỉ, ghi chú hoặc tên hàng. Audit lại cả bốn shape cho 280/280 trace hợp lệ và mới, 0 leak trial theo index public. Trace và DB không commit; JSON tổng hợp được sinh bằng công cụ và có lệnh tái tạo trong báo cáo nhóm.

## Giới hạn

Commit trên chứa code và bằng chứng chạy; commit tài liệu sau đó chỉ hoàn thiện hồ sơ. Không sửa host, policy, durability, metrics, public data/config hoặc rules gốc. Bộ public có 14 claim và 5 seed; test retry/crash là kịch bản nhiều lượt riêng, không thay bằng benchmark một lượt. Chưa có student_drop hay nhãn hidden nên không báo pass rate hidden, điểm chính thức hoặc bonus. MockLLM không phải model sản xuất, cost là mô phỏng và latency unavailable. Prompt, số lời gọi tool và độ chi tiết trace khác nhau giữa shape nên bảng chính so các triển khai cụ thể, không cô lập riêng topology. Steps đếm event trace, không phải số vòng ReAct hoặc latency.
