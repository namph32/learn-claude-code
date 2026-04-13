# s06: Context Compact

`s01 > s02 > s03 > s04 > s05 > [ s06 ] > s07 > s08 > s09 > s10 > s11 > s12 > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Tại sao các phiên làm việc dài không tránh khỏi hết không gian Context, và điều gì xảy ra khi điều đó xảy ra
- Một chiến lược nén bốn đòn bẩy: persisted output, micro-compact, auto-compact và manual compact
- Cách chuyển thông tin chi tiết ra khỏi bộ nhớ hoạt động mà không mất đi
- Cách giữ cho phiên làm việc hoạt động vô thời hạn bằng cách tóm tắt và tiếp tục

Agent của bạn từ s05 đã có năng lực. Nó đọc file, chạy lệnh, chỉnh sửa code và ủy thác các tác vụ phụ. Nhưng hãy thử điều gì đó tham vọng -- yêu cầu nó tái cấu trúc một module liên quan đến 30 file. Sau khi đọc tất cả chúng và chạy 20 lệnh shell, bạn sẽ nhận thấy các phản hồi trở nên tệ hơn. Mô hình bắt đầu quên những gì nó đã đọc. Nó lặp lại công việc. Cuối cùng API từ chối yêu cầu của bạn hoàn toàn. Bạn đã chạm đến giới hạn Context window, và nếu không có kế hoạch cho điều đó, Agent của bạn sẽ bị kẹt.

## Vấn Đề

Mỗi lần gọi API đến mô hình đều bao gồm toàn bộ cuộc hội thoại cho đến nay: mỗi tin nhắn người dùng, mỗi phản hồi trợ lý, mỗi lần gọi Tool và kết quả của nó. Context window của mô hình (tổng lượng văn bản nó có thể giữ trong bộ nhớ làm việc tại một thời điểm) là hữu hạn. Một lần `read_file` trên file nguồn 1000 dòng tốn khoảng 4.000 Token (những mảnh vỡ kích thước từ -- một file 1.000 dòng dùng khoảng 4.000 Token). Đọc 30 file và chạy 20 lệnh bash, và bạn đã đốt qua 100.000+ Token. Context đầy, nhưng công việc mới chỉ hoàn thành một nửa.

Cách sửa đơn giản -- chỉ cắt bớt các tin nhắn cũ -- ném đi thông tin mà Agent có thể cần sau này. Một cách tiếp cận thông minh hơn là nén có chiến lược: giữ lại những phần quan trọng, chuyển các chi tiết cồng kềnh ra đĩa, và tóm tắt khi cuộc hội thoại trở nên quá dài. Đó là những gì chương này xây dựng.

## Giải Pháp

Chúng ta sử dụng bốn đòn bẩy, mỗi đòn bẩy hoạt động ở một giai đoạn khác nhau của pipeline, từ lọc tại thời điểm đầu ra đến tóm tắt toàn bộ cuộc hội thoại.

```
Every tool call:
+------------------+
| Tool call result |
+------------------+
        |
        v
[Lever 0: persisted-output]     (at tool execution time)
  Large outputs (>50KB, bash >30KB) are written to disk
  and replaced with a <persisted-output> preview marker.
        |
        v
[Lever 1: micro_compact]        (silent, every turn)
  Replace tool_result > 3 turns old
  with "[Previous: used {tool_name}]"
  (preserves read_file results as reference material)
        |
        v
[Check: tokens > 50000?]
   |               |
   no              yes
   |               |
   v               v
continue    [Lever 2: auto_compact]
              Save transcript to .transcripts/
              LLM summarizes conversation.
              Replace all messages with [summary].
                    |
                    v
            [Lever 3: compact tool]
              Model calls compact explicitly.
              Same summarization as auto_compact.
```

## Cách Hoạt Động

### Bước 1: Đòn Bẩy 0 -- Persisted Output

Tuyến phòng thủ đầu tiên chạy tại thời điểm thực thi Tool, trước khi kết quả thậm chí vào cuộc hội thoại. Khi kết quả Tool vượt quá ngưỡng kích thước, chúng ta ghi đầu ra đầy đủ ra đĩa và thay thế nó bằng một bản xem trước ngắn. Điều này ngăn một đầu ra lệnh khổng lồ duy nhất tiêu thụ một nửa Context window.

```python
PERSIST_OUTPUT_TRIGGER_CHARS_DEFAULT = 50000
PERSIST_OUTPUT_TRIGGER_CHARS_BASH = 30000   # bash uses a lower threshold

def maybe_persist_output(tool_use_id, output, trigger_chars=None):
    if len(output) <= trigger:
        return output                                    # small enough -- keep inline
    stored_path = _persist_tool_result(tool_use_id, output)
    return _build_persisted_marker(stored_path, output)  # swap in a compact preview
    # Returns: <persisted-output>
    #   Output too large (48.8KB). Full output saved to: .task_outputs/tool-results/abc123.txt
    #   Preview (first 2.0KB):
    #   ... first 2000 chars ...
    # </persisted-output>
```

Mô hình có thể dùng `read_file` trên đường dẫn đã lưu để truy cập nội dung đầy đủ nếu cần. Không có gì bị mất -- chi tiết chỉ nằm trên đĩa thay vì trong cuộc hội thoại.

### Bước 2: Đòn Bẩy 1 -- Micro-Compact

Trước mỗi lần gọi LLM, chúng ta quét tìm các kết quả Tool cũ và thay thế chúng bằng các placeholder một dòng. Điều này vô hình với người dùng và chạy mỗi lượt. Điểm tinh tế quan trọng: chúng ta giữ lại các kết quả `read_file` vì chúng phục vụ như tài liệu tham khảo mà mô hình thường cần tra cứu lại.

```python
PRESERVE_RESULT_TOOLS = {"read_file"}

def micro_compact(messages: list) -> list:
    tool_results = [...]  # collect all tool_result entries
    if len(tool_results) <= KEEP_RECENT:
        return messages                                  # not enough results to compact yet
    for part in tool_results[:-KEEP_RECENT]:
        if tool_name in PRESERVE_RESULT_TOOLS:
            continue   # keep reference material
        part["content"] = f"[Previous: used {tool_name}]"  # replace with short placeholder
    return messages
```

### Bước 3: Đòn Bẩy 2 -- Auto-Compact

Khi micro-compact không đủ và số lượng Token vượt ngưỡng, harness thực hiện một bước lớn hơn: nó lưu toàn bộ bản ghi vào đĩa để phục hồi, yêu cầu LLM tóm tắt toàn bộ cuộc hội thoại, sau đó thay thế tất cả tin nhắn bằng bản tóm tắt đó. Agent tiếp tục từ bản tóm tắt như thể không có gì xảy ra.

```python
def auto_compact(messages: list) -> list:
    # Save transcript for recovery
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")
    # LLM summarizes
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity..."
            + json.dumps(messages, default=str)[:80000]}],  # cap at 80K chars for the summary call
        max_tokens=2000,
    )
    return [
        {"role": "user", "content": f"[Compressed]\n\n{response.content[0].text}"},
    ]
```

### Bước 4: Đòn Bẩy 3 -- Manual Compact

Tool `compact` cho phép mô hình tự kích hoạt việc tóm tắt theo yêu cầu. Nó sử dụng chính xác cùng cơ chế như auto-compact. Sự khác biệt là ai quyết định: auto-compact kích hoạt theo ngưỡng, manual compact kích hoạt khi Agent đánh giá đây là thời điểm thích hợp để nén.

### Bước 5: Tích Hợp Trong Agent Loop

Cả bốn đòn bẩy kết hợp tự nhiên trong vòng lặp chính:

```python
def agent_loop(messages: list):
    while True:
        micro_compact(messages)                        # Lever 1
        if estimate_tokens(messages) > THRESHOLD:
            messages[:] = auto_compact(messages)       # Lever 2
        response = client.messages.create(...)
        # ... tool execution with persisted-output ... # Lever 0
        if manual_compact:
            messages[:] = auto_compact(messages)       # Lever 3
```

Các bản ghi lưu giữ lịch sử đầy đủ trên đĩa. Các đầu ra lớn được lưu vào `.task_outputs/tool-results/`. Không có gì thực sự bị mất -- chỉ được chuyển ra khỏi Context hoạt động.

## Những Gì Đã Thay Đổi So Với s05

| Thành phần         | Trước đây (s05)  | Sau này (s06)              |
|--------------------|------------------|----------------------------|
| Tools              | 5                | 5 (cơ bản + compact)       |
| Quản lý Context    | Không có         | Nén bốn đòn bẩy            |
| Persisted-output   | Không có         | Đầu ra lớn -> đĩa + xem trước |
| Micro-compact      | Không có         | Kết quả cũ -> placeholder  |
| Auto-compact       | Không có         | Kích hoạt theo ngưỡng Token|
| Bản ghi            | Không có         | Lưu vào .transcripts/      |

## Thử Ngay

```sh
cd learn-claude-code
python agents/s06_context_compact.py
```

1. `Read every Python file in the agents/ directory one by one` (xem micro-compact thay thế các kết quả cũ)
2. `Keep reading files until compression triggers automatically`
3. `Use the compact tool to manually compress the conversation`

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Giải thích tại sao một phiên làm việc Agent dài bị suy giảm và cuối cùng thất bại mà không có nén
- Chặn các đầu ra Tool quá lớn trước khi chúng vào Context window
- Lặng lẽ thay thế các kết quả Tool cũ bằng các placeholder nhẹ mỗi lượt
- Kích hoạt tóm tắt cuộc hội thoại đầy đủ -- tự động theo ngưỡng hoặc thủ công qua lần gọi Tool
- Lưu giữ toàn bộ bản ghi trên đĩa để không có gì bị mất vĩnh viễn

## Stage 1 Hoàn Thành

Bây giờ bạn có một hệ thống Agent đơn hoàn chỉnh. Bắt đầu từ một lần gọi API thuần túy trong s01, bạn đã xây dựng lên Tool Use, lập kế hoạch có cấu trúc, ủy thác Subagent, tải kỹ năng động và nén Context. Agent của bạn có thể đọc, ghi, thực thi, lập kế hoạch, ủy thác và hoạt động vô thời hạn mà không bị hết bộ nhớ. Đó là một coding Agent thực sự.

Trước khi tiếp tục, hãy cân nhắc quay lại s01 và xây dựng lại toàn bộ stack từ đầu mà không nhìn vào code. Nếu bạn có thể viết cả sáu tầng từ trí nhớ, bạn thực sự làm chủ các ý tưởng -- không chỉ là triển khai.

Stage 2 bắt đầu với s07 và củng cố nền tảng này. Bạn sẽ thêm kiểm soát quyền, hệ thống Hook, Memory bền vững, phục hồi lỗi và nhiều hơn nữa. Agent đơn bạn đã xây dựng ở đây trở thành kernel mà mọi thứ khác bao bọc xung quanh.

## Điểm Mấu Chốt

> Compaction không phải là xóa lịch sử -- đó là việc di dời thông tin chi tiết để Agent có thể tiếp tục làm việc.
