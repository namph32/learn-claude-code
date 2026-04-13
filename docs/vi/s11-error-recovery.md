# s11: Error Recovery

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > [ s11 ] > s12 > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Ba danh mục lỗi có thể khôi phục: cắt ngắn, tràn Context, và lỗi vận chuyển tạm thời
- Cách định tuyến mỗi lỗi đến nhánh khôi phục đúng (tiếp nối, Compaction, hoặc rút lui)
- Tại sao ngân sách thử lại ngăn chặn các vòng lặp vô hạn
- Cách trạng thái khôi phục giữ "lý do tại sao" hiển thị thay vì chôn vùi trong một khối catch

Agent của bạn đang làm công việc thực sự -- đọc tệp, viết code, gọi Tool qua nhiều lượt. Và công việc thực sự tạo ra lỗi thực sự. Đầu ra bị cắt ngắn giữa câu. Prompt phát triển vượt quá cửa sổ Context của mô hình. API hết thời gian hoặc chạm giới hạn tốc độ. Nếu mọi lỗi trong số này kết thúc quá trình chạy ngay lập tức, hệ thống của bạn cảm thấy dễ vỡ và người dùng học được không tin tưởng vào nó. Nhưng đây là nhận thức quan trọng: hầu hết các lỗi này không phải là lỗi tác vụ thực sự. Chúng là tín hiệu rằng bước tiếp theo cần một đường tiếp nối khác.

## Vấn Đề

Người dùng của bạn yêu cầu Agent tái cấu trúc một tệp lớn. Mô hình bắt đầu viết phiên bản mới, nhưng đầu ra chạm `max_tokens` và dừng giữa hàm. Không có khôi phục, Agent chỉ dừng lại với một tệp viết dở. Người dùng phải để ý, nhắc lại, và hy vọng mô hình tiếp tục từ nơi nó dừng lại.

Hoặc: cuộc hội thoại đã chạy 40 lượt. Các tin nhắn tích lũy đẩy prompt vượt quá giới hạn Context của mô hình. API trả về lỗi. Không có khôi phục, toàn bộ phiên bị mất.

Hoặc: một sự cố mạng tạm thời làm ngắt kết nối. Không có khôi phục, Agent bị sập mặc dù cùng một yêu cầu sẽ thành công một giây sau.

Mỗi loại là một loại lỗi khác nhau, và mỗi loại cần một hành động khôi phục khác nhau. Một lần thử lại bắt tất cả không thể xử lý cả ba một cách đúng đắn.

## Giải Pháp

Phân loại lỗi trước, chọn nhánh khôi phục sau, và thực thi ngân sách thử lại để hệ thống không thể lặp mãi.

```text
LLM call
  |
  +-- stop_reason == "max_tokens"
  |      -> append continuation reminder
  |      -> retry
  |
  +-- prompt too long
  |      -> compact context
  |      -> retry
  |
  +-- timeout / rate limit / connection error
         -> back off
         -> retry
```

## Cách Hoạt Động

**Bước 1. Theo dõi trạng thái khôi phục.** Trước khi có thể khôi phục, bạn cần biết đã thử bao nhiêu lần. Một bộ đếm đơn giản mỗi danh mục ngăn chặn các vòng lặp vô hạn:

```python
recovery_state = {
    "continuation_attempts": 0,
    "compact_attempts": 0,
    "transport_attempts": 0,
}
```

**Bước 2. Phân loại lỗi.** Mỗi lỗi ánh xạ đến đúng một loại khôi phục. Bộ phân loại kiểm tra stop reason và văn bản lỗi, rồi trả về một quyết định có cấu trúc:

```python
def choose_recovery(stop_reason: str | None, error_text: str | None) -> dict:
    if stop_reason == "max_tokens":
        return {"kind": "continue", "reason": "output truncated"}

    if error_text and "prompt" in error_text and "long" in error_text:
        return {"kind": "compact", "reason": "context too large"}

    if error_text and any(word in error_text for word in [
        "timeout", "rate", "unavailable", "connection"
    ]):
        return {"kind": "backoff", "reason": "transient transport failure"}

    return {"kind": "fail", "reason": "unknown or non-recoverable error"}
```

Sự phân tách quan trọng: phân loại trước, hành động sau. Bằng cách đó lý do khôi phục vẫn hiển thị trong trạng thái thay vì biến mất bên trong một khối catch.

**Bước 3. Xử lý tiếp nối (đầu ra bị cắt ngắn).** Khi mô hình hết không gian đầu ra, tác vụ không thất bại -- lượt chỉ kết thúc quá sớm. Bạn chèn một nhắc nhở tiếp nối và thử lại:

```python
CONTINUE_MESSAGE = (
    "Output limit hit. Continue directly from where you stopped. "
    "Do not restart or repeat."
)
```

Nếu không có nhắc nhở này, các mô hình có xu hướng bắt đầu lại từ đầu hoặc lặp lại những gì chúng đã viết. Hướng dẫn rõ ràng "tiếp tục trực tiếp" giữ đầu ra tiến về phía trước.

**Bước 4. Xử lý Compaction (tràn Context).** Khi prompt trở nên quá lớn, vấn đề không phải là tác vụ -- Context tích lũy cần thu nhỏ trước khi lượt tiếp theo có thể tiến hành. Bạn gọi cùng cơ chế `auto_compact` từ s06 để tóm tắt lịch sử, rồi thử lại:

```python
if decision["kind"] == "compact":
    messages = auto_compact(messages)
    continue
```

**Bước 5. Xử lý rút lui (lỗi tạm thời).** Khi lỗi có thể là tạm thời -- hết thời gian, giới hạn tốc độ, ngừng hoạt động ngắn -- bạn chờ và thử lại. Rút lui theo hàm mũ (tăng gấp đôi độ trễ mỗi lần thử, cộng với jitter ngẫu nhiên để tránh vấn đề thundering-herd khi nhiều client thử lại cùng một lúc) giữ hệ thống không tấn công một máy chủ đang gặp khó khăn:

```python
def backoff_delay(attempt: int) -> float:
    delay = min(BACKOFF_BASE_DELAY * (2 ** attempt), BACKOFF_MAX_DELAY)
    jitter = random.uniform(0, 1)
    return delay + jitter
```

**Bước 6. Kết nối vào vòng lặp.** Logic khôi phục nằm ngay bên trong Agent Loop. Mỗi nhánh điều chỉnh tin nhắn và tiếp tục, hoặc từ bỏ:

```python
while True:
    try:
        response = client.messages.create(...)
        decision = choose_recovery(response.stop_reason, None)
    except Exception as e:
        response = None
        decision = choose_recovery(None, str(e).lower())

    if decision["kind"] == "continue":
        messages.append({"role": "user", "content": CONTINUE_MESSAGE})
        continue

    if decision["kind"] == "compact":
        messages = auto_compact(messages)
        continue

    if decision["kind"] == "backoff":
        time.sleep(backoff_delay(...))
        continue

    if decision["kind"] == "fail":
        break
```

Điểm mấu chốt không phải là code thông minh. Điểm mấu chốt là: phân loại, chọn, thử lại với ngân sách.

## Những Thay Đổi So Với s10

| Khía cạnh | s10: System Prompt | s11: Error Recovery |
|-----------|--------------------|--------------------|
| Mối quan tâm cốt lõi | Lắp ráp đầu vào mô hình từ các phần | Xử lý lỗi mà không bị sập |
| Hành vi vòng lặp | Chạy cho đến end_turn hoặc tool_use | Thêm các nhánh khôi phục trước khi từ bỏ |
| Compaction | Chưa được đề cập | Được kích hoạt phản ứng khi tràn Context |
| Logic thử lại | Chưa được đề cập | Có ngân sách mỗi danh mục lỗi |
| Theo dõi trạng thái | Các phần prompt | Các bộ đếm khôi phục |

## Ghi Chú Về Các Hệ Thống Thực Tế

Các hệ thống Agent thực tế cũng lưu trữ trạng thái phiên lên đĩa, để sự cố không phá hủy một cuộc hội thoại chạy dài. Lưu trữ phiên, checkpoint, và tiếp tục là các mối quan tâm riêng biệt so với khôi phục lỗi -- nhưng chúng bổ sung cho nó. Khôi phục xử lý các lỗi bạn có thể thử lại trong quá trình; lưu trữ xử lý các lỗi bạn không thể. Harness dạy học này tập trung vào các đường khôi phục trong quá trình, nhưng hãy nhớ rằng các hệ thống sản xuất cần cả hai tầng.

## Đọc Cùng Nhau

- Nếu bạn bắt đầu mất dấu tại sao Query hiện tại vẫn tiếp tục, hãy quay lại [`s00c-query-transition-model.md`](./s00c-query-transition-model.md).
- Nếu Compaction Context và khôi phục lỗi bắt đầu trông giống nhau, hãy đọc lại [`s06-context-compact.md`](./s06-context-compact.md) để tách "thu nhỏ Context" khỏi "khôi phục sau lỗi."
- Nếu bạn sắp chuyển sang `s12`, hãy giữ [`data-structures.md`](./data-structures.md) ở gần vì hệ thống tác vụ thêm một tầng công việc bền vững mới trên trạng thái khôi phục.

## Các Lỗi Phổ Biến Của Người Mới

**Lỗi 1: dùng một quy tắc thử lại cho mọi lỗi.** Các lỗi khác nhau cần các hành động khôi phục khác nhau. Thử lại một lỗi tràn Context mà không Compact trước sẽ chỉ tạo ra cùng lỗi đó lại.

**Lỗi 2: không có ngân sách thử lại.** Nếu không có ngân sách, hệ thống có thể lặp mãi. Mỗi danh mục khôi phục cần bộ đếm riêng và giới hạn riêng.

**Lỗi 3: ẩn lý do khôi phục.** Hệ thống nên biết *tại sao* nó đang thử lại. Lý do đó nên giữ hiển thị trong trạng thái -- như một đối tượng quyết định có cấu trúc -- không biến mất bên trong một khối catch.

## Thử Nghiệm

```sh
cd learn-claude-code
python agents/s11_error_recovery.py
```

Thử ép buộc:

- một phản hồi dài (để kích hoạt tiếp nối max_tokens)
- một Context lớn (để kích hoạt Compaction)
- một hết thời gian tạm thời (để kích hoạt rút lui)

Sau đó quan sát nhánh khôi phục nào hệ thống chọn và bộ đếm thử lại tăng như thế nào.

## Những Gì Bạn Đã Thành Thạo

Tại thời điểm này, bạn có thể:

- Phân loại lỗi Agent thành ba danh mục có thể khôi phục và một danh mục cuối cùng
- Định tuyến mỗi lỗi đến nhánh khôi phục đúng: tiếp nối, Compaction, hoặc rút lui
- Thực thi ngân sách thử lại để hệ thống không bao giờ lặp mãi
- Giữ các quyết định khôi phục hiển thị như trạng thái có cấu trúc thay vì chôn vùi trong các bộ xử lý ngoại lệ
- Giải thích tại sao các loại lỗi khác nhau cần các hành động khôi phục khác nhau

## Stage 2 Hoàn Thành

Bạn đã hoàn thành Stage 2 của Harness. Hãy nhìn vào những gì bạn đã xây dựng kể từ Stage 1:

- **s07 Permission System** -- Harness hỏi trước khi hành động, và người dùng kiểm soát những gì được tự động phê duyệt
- **s08 Hook System** -- các script bên ngoài chạy tại các điểm vòng đời mà không cần chạm vào Agent Loop
- **s09 Memory System** -- các sự kiện bền vững tồn tại qua các phiên
- **s10 System Prompt** -- prompt là một quy trình lắp ráp với các phần rõ ràng, không phải một chuỗi lớn
- **s11 Error Recovery** -- các lỗi được định tuyến đến đường khôi phục đúng thay vì bị sập

Agent của bạn bắt đầu Stage 2 như một vòng lặp hoạt động có thể gọi Tool và quản lý Context. Nó kết thúc Stage 2 như một hệ thống tự quản lý: nó kiểm tra quyền, chạy Hook, nhớ những gì quan trọng, lắp ráp hướng dẫn của chính nó, và khôi phục từ lỗi mà không cần can thiệp của con người.

Đó là một Harness Agent thực sự. Nếu bạn dừng ở đây và xây dựng một sản phẩm trên nó, bạn sẽ có thứ gì đó thực sự hữu ích.

Nhưng còn nhiều thứ để xây dựng. Stage 3 giới thiệu quản lý công việc có cấu trúc -- danh sách tác vụ, thực thi nền, và công việc theo lịch. Agent ngừng phản ứng thuần túy và bắt đầu tổ chức công việc của chính nó theo thời gian. Hẹn gặp bạn ở [s12: Task System](./s12-task-system.md).

## Điểm Mấu Chốt

> Hầu hết các lỗi Agent không phải là lỗi tác vụ thực sự -- chúng là tín hiệu để thử một đường tiếp nối khác, và Harness nên phân loại chúng và tự động khôi phục.
