# s00f: Thứ Tự Đọc Code

> **Deep Dive** -- Đọc tài liệu này khi bạn sắp mở các tệp Python của Agent và muốn có chiến lược đọc chúng.

Trang này không nói về việc đọc nhiều code hơn. Nó trả lời một câu hỏi hẹp hơn: khi thứ tự chương đã ổn định, thứ tự sạch nhất để đọc code của repo này mà không làm rối mô hình tư duy một lần nữa là gì?

## Kết Luận Trước

Đừng đọc code theo cách này:

- đừng bắt đầu với tệp dài nhất
- đừng nhảy thẳng vào chương "nâng cao" nhất
- đừng mở `web/` trước rồi đoán nhánh chính
- đừng coi tất cả các tệp `agents/*.py` như một pool nguồn phẳng

Quy tắc ổn định rất đơn giản:

**đọc code theo cùng thứ tự như chương trình học.**

Bên trong mỗi tệp chương, giữ cùng thứ tự đọc:

1. cấu trúc trạng thái
2. định nghĩa Tool hoặc registry
3. hàm tiến một lượt
4. entry CLI cuối cùng

## Tại Sao Trang Này Tồn Tại

Bạn có thể sẽ không bị lạc trong phần văn xuôi trước. Bạn sẽ bị lạc khi cuối cùng mở code và ngay lập tức bắt đầu quét những thứ sai.

Các lỗi điển hình:

- nhìn chằm chằm vào nửa dưới của một tệp dài trước
- đọc một đống helper `run_*` trước khi biết chúng kết nối ở đâu
- nhảy vào các chương nền tảng cuối và coi các chương đầu là "quá đơn giản"
- gộp `task`, `runtime task`, `teammate` và `worktree` lại thành một ý tưởng mơ hồ

## Sử Dụng Cùng Một Mẫu Đọc Cho Mỗi Tệp Agent

Với bất kỳ `agents/sXX_*.py` nào, đọc theo thứ tự này:

### 1. Phần đầu tệp

Trả lời hai câu hỏi trước bất cứ điều gì khác:

- chương này đang dạy gì
- nó cố ý chưa dạy gì

### 2. Cấu trúc trạng thái hoặc lớp manager

Tìm những thứ như:

- `LoopState`
- `PlanningState`
- `CompactState`
- `TaskManager`
- `BackgroundManager`
- `TeammateManager`
- `WorktreeManager`

### 3. Danh sách Tool hoặc registry

Tìm:

- `TOOLS`
- `TOOL_HANDLERS`
- `build_tool_pool()`
- các entrypoint `run_*` quan trọng

### 4. Hàm tiến lượt

Thường là một trong các hàm:

- `run_one_turn(...)`
- `agent_loop(...)`
- một `handle_*` theo chương cụ thể

### 5. Entry CLI cuối cùng

`if __name__ == "__main__"` quan trọng, nhưng không nên là thứ đầu tiên bạn nghiên cứu.

## Stage 1: `s01-s06`

Giai đoạn này là xương sống Agent đơn đang hình thành.

| Chương | Tệp | Đọc Trước | Rồi Đọc | Xác Nhận Trước Khi Chuyển Tiếp |
|---|---|---|---|---|
| `s01` | `agents/s01_agent_loop.py` | `LoopState` | `TOOLS` -> `execute_tool_calls()` -> `run_one_turn()` -> `agent_loop()` | Bạn có thể theo dõi `messages -> model -> tool_result -> next turn` |
| `s02` | `agents/s02_tool_use.py` | `safe_path()` | tool handlers -> `TOOL_HANDLERS` -> `agent_loop()` | Bạn hiểu cách Tool phát triển mà không viết lại vòng lặp |
| `s03` | `agents/s03_todo_write.py` | các kiểu trạng thái lập kế hoạch | đường todo handler -> reminder injection -> `agent_loop()` | Bạn hiểu trạng thái lập kế hoạch phiên có thể nhìn thấy |
| `s04` | `agents/s04_subagent.py` | `AgentTemplate` | `run_subagent()` -> parent `agent_loop()` | Bạn hiểu Subagent chủ yếu là cô lập Context |
| `s05` | `agents/s05_skill_loading.py` | các kiểu skill registry | registry methods -> `agent_loop()` | Bạn hiểu khám phá nhẹ, tải sâu |
| `s06` | `agents/s06_context_compact.py` | `CompactState` | persist / micro compact / history compact -> `agent_loop()` | Bạn hiểu Compaction di chuyển chi tiết thay vì xóa tính liên tục |

## Stage 2: `s07-s11`

Giai đoạn này củng cố control plane xung quanh một Agent đơn hoạt động.

| Chương | Tệp | Đọc Trước | Rồi Đọc | Xác Nhận Trước Khi Chuyển Tiếp |
|---|---|---|---|---|
| `s07` | `agents/s07_permission_system.py` | validator / manager | permission path -> `run_bash()` -> `agent_loop()` | Bạn hiểu cổng trước thực thi |
| `s08` | `agents/s08_hook_system.py` | `HookManager` | hook registration and dispatch -> `agent_loop()` | Bạn hiểu các điểm mở rộng cố định |
| `s09` | `agents/s09_memory_system.py` | memory managers | save path -> prompt build -> `agent_loop()` | Bạn hiểu Memory như một tầng thông tin dài hạn |
| `s10` | `agents/s10_system_prompt.py` | `SystemPromptBuilder` | reminder builder -> `agent_loop()` | Bạn hiểu tập hợp đầu vào như một pipeline |
| `s11` | `agents/s11_error_recovery.py` | compact / backoff helpers | recovery branches -> `agent_loop()` | Bạn hiểu tiếp tục sau thất bại |

## Stage 3: `s12-s14`

Giai đoạn này biến harness thành một work Runtime.

| Chương | Tệp | Đọc Trước | Rồi Đọc | Xác Nhận Trước Khi Chuyển Tiếp |
|---|---|---|---|---|
| `s12` | `agents/s12_task_system.py` | `TaskManager` | task create / dependency / unlock -> `agent_loop()` | Bạn hiểu mục tiêu công việc lâu bền |
| `s13` | `agents/s13_background_tasks.py` | `NotificationQueue` / `BackgroundManager` | background registration -> notification drain -> `agent_loop()` | Bạn hiểu Runtime slot |
| `s14` | `agents/s14_cron_scheduler.py` | `CronLock` / `CronScheduler` | cron match -> trigger -> `agent_loop()` | Bạn hiểu các điều kiện khởi động trong tương lai |

## Stage 4: `s15-s19`

Giai đoạn này về ranh giới nền tảng.

| Chương | Tệp | Đọc Trước | Rồi Đọc | Xác Nhận Trước Khi Chuyển Tiếp |
|---|---|---|---|---|
| `s15` | `agents/s15_agent_teams.py` | `MessageBus` / `TeammateManager` | roster / inbox / loop -> `agent_loop()` | Bạn hiểu thành viên nhóm lâu bền |
| `s16` | `agents/s16_team_protocols.py` | `RequestStore` / `TeammateManager` | request handlers -> `agent_loop()` | Bạn hiểu request-response cộng với `request_id` |
| `s17` | `agents/s17_autonomous_agents.py` | claim and identity helpers | claim path -> resume path -> `agent_loop()` | Bạn hiểu idle check -> safe claim -> resume work |
| `s18` | `agents/s18_worktree_task_isolation.py` | `TaskManager` / `WorktreeManager` / `EventBus` | worktree lifecycle -> `agent_loop()` | Bạn hiểu mục tiêu so với luồng thực thi |
| `s19` | `agents/s19_mcp_plugin.py` | capability gate / MCP client / plugin loader / router | tool pool build -> route -> normalize -> `agent_loop()` | Bạn hiểu cách năng lực bên ngoài vào cùng một control plane |

## Vòng Lặp Tài Liệu + Code Tốt Nhất

Với mỗi chương:

1. đọc văn xuôi chương
2. đọc ghi chú cầu nối cho chương đó
3. mở `agents/sXX_*.py` tương ứng
4. theo thứ tự: state -> tools -> turn driver -> CLI entry
5. chạy demo một lần
6. viết lại phiên bản nhỏ nhất từ đầu

## Điểm Mấu Chốt

**Thứ tự đọc code phải tuân thủ thứ tự giảng dạy: đọc ranh giới trước, rồi trạng thái, rồi đường tiến vòng lặp.**
