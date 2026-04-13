# s00c: Mô Hình Chuyển Tiếp Query

> **Deep Dive** -- Nên đọc cùng với s11 (Phục Hồi Lỗi). Tài liệu này đào sâu mô hình Transition được giới thiệu trong s00a.

### Khi Nào Nên Đọc Tài Liệu Này

Khi bạn đang làm việc về phục hồi lỗi và muốn hiểu tại sao mỗi lần tiếp tục cần một lý do rõ ràng.

---

> Ghi chú cầu nối này trả lời một câu hỏi hẹp nhưng quan trọng:
>
> **Tại sao một Agent hoàn thành tốt cần biết _lý do_ một Query tiếp tục sang lượt tiếp theo, thay vì coi mọi `continue` là cùng một thứ?**

## Tại Sao Ghi Chú Này Tồn Tại

Nhánh chính đã dạy:

- `s01`: vòng lặp nhỏ nhất
- `s06`: Compaction và kiểm soát Context
- `s11`: phục hồi lỗi

Chuỗi đó là đúng.

Vấn đề là những gì bạn thường mang trong đầu sau khi đọc các chương đó riêng lẻ:

> "Vòng lặp tiếp tục vì nó tiếp tục."

Điều đó đủ cho một demo đồ chơi, nhưng nó sẽ nhanh chóng gặp vấn đề trong một hệ thống lớn hơn.

Một Query có thể tiếp tục vì những lý do rất khác nhau:

- một Tool vừa hoàn thành và mô hình cần kết quả
- đầu ra đạt giới hạn Token và mô hình nên tiếp tục
- Compaction đã thay đổi Context đang hoạt động và hệ thống nên thử lại
- tầng vận chuyển thất bại và backoff nói "thử lại"
- một stop Hook nói lượt không nên kết thúc hoàn toàn
- chính sách ngân sách vẫn cho phép hệ thống tiếp tục

Nếu tất cả những điều đó thu gọn thành một `continue` mơ hồ, ba điều sẽ nhanh chóng xấu đi:

- nhật ký ngừng đọc được
- kiểm thử ngừng chính xác
- mô hình tư duy giảng dạy trở nên mờ nhạt

## Thuật Ngữ Trước Tiên

### Transition là gì

Ở đây, một Transition có nghĩa là:

> lý do lượt trước trở thành lượt tiếp theo

Đó không phải là nội dung tin nhắn chính. Đó là nguyên nhân luồng điều khiển.

### Continuation là gì

Một continuation có nghĩa là:

> Query này vẫn còn sống và nên tiếp tục tiến lên

Nhưng continuation không phải là một thứ. Đó là một tập hợp các lý do.

### Query boundary là gì

Một query boundary là ranh giới giữa một lượt và lượt tiếp theo.

Bất cứ khi nào hệ thống vượt qua ranh giới đó, nó nên biết:

- tại sao nó đang vượt qua
- trạng thái nào đã được thay đổi trước khi vượt qua
- lượt tiếp theo nên diễn giải sự thay đổi đó như thế nào

## Mô Hình Tư Duy Tối Thiểu

Đừng hình dung một Query như một đường thẳng duy nhất.

Một mô hình tư duy tốt hơn là:

```text
một query
  = một chuỗi chuyển tiếp trạng thái
    với lý do tiếp tục rõ ràng
```

Ví dụ:

```text
user input
  ->
model emits tool_use
  ->
tool finishes
  ->
tool_result_continuation
  ->
model output is truncated
  ->
max_tokens_recovery
  ->
compaction happens
  ->
compact_retry
  ->
final completion
```

Đó là lý do tại sao bài học thực sự không phải là:

> "vòng lặp tiếp tục quay"

Bài học thực sự là:

> "hệ thống đang tiến lên qua các lý do Transition có kiểu"

## Các Bản Ghi Cốt Lõi

### 1. `transition` bên trong query state

Ngay cả một triển khai giảng dạy cũng nên mang một trường Transition rõ ràng:

```python
state = {
    "messages": [...],
    "turn_count": 3,
    "continuation_count": 1,
    "has_attempted_compact": False,
    "transition": None,
}
```

Trường này không phải là trang trí.

Nó cho bạn biết:

- tại sao lượt này tồn tại
- nhật ký nên giải thích nó như thế nào
- một kiểm thử nên khẳng định đường nào

### 2. `TransitionReason`

Một tập giảng dạy tối thiểu có thể trông như thế này:

```python
TRANSITIONS = (
    "tool_result_continuation",
    "max_tokens_recovery",
    "compact_retry",
    "transport_retry",
    "stop_hook_continuation",
    "budget_continuation",
)
```

Các lý do này không tương đương:

- `tool_result_continuation`
  là tiến trình vòng lặp bình thường
- `max_tokens_recovery`
  là tiếp tục sau đầu ra bị cắt ngắn
- `compact_retry`
  là tiếp tục sau khi tái cấu trúc Context
- `transport_retry`
  là tiếp tục sau lỗi hạ tầng
- `stop_hook_continuation`
  là tiếp tục bị buộc bởi logic điều khiển bên ngoài
- `budget_continuation`
  là tiếp tục được cho phép bởi chính sách và ngân sách còn lại

### 3. Ngân sách continuation

Các hệ thống hoàn thành tốt không chỉ tiếp tục. Chúng giới hạn continuation.

Các trường điển hình trông như:

```python
state = {
    "max_output_tokens_recovery_count": 2,
    "has_attempted_reactive_compact": True,
}
```

Nguyên tắc là:

> continuation là một tài nguyên được kiểm soát, không phải một lối thoát vô hạn

## Các Bước Triển Khai Tối Thiểu

### Bước 1: làm rõ ràng mọi điểm tiếp tục

Nhiều vòng lặp người mới bắt đầu vẫn trông như thế này:

```python
continue
```

Tiến một bước về phía trước:

```python
state["transition"] = "tool_result_continuation"
continue
```

### Bước 2: ghép mỗi continuation với bản vá trạng thái của nó

```python
if response.stop_reason == "tool_use":
    state["messages"] = append_tool_results(...)
    state["turn_count"] += 1
    state["transition"] = "tool_result_continuation"
    continue

if response.stop_reason == "max_tokens":
    state["messages"].append({
        "role": "user",
        "content": CONTINUE_MESSAGE,
    })
    state["max_output_tokens_recovery_count"] += 1
    state["transition"] = "max_tokens_recovery"
    continue
```

Phần quan trọng không phải là "thêm một dòng code."

Phần quan trọng là:

> trước mỗi continuation, hệ thống biết cả lý do và sự thay đổi trạng thái

### Bước 3: tách tiến trình bình thường khỏi phục hồi

```python
if should_retry_transport(error):
    time.sleep(backoff(...))
    state["transition"] = "transport_retry"
    continue

if should_recompact(error):
    state["messages"] = compact_messages(state["messages"])
    state["transition"] = "compact_retry"
    continue
```

Khi bạn làm điều này, "continue" ngừng là một hành động mơ hồ và trở thành một Transition điều khiển có kiểu.

## Những Gì Cần Kiểm Thử

Repo giảng dạy của bạn nên làm cho các khẳng định này trở nên đơn giản:

- kết quả Tool viết `tool_result_continuation`
- đầu ra mô hình bị cắt ngắn viết `max_tokens_recovery`
- thử lại Compaction không âm thầm tái sử dụng lý do cũ
- thử lại vận chuyển tăng trạng thái thử lại và không trông giống lượt bình thường

Nếu những đường đó không dễ kiểm thử, mô hình có thể vẫn còn quá ngầm ẩn.

## Những Gì Không Nên Dạy Quá Mức

Bạn không cần chôn mình trong các chi tiết vận chuyển cụ thể của nhà cung cấp hoặc mọi enum trường hợp góc.

Đối với một repo giảng dạy, bài học cốt lõi hẹp hơn:

> một Query là một chuỗi các Transition rõ ràng, và mỗi Transition nên mang một lý do, một bản vá trạng thái và một quy tắc ngân sách

Đó là phần bạn thực sự cần nếu muốn tái xây dựng một Agent hoàn thành tốt từ đầu.

## Điểm Mấu Chốt

**Mỗi continuation cần một lý do có kiểu. Không có nó, nhật ký mờ dần, kiểm thử yếu đi, và mô hình tư duy sụp đổ thành "vòng lặp tiếp tục quay."**
