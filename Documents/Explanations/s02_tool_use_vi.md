# Giải thích `agents/s02_tool_use.py`

`s02_tool_use.py` là bước tiếp theo của `s01_agent_loop.py`. File này không thay đổi cấu trúc agent loop mà **mở rộng khả năng của model bằng cách thêm nhiều tool hơn**. Ngoài `bash`, giờ model có thể đọc, ghi, và chỉnh sửa file trực tiếp.

Điểm mấu chốt được nhấn ngay từ đầu:

> "The loop didn't change at all. I just added tools."

## 1. Mục tiêu của file

File này dạy hai điều:

- **Tool dispatch**: làm sao để đăng ký nhiều tool, mỗi tool có handler riêng.
- **Message normalization**: trước mỗi lần gọi API, cần dọn dẹp messages để đúng format mà API yêu cầu.

So với `s01`, model giờ không còn phải dùng `bash` cho mọi thao tác file — nó có thể dùng tool chuyên biệt.

## 2. Các tool mới

### 2.1. `bash`

Giữ nguyên từ `s01`:

```python
"bash": lambda **kw: run_bash(kw["command"])
```

### 2.2. `read_file`

Đọc nội dung file với optional limit dòng:

```python
"read_file": lambda **kw: run_read(kw["path"], kw.get("limit"))
```

Hàm `run_read`:

```python
def run_read(path: str, limit: int = None) -> str:
    try:
        text = safe_path(path).read_text()
        lines = text.splitlines()
        if limit and limit < len(lines):
            lines = lines[:limit] + [f"... ({len(lines) - limit} lines)"]
        return "\n".join(lines)[:50000]
    except Exception as e:
        return f"Error: {e}"
```

Lưu ý:

- Dùng `safe_path()` để ngăn truy cập file ngoài workspace.
- Có `limit` để chỉ đọc N dòng đầu — tránh load file quá lớn.
- Luôn cắt output tối đa 50000 ký tự.

### 2.3. `write_file`

Ghi nội dung mới vào file, tự tạo thư mục cha nếu chưa tồn tại:

```python
"write_file": lambda **kw: run_write(kw["path"], kw["content"])
```

Hàm `run_write`:

```python
def run_write(path: str, content: str) -> str:
    try:
        fp = safe_path(path)
        fp.parent.mkdir(parents=True, exist_ok=True)
        fp.write_text(content)
        return f"Wrote {len(content)} bytes to {path}"
    except Exception as e:
        return f"Error: {e}"
```

Điểm đáng chú ý: `fp.parent.mkdir(parents=True, exist_ok=True)` — tự tạo thư mục cha giúp model ghi file vào thư mục mới mà không cần chạy `mkdir` trước.

### 2.4. `edit_file`

Thay thế chính xác một đoạn text trong file:

```python
"edit_file": lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"])
```

Hàm `run_edit`:

```python
def run_edit(path: str, old_text: str, new_text: str) -> str:
    try:
        fp = safe_path(path)
        content = fp.read_text()
        if old_text not in content:
            return f"Error: Text not found in {path}"
        fp.write_text(content.replace(old_text, new_text, 1))
        return f"Edited {path}"
    except Exception as e:
        return f"Error: {e}"
```

Điểm quan trọng: `replace(old_text, new_text, 1)` — chỉ thay thế **đúng một vị trí** đầu tiên, tránh thay nhầm nhiều chỗ.

## 3. Cấu trúc dispatch — `TOOL_HANDLERS`

Thay vì dùng `if/elif` cho từng tool, `s02` dùng dictionary dispatch:

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

Ưu điểm của dispatch map:

- Thêm tool mới chỉ cần thêm một dòng, không sửa logic hiện có.
- Nếu tool không tồn tại trong map, trả về `f"Unknown tool: {block.name}"` — không crash.

## 4. Khai báo tool cho API — `TOOLS`

Biến `TOOLS` là danh sách JSON schema gửi lên API Anthropic để mô tả tool cho model:

```python
TOOLS = [
    {"name": "bash", "description": "Run a shell command.",
     "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}},
    {"name": "read_file", "description": "Read file contents.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "limit": {"type": "integer"}}, "required": ["path"]}},
    {"name": "write_file", "description": "Write content to file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "content": {"type": "string"}}, "required": ["path", "content"]}},
    {"name": "edit_file", "description": "Replace exact text in file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "old_text": {"type": "string"}, "new_text": {"type": "string"}}, "required": ["path", "old_text", "new_text"]}},
]
```

Mỗi tool cần:

- `name`: tên duy nhất, trùng với key trong `TOOL_HANDLERS`
- `description`: mô tả bằng tiếng Anh tự nhiên — model dùng mô tả này để quyết định dùng tool nào
- `input_schema`: định nghĩa các tham số và kiểu dữ liệu

## 5. Phân loại concurrency safety

```python
CONCURRENCY_SAFE = {"read_file"}
CONCURRENCY_UNSAFE = {"write_file", "edit_file"}
```

Ý tưởng:

- `read_file` chỉ đọc, không thay đổi gì → **an toàn khi chạy song song**
- `write_file` và `edit_file` thay đổi filesystem → **phải chạy tuần tự**

Đây là bước chuẩn bị cho khả năng chạy nhiều tool song song ở chương sau. Hiện tại tất cả chạy tuần tự, nhưng thiết kế đã tách biệt sẵn.

## 6. Hàm `normalize_messages`

Đây là phần hoàn toàn mới so với `s01`. Hàm này dọn dẹp messages trước khi gửi lên API, làm 3 việc:

### 6.1. Strip metadata fields

```python
clean["content"] = [
    {k: v for k, v in block.items()
     if not k.startswith("_")}
    for block in msg["content"]
    if isinstance(block, dict)
]
```

Loại bỏ các trường internal bắt đầu bằng `_` — API không hiểu những trường đó.

### 6.2. Chèn placeholder cho orphaned tool_use

```python
# Collect existing tool_result IDs
existing_results = set()
for msg in cleaned:
    if isinstance(msg.get("content"), list):
        for block in msg["content"]:
            if isinstance(block, dict) and block.get("type") == "tool_result":
                existing_results.add(block.get("tool_use_id"))

# Find orphaned tool_use blocks and insert placeholder results
for msg in cleaned:
    if msg["role"] != "assistant" or not isinstance(msg.get("content"), list):
        continue
    for block in msg["content"]:
        if not isinstance(block, dict):
            continue
        if block.get("type") == "tool_use" and block.get("id") not in existing_results:
            cleaned.append({"role": "user", "content": [
                {"type": "tool_result", "tool_use_id": block["id"],
                 "content": "(cancelled)"}
            ]})
```

Nếu model gọi tool mà harness không thực thi (ví dụ: tool bị chặn hoặc lỗi), hàm tự chèn một `tool_result` placeholder để API không bị lệch giữa số `tool_use` và `tool_result`.

### 6.3. Gộp consecutive same-role messages

```python
if msg["role"] == merged[-1]["role"]:
    prev = merged[-1]
    prev_c = prev["content"] if isinstance(prev["content"], list) \
        else [{"type": "text", "text": str(prev["content"])}]
    curr_c = msg["content"] if isinstance(msg["content"], list) \
        else [{"type": "text", "text": str(msg["content"])}]
    prev["content"] = prev_c + curr_c
else:
    merged.append(msg)
```

API Anthropic yêu cầu messages phải **đan xen role** (user → assistant → user → assistant...). Nếu có nhiều user messages liên tiếp, chúng được gộp lại thành một.

## 7. Hàm `agent_loop` — so sánh với `s01`

Trong `s01`:

```python
def run_one_turn(state: LoopState) -> bool:
    response = client.messages.create(
        model=MODEL, system=SYSTEM,
        messages=state.messages,
        tools=TOOLS, max_tokens=8000,
    )
    state.messages.append({"role": "assistant", "content": response.content})
    ...
```

Trong `s02`:

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM,
            messages=normalize_messages(messages),   # ← thêm normalize
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)  # ← dùng dispatch map
                output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                print(f"> {block.name}:")
                print(output[:200])
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})
        messages.append({"role": "user", "content": results})
```

Sự khác biệt chính:

| Thay đổi | Chi tiết |
|---|---|
| Bỏ `LoopState` | Dùng `list` trực tiếp cho messages |
| Thêm `normalize_messages(messages)` | Dọn dẹp trước khi gửi API |
| Dùng `TOOL_HANDLERS` dispatch | Thay vì viết từng `if/elif` |
| In debug `> tool_name` | Cho thấy tool nào đang chạy |

## 8. Phần main — so sánh với `s01`

```python
if __name__ == "__main__":
    history = []
    while True:
        try:
            query = input("\033[36ms02 >> \033[0m")  # ← prompt màu cyan
        except (EOFError, KeyboardInterrupt):
            break
        if query.strip().lower() in ("q", "exit", ""):
            break
        history.append({"role": "user", "content": query})
        agent_loop(history)
        response_content = history[-1]["content"]
        if isinstance(response_content, list):
            for block in response_content:
                if hasattr(block, "text"):
                    print(block.text)
        print()
```

- Prompt `s02 >>` khác với `s01 >>` để biết mình đang chạy chương nào.
- Phần in kết quả vẫn duyệt list of blocks như `s01`.

## 9. So sánh tổng quan `s01` vs `s02`

| Khía cạnh | `s01` | `s02` |
|---|---|---|
| Số tool | 1 (`bash`) | 4 (`bash`, `read_file`, `write_file`, `edit_file`) |
| Dispatch | `if/elif` | Dictionary (`TOOL_HANDLERS`) |
| Message handling | Trực tiếp | `normalize_messages()` |
| Concurrency safety | Không có | Có (`CONCURRENCY_SAFE` / `CONCURRENCY_UNSAFE`) |
| File safety | `run_bash` chặn lệnh nguy hiểm | `safe_path()` ngăn truy cập ngoài workspace |
| Cấu trúc loop | `LoopState` dataclass | List trực tiếp |

## 10. Điểm đáng chú ý trong thiết kế

### Tool là phần mở rộng, không phải thay đổi loop

Thông điệp "the loop didn't change" là nguyên tắc quan trọng:

- Loop cũ: nhận message → gọi model → kiểm tra `tool_use` → chạy tool → quay lại
- Thêm tool mới không cần sửa loop, chỉ cần thêm handler vào `TOOL_HANDLERS`

### `safe_path` là lớp bảo vệ filesystem

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

Dù model muốn đọc/ghi file ở đâu, nó bị giới hạn trong `WORKDIR`. Nếu path đi ra ngoài, lập tức raise `ValueError`.

### Tất cả tool đều trả về string

Dù là đọc file, ghi file, hay bash — mọi handler đều trả về `str`. Điều này giúp `agent_loop` xử lý đồng nhất: nhận string, đưa vào `tool_result.content`.

## 11. Tóm tắt ngắn

`s02_tool_use.py` mở rộng `s01` bằng cách thêm ba tool thao tác file:

1. `read_file` — đọc nội dung với optional limit
2. `write_file` — ghi file mới, tự tạo thư mục
3. `edit_file` — thay đúng một đoạn text

Cấu trúc loop giữ nguyên từ `s01`. Ba điểm mới:

- **Dispatch map** thay thế `if/elif` — thêm tool dễ dàng hơn.
- **`normalize_messages`** dọn dẹp format messages trước khi gửi API.
- **Concurrency classification** chuẩn bị cho khả năng chạy song song ở chương sau.

Nếu hiểu `s01`, file này chỉ thêm hai phần: nhiều handler hơn và một hàm `normalize_messages`. Logic loop cốt lõi không thay đổi.