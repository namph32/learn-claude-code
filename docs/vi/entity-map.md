# Bản Đồ Thực Thể

> **Tài liệu tham khảo** -- Sử dụng khi các khái niệm bắt đầu bị nhòe lẫn nhau. Tài liệu này cho bạn biết mỗi thứ thuộc tầng nào.

Khi bạn tiến vào nửa sau của repo, bạn sẽ nhận thấy rằng nguồn gây nhầm lẫn chính thường không phải là code. Đó là thực tế rằng nhiều thực thể trông giống nhau trong khi lại tồn tại trên các tầng khác nhau. Bản đồ này giúp bạn phân biệt chúng.

## Tài Liệu Này Khác Gì So Với Các Tài Liệu Khác

- bản đồ này trả lời: **thứ này thuộc tầng nào?**
- [`glossary.md`](./glossary.md) trả lời: **từ này có nghĩa là gì?**
- [`data-structures.md`](./data-structures.md) trả lời: **hình dạng trạng thái trông như thế nào?**

## Hình Ảnh Tổng Quan Theo Tầng

```text
conversation layer
  - message
  - prompt block
  - reminder

action layer
  - tool call
  - tool result
  - hook event

work layer
  - work-graph task
  - runtime task
  - protocol request

execution layer
  - subagent
  - teammate
  - worktree lane

platform layer
  - MCP server
  - memory record
  - capability router
```

## Các Cặp Hay Bị Nhầm Lẫn Nhất

### `Message` vs `PromptBlock`

| Thực thể | Nó là gì | Nó không phải là |
|---|---|---|
| `Message` | nội dung hội thoại trong lịch sử | không phải quy tắc hệ thống ổn định |
| `PromptBlock` | đoạn hướng dẫn prompt ổn định | không phải sự kiện mới nhất của một lượt |

### `Todo / Plan` vs `Task`

| Thực thể | Nó là gì | Nó không phải là |
|---|---|---|
| `todo / plan` | hướng dẫn phiên tạm thời | không phải đồ thị công việc bền vững |
| `task` | nút công việc bền vững | không phải suy nghĩ cục bộ của một lượt |

### `Work-Graph Task` vs `RuntimeTaskState`

| Thực thể | Nó là gì | Nó không phải là |
|---|---|---|
| work-graph task | nút mục tiêu và phụ thuộc bền vững | không phải bộ thực thi trực tiếp |
| runtime task | slot thực thi đang chạy hiện tại | không phải nút phụ thuộc bền vững |

### `Subagent` vs `Teammate`

| Thực thể | Nó là gì | Nó không phải là |
|---|---|---|
| subagent | worker được ủy quyền dùng một lần | không phải thành viên nhóm tồn tại lâu dài |
| teammate | cộng tác viên bền vững có danh tính và hộp thư | không phải công cụ tóm tắt có thể loại bỏ |

### `ProtocolRequest` vs tin nhắn thông thường

| Thực thể | Nó là gì | Nó không phải là |
|---|---|---|
| tin nhắn thông thường | giao tiếp tự do | không phải luồng phê duyệt có thể theo dõi |
| protocol request | yêu cầu có cấu trúc với `request_id` | không phải văn bản trò chuyện thông thường |

### `Task` vs `Worktree`

| Thực thể | Nó là gì | Nó không phải là |
|---|---|---|
| task | điều cần được thực hiện | không phải thư mục |
| worktree | nơi thực thi độc lập xảy ra | không phải mục tiêu bản thân |

### `Memory` vs `CLAUDE.md`

| Thực thể | Nó là gì | Nó không phải là |
|---|---|---|
| memory | các sự kiện xuyên phiên bền vững | không phải tệp quy tắc dự án |
| `CLAUDE.md` | bề mặt quy tắc/hướng dẫn cục bộ ổn định | không phải lưu trữ sự kiện dài hạn theo người dùng |

### `MCPServer` vs `MCPTool`

| Thực thể | Nó là gì | Nó không phải là |
|---|---|---|
| MCP server | nhà cung cấp khả năng bên ngoài | không phải một Tool cụ thể |
| MCP tool | một khả năng được hiển thị | không phải toàn bộ bề mặt kết nối |

## Bảng "Cái Gì / Ở Đâu" Nhanh

| Thực thể | Công việc chính | Vị trí thông thường |
|---|---|---|
| `Message` | Context hội thoại hiển thị | `messages[]` |
| `PromptParts` | các đoạn lắp ráp đầu vào | prompt builder |
| `PermissionRule` | quy tắc quyết định thực thi | cài đặt / trạng thái phiên |
| `HookEvent` | điểm mở rộng vòng đời | hệ thống Hook |
| `MemoryEntry` | sự kiện bền vững | kho Memory |
| `TaskRecord` | nút mục tiêu công việc | bảng tác vụ |
| `RuntimeTaskState` | slot thực thi trực tiếp | runtime manager |
| `TeamMember` | danh tính worker bền vững | cấu hình nhóm |
| `MessageEnvelope` | tin nhắn có cấu trúc giữa các Teammate | hộp thư |
| `RequestRecord` | trạng thái luồng protocol | request tracker |
| `WorktreeRecord` | lane thực thi độc lập | worktree index |
| `MCPServerConfig` | cấu hình nhà cung cấp khả năng bên ngoài | plugin / cài đặt |

## Kết Luận Chính

**Hệ thống càng có khả năng cao, ranh giới thực thể rõ ràng càng trở nên quan trọng.**
