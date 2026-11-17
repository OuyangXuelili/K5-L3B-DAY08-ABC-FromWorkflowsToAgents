# Team

## Thông tin nhóm

- Tên nhóm: ABC
- Lớp: L3B, theo xác nhận của nhóm; giữ tên repo hiện tại.
- Repo nhóm: `K5-L3B-DAY08-ABC-FromWorkflowsToAgents`.
- URL repo: https://github.com/OuyangXuelili/K5-L3B-DAY08-ABC-FromWorkflowsToAgents
- Nhánh nộp: `main`.
- Commit code và bằng chứng chốt: `c68da30d448de039686252a9c519cd39c316d524`; commit tài liệu tiếp theo chỉ hoàn thiện hồ sơ.

## Thành viên

| STT | Họ tên | MSSV | Email | GitHub | Vai trò |
| --: | ------ | ---- | ----- | ------ | ------- |
|   1 | Trương Văn Phong | 2A202604091 | phongtruongvan2005@gmail.com | vanphong205 | Orchestrator + an toàn dữ liệu |
|   2 | Nguyễn Hoàng Ân | 2A202604025 | Không công khai tại đây | hoangan06 | Workflow + nghiệp vụ + durability |
|   3 | Trần Văn Quang | 2A202604032 | Không công khai | OuyangXuelili (tài khoản commit) | Single Agent/ReAct + kiểm thử phần 2 |

Ba người tích hợp và đối chiếu kết quả chung: 137 test host, 248 test student; benchmark 4 shape × 14 claim × 5 seed. Chi tiết trong [báo cáo nhóm](report/team/group_report.md), [ADR](report/team/decision_adr.md) và [research card](report/team/research_card.md).

## Bảng đóng góp

`python -m lab8.check_submission` đọc bảng này bằng máy: giữ nguyên tiêu đề `## Bảng đóng góp` và các cột. Mỗi người một dòng: commit có nội dung và ít nhất một test đã thêm hoặc sửa.

| Người | Vai | Commit | Test đã thêm hoặc sửa |
| --- | --- | --- | --- |
| Trương Văn Phong (2A202604091) | Orchestrator + an toàn dữ liệu | `64a747d` — feat(orchestrator): sanitized handoff, read-only investigators, trace audit | `src/tests/test_orchestrator_handoff.py` (19 test), `src/tests/test_trace_audit.py` (5 test) |
| Nguyễn Hoàng Ân (2A202604025) | Workflow + nghiệp vụ + durability | `8e2e45b` — feat(workflow): hoàn thành shape workflow, test riêng và báo cáo phần nghiệp vụ | `src/tests/test_workflow_rules.py` (35 test) |
| Trần Văn Quang (2A202604032) | Single Agent/ReAct | `c4805f3` — Single agent | `src/tests/test_team_single.py` (24 trường hợp) |

## Cá nhân

Mỗi thành viên đồng thời tạo một báo cáo riêng trong `report/` theo dạng:

```text
report/team/MSSV_HoTen.md
```

- [Nguyễn Hoàng Ân](report/team/2A202604025_NguyenHoangAn.md)
- [Trần Văn Quang](report/team/2A202604032_TranVanQuang.md)
- [Trương Văn Phong](report/team/2A202604091_TruongVanPhong.md)

Giữ file mẫu để đối chiếu cấu trúc. Email không được tự điền khi chưa được chủ tài khoản cung cấp. Mỗi thành viên tự nộp link repo nhóm trên LMS; việc push repo không thay việc nộp cá nhân.
