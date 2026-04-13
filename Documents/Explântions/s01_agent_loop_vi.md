# Giải thích `agents/s01_agent_loop.py`

`agents/s01_agent_loop.py` là phiên bản tối giản của một coding agent. Ý tưởng chính của file này là:

1. Nhận yêu cầu từ người dùng.
2. Gửi lịch sử hội thoại vào model.
3. Nếu model yêu cầu dùng tool, chương trình sẽ chạy tool thật.
4. Kết quả tool được đưa ngược lại vào lịch sử hội thoại.
5. Tiếp tục lặp cho đến khi model không yêu cầu tool nữa.

## 1. Mục tiêu của file

File này dạy cơ chế cốt lõi nhất của agent loop:

```text
user message
  -> model reply
  -> if tool_use: execute tools
  -> write tool_result back to messages
  -> continue
```

Điểm quan trọng là file giữ cho logic đủ nhỏ để dễ học, nhưng vẫn tách state rõ ràng để các chương sau có thể mở rộng từ cùng cấu trúc đó.

## 2. Phần khởi tạo

Đoạn đầu file làm ba việc:

- Nạp biến môi trường từ `.env` bằng `load_dotenv(override=True)`.
- Tạo client Anthropic bằng:

```python
client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
```

- Lấy model đang dùng từ:

```python
MODEL = os.environ["MODEL_ID"]
```

Ngoài ra còn có `SYSTEM`, là system prompt nói cho model biết:

- nó đang hoạt động trong thư mục hiện tại
- nó có thể dùng `bash` để kiểm tra và sửa workspace
- nó nên hành động trước rồi báo cáo rõ ràng

## 3. Tool được khai báo

Biến `TOOLS` chỉ khai báo đúng một tool:

- tên tool: `bash`
- mô tả: chạy lệnh shell trong workspace hiện tại
- input schema: nhận một object có trường `command`

Điều này có nghĩa là model không tự sửa file trực tiếp. Muốn thao tác với môi trường, nó phải tạo `tool_use` để xin chạy lệnh.

## 4. State tối thiểu của vòng lặp

`LoopState` là `dataclass` dùng để giữ state của agent loop:

- `messages`: toàn bộ lịch sử hội thoại
- `turn_count`: số vòng đã chạy
- `transition_reason`: lý do vòng lặp tiếp tục

Trong chương này, đây là state tối thiểu cần thiết. Những chương sau sẽ mở rộng thêm dựa trên chính cấu trúc này.

## 5. Hàm `run_bash`

Hàm `run_bash(command: str) -> str` là lớp thực thi tool thật.

Nó làm các việc sau:

- chặn một số chuỗi bị xem là nguy hiểm như `rm -rf /`, `sudo`, `shutdown`, `reboot`, `> /dev/`
- chạy lệnh bằng `subprocess.run(...)`
- đặt thư mục chạy là `os.getcwd()`
- gom cả `stdout` và `stderr`
- timeout sau 120 giây
- cắt output còn tối đa 50000 ký tự

Ý nghĩa của hàm này:

- model chỉ “ra ý định”
- harness mới là nơi thật sự chạy lệnh

Đây là ranh giới cơ bản giữa reasoning và execution.

## 6. Hàm `extract_text`

Response của Anthropic không nhất thiết chỉ có text. Nó có thể chứa nhiều block như:

- `text`
- `tool_use`

Vì vậy `extract_text(content)` sẽ:

- kiểm tra xem `content` có phải list không
- duyệt từng block
- lấy các block có thuộc tính `text`
- nối chúng lại thành một chuỗi

Hàm này được dùng ở cuối chương trình để in câu trả lời cuối cùng của assistant cho người dùng.

## 7. Hàm `execute_tool_calls`

Đây là nơi thực thi các lệnh mà model vừa yêu cầu.

Luồng chạy:

1. Duyệt qua từng block trong `response_content`
2. Chỉ giữ các block có `block.type == "tool_use"`
3. Lấy `command = block.input["command"]`
4. In lệnh ra terminal
5. Gọi `run_bash(command)`
6. Đóng gói output thành một object `tool_result`

Dạng kết quả được append có cấu trúc:

```python
{
    "type": "tool_result",
    "tool_use_id": block.id,
    "content": output,
}
```

`tool_use_id` là điểm rất quan trọng. Nó giúp model biết kết quả này thuộc về lời gọi tool nào.

## 8. Hàm `run_one_turn`

Đây là trung tâm thật sự của agent loop.

### Bước 1: gọi model

```python
response = client.messages.create(
    model=MODEL,
    system=SYSTEM,
    messages=state.messages,
    tools=TOOLS,
    max_tokens=8000,
)
```

Model nhận:

- system prompt
- toàn bộ message history
- danh sách tool được phép dùng

### Bước 2: lưu phản hồi của assistant

Ngay sau đó, code append phản hồi vào `state.messages`:

```python
state.messages.append({"role": "assistant", "content": response.content})
```

### Bước 3: kiểm tra model có muốn dùng tool không

Nếu:

```python
response.stop_reason != "tool_use"
```

thì hàm trả `False`, nghĩa là vòng lặp dừng. Lúc này model đã trả lời xong bằng text thường.

### Bước 4: nếu có tool_use thì chạy tool

Code gọi:

```python
results = execute_tool_calls(response.content)
```

Nếu không có kết quả tool hợp lệ thì dừng.

### Bước 5: đưa `tool_result` trở lại history

Nếu có kết quả, code append:

```python
state.messages.append({"role": "user", "content": results})
```

Rồi tăng `turn_count`, đặt `transition_reason = "tool_result"` và trả `True` để vòng lặp chạy tiếp.

Đây là ý tưởng cốt lõi nhất của cả file:

- model yêu cầu tool
- chương trình chạy tool thật
- kết quả được đưa lại cho model như một message mới
- model tiếp tục suy luận dựa trên kết quả đó

## 9. Hàm `agent_loop`

Hàm này rất nhỏ:

```python
while run_one_turn(state):
    pass
```

Nó chỉ làm đúng một việc: cứ còn lý do để tiếp tục thì chạy thêm một turn nữa.

Thiết kế nhỏ như vậy giúp người đọc nhìn rõ phần “loop” nằm ở đâu, thay vì bị lẫn vào logic phụ.

## 10. Phần chạy từ terminal

Khối:

```python
if __name__ == "__main__":
```

biến file này thành một CLI đơn giản.

Luồng chạy:

1. Tạo `history = []`
2. Lặp vô hạn để nhận input từ người dùng
3. Nếu người dùng nhập `q`, `exit`, hoặc chuỗi rỗng thì thoát
4. Append câu hỏi user vào `history`
5. Tạo `state = LoopState(messages=history)`
6. Gọi `agent_loop(state)`
7. Lấy text cuối cùng từ message mới nhất và in ra terminal

## 11. Điểm đáng chú ý trong thiết kế

### `history` và `state.messages` dùng chung một list

Dòng:

```python
state = LoopState(messages=history)
```

không tạo bản sao. `state.messages` và `history` trỏ vào cùng một list.

Vì vậy khi `run_one_turn()` append assistant response hoặc `tool_result`, các phần đó cũng xuất hiện luôn trong `history`.

Điều này là hợp lý, vì lịch sử hội thoại cần được giữ lại giữa các lượt chat.

### `tool_result` là trung tâm của loop

Điểm quan trọng nhất của file không phải là gọi model, mà là cơ chế:

- tool được gọi ngoài model
- kết quả quay lại thành message
- model tiếp tục từ trạng thái mới

Đó là lý do repo này nhấn mạnh rằng `tool_result` là mắt xích trung tâm của agent loop.

### Chặn lệnh nguy hiểm chỉ ở mức rất đơn giản

Danh sách chặn trong `run_bash` chỉ là lớp bảo vệ tối thiểu để dạy học.

Trong hệ thống thật, phần permission và execution safety sẽ phải chặt chẽ hơn rất nhiều. Repo này xử lý những phần đó ở các chương sau.

## 12. Tóm tắt ngắn

`s01_agent_loop.py` minh họa mẫu agent nhỏ nhất nhưng đã hữu ích:

1. user gửi yêu cầu
2. model trả lời hoặc phát sinh `tool_use`
3. harness chạy tool thật
4. harness trả `tool_result` lại vào history
5. model tiếp tục cho đến khi hoàn tất

Nếu hiểu rõ file này, bạn đã nắm được xương sống của một coding agent cơ bản.
