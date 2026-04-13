# s02b: Tool Execution Runtime

> **Deep Dive** -- Nên đọc sau s02, khi bạn muốn hiểu về thực thi Tool đồng thời.

### Khi Nào Nên Đọc Tài Liệu Này

Khi bạn bắt đầu tự hỏi nhiều lần gọi Tool trong một lượt được thực thi an toàn như thế nào.

---

> Ghi chú cầu nối này không nói về cách đăng ký Tool.
>
> Nó đề cập đến một câu hỏi sâu hơn:
>
> **Khi mô hình phát ra nhiều lần gọi Tool, những quy tắc nào quyết định sự đồng thời, cập nhật tiến trình, thứ tự kết quả và việc hợp nhất Context?**

## Tại Sao Ghi Chú Này Tồn Tại

`s02` dạy đúng về:

- Schema của Tool
- Dispatch Map
- `tool_result` được đưa trở lại vào vòng lặp

Đó là điểm khởi đầu đúng đắn.

Nhưng khi hệ thống phát triển, các câu hỏi khó chuyển xuống một tầng sâu hơn:

- Tool nào có thể chạy song song
- Tool nào nên giữ tuần tự
- liệu các Tool chạy lâu có nên phát ra tiến trình trước không
- liệu các kết quả đồng thời có nên write-back theo thứ tự hoàn thành hay thứ tự ban đầu
- liệu việc thực thi Tool có làm biến đổi Context dùng chung không
- cách các biến đổi đồng thời nên hợp nhất an toàn

Những câu hỏi đó không còn về đăng ký nữa.

Chúng thuộc về **tool execution runtime** -- tập hợp các quy tắc mà hệ thống tuân theo khi các lần gọi Tool thực sự bắt đầu thực thi, bao gồm lập lịch, theo dõi, tạo ra tiến trình và hợp nhất kết quả.

## Các Khái Niệm Cơ Bản

### "Tool execution runtime" có nghĩa là gì ở đây

Đây không phải là Runtime của ngôn ngữ lập trình.

Ở đây nó có nghĩa là:

> các quy tắc mà hệ thống sử dụng khi các lần gọi Tool thực sự bắt đầu thực thi

Các quy tắc đó bao gồm lập lịch, theo dõi, tạo ra tiến trình và hợp nhất kết quả.

### "Concurrency safe" có nghĩa là gì

Một Tool là concurrency safe khi:

> nó có thể chạy song song với công việc tương tự mà không làm hỏng trạng thái dùng chung

Các Tool chỉ đọc thường an toàn:

- `read_file`
- một số Tool tìm kiếm
- Tool MCP chỉ Query

Nhiều Tool ghi thì không:

- `write_file`
- `edit_file`
- các Tool sửa đổi trạng thái ứng dụng dùng chung

### Progress message là gì

Một progress message có nghĩa là:

> Tool chưa xong, nhưng hệ thống đã hiển thị những gì nó đang làm

Điều này giúp người dùng được thông báo trong các thao tác chạy lâu thay vì để họ nhìn chằm chằm vào màn hình trống.

### Context modifier là gì

Một số Tool làm nhiều hơn là trả về văn bản.

Chúng cũng sửa đổi Context Runtime dùng chung, ví dụ:

- cập nhật hàng đợi thông báo
- ghi lại các Tool đang hoạt động
- biến đổi trạng thái ứng dụng

Việc biến đổi trạng thái dùng chung đó được gọi là context modifier.

## Mô Hình Tư Duy Tối Giản

Đừng làm phẳng việc thực thi Tool thành:

```text
tool_use -> handler -> result
```

Mô hình tư duy tốt hơn là:

```text
tool_use blocks
  ->
partition by concurrency safety
  ->
choose concurrent or serial execution
  ->
emit progress if needed
  ->
write results back in stable order
  ->
merge queued context modifiers
```

Hai nâng cấp quan trọng nhất:

- tính đồng thời không phải là "tất cả Tool chạy cùng nhau"
- Context dùng chung không nên bị biến đổi theo thứ tự hoàn thành ngẫu nhiên

## Các Bản Ghi Cốt Lõi

### 1. `ToolExecutionBatch`

Một batch dạy học tối giản có thể trông như sau:

```python
batch = {
    "is_concurrency_safe": True,
    "blocks": [tool_use_1, tool_use_2, tool_use_3],
}
```

Điểm quan trọng rất đơn giản:

- các Tool không phải lúc nào cũng được xử lý từng cái một
- Runtime nhóm chúng vào các execution batch trước

### 2. `TrackedTool`

Nếu bạn muốn một tầng thực thi hoàn chỉnh hơn, hãy theo dõi từng Tool một cách rõ ràng:

```python
tracked_tool = {
    "id": "toolu_01",
    "name": "read_file",
    "status": "queued",   # queued / executing / completed / yielded
    "is_concurrency_safe": True,
    "pending_progress": [],
    "results": [],
    "context_modifiers": [],
}
```

Điều này cho phép Runtime có thể trả lời:

- cái gì vẫn đang chờ
- cái gì đang chạy
- cái gì đã hoàn thành
- cái gì đã tạo ra tiến trình rồi

### 3. `MessageUpdate`

Việc thực thi Tool có thể tạo ra nhiều hơn một kết quả cuối cùng.

Một bản cập nhật tối giản có thể được xử lý như:

```python
update = {
    "message": maybe_message,
    "new_context": current_context,
}
```

Trong một Runtime lớn hơn, các bản cập nhật thường tách thành hai kênh:

- các tin nhắn cần được đưa lên trên ngay lập tức
- các thay đổi Context nên giữ nội bộ cho đến thời điểm hợp nhất

### 4. Queued context modifiers

Điều này dễ bỏ qua, nhưng đây là một trong những ý tưởng quan trọng nhất.

Trong một batch đồng thời, chiến lược an toàn hơn không phải là:

> "Tool nào hoàn thành trước sẽ biến đổi Context dùng chung trước"

Chiến lược an toàn hơn là:

> xếp hàng các context modifier trước, sau đó hợp nhất chúng theo thứ tự Tool ban đầu

Ví dụ:

```python
queued_context_modifiers = {
    "toolu_01": [modify_ctx_a],
    "toolu_02": [modify_ctx_b],
}
```

## Các Bước Triển Khai Tối Giản

### Bước 1: phân loại mức độ an toàn đồng thời

```python
def is_concurrency_safe(tool_name: str, tool_input: dict) -> bool:
    return tool_name in {"read_file", "search_files"}
```

### Bước 2: phân vùng trước khi thực thi

```python
batches = partition_tool_calls(tool_uses)

for batch in batches:
    if batch["is_concurrency_safe"]:
        run_concurrently(batch["blocks"])
    else:
        run_serially(batch["blocks"])
```

### Bước 3: cho phép các batch đồng thời phát ra tiến trình

```python
for update in run_concurrently(...):
    if update.get("message"):
        yield update["message"]
```

### Bước 4: hợp nhất Context theo thứ tự ổn định

```python
queued_modifiers = {}

for update in concurrent_updates:
    if update.get("context_modifier"):
        queued_modifiers[update["tool_id"]].append(update["context_modifier"])

for tool in original_batch_order:
    for modifier in queued_modifiers.get(tool["id"], []):
        context = modifier(context)
```

Đây là một trong những nơi mà một repo dạy học vẫn có thể giữ đơn giản trong khi vẫn trung thực về hình dạng hệ thống thực tế.

## Hình Ảnh Bạn Nên Nắm Giữ

```text
tool_use blocks
  |
  v
partition by concurrency safety
  |
  +-- safe batch ----------> concurrent execution
  |                            |
  |                            +-- progress updates
  |                            +-- final results
  |                            +-- queued context modifiers
  |
  +-- exclusive batch -----> serial execution
                               |
                               +-- direct result
                               +-- direct context update
```

## Tại Sao Điều Này Quan Trọng Hơn Dispatch Map

Trong một bản demo nhỏ:

```python
handlers[tool_name](tool_input)
```

là đủ.

Nhưng trong một Agent hoàn chỉnh hơn, phần khó không còn là gọi đúng hàm xử lý nữa.

Phần khó là:

- lập lịch nhiều Tool một cách an toàn
- giữ cho tiến trình hiển thị
- làm cho thứ tự kết quả ổn định
- ngăn Context dùng chung trở nên không xác định

Đó là lý do tại sao tool execution runtime xứng đáng có phần Deep Dive riêng.

## Điểm Mấu Chốt

**Khi mô hình phát ra nhiều lần gọi Tool mỗi lượt, vấn đề khó chuyển từ dispatch sang thực thi đồng thời an toàn với thứ tự kết quả ổn định.**
