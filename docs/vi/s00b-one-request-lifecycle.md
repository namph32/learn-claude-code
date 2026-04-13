# s00b: Vòng Đời Của Một Yêu Cầu

> **Deep Dive** -- Nên đọc sau Stage 2 (s07-s11) khi bạn muốn xem cách tất cả các phần kết nối từ đầu đến cuối.

### Khi Nào Nên Đọc Tài Liệu Này

Khi bạn đã học một số hệ thống con và muốn xem luồng dọc đầy đủ của một yêu cầu duy nhất.

---

> Tài liệu cầu nối này kết nối toàn bộ hệ thống thành một chuỗi thực thi liên tục.
>
> Nó trả lời:
>
> **Điều gì thực sự xảy ra sau khi một tin nhắn người dùng vào hệ thống?**

## Tại Sao Tài Liệu Này Tồn Tại

Khi bạn đọc từng chương, bạn có thể hiểu từng cơ chế riêng lẻ:

- `s01` vòng lặp
- `s02` Tool
- `s07` quyền
- `s09` Memory
- `s12-s19` tác vụ, nhóm, Worktree, MCP

Nhưng việc triển khai trở nên khó khăn khi bạn không thể trả lời:

- điều gì đến trước?
- Memory và tập hợp System Prompt xảy ra khi nào?
- quyền nằm ở đâu so với Tool?
- khi nào tác vụ, Runtime slot, thành viên nhóm, Worktree và MCP xuất hiện?

Tài liệu này cung cấp cho bạn luồng dọc.

## Bức Tranh Đầy Đủ Quan Trọng Nhất

```text
user request
  |
  v
initialize query state
  |
  v
assemble system prompt / messages / reminders
  |
  v
call model
  |
  +-- normal answer --------------------------> finish request
  |
  +-- tool_use
        |
        v
    tool router
        |
        +-- permission gate
        +-- hooks
        +-- native tool / MCP / agent / task / team
        |
        v
    execution result
        |
        +-- may update task / runtime / memory / worktree state
        |
        v
    write tool_result back to messages
        |
        v
    patch query state
        |
        v
    continue next turn
```

## Phân Đoạn 1: Một Yêu Cầu Người Dùng Trở Thành Query State

Hệ thống không xử lý một yêu cầu người dùng như một lời gọi API duy nhất.

Đầu tiên nó tạo một query state cho một quá trình có thể kéo dài nhiều lượt:

```python
query_state = {
    "messages": [{"role": "user", "content": user_text}],
    "turn_count": 1,
    "transition": None,
    "tool_use_context": {...},
}
```

Sự thay đổi tư duy quan trọng:

**một yêu cầu là một quá trình Runtime đa lượt, không phải một phản hồi mô hình đơn.**

Tài liệu liên quan:

- [`s01-the-agent-loop.md`](./s01-the-agent-loop.md)
- [`s00a-query-control-plane.md`](./s00a-query-control-plane.md)

## Phân Đoạn 2: Đầu Vào Thực Sự Cho Mô Hình Được Tập Hợp

Harness thường không gửi `messages` thô trực tiếp.

Nó tập hợp:

- các khối System Prompt
- các tin nhắn đã chuẩn hóa
- các tệp đính kèm Memory
- các nhắc nhở
- các định nghĩa Tool

Vì vậy, payload thực tế gần hơn với:

```text
system prompt
+ normalized messages
+ tools
+ optional reminders and attachments
```

Các chương liên quan:

- `s09`
- `s10`
- [`s10a-message-prompt-pipeline.md`](./s10a-message-prompt-pipeline.md)

## Phân Đoạn 3: Mô Hình Tạo Ra Câu Trả Lời Hoặc Ý Định Hành Động

Có hai lớp đầu ra quan trọng.

### Câu trả lời bình thường

Yêu cầu có thể kết thúc ở đây.

### Ý định hành động

Thường có nghĩa là một lời gọi Tool, ví dụ:

- `read_file(...)`
- `bash(...)`
- `task_create(...)`
- `mcp__server__tool(...)`

Hệ thống không còn chỉ nhận văn bản.

Nó đang nhận một hướng dẫn cần ảnh hưởng đến thế giới thực.

## Phân Đoạn 4: Tool Control Plane Tiếp Quản

Khi `tool_use` xuất hiện, hệ thống vào Tool control plane (tầng quyết định cách một lời gọi Tool được định tuyến, kiểm tra và thực thi).

Nó trả lời:

1. đây là Tool nào?
2. nó nên định tuyến ở đâu?
3. nó có cần qua cổng quyền không?
4. Hook quan sát hay sửa đổi hành động không?
5. Context Runtime được chia sẻ nào nó có thể truy cập?

Sơ đồ tối thiểu:

```text
tool_use
  |
  v
tool router
  |
  +-- native handler
  +-- MCP client
  +-- agent / team / task runtime
```

Tài liệu liên quan:

- [`s02-tool-use.md`](./s02-tool-use.md)
- [`s02a-tool-control-plane.md`](./s02a-tool-control-plane.md)

## Phân Đoạn 5: Thực Thi Có Thể Cập Nhật Nhiều Hơn Chỉ Messages

Kết quả Tool không chỉ trả về văn bản.

Thực thi cũng có thể cập nhật:

- trạng thái bảng tác vụ
- trạng thái tác vụ Runtime
- bản ghi Memory
- bản ghi yêu cầu
- bản ghi Worktree

Đó là lý do tại sao các chương giữa và cuối không phải là tính năng phụ tùy chọn. Chúng trở thành một phần của vòng đời yêu cầu.

## Phân Đoạn 6: Kết Quả Tái Gia Nhập Vòng Lặp Chính

Bước quan trọng luôn giống nhau:

```text
real execution result
  ->
tool_result or structured write-back
  ->
messages / query state updated
  ->
next turn
```

Nếu kết quả không bao giờ vào lại vòng lặp, mô hình không thể lý luận trên thực tế.

## Một Cách Nén Hữu Ích

Khi bạn bị lạc, hãy nén toàn bộ vòng đời thành ba tầng:

### Query loop

Sở hữu quá trình yêu cầu đa lượt.

### Tool control plane

Sở hữu định tuyến, quyền, Hook và Context thực thi.

### Trạng thái nền tảng

Sở hữu các bản ghi lâu bền như tác vụ, Runtime slot, thành viên nhóm, Worktree và cấu hình năng lực bên ngoài.

## Điểm Mấu Chốt

**Một yêu cầu người dùng đi vào như query state, di chuyển qua đầu vào được tập hợp, trở thành ý định hành động, vượt qua Tool control plane, chạm đến trạng thái nền tảng, và sau đó quay lại vòng lặp như Context mới có thể nhìn thấy.**
