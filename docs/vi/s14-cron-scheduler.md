# s14: Bộ Lập Lịch Cron

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 > s12 > s13 > [ s14 ] > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Cách các bản ghi lịch trình lưu trữ ý định tương lai dưới dạng dữ liệu bền vững
- Cách một bộ kiểm tra dựa trên thời gian biến các biểu thức cron thành các thông báo được kích hoạt
- Sự khác biệt giữa các công việc bền vững (tồn tại qua khởi động lại) và các công việc chỉ trong phiên (mất khi tiến trình kết thúc)
- Cách công việc đã lên lịch trình quay lại hệ thống agent thông qua cùng hàng đợi thông báo từ s13

Trong s13, bạn đã học cách chạy công việc chậm ở chế độ nền để agent không bị chặn. Nhưng công việc đó vẫn bắt đầu ngay lập tức -- người dùng nói "chạy cái này" và nó chạy ngay bây giờ. Các quy trình làm việc thực tế thường cần công việc bắt đầu muộn hơn: "chạy cái này mỗi đêm," "tạo báo cáo mỗi sáng thứ Hai," "nhắc tôi kiểm tra lại trong 30 phút." Nếu không có lập lịch trình, người dùng phải gửi lại cùng một yêu cầu mỗi lần. Chương này thêm một ý tưởng mới: lưu trữ ý định tương lai ngay bây giờ, kích hoạt nó sau. Và nó kết thúc Stage 3 bằng cách hoàn thiện tiến trình từ tác vụ bền vững (s12) đến thực thi nền (s13) đến kích hoạt dựa trên thời gian (s14).

## Vấn Đề

Agent của bạn giờ đây có thể quản lý đồ thị tác vụ và chạy lệnh ở chế độ nền. Nhưng mọi công việc đều bắt đầu khi người dùng yêu cầu một cách tường minh. Nếu người dùng muốn chạy kiểm thử hàng đêm, họ phải nhớ gõ "chạy kiểm thử" mỗi tối. Nếu họ muốn báo cáo trạng thái hàng tuần, họ phải mở phiên mỗi sáng thứ Hai. Agent không có khái niệm về thời gian tương lai -- nó phản ứng với những gì bạn nói ngay bây giờ, và nó không thể hành động theo điều gì đó bạn muốn xảy ra vào ngày mai. Bạn cần một cách để ghi lại "làm X vào thời điểm Y" và để hệ thống tự động kích hoạt nó.

## Giải Pháp

Thêm ba bộ phận chuyển động: các bản ghi lịch trình mô tả khi nào và làm gì, một bộ kiểm tra thời gian chạy ở chế độ nền và kiểm tra xem lịch trình nào khớp với thời gian hiện tại, và cùng hàng đợi thông báo từ s13 để đưa công việc được kích hoạt trở lại vòng lặp chính.

```text
schedule_create(...)
  ->
ghi một bản ghi lịch trình bền vững
  ->
bộ kiểm tra thời gian thức dậy và kiểm tra "quy tắc này có khớp ngay bây giờ không?"
  ->
nếu có, đưa thông báo đã lên lịch vào hàng đợi
  ->
vòng lặp chính đưa thông báo đó vào như công việc mới
```

Điểm mấu chốt là bộ lập lịch trình không phải là vòng lặp agent thứ hai. Nó đưa các prompt được kích hoạt vào cùng hệ thống mà agent đã sử dụng. Vòng lặp chính không biết hoặc không quan tâm liệu một công việc đến từ người dùng gõ hay từ một kích hoạt cron -- nó xử lý cả hai theo cùng một cách.

## Cách Thức Hoạt Động

**Bước 1.** Định nghĩa bản ghi lịch trình. Mỗi công việc lưu trữ một biểu thức cron (cú pháp khớp thời gian gọn như `0 9 * * 1` có nghĩa là "9:00 SA mỗi thứ Hai"), prompt để thực thi, liệu nó có định kỳ hay chỉ kích hoạt một lần, và dấu thời gian `last_fired_at` để ngăn kích hoạt kép.

```python
schedule = {
    "id": "job_001",
    "cron": "0 9 * * 1",
    "prompt": "Run the weekly status report.",
    "recurring": True,
    "durable": True,
    "created_at": 1710000000.0,
    "last_fired_at": None,
}
```

Một công việc bền vững được ghi vào đĩa và tồn tại qua các lần khởi động lại tiến trình. Một công việc chỉ trong phiên tồn tại trong Memory và mất khi agent thoát. Các công việc một lần (`recurring: False`) kích hoạt một lần rồi tự xóa.

**Bước 2.** Tạo lịch trình thông qua một lần gọi Tool. Phương thức lưu trữ bản ghi và trả về nó để mô hình có thể xác nhận những gì đã được lên lịch.

```python
def create(self, cron_expr: str, prompt: str, recurring: bool = True):
    job = {
        "id": new_id(),
        "cron": cron_expr,
        "prompt": prompt,
        "recurring": recurring,
        "created_at": time.time(),
        "last_fired_at": None,
    }
    self.jobs.append(job)
    return job
```

**Bước 3.** Chạy một vòng lặp bộ kiểm tra nền thức dậy mỗi 60 giây và kiểm tra từng lịch trình với thời gian hiện tại.

```python
def check_loop(self):
    while True:
        now = datetime.now()
        self.check_jobs(now)
        time.sleep(60)
```

**Bước 4.** Khi một lịch trình khớp, đưa thông báo vào hàng đợi. Trường `last_fired_at` được cập nhật để ngăn cùng một phút kích hoạt công việc hai lần.

```python
def check_jobs(self, now):
    for job in self.jobs:
        if cron_matches(job["cron"], now):
            self.queue.put({
                "type": "scheduled_prompt",
                "schedule_id": job["id"],
                "prompt": job["prompt"],
            })
            job["last_fired_at"] = now.timestamp()
```

**Bước 5.** Đưa các thông báo đã lên lịch trở lại vòng lặp chính sử dụng cùng mẫu drain từ s13. Từ góc độ của agent, một prompt đã lên lịch trông giống hệt một tin nhắn từ người dùng.

```python
notifications = scheduler.drain()
for item in notifications:
    messages.append({
        "role": "user",
        "content": f"[scheduled:{item['schedule_id']}] {item['prompt']}",
    })
```

## Đọc Cùng Nhau

- Nếu `schedule`, `task`, và `runtime task` vẫn cảm thấy như cùng một đối tượng, hãy đọc lại [`s13a-runtime-task-model.md`](./s13a-runtime-task-model.md) -- nó vẽ ranh giới giữa các bản ghi lập kế hoạch, bản ghi thực thi, và bản ghi lịch trình.
- Nếu bạn muốn thấy cách một kích hoạt cuối cùng trở về luồng chính, hãy ghép chương này với [`s00b-one-request-lifecycle.md`](./s00b-one-request-lifecycle.md).
- Nếu các kích hoạt tương lai bắt đầu cảm thấy như một hệ thống thực thi thứ hai hoàn toàn, hãy đặt lại với [`data-structures.md`](./data-structures.md) và tách các bản ghi lịch trình khỏi các bản ghi Runtime.

## Những Gì Đã Thay Đổi

| Cơ chế | Câu hỏi chính |
|---|---|
| Tác vụ nền (s13) | "Công việc chậm tiếp tục mà không bị chặn như thế nào?" |
| Lập lịch trình (s14) | "Khi nào công việc tương lai nên bắt đầu?" |

| Thành phần | Trước (s13) | Sau (s14) |
|---|---|---|
| Tool | 6 (base + background) | 8 (+ schedule_create, schedule_list, schedule_delete) |
| Nhận thức thời gian | Không có | Kích hoạt tương lai dựa trên cron |
| Bền vững | Tác vụ nền trong Memory | Lịch trình bền vững tồn tại qua khởi động lại |
| Mô hình kích hoạt | Chỉ do người dùng khởi tạo | Do người dùng + dựa trên thời gian |

## Thử Ngay

```sh
cd learn-claude-code
python agents/s14_cron_scheduler.py
```

1. Tạo lịch trình lặp lại: `Schedule "echo hello" to run every 2 minutes`
2. Tạo lời nhắc một lần: `Remind me in 1 minute to check the build`
3. Tạo theo dõi trì hoãn: `In 5 minutes, run the test suite and report results`

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Định nghĩa các bản ghi lịch trình lưu trữ ý định tương lai dưới dạng dữ liệu bền vững
- Chạy một bộ kiểm tra thời gian nền khớp các biểu thức cron với đồng hồ hiện tại
- Phân biệt các công việc bền vững (lưu trữ trên đĩa) với các công việc chỉ trong phiên (trong Memory)
- Đưa các kích hoạt đã lên lịch trở lại vòng lặp chính thông qua cùng hàng đợi thông báo được sử dụng bởi các tác vụ nền
- Ngăn kích hoạt kép với theo dõi `last_fired_at`

## Stage 3 Hoàn Thành

Bạn đã hoàn thành Stage 3: tầng thực thi và lập lịch trình. Nhìn lại ba chương cùng nhau:

- **s12** đã cung cấp cho agent một đồ thị tác vụ với các phụ thuộc và sự bền vững -- nó có thể lập kế hoạch công việc có cấu trúc tồn tại qua các lần khởi động lại.
- **s13** đã thêm thực thi nền -- công việc chậm chạy song song thay vì chặn vòng lặp.
- **s14** đã thêm kích hoạt dựa trên thời gian -- agent có thể lên lịch công việc tương lai mà người dùng không cần phải nhớ.

Ba chương này cùng nhau biến agent từ một thứ chỉ phản ứng với những gì bạn gõ ngay bây giờ thành một thứ có thể lập kế hoạch trước, làm việc song song, và hành động theo lịch trình của riêng mình. Trong Stage 4 (s15-s18), bạn sẽ sử dụng nền tảng này để điều phối nhiều agent làm việc như một nhóm.

## Điểm Mấu Chốt

> Một bộ lập lịch trình lưu trữ ý định tương lai dưới dạng bản ghi, kiểm tra nó với đồng hồ trong một vòng lặp nền, và đưa công việc được kích hoạt trở lại cùng hệ thống agent -- không cần vòng lặp thứ hai.
