# Các Cấu Trúc Dữ Liệu Lõi

> **Tài liệu tham khảo** -- Sử dụng khi bạn mất dấu trạng thái tồn tại ở đâu. Mỗi bản ghi có một công việc rõ ràng.

Cách dễ nhất để bị lạc trong hệ thống Agent không phải là số lượng tính năng -- mà là mất dấu trạng thái thực sự tồn tại ở đâu. Tài liệu này tập hợp các bản ghi lõi xuất hiện lặp đi lặp lại trong các tài liệu mainline và bridge để bạn luôn có một nơi để tra cứu.

## Nên Đọc Cùng Nhau

- [`glossary.md`](./glossary.md) để tra nghĩa thuật ngữ
- [`entity-map.md`](./entity-map.md) để xem ranh giới các tầng
- [`s13a-runtime-task-model.md`](./s13a-runtime-task-model.md) để phân biệt tác vụ và runtime-slot
- [`s19a-mcp-capability-layers.md`](./s19a-mcp-capability-layers.md) để hiểu MCP vượt ra ngoài Tool

## Hai Nguyên Tắc Cần Ghi Nhớ

### Nguyên tắc 1: tách trạng thái nội dung khỏi trạng thái kiểm soát quy trình

- `messages`, `tool_result`, và văn bản Memory là trạng thái nội dung
- `turn_count`, `transition`, và cờ retry là trạng thái kiểm soát quy trình

### Nguyên tắc 2: tách trạng thái bền vững khỏi trạng thái chỉ dùng trong Runtime

- tác vụ, Memory và lịch trình thường là bền vững
- Runtime slot, quyết định quyền và kết nối MCP trực tiếp thường là trạng thái Runtime

## Trạng Thái Query Và Cuộc Hội Thoại

### `Message`

Lưu lịch sử cuộc hội thoại và vòng gọi Tool.

### `NormalizedMessage`

Hình dạng tin nhắn ổn định sẵn sàng cho API mô hình.

### `QueryParams`

Đầu vào bên ngoài dùng để bắt đầu một quá trình Query.

### `QueryState`

Trạng thái có thể thay đổi qua các lượt.

### `TransitionReason`

Giải thích tại sao lượt tiếp theo tồn tại.

### `CompactSummary`

Bản tóm tắt nén chuyển tiếp khi Context cũ rời khỏi cửa sổ hot.

## Trạng Thái Prompt Và Đầu Vào

### `SystemPromptBlock`

Một đoạn prompt ổn định.

### `PromptParts`

Các đoạn prompt đã tách trước khi lắp ráp cuối cùng.

### `ReminderMessage`

Injection tạm thời cho một lượt hoặc một chế độ.

## Trạng Thái Tool Và Control Plane

### `ToolSpec`

Những gì mô hình biết về một Tool.

### `ToolDispatchMap`

Bảng định tuyến từ tên đến handler.

### `ToolUseContext`

Môi trường thực thi dùng chung hiển thị với các Tool.

### `ToolResultEnvelope`

Kết quả được chuẩn hóa trả về vào vòng lặp chính.

### `PermissionRule`

Chính sách quyết định cho phép / từ chối / hỏi.

### `PermissionDecision`

Đầu ra có cấu trúc của cổng quyền.

### `HookEvent`

Sự kiện vòng đời được chuẩn hóa phát ra xung quanh vòng lặp.

## Trạng Thái Công Việc Bền Vững

### `TaskRecord`

Nút đồ thị công việc bền vững với mục tiêu, trạng thái và các cạnh phụ thuộc.

### `ScheduleRecord`

Quy tắc mô tả khi nào công việc nên được kích hoạt.

### `MemoryEntry`

Sự kiện xuyên phiên đáng được giữ lại.

## Trạng Thái Thực Thi Runtime

### `RuntimeTaskState`

Bản ghi slot thực thi trực tiếp cho công việc nền hoặc chạy lâu dài.

### `Notification`

Cầu nối kết quả nhỏ mang kết quả Runtime trở lại vòng lặp chính.

### `RecoveryState`

Trạng thái dùng để tiếp tục mạch lạc sau khi xảy ra lỗi.

## Trạng Thái Nhóm Và Nền Tảng

### `TeamMember`

Danh tính Teammate bền vững.

### `MessageEnvelope`

Tin nhắn có cấu trúc giữa các Teammate.

### `RequestRecord`

Bản ghi bền vững cho các luồng phê duyệt, tắt máy, bàn giao hoặc các luồng protocol khác.

### `WorktreeRecord`

Bản ghi cho một lane thực thi độc lập.

### `MCPServerConfig`

Cấu hình cho một nhà cung cấp khả năng bên ngoài.

### `CapabilityRoute`

Quyết định định tuyến cho khả năng native, plugin hoặc được hỗ trợ bởi MCP.

## Bảng Tham Chiếu Nhanh

| Bản ghi | Công việc chính | Thường tồn tại trong |
|---|---|---|
| `Message` | lịch sử cuộc hội thoại | `messages[]` |
| `QueryState` | kiểm soát theo từng lượt | query engine |
| `ToolUseContext` | môi trường thực thi Tool | tool control plane |
| `PermissionDecision` | kết quả cổng thực thi | tầng quyền |
| `TaskRecord` | mục tiêu công việc bền vững | bảng tác vụ |
| `RuntimeTaskState` | slot thực thi trực tiếp | runtime manager |
| `TeamMember` | Teammate bền vững | cấu hình nhóm |
| `RequestRecord` | trạng thái protocol | request tracker |
| `WorktreeRecord` | lane thực thi độc lập | worktree index |
| `MCPServerConfig` | cấu hình khả năng bên ngoài | cài đặt / cấu hình plugin |

## Kết Luận Chính

**Các hệ thống hoàn thiện cao trở nên dễ hiểu hơn nhiều khi mỗi bản ghi quan trọng có một công việc rõ ràng và một tầng rõ ràng.**
