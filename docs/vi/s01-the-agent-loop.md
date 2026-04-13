# s01: The Agent Loop

`[ s01 ] > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 > s12 > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Agent Loop cốt lõi hoạt động như thế nào: gửi tin nhắn, chạy Tool, đưa kết quả trở lại
- Tại sao bước "write-back" là ý tưởng quan trọng nhất trong thiết kế Agent
- Cách xây dựng một Agent hoạt động được trong chưa đến 30 dòng Python

Hãy tưởng tượng bạn có một trợ lý xuất sắc có khả năng suy luận về code, lập kế hoạch giải pháp và viết câu trả lời chất lượng cao -- nhưng không thể chạm vào bất cứ thứ gì. Mỗi lần nó đề xuất chạy một lệnh, bạn phải tự sao chép lệnh đó, chạy nó, dán kết quả trở lại và chờ gợi ý tiếp theo. Bạn chính là vòng lặp. Chương này sẽ loại bỏ bạn ra khỏi vòng lặp đó.

## Vấn Đề

Nếu không có vòng lặp, mỗi lần gọi Tool đều cần có con người ở giữa. Mô hình nói "chạy bài kiểm thử này." Bạn chạy nó. Bạn dán kết quả vào. Mô hình nói "bây giờ hãy sửa dòng 12." Bạn sửa nó. Bạn cho mô hình biết điều gì đã xảy ra. Cách làm thủ công qua lại này có thể phù hợp với một câu hỏi đơn lẻ, nhưng sẽ hoàn toàn thất bại khi một tác vụ yêu cầu 10, 20 hoặc 50 lần gọi Tool liên tiếp.

Giải pháp rất đơn giản: để code thực hiện vòng lặp.

## Giải Pháp

Đây là toàn bộ hệ thống trong một hình ảnh:

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> |  Tool   |
| prompt |      |       |      | execute |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                    (loop until the model stops calling tools)
```

Mô hình nói chuyện, harness (đoạn code bọc quanh mô hình) thực thi các Tool, và kết quả được đưa thẳng trở lại vào cuộc hội thoại. Vòng lặp tiếp tục quay cho đến khi mô hình quyết định đã xong.

## Cách Hoạt Động

**Bước 1.** Câu hỏi của người dùng trở thành tin nhắn đầu tiên.

```python
messages.append({"role": "user", "content": query})
```

**Bước 2.** Gửi cuộc hội thoại đến mô hình, cùng với các định nghĩa Tool.

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

**Bước 3.** Thêm phản hồi của mô hình vào cuộc hội thoại. Sau đó kiểm tra: nó có gọi Tool không, hay đã xong?

```python
messages.append({"role": "assistant", "content": response.content})

# If the model didn't call a tool, the task is finished
if response.stop_reason != "tool_use":
    return
```

**Bước 4.** Thực thi từng lần gọi Tool, thu thập kết quả, và đưa chúng trở lại vào cuộc hội thoại dưới dạng tin nhắn mới. Sau đó lặp lại từ Bước 2.

```python
results = []
for block in response.content:
    if block.type == "tool_use":
        output = run_bash(block.input["command"])
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,  # links result to the tool call
            "content": output,
        })
# This is the "write-back" -- the model can now see the real-world result
messages.append({"role": "user", "content": results})
```

Kết hợp tất cả lại, toàn bộ Agent nằm gọn trong một hàm:

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return  # model is done

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

Đó là toàn bộ Agent trong chưa đến 30 dòng. Mọi thứ khác trong khóa học này đều được xây dựng chồng lên vòng lặp này -- mà không thay đổi hình dạng cốt lõi của nó.

> **Lưu ý về hệ thống thực tế:** Các Agent trong môi trường production thường sử dụng phản hồi dạng streaming, trong đó đầu ra của mô hình được trả về từng Token một thay vì tất cả cùng một lúc. Điều này thay đổi trải nghiệm người dùng (bạn thấy văn bản xuất hiện theo thời gian thực), nhưng vòng lặp cơ bản -- gửi, thực thi, write-back -- vẫn hoàn toàn giống nhau. Chúng ta bỏ qua streaming ở đây để giữ ý tưởng cốt lõi thật rõ ràng.

## Những Gì Đã Thay Đổi

| Thành phần    | Trước đây  | Sau này                        |
|---------------|------------|--------------------------------|
| Agent Loop    | (không có) | `while True` + stop_reason     |
| Tools         | (không có) | `bash` (một Tool)              |
| Tin nhắn      | (không có) | Danh sách tích lũy             |
| Luồng điều khiển | (không có) | `stop_reason != "tool_use"` |

## Thử Ngay

```sh
cd learn-claude-code
python agents/s01_agent_loop.py
```

1. `Create a file called hello.py that prints "Hello, World!"`
2. `List all Python files in this directory`
3. `What is the current git branch?`
4. `Create a directory called test_output and write 3 files in it`

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Xây dựng một Agent Loop hoạt động được từ đầu
- Giải thích tại sao kết quả Tool phải được đưa trở lại vào cuộc hội thoại (bước "write-back")
- Vẽ lại vòng lặp từ trí nhớ: tin nhắn -> mô hình -> thực thi Tool -> write-back -> lượt tiếp theo

## Tiếp Theo Là Gì

Hiện tại, Agent chỉ có thể chạy các lệnh bash. Điều đó có nghĩa là mỗi lần đọc file đều dùng `cat`, mỗi lần chỉnh sửa đều dùng `sed`, và không có ranh giới an toàn nào cả. Trong chương tiếp theo, bạn sẽ thêm các Tool chuyên dụng với một hệ thống định tuyến rõ ràng -- và bản thân vòng lặp sẽ không cần thay đổi gì cả.

## Điểm Mấu Chốt

> Một Agent chỉ là một vòng lặp: gửi tin nhắn đến mô hình, thực thi các Tool mà nó yêu cầu, đưa kết quả trở lại, và lặp lại cho đến khi xong.
