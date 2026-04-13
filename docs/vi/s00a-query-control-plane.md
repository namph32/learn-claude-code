# s00a: Query Control Plane

> **Deep Dive** -- Nên đọc sau khi hoàn thành Stage 1 (s01-s06). Tài liệu này giải thích tại sao vòng lặp đơn giản cần một tầng phối hợp khi hệ thống phát triển.

### Khi Nào Nên Đọc Tài Liệu Này

Sau khi bạn đã xây dựng vòng lặp cơ bản và các Tool, và trước khi bạn bắt đầu các chương củng cố của Stage 2.

---

> Tài liệu cầu nối này trả lời một câu hỏi nền tảng:
>
> **Tại sao `messages[] + while True` không đủ cho một Agent hoàn thành tốt?**

## Tại Sao Tài Liệu Này Tồn Tại

`s01` đúng khi dạy vòng lặp hoạt động nhỏ nhất:

```text
user input
  ->
model response
  ->
if tool_use then execute
  ->
append result
  ->
continue
```

Đó là điểm khởi đầu đúng đắn.

Nhưng khi hệ thống phát triển, harness cần một tầng riêng biệt để quản lý **chính quá trình Query**. Một "control plane" (phần của hệ thống phối hợp hành vi thay vì thực hiện công việc trực tiếp) nằm phía trên đường dữ liệu và quyết định khi nào, tại sao và bằng cách nào vòng lặp nên tiếp tục chạy:

- lượt hiện tại
- lý do tiếp tục
- trạng thái phục hồi
- trạng thái Compaction
- thay đổi ngân sách
- tiếp tục do Hook điều khiển

Tầng đó là **Query control plane**.

## Thuật Ngữ Trước Tiên

### Query là gì?

Ở đây, Query không phải là tra cứu cơ sở dữ liệu.

Nó có nghĩa là:

> toàn bộ quá trình đa lượt mà hệ thống chạy để hoàn thành một yêu cầu người dùng

### Control plane là gì?

Control plane không tự thực hiện hành động nghiệp vụ.

Nó phối hợp:

- khi nào việc thực thi tiếp tục
- tại sao nó tiếp tục
- trạng thái nào được cập nhật trước lượt tiếp theo

Nếu bạn đã làm việc với mạng hoặc hạ tầng, thuật ngữ này quen thuộc -- control plane quyết định traffic đi đâu, trong khi data plane mang các gói tin thực tế. Ý tưởng tương tự áp dụng ở đây: control plane quyết định liệu vòng lặp có nên tiếp tục chạy hay không và tại sao, trong khi tầng thực thi thực hiện các cuộc gọi mô hình và công việc Tool thực tế.

### Transition là gì?

Một Transition giải thích:

> tại sao lượt trước không kết thúc và tại sao lượt tiếp theo tồn tại

Các lý do phổ biến:

- write-back kết quả Tool
- phục hồi đầu ra bị cắt ngắn
- thử lại sau Compaction
- thử lại sau lỗi vận chuyển

## Mô Hình Tư Duy Hữu Ích Nhỏ Nhất

Hãy nghĩ về đường dẫn Query trong ba tầng:

```text
1. Tầng đầu vào
   - messages
   - system prompt
   - user/system context

2. Tầng điều khiển
   - query state
   - turn count
   - transition reason
   - cờ recovery / compaction / budget

3. Tầng thực thi
   - model call
   - tool execution
   - write-back
```

Control plane không thay thế vòng lặp.

Nó làm cho vòng lặp có khả năng xử lý nhiều hơn một nhánh happy-path.

## Tại Sao `messages[]` Một Mình Không Còn Đủ

Ở quy mô demo, nhiều người học đặt mọi thứ vào `messages[]`.

Điều đó sẽ gặp vấn đề khi hệ thống cần biết:

- liệu Compaction phản ứng đã chạy chưa
- đã thực hiện bao nhiêu lần thử tiếp tục
- liệu lượt này là thử lại hay write-back bình thường
- liệu ngân sách đầu ra tạm thời có đang hoạt động không

Đây không phải là nội dung cuộc hội thoại.

Chúng là **trạng thái kiểm soát quá trình**.

## Các Cấu Trúc Cốt Lõi

### `QueryParams`

Đầu vào bên ngoài được truyền vào Query engine:

```python
params = {
    "messages": [...],
    "system_prompt": "...",
    "tool_use_context": {...},
    "max_output_tokens_override": None,
    "max_turns": None,
}
```

### `QueryState`

Trạng thái có thể thay đổi biến đổi qua các lượt:

```python
state = {
    "messages": [...],
    "tool_use_context": {...},
    "turn_count": 1,
    "continuation_count": 0,
    "has_attempted_compact": False,
    "max_output_tokens_override": None,
    "transition": None,
}
```

### `TransitionReason`

Một lý do rõ ràng để tiếp tục:

```python
TRANSITIONS = (
    "tool_result_continuation",
    "max_tokens_recovery",
    "compact_retry",
    "transport_retry",
)
```

Đây không phải là hình thức. Nó làm cho nhật ký, kiểm thử, gỡ lỗi và giảng dạy rõ ràng hơn nhiều.

## Mẫu Triển Khai Tối Thiểu

### 1. Tách tham số đầu vào khỏi trạng thái trực tiếp

```python
def query(params):
    state = {
        "messages": params["messages"],
        "tool_use_context": params["tool_use_context"],
        "turn_count": 1,
        "transition": None,
    }
```

### 2. Để mọi điểm tiếp tục cập nhật trạng thái một cách rõ ràng

```python
state["transition"] = "tool_result_continuation"
state["turn_count"] += 1
```

### 3. Làm cho lượt tiếp theo bước vào với một lý do

Vòng lặp lặp tiếp theo nên biết liệu nó tồn tại vì:

- write-back bình thường
- thử lại
- Compaction
- tiếp tục sau đầu ra bị cắt ngắn

## Điều Này Thay Đổi Gì Đối Với Bạn

Khi bạn thấy rõ Query control plane, các chương sau không còn cảm giác như các tính năng ngẫu nhiên.

- Compaction `s06` trở thành một bản vá trạng thái, không phải một bước nhảy kỳ diệu
- Phục hồi `s11` trở thành tiếp tục có cấu trúc, không chỉ là `try/except`
- Tính tự chủ `s17` trở thành một đường tiếp tục được kiểm soát khác, không phải một vòng lặp bí ẩn riêng biệt

## Điểm Mấu Chốt

**Một Query không chỉ là các tin nhắn chảy qua một vòng lặp. Đó là một quá trình được kiểm soát với trạng thái tiếp tục rõ ràng.**
