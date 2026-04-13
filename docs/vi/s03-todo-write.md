# s03: TodoWrite

`s01 > s02 > [ s03 ] > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 > s12 > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Cách lập kế hoạch phiên làm việc giúp mô hình bám sát tiến trình trong các tác vụ nhiều bước
- Cách một danh sách todo có cấu trúc với theo dõi trạng thái thay thế các kế hoạch tự do dễ bị lỗi
- Cách các nhắc nhở nhẹ nhàng (nag injection) kéo mô hình trở lại khi nó bị lạc hướng

Bạn đã bao giờ yêu cầu một AI thực hiện một tác vụ phức tạp và xem nó bị lạc hướng giữa chừng chưa? Bạn nói "tái cấu trúc module này: thêm type hint, docstring, kiểm thử và một main guard" và nó hoàn thành hai bước đầu tiên, sau đó lang thang sang điều gì đó bạn chưa từng yêu cầu. Đây không phải là vấn đề về trí thông minh của mô hình -- đây là vấn đề về bộ nhớ làm việc. Khi các kết quả Tool tích lũy trong cuộc hội thoại, kế hoạch ban đầu mờ dần. Đến bước 4, mô hình đã thực sự quên mất các bước 5 đến 10. Bạn cần một cách để giữ kế hoạch luôn hiển thị.

## Vấn Đề

Trong các tác vụ nhiều bước, mô hình bị lạc hướng. Nó lặp lại công việc, bỏ qua các bước, hoặc ứng biến khi System Prompt mờ dần sau nhiều trang kết quả Tool. Context window (tổng lượng văn bản mà mô hình có thể giữ trong bộ nhớ làm việc tại một thời điểm) là hữu hạn, và các hướng dẫn trước đó bị đẩy xa hơn với mỗi lần gọi Tool. Một quy trình tái cấu trúc 10 bước có thể hoàn thành các bước 1-3, sau đó mô hình bắt đầu bịa đặt vì nó đơn giản là không thể "nhìn thấy" các bước 4-10 nữa.

## Giải Pháp

Cung cấp cho mô hình một Tool `todo` duy trì một danh sách kiểm tra có cấu trúc. Sau đó chèn các nhắc nhở nhẹ nhàng khi mô hình đi quá lâu mà không cập nhật kế hoạch.

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> | Tools   |
| prompt |      |       |      | + todo  |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                          |
              +-----------+-----------+
              | TodoManager state     |
              | [ ] task A            |
              | [>] task B  <- doing  |
              | [x] task C            |
              +-----------------------+
                          |
              if rounds_since_todo >= 3:
                inject <reminder> into tool_result
```

## Cách Hoạt Động

**Bước 1.** TodoManager lưu trữ các mục với trạng thái. Ràng buộc "chỉ một `in_progress` tại một thời điểm" buộc mô hình phải hoàn thành điều nó đang làm trước khi tiếp tục.

```python
class TodoManager:
    def update(self, items: list) -> str:
        validated, in_progress_count = [], 0
        for item in items:
            status = item.get("status", "pending")
            if status == "in_progress":
                in_progress_count += 1
            validated.append({"id": item["id"], "text": item["text"],
                              "status": status})
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress")
        self.items = validated
        return self.render()  # returns the checklist as formatted text
```

**Bước 2.** Tool `todo` được đưa vào Dispatch Map như bất kỳ Tool nào khác -- không cần kết nối đặc biệt, chỉ là thêm một mục vào từ điển bạn đã xây dựng trong s02.

```python
TOOL_HANDLERS = {
    # ...base tools...
    "todo": lambda **kw: TODO.update(kw["items"]),
}
```

**Bước 3.** Một nhắc nhở nag injection chèn một gợi ý nếu mô hình đi qua 3+ vòng mà không gọi `todo`. Đây là thủ thuật write-back (đưa kết quả Tool trở lại vào cuộc hội thoại) được dùng cho mục đích mới: harness (đoạn code bọc quanh mô hình) lặng lẽ chèn một nhắc nhở vào payload kết quả trước khi nó được thêm vào tin nhắn.

```python
if rounds_since_todo >= 3:
    results.insert(0, {
        "type": "text",
        "text": "<reminder>Update your todos.</reminder>",
    })
messages.append({"role": "user", "content": results})
```

Ràng buộc "chỉ một in_progress tại một thời điểm" buộc phải tập trung tuần tự. Nhắc nhở nag tạo ra trách nhiệm. Cùng nhau, chúng giữ cho mô hình làm việc theo kế hoạch thay vì bị lạc hướng.

## Những Gì Đã Thay Đổi So Với s02

| Thành phần     | Trước đây (s02)  | Sau này (s03)              |
|----------------|------------------|----------------------------|
| Tools          | 4                | 5 (+todo)                  |
| Lập kế hoạch   | Không có         | TodoManager với trạng thái |
| Nag injection  | Không có         | `<reminder>` sau 3 vòng    |
| Agent Loop     | Dispatch đơn giản| + bộ đếm rounds_since_todo |

## Thử Ngay

```sh
cd learn-claude-code
python agents/s03_todo_write.py
```

1. `Refactor the file hello.py: add type hints, docstrings, and a main guard`
2. `Create a Python package with __init__.py, utils.py, and tests/test_utils.py`
3. `Review all Python files and fix any style issues`

Hãy xem mô hình tạo một kế hoạch, làm việc từng bước một, và đánh dấu các mục khi hoàn thành. Nếu nó quên cập nhật kế hoạch trong vài vòng, bạn sẽ thấy gợi ý `<reminder>` xuất hiện trong cuộc hội thoại.

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Thêm lập kế hoạch phiên làm việc vào bất kỳ Agent nào bằng cách đưa Tool `todo` vào Dispatch Map.
- Thực thi tập trung tuần tự với ràng buộc "chỉ một in_progress tại một thời điểm".
- Sử dụng nag injection để kéo mô hình trở lại đúng hướng khi nó bị lạc.
- Giải thích tại sao trạng thái có cấu trúc tốt hơn văn xuôi tự do cho các kế hoạch nhiều bước.

Hãy ghi nhớ ba ranh giới: `todo` ở đây có nghĩa là "kế hoạch cho cuộc hội thoại hiện tại", không phải cơ sở dữ liệu tác vụ lâu dài. Schema nhỏ `{id, text, status}` là đủ. Một nhắc nhở trực tiếp là đủ -- bạn chưa cần một giao diện lập kế hoạch phức tạp.

## Tiếp Theo Là Gì

Agent của bạn giờ đây có thể lập kế hoạch công việc và bám sát tiến trình. Nhưng mỗi file nó đọc, mỗi đầu ra bash nó tạo ra -- tất cả đều ở lại trong cuộc hội thoại mãi mãi, ngốn dần Context window. Một cuộc điều tra năm file có thể đốt hàng nghìn Token (những mảnh vỡ kích thước từ -- một file 1000 dòng dùng khoảng 4000 Token) mà cuộc hội thoại cha không bao giờ cần đến nữa. Trong s04, bạn sẽ học cách khởi tạo các Subagent với Context mới, cô lập -- để cha giữ sạch và mô hình luôn sắc bén.

## Điểm Mấu Chốt

> Khi kế hoạch nằm trong trạng thái có cấu trúc thay vì văn xuôi tự do, Agent sẽ ít bị lạc hướng hơn nhiều.
