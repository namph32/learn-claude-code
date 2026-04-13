# s13: Tác Vụ Nền

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 > s12 > [ s13 ] > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Cách chạy các lệnh chậm trong các luồng nền trong khi vòng lặp chính vẫn phản hồi
- Cách một hàng đợi thông báo an toàn với luồng gửi kết quả trở lại cho agent
- Cách daemon thread giữ tiến trình gọn gàng khi thoát
- Cách mẫu drain-before-call đưa kết quả nền vào đúng thời điểm

Bạn đã có một đồ thị tác vụ rồi, và mỗi tác vụ có thể biểu diễn những gì nó phụ thuộc vào. Nhưng có một vấn đề thực tế: một số tác vụ liên quan đến các lệnh mất hàng phút. `npm install`, `pytest`, `docker build` -- những thứ này chặn vòng lặp chính, và trong khi agent chờ, người dùng cũng phải chờ. Nếu người dùng nói "cài đặt các phụ thuộc và trong khi đó tạo tệp config," agent từ s12 thực hiện chúng tuần tự vì nó không có cách nào để bắt đầu điều gì đó và quay lại sau. Chương này khắc phục điều đó bằng cách thêm khả năng thực thi nền.

## Vấn Đề

Hãy xem xét một quy trình làm việc thực tế: người dùng yêu cầu agent chạy toàn bộ bộ kiểm thử (mất 90 giây) và sau đó thiết lập tệp cấu hình. Với vòng lặp chặn, agent gửi lệnh kiểm thử, nhìn chằm chằm vào subprocess quay vòng trong 90 giây, nhận kết quả, rồi mới bắt đầu tệp config. Người dùng xem mọi thứ xảy ra tuần tự. Tệ hơn nữa, nếu có ba lệnh chậm, tổng thời gian thực là tổng của cả ba -- mặc dù chúng có thể chạy song song. Agent cần một cách để bắt đầu công việc chậm, trả quyền kiểm soát lại cho vòng lặp chính ngay lập tức, và lấy kết quả sau.

## Giải Pháp

Giữ vòng lặp chính đơn luồng, nhưng chạy các subprocess chậm trên các daemon thread nền. Khi một lệnh nền hoàn thành, kết quả của nó đi vào một hàng đợi thông báo an toàn với luồng. Trước mỗi lần gọi LLM, vòng lặp chính xả hàng đợi đó và đưa bất kỳ kết quả đã hoàn thành nào vào cuộc hội thoại.

```
Main thread                Background thread
+-----------------+        +-----------------+
| agent loop      |        | subprocess runs |
| ...             |        | ...             |
| [LLM call] <---+------- | enqueue(result) |
|  ^drain queue   |        +-----------------+
+-----------------+

Timeline:
Agent --[spawn A]--[spawn B]--[other work]----
             |          |
             v          v
          [A runs]   [B runs]      (parallel)
             |          |
             +-- results injected before next LLM call --+
```

## Cách Thức Hoạt Động

**Bước 1.** Tạo một `BackgroundManager` theo dõi các tác vụ đang chạy với một hàng đợi thông báo an toàn với luồng. Khóa đảm bảo rằng luồng chính và các luồng nền không bao giờ làm hỏng hàng đợi cùng lúc.

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}
        self._notification_queue = []
        self._lock = threading.Lock()
```

**Bước 2.** Phương thức `run()` khởi động một daemon thread và trả về ngay lập tức. Daemon thread là luồng mà Python Runtime tự động tắt khi chương trình chính thoát -- bạn không cần phải join hoặc dọn dẹp nó.

```python
def run(self, command: str) -> str:
    task_id = str(uuid.uuid4())[:8]
    self.tasks[task_id] = {"status": "running", "command": command}
    thread = threading.Thread(
        target=self._execute, args=(task_id, command), daemon=True)
    thread.start()
    return f"Background task {task_id} started"
```

**Bước 3.** Khi subprocess hoàn thành, luồng nền đặt kết quả của nó vào hàng đợi thông báo. Khóa làm cho điều này an toàn ngay cả khi luồng chính đang xả hàng đợi cùng lúc.

```python
def _execute(self, task_id, command):
    try:
        r = subprocess.run(command, shell=True, cwd=WORKDIR,
            capture_output=True, text=True, timeout=300)
        output = (r.stdout + r.stderr).strip()[:50000]
    except subprocess.TimeoutExpired:
        output = "Error: Timeout (300s)"
    with self._lock:
        self._notification_queue.append({
            "task_id": task_id, "result": output[:500]})
```

**Bước 4.** Vòng lặp agent xả các thông báo trước mỗi lần gọi LLM. Đây là mẫu drain-before-call: ngay trước khi bạn yêu cầu mô hình suy nghĩ, thu thập mọi kết quả nền và thêm chúng vào cuộc hội thoại để mô hình thấy chúng ở lượt tiếp theo.

```python
def agent_loop(messages: list):
    while True:
        notifs = BG.drain_notifications()
        if notifs:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['result']}" for n in notifs)
            messages.append({"role": "user",
                "content": f"<background-results>\n{notif_text}\n"
                           f"</background-results>"})
            messages.append({"role": "assistant",
                "content": "Noted background results."})
        response = client.messages.create(...)
```

Demo dạy học này giữ vòng lặp cốt lõi đơn luồng; chỉ việc chờ subprocess được song song hóa. Một hệ thống production thường sẽ chia công việc nền thành nhiều làn Runtime, nhưng bắt đầu với một mẫu sạch giúp cơ chế dễ theo dõi hơn.

## Đọc Cùng Nhau

- Nếu bạn chưa hoàn toàn phân tách "mục tiêu tác vụ" khỏi "slot thực thi đang chạy," hãy đọc [`s13a-runtime-task-model.md`](./s13a-runtime-task-model.md) trước -- nó làm rõ tại sao bản ghi tác vụ và bản ghi Runtime là các đối tượng khác nhau.
- Nếu bạn không chắc trạng thái nào thuộc về `RuntimeTaskRecord` và trạng thái nào vẫn thuộc về bảng tác vụ, hãy giữ [`data-structures.md`](./data-structures.md) gần bên.
- Nếu thực thi nền bắt đầu cảm thấy như "một vòng lặp chính khác," hãy quay lại [`s02b-tool-execution-runtime.md`](./s02b-tool-execution-runtime.md) và đặt lại ranh giới: thực thi và chờ đợi có thể chạy song song, nhưng vòng lặp chính vẫn là một luồng chính.

## Những Gì Đã Thay Đổi

| Thành phần     | Trước (s12)      | Sau (s13)                  |
|----------------|------------------|----------------------------|
| Tool           | 8                | 6 (base + background_run + check)|
| Thực thi       | Chỉ chặn        | Chặn + luồng nền           |
| Thông báo      | Không có         | Hàng đợi xả mỗi vòng lặp  |
| Đồng thời      | Không có         | Daemon thread              |

## Thử Ngay

```sh
cd learn-claude-code
python agents/s13_background_tasks.py
```

1. `Run "sleep 5 && echo done" in the background, then create a file while it runs`
2. `Start 3 background tasks: "sleep 2", "sleep 4", "sleep 6". Check their status.`
3. `Run pytest in the background and keep working on other things`

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Chạy các subprocess chậm trên daemon thread mà không chặn vòng lặp agent chính
- Thu thập kết quả thông qua hàng đợi thông báo an toàn với luồng
- Đưa kết quả nền vào cuộc hội thoại sử dụng mẫu drain-before-call
- Cho phép agent làm việc khác trong khi các lệnh chạy lâu hoàn thành song song

## Tiếp Theo Là Gì

Các tác vụ nền giải quyết vấn đề công việc chậm bắt đầu ngay bây giờ. Nhưng còn công việc nên bắt đầu sau thì sao -- "chạy cái này mỗi đêm" hay "nhắc tôi trong 30 phút"? Trong s14, bạn sẽ thêm một bộ lập lịch trình cron lưu trữ ý định tương lai và kích hoạt nó khi đến thời điểm.

## Điểm Mấu Chốt

> Thực thi nền là một làn Runtime, không phải vòng lặp chính thứ hai -- công việc chậm chạy trên daemon thread và gửi kết quả trở lại thông qua một hàng đợi thông báo duy nhất.
