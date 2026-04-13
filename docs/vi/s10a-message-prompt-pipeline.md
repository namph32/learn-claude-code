# s10a: Message & Prompt Pipeline

> **Deep Dive** -- Nên đọc cùng với s10. Tài liệu này cho thấy tại sao System Prompt chỉ là một phần trong đầu vào đầy đủ của mô hình.

### Khi Nào Đọc Tài Liệu Này

Khi bạn đang làm việc về lắp ráp prompt và muốn xem quy trình đầu vào hoàn chỉnh.

---

> Tài liệu cầu nối này mở rộng `s10`.
>
> Nó tồn tại để làm rõ một ý tưởng quan trọng:
>
> **System Prompt quan trọng, nhưng nó không phải là toàn bộ đầu vào của mô hình.**

## Tại Sao Tài Liệu Này Tồn Tại

`s10` đã nâng cấp System Prompt từ một chuỗi khổng lồ thành một quy trình lắp ráp có thể duy trì.

Điều đó quan trọng.

Nhưng một hệ thống hoàn chỉnh hơn đi thêm một bước nữa và coi toàn bộ đầu vào mô hình như một quy trình được tạo từ nhiều nguồn:

- các khối System Prompt
- các tin nhắn được chuẩn hóa
- các đính kèm Memory
- các lần chèn nhắc nhở
- Context Runtime động

Vì vậy cấu trúc thực sự là:

**một prompt pipeline, không chỉ là một prompt builder.**

## Thuật Ngữ Trước

### Prompt block

Một phần có cấu trúc bên trong System Prompt, chẳng hạn như:

- core identity
- hướng dẫn Tool
- phần Memory
- phần CLAUDE.md

### Normalized message

Một tin nhắn đã được chuyển đổi sang dạng ổn định phù hợp với API mô hình.

Điều này cần thiết vì hệ thống thô có thể chứa:

- tin nhắn người dùng
- phản hồi của trợ lý
- kết quả Tool
- các lần chèn nhắc nhở
- nội dung giống như đính kèm

Chuẩn hóa đảm bảo tất cả chúng phù hợp với cùng một hợp đồng cấu trúc trước khi chúng đến API.

### System reminder

Một hướng dẫn tạm thời nhỏ được chèn vào cho lượt hiện tại hoặc chế độ hiện tại.

Không giống như một prompt block tồn tại lâu dài, một nhắc nhở thường ngắn hạn và theo tình huống -- ví dụ: cho mô hình biết nó hiện đang ở "chế độ plan" hoặc một Tool nào đó tạm thời không có sẵn.

## Mô Hình Tư Duy Nhỏ Nhất Hữu Ích

Hãy nghĩ về đầu vào đầy đủ như một quy trình:

```text
multiple sources
  |
  +-- system prompt blocks
  +-- messages
  +-- attachments
  +-- reminders
  |
  v
normalize
  |
  v
final API payload
```

Điểm dạy học quan trọng là:

**tách biệt các nguồn trước, sau đó chuẩn hóa chúng thành một đầu vào ổn định duy nhất.**

## Tại Sao System Prompt Không Phải Là Tất Cả

System Prompt là đúng chỗ cho:

- identity
- các quy tắc ổn định
- các ràng buộc lâu dài
- mô tả khả năng Tool

Nhưng nó thường là sai chỗ cho:

- `tool_result` mới nhất
- các lần chèn Hook một lượt
- các nhắc nhở tạm thời
- các đính kèm Memory động

Những thứ đó thuộc về luồng tin nhắn hoặc các bề mặt đầu vào liền kề.

## Các Cấu Trúc Cốt Lõi

### `SystemPromptBlock`

```python
block = {
    "text": "...",
    "cache_scope": None,
}
```

### `PromptParts`

```python
parts = {
    "core": "...",
    "tools": "...",
    "skills": "...",
    "memory": "...",
    "claude_md": "...",
    "dynamic": "...",
}
```

### `NormalizedMessage`

```python
message = {
    "role": "user" | "assistant",
    "content": [...],
}
```

Hãy coi `content` là một danh sách các khối, không chỉ là một chuỗi.

### `ReminderMessage`

```python
reminder = {
    "role": "system",
    "content": "Current mode: plan",
}
```

Ngay cả khi việc triển khai dạy học của bạn không thực sự sử dụng `role="system"` ở đây, bạn vẫn nên giữ sự phân tách về mặt tư duy:

- prompt block tồn tại lâu dài
- nhắc nhở ngắn hạn

## Đường Triển Khai Tối Thiểu

### 1. Giữ `SystemPromptBuilder`

Đừng bỏ bước prompt-builder.

### 2. Tạo tin nhắn như một quy trình riêng biệt

```python
def build_messages(raw_messages, attachments, reminders):
    messages = normalize_messages(raw_messages)
    messages = attach_memory(messages, attachments)
    messages = append_reminders(messages, reminders)
    return messages
```

### 3. Lắp ráp payload cuối cùng chỉ ở cuối

```python
payload = {
    "system": build_system_prompt(),
    "messages": build_messages(...),
    "tools": build_tools(...),
}
```

Đây là sự nâng cấp tư duy quan trọng:

**System Prompt, tin nhắn, và Tool là các bề mặt đầu vào song song, không phải là sự thay thế cho nhau.**

## Điểm Mấu Chốt

**Đầu vào mô hình là một quy trình các nguồn được chuẩn hóa muộn, không phải một khối prompt huyền bí. System Prompt, tin nhắn, và Tool là các bề mặt song song chỉ hội tụ tại thời điểm gửi.**
