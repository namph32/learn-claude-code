# s04: Subagents

`s01 > s02 > s03 > [ s04 ] > s05 > s06 > s07 > s08 > s09 > s10 > s11 > s12 > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì
- Tại sao việc khám phá một câu hỏi phụ có thể làm ô nhiễm Context của Agent cha
- Cách một Subagent có được lịch sử tin nhắn mới, rỗng
- Cách chỉ một bản tóm tắt ngắn được gửi trở lại cho Agent cha
- Tại sao lịch sử tin nhắn đầy đủ của Agent con bị loại bỏ sau khi sử dụng

Hãy tưởng tượng bạn hỏi Agent của mình "Dự án này sử dụng framework kiểm thử nào?" Để trả lời, nó đọc năm file, phân tích các khối cấu hình và so sánh các câu lệnh import. Tất cả quá trình khám phá đó hữu ích trong một khoảnh khắc -- nhưng khi câu trả lời đã là "pytest," bạn thực sự không muốn những đoạn dump năm file đó nằm mãi trong cuộc hội thoại. Mỗi lần gọi API trong tương lai đều phải gánh thêm trọng tải đó, đốt Token và phân tán sự chú ý của mô hình. Bạn cần một cách để đặt câu hỏi phụ trong một phòng sạch sẽ và chỉ mang về câu trả lời.

## Vấn Đề

Khi Agent làm việc, mảng `messages` của nó ngày càng phình to. Mỗi lần đọc file, mỗi đầu ra bash đều ở lại trong Context vĩnh viễn. Một câu hỏi đơn giản như "framework kiểm thử là gì?" có thể yêu cầu đọc năm file, nhưng Agent cha chỉ cần một từ trở về: "pytest." Nếu không cô lập, những artifact trung gian đó ở lại trong Context suốt phiên làm việc, lãng phí Token cho mỗi lần gọi API tiếp theo và làm mờ sự chú ý của mô hình. Phiên làm việc càng dài, vấn đề này càng tệ -- Context đầy ắp rác thải khám phá không liên quan đến tác vụ hiện tại.

## Giải Pháp

Agent cha ủy thác các tác vụ phụ cho một Agent con bắt đầu với `messages=[]` rỗng. Agent con thực hiện tất cả quá trình khám phá lộn xộn, sau đó chỉ có bản tóm tắt văn bản cuối cùng của nó được gửi trở lại. Lịch sử đầy đủ của Agent con bị loại bỏ.

```
Parent agent                     Subagent
+------------------+             +------------------+
| messages=[...]   |             | messages=[]      | <-- fresh
|                  |  dispatch   |                  |
| tool: task       | ----------> | while tool_use:  |
|   prompt="..."   |             |   call tools     |
|                  |  summary    |   append results |
|   result = "..." | <---------- | return last text |
+------------------+             +------------------+

Parent context stays clean. Subagent context is discarded.
```

## Cách Hoạt Động

**Bước 1.** Agent cha có một Tool `task` mà Agent con không có. Điều này ngăn việc tạo ra đệ quy -- một Agent con không thể tạo ra Agent con của riêng nó.

```python
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task",
     "description": "Spawn a subagent with fresh context.",
     "input_schema": {
         "type": "object",
         "properties": {"prompt": {"type": "string"}},
         "required": ["prompt"],
     }},
]
```

**Bước 2.** Subagent bắt đầu với `messages=[]` và chạy Agent Loop của riêng nó. Chỉ có khối văn bản cuối cùng được trả về cho Agent cha dưới dạng `tool_result`.

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
    for _ in range(30):  # safety limit
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM,
            messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant",
                             "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input)
                results.append({"type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    # Extract only the final text -- everything else is thrown away
    return "".join(
        b.text for b in response.content if hasattr(b, "text")
    ) or "(no summary)"
```

Toàn bộ lịch sử tin nhắn của Agent con (có thể là 30+ lần gọi Tool đáng giá các lần đọc file và đầu ra bash) bị loại bỏ ngay khi `run_subagent` trả về. Agent cha nhận được một bản tóm tắt một đoạn văn như một `tool_result` bình thường, giữ cho Context của chính nó sạch sẽ.

## Những Gì Đã Thay Đổi So Với s03

| Thành phần     | Trước đây (s03)  | Sau này (s04)             |
|----------------|------------------|---------------------------|
| Tools          | 5                | 5 (cơ bản) + task (cha)   |
| Context        | Dùng chung duy nhất | Cô lập cha + con       |
| Subagent       | Không có         | Hàm `run_subagent()`      |
| Giá trị trả về | N/A              | Chỉ văn bản tóm tắt       |

## Thử Ngay

```sh
cd learn-claude-code
python agents/s04_subagent.py
```

1. `Use a subtask to find what testing framework this project uses`
2. `Delegate: read all .py files and summarize what each one does`
3. `Use a task to create a new module, then verify it from here`

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Giải thích tại sao một Subagent chủ yếu là một **ranh giới Context**, không phải một thủ thuật tiến trình
- Tạo ra một Agent con một lần dùng với `messages=[]` mới
- Chỉ trả về bản tóm tắt cho Agent cha, loại bỏ tất cả quá trình khám phá trung gian
- Quyết định Agent con nên và không nên có quyền truy cập vào các Tool nào

Bạn chưa cần các worker tồn tại lâu dài, các phiên có thể tiếp tục, hay cô lập Worktree. Ý tưởng cốt lõi rất đơn giản: cung cấp cho tác vụ phụ một không gian làm việc sạch sẽ trong bộ nhớ, sau đó chỉ mang về câu trả lời mà Agent cha vẫn cần.

## Tiếp Theo Là Gì

Cho đến nay bạn đã học cách giữ Context sạch sẽ bằng cách cô lập các tác vụ phụ. Nhưng còn kiến thức mà Agent mang sẵn thì sao? Trong s05, bạn sẽ thấy cách tránh làm phình to System Prompt với các chuyên môn miền mà mô hình có thể không bao giờ sử dụng -- tải các kỹ năng theo yêu cầu thay vì trước đó.

## Điểm Mấu Chốt

> Một Subagent là một tờ giấy nháp dùng xong bỏ: Context mới vào, bản tóm tắt ngắn ra, tất cả còn lại bị loại bỏ.
