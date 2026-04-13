# s02: Tool Use

`s01 > [ s02 ] > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 > s12 > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Cách xây dựng một Dispatch Map (bảng định tuyến ánh xạ tên Tool tới hàm xử lý)
- Cách path sandboxing ngăn mô hình thoát ra ngoài không gian làm việc
- Cách thêm Tool mới mà không cần chỉnh sửa Agent Loop

Nếu bạn đã chạy Agent s01 được vài phút, hẳn bạn đã nhận ra những điểm yếu. `cat` lặng lẽ cắt bớt các file dài. `sed` bị nghẹt với các ký tự đặc biệt. Mỗi lệnh bash đều là một cánh cửa mở -- không có gì ngăn mô hình chạy `rm -rf /` hay đọc các khóa SSH của bạn. Bạn cần các Tool chuyên dụng với các rào chắn an toàn, và bạn cần một cách rõ ràng để thêm chúng vào.

## Vấn Đề

Khi chỉ có `bash`, Agent sử dụng shell cho mọi thứ. Không có cách nào giới hạn những gì nó đọc, nơi nó ghi, hay lượng đầu ra nó trả về. Một lệnh xấu duy nhất có thể làm hỏng file, làm lộ thông tin bí mật, hoặc vượt quá ngân sách Token với một luồng stdout khổng lồ. Điều bạn thực sự muốn là một tập hợp nhỏ các Tool có mục đích rõ ràng -- `read_file`, `write_file`, `edit_file` -- mỗi Tool có các kiểm tra an toàn riêng. Câu hỏi đặt ra là: làm sao kết nối chúng vào mà không cần viết lại vòng lặp mỗi lần?

## Giải Pháp

Câu trả lời là một Dispatch Map -- một từ điển duy nhất định tuyến tên Tool tới các hàm xử lý. Thêm một Tool đồng nghĩa với thêm một mục. Bản thân vòng lặp không bao giờ thay đổi.

```
+--------+      +-------+      +------------------+
|  User  | ---> |  LLM  | ---> | Tool Dispatch    |
| prompt |      |       |      | {                |
+--------+      +---+---+      |   bash: run_bash |
                    ^           |   read: run_read |
                    |           |   write: run_wr  |
                    +-----------+   edit: run_edit |
                    tool_result | }                |
                                +------------------+

The dispatch map is a dict: {tool_name: handler_function}.
One lookup replaces any if/elif chain.
```

## Cách Hoạt Động

**Bước 1.** Mỗi Tool có một hàm xử lý. Path sandboxing ngăn mô hình thoát ra ngoài không gian làm việc -- mỗi đường dẫn được yêu cầu đều được giải quyết và kiểm tra so với thư mục làm việc trước khi bất kỳ thao tác I/O nào xảy ra.

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path

def run_read(path: str, limit: int = None) -> str:
    text = safe_path(path).read_text()
    lines = text.splitlines()
    if limit and limit < len(lines):
        lines = lines[:limit]
    return "\n".join(lines)[:50000]  # hard cap to avoid blowing up the context
```

**Bước 2.** Dispatch Map liên kết tên Tool với các hàm xử lý. Đây là toàn bộ tầng định tuyến -- không có chuỗi if/elif, không có phân cấp lớp, chỉ là một từ điển.

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"],
                                        kw["new_text"]),
}
```

**Bước 3.** Trong vòng lặp, tra cứu hàm xử lý theo tên. Thân vòng lặp không thay đổi so với s01 -- chỉ có dòng điều phối là mới.

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler \
            else f"Unknown tool: {block.name}"
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
```

Thêm Tool = thêm hàm xử lý + thêm mục Schema. Vòng lặp không bao giờ thay đổi.

## Những Gì Đã Thay Đổi So Với s01

| Thành phần     | Trước đây (s01)    | Sau này (s02)              |
|----------------|--------------------|----------------------------|
| Tools          | 1 (chỉ bash)       | 4 (bash, read, write, edit)|
| Dispatch       | Gọi bash cứng      | Từ điển `TOOL_HANDLERS`    |
| An toàn đường dẫn | Không có        | Hộp cát `safe_path()`      |
| Agent Loop     | Không thay đổi     | Không thay đổi             |

## Thử Ngay

```sh
cd learn-claude-code
python agents/s02_tool_use.py
```

1. `Read the file requirements.txt`
2. `Create a file called greet.py with a greet(name) function`
3. `Edit greet.py to add a docstring to the function`
4. `Read greet.py to verify the edit worked`

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Kết nối bất kỳ Tool mới nào vào Agent bằng cách thêm một hàm xử lý và một mục Schema -- mà không cần chỉnh sửa vòng lặp.
- Áp dụng path sandboxing để mô hình không thể đọc hay ghi ra ngoài không gian làm việc.
- Giải thích tại sao Dispatch Map mở rộng tốt hơn so với chuỗi if/elif.

Giữ ranh giới rõ ràng: Schema của Tool là đủ cho lúc này. Bạn chưa cần các tầng chính sách, giao diện phê duyệt hay hệ sinh thái plugin. Nếu bạn có thể thêm một Tool mới mà không cần viết lại vòng lặp, bạn đã nắm được mẫu cốt lõi.

## Tiếp Theo Là Gì

Agent của bạn giờ đây có thể đọc, ghi và chỉnh sửa file một cách an toàn. Nhưng điều gì xảy ra khi bạn yêu cầu nó thực hiện một quy trình tái cấu trúc 10 bước? Nó hoàn thành các bước 1 đến 3 và sau đó bắt đầu ứng biến vì đã quên phần còn lại. Trong s03, bạn sẽ trang bị cho Agent một kế hoạch phiên làm việc -- một danh sách todo có cấu trúc giúp nó bám sát tiến trình trong các tác vụ phức tạp, nhiều bước.

## Điểm Mấu Chốt

> Vòng lặp không nên quan tâm đến cách hoạt động nội tại của một Tool. Nó chỉ cần một tuyến đường đáng tin cậy từ tên Tool đến hàm xử lý.
