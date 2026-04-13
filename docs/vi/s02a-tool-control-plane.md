# s02a: Tool Control Plane

> **Deep Dive** -- Nên đọc sau s02 và trước s07. Tài liệu này giải thích tại sao Tool trở thành nhiều hơn một bảng tra cứu đơn giản.

### Khi Nào Nên Đọc Tài Liệu Này

Sau khi bạn hiểu về Tool dispatch cơ bản và trước khi bạn thêm các quyền.

---

> Tài liệu cầu nối này trả lời một câu hỏi quan trọng khác:
>
> **Tại sao hệ thống Tool lại nhiều hơn một bảng `tool_name -> handler`?**

## Tại Sao Tài Liệu Này Tồn Tại

`s02` dạy đúng về đăng ký và điều phối Tool trước tiên.

Đó là bước dạy học đúng đắn vì bạn nên hiểu trước cách mô hình biến ý định thành hành động.

Nhưng về sau, tầng Tool bắt đầu gánh vác nhiều trách nhiệm hơn:

- kiểm tra quyền
- định tuyến MCP
- thông báo
- trạng thái Runtime dùng chung
- truy cập tin nhắn
- trạng thái ứng dụng
- các hạn chế theo loại khả năng

Tại thời điểm đó, tầng Tool không còn chỉ là một bảng hàm nữa.

Nó trở thành một control plane (tầng điều phối quyết định *cách* mỗi lần gọi Tool được định tuyến và thực thi, thay vì tự thực hiện công việc của Tool).

## Các Khái Niệm Cơ Bản

### Tool control plane

Phần của hệ thống quyết định **cách** một lần gọi Tool được thực thi:

- nó chạy ở đâu
- nó có được phép không
- nó có thể truy cập trạng thái nào
- nó là native hay bên ngoài

### Execution context

Môi trường Runtime hiển thị cho Tool:

- thư mục làm việc hiện tại
- chế độ quyền hiện tại
- tin nhắn hiện tại
- các MCP client có sẵn
- trạng thái ứng dụng và các kênh thông báo

### Capability source

Không phải mọi Tool đều đến từ cùng một nơi. Các nguồn phổ biến:

- Tool local native
- Tool MCP
- Tool nền tảng agent/team/task/worktree

## Mô Hình Tư Duy Hữu Ích Nhỏ Gọn Nhất

Hãy coi hệ thống Tool gồm bốn tầng:

```text
1. ToolSpec
   những gì mô hình thấy

2. Tool Router
   nơi yêu cầu được gửi đến

3. ToolUseContext
   môi trường mà Tool có thể truy cập

4. Tool Result Envelope
   cách đầu ra trả về vòng lặp chính
```

Bước nâng cấp lớn nhất là tầng 3:

**các hệ thống hoàn chỉnh cao được định nghĩa ít hơn bởi bảng dispatch và nhiều hơn bởi execution context dùng chung.**

## Các Cấu Trúc Cốt Lõi

### `ToolSpec`

```python
tool = {
    "name": "read_file",
    "description": "Read file contents.",
    "input_schema": {...},
}
```

### `ToolDispatchMap`

```python
handlers = {
    "read_file": read_file,
    "write_file": write_file,
    "bash": run_bash,
}
```

Cần thiết, nhưng chưa đủ.

### `ToolUseContext`

```python
tool_use_context = {
    "tools": handlers,
    "permission_context": {...},
    "mcp_clients": {},
    "messages": [...],
    "app_state": {...},
    "notifications": [],
    "cwd": "...",
}
```

Điểm mấu chốt:

Các Tool ngừng chỉ nhận các tham số đầu vào.
Chúng bắt đầu nhận một môi trường Runtime dùng chung.

### `ToolResultEnvelope`

```python
result = {
    "ok": True,
    "content": "...",
    "is_error": False,
    "attachments": [],
}
```

Điều này giúp dễ dàng hỗ trợ hơn:

- đầu ra văn bản thuần túy
- đầu ra có cấu trúc
- đầu ra lỗi
- kết quả dạng tệp đính kèm

## Tại Sao `ToolUseContext` Cuối Cùng Trở Nên Cần Thiết

So sánh hai hệ thống.

### Hệ thống A: chỉ có dispatch map

```python
output = handlers[tool_name](**tool_input)
```

Ổn cho một bản demo.

### Hệ thống B: dispatch map cộng với execution context

```python
output = handlers[tool_name](tool_input, tool_use_context)
```

Gần với một nền tảng thực tế hơn.

Tại sao?

Vì lúc này:

- `bash` cần quyền
- `mcp__...` cần một client
- Tool `agent` cần thiết lập môi trường thực thi
- `task_output` có thể cần ghi file cộng với write-back thông báo

## Lộ Trình Triển Khai Tối Giản

### 1. Giữ `ToolSpec` và các hàm xử lý

Đừng bỏ mô hình đơn giản đi.

### 2. Giới thiệu một đối tượng context dùng chung

```python
class ToolUseContext:
    def __init__(self):
        self.handlers = {}
        self.permission_context = {}
        self.mcp_clients = {}
        self.messages = []
        self.app_state = {}
        self.notifications = []
```

### 3. Cho phép tất cả hàm xử lý nhận context

```python
def run_tool(tool_name: str, tool_input: dict, ctx: ToolUseContext):
    handler = ctx.handlers[tool_name]
    return handler(tool_input, ctx)
```

### 4. Định tuyến theo capability source

```python
def route_tool(tool_name: str, tool_input: dict, ctx: ToolUseContext):
    if tool_name.startswith("mcp__"):
        return run_mcp_tool(tool_name, tool_input, ctx)
    return run_native_tool(tool_name, tool_input, ctx)
```

## Điểm Mấu Chốt

**Một hệ thống Tool trưởng thành không chỉ là bảng ánh xạ tên-sang-hàm. Nó là một execution plane dùng chung quyết định cách ý định hành động của mô hình trở thành công việc thực tế.**
