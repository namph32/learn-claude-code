# s15: Nhóm Agent

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 > s12 > s13 > s14 > [ s15 ] > s16 > s17 > s18 > s19`

## Những Gì Bạn Sẽ Học
- Cách các thành viên nhóm bền vững khác với các Subagent dùng một lần
- Cách các hộp thư đến dựa trên JSONL cung cấp cho Agent một kênh giao tiếp lâu dài
- Cách vòng đời nhóm chuyển qua các trạng thái: tạo ra, đang làm việc, nhàn rỗi và tắt
- Cách phối hợp dựa trên file cho phép nhiều Agent Loop chạy song song

Đôi khi một Agent là không đủ. Một dự án phức tạp -- ví dụ, xây dựng một tính năng bao gồm frontend, backend và các bài kiểm thử -- cần nhiều worker chạy song song, mỗi worker có danh tính và Memory riêng. Trong chương này bạn sẽ xây dựng một hệ thống nhóm nơi các Agent tồn tại vượt ra ngoài một lần prompt đơn lẻ, giao tiếp thông qua các hộp thư dựa trên file, và phối hợp mà không cần chia sẻ một luồng cuộc hội thoại duy nhất.

## Vấn Đề

Các Subagent từ s04 là dùng một lần: bạn tạo ra một cái, nó làm việc, trả về tóm tắt và biến mất. Nó không có danh tính và không có Memory giữa các lần gọi. Các tác vụ nền từ s13 có thể giữ công việc chạy trong nền, nhưng chúng không phải là các thành viên nhóm bền vững đưa ra quyết định được hướng dẫn bởi LLM.

Làm việc nhóm thực sự cần ba thứ: (1) các Agent bền vững tồn tại lâu hơn một lần prompt đơn lẻ, (2) quản lý danh tính và vòng đời để bạn biết ai đang làm gì, và (3) một kênh giao tiếp giữa các Agent để chúng có thể trao đổi thông tin mà không cần người dẫn đầu chuyển tiếp thủ công từng tin nhắn.

## Giải Pháp

Harness duy trì danh sách nhóm trong một file cấu hình dùng chung và cung cấp cho mỗi thành viên nhóm một hộp thư đến JSONL chỉ ghi thêm. Khi một Agent gửi tin nhắn cho Agent khác, nó chỉ đơn giản là ghi thêm một dòng JSON vào file hộp thư đến của người nhận. Người nhận đọc hết file đó trước mỗi lần gọi LLM.

```
Vòng đời thành viên nhóm:
  spawn -> WORKING -> IDLE -> WORKING -> ... -> SHUTDOWN

Giao tiếp:
  .team/
    config.json           <- danh sách nhóm + trạng thái
    inbox/
      alice.jsonl         <- chỉ ghi thêm, đọc hết khi đọc
      bob.jsonl
      lead.jsonl

              +--------+    send("alice","bob","...")    +--------+
              | alice  | -----------------------------> |  bob   |
              | loop   |    bob.jsonl << {json_line}    |  loop  |
              +--------+                                +--------+
                   ^                                         |
                   |        BUS.read_inbox("alice")          |
                   +---- alice.jsonl -> read + drain ---------+
```

## Cách Hoạt Động

**Bước 1.** `TeammateManager` duy trì `config.json` với danh sách nhóm. Nó theo dõi tên, vai trò và trạng thái hiện tại của mỗi thành viên nhóm.

```python
class TeammateManager:
    def __init__(self, team_dir: Path):
        self.dir = team_dir
        self.dir.mkdir(exist_ok=True)
        self.config_path = self.dir / "config.json"
        self.config = self._load_config()
        self.threads = {}
```

**Bước 2.** `spawn()` tạo một mục thành viên nhóm trong danh sách và khởi động Agent Loop của nó trong một luồng riêng biệt. Từ thời điểm này, thành viên nhóm chạy độc lập -- nó có lịch sử cuộc hội thoại riêng, các lần gọi Tool riêng và các tương tác LLM riêng.

```python
def spawn(self, name: str, role: str, prompt: str) -> str:
    member = {"name": name, "role": role, "status": "working"}
    self.config["members"].append(member)
    self._save_config()
    thread = threading.Thread(
        target=self._teammate_loop,
        args=(name, role, prompt), daemon=True)
    thread.start()
    return f"Spawned teammate '{name}' (role: {role})"
```

**Bước 3.** `MessageBus` cung cấp các hộp thư đến JSONL chỉ ghi thêm. `send()` ghi thêm một dòng JSON vào file của người nhận; `read_inbox()` đọc tất cả các tin nhắn đã tích lũy rồi làm trống file (đọc hết). Định dạng lưu trữ đơn giản có chủ đích -- trọng tâm giảng dạy ở đây là ranh giới hộp thư, không phải sự khéo léo về lưu trữ.

```python
class MessageBus:
    def send(self, sender, to, content, msg_type="message", extra=None):
        msg = {"type": msg_type, "from": sender,
               "content": content, "timestamp": time.time()}
        if extra:
            msg.update(extra)
        with open(self.dir / f"{to}.jsonl", "a") as f:
            f.write(json.dumps(msg) + "\n")

    def read_inbox(self, name):
        path = self.dir / f"{name}.jsonl"
        if not path.exists(): return "[]"
        msgs = [json.loads(l) for l in path.read_text().strip().splitlines() if l]
        path.write_text("")  # drain
        return json.dumps(msgs, indent=2)
```

**Bước 4.** Mỗi thành viên nhóm kiểm tra hộp thư đến của mình trước mỗi lần gọi LLM. Bất kỳ tin nhắn nhận được nào đều được đưa vào Context cuộc hội thoại để mô hình có thể thấy và phản hồi.

```python
def _teammate_loop(self, name, role, prompt):
    messages = [{"role": "user", "content": prompt}]
    for _ in range(50):
        inbox = BUS.read_inbox(name)
        if inbox != "[]":
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            messages.append({"role": "assistant",
                "content": "Noted inbox messages."})
        response = client.messages.create(...)
        if response.stop_reason != "tool_use":
            break
        # execute tools, append results...
    self._find_member(name)["status"] = "idle"
```

## Đọc Cùng Nhau

- Nếu bạn vẫn xem thành viên nhóm như Subagent dùng một lần của s04, hãy xem lại [`entity-map.md`](./entity-map.md) để thấy sự khác biệt.
- Nếu bạn dự định tiếp tục sang s16-s18, hãy giữ [`team-task-lane-model.md`](./team-task-lane-model.md) mở -- nó phân tách thành viên nhóm, yêu cầu giao thức, tác vụ, Runtime slot và lane Worktree thành các khái niệm riêng biệt.
- Nếu bạn không chắc thành viên nhóm tồn tại lâu dài khác với một Runtime slot đang hoạt động như thế nào, hãy kết hợp chương này với [`s13a-runtime-task-model.md`](./s13a-runtime-task-model.md).

## Cách Kết Nối Với Hệ Thống Trước Đó

Chương này không chỉ là "thêm các lần gọi mô hình." Nó thêm các executor bền vững lên trên các cấu trúc công việc bạn đã xây dựng trong s12-s14.

```text
lead xác định công việc cần một worker tồn tại lâu dài
  ->
tạo ra thành viên nhóm
  ->
ghi mục danh sách trong .team/config.json
  ->
gửi tin nhắn hộp thư đến / gợi ý tác vụ
  ->
thành viên nhóm đọc hết hộp thư đến trước vòng lặp tiếp theo
  ->
thành viên nhóm chạy Agent Loop và Tool riêng của mình
  ->
kết quả trả về qua tin nhắn nhóm hoặc cập nhật tác vụ
```

Giữ ranh giới rõ ràng:

- s12-s14 cung cấp cho bạn các tác vụ, Runtime slot và lịch trình
- s15 thêm các worker có tên bền vững
- s15 vẫn chủ yếu là công việc được lead giao
- các giao thức có cấu trúc xuất hiện trong s16
- việc nhận tác vụ tự chủ xuất hiện trong s17

## Thành Viên Nhóm vs Subagent vs Runtime Slot

| Cơ chế | Hãy nghĩ đó là | Vòng đời | Ranh giới chính |
|---|---|---|---|
| Subagent | một helper dùng một lần | spawn -> làm việc -> tóm tắt -> biến mất | cô lập một nhánh khám phá |
| Runtime slot | một slot thực thi đang hoạt động | tồn tại khi công việc nền đang chạy | theo dõi việc thực thi dài hạn, không phải danh tính |
| thành viên nhóm | một worker bền vững | có thể nhàn rỗi, tiếp tục và tiếp tục nhận việc | có tên, hộp thư đến và vòng lặp độc lập |

## Những Gì Thay Đổi Từ s14

| Thành phần     | Trước (s14)      | Sau (s15)                  |
|----------------|------------------|----------------------------|
| Tool           | 6                | 9 (+spawn/send/read_inbox) |
| Agent          | Đơn lẻ          | Lead + N thành viên nhóm   |
| Tính bền vững  | Không có         | config.json + hộp thư đến JSONL |
| Luồng          | Lệnh nền        | Toàn bộ Agent Loop mỗi luồng |
| Vòng đời       | Fire-and-forget  | idle -> working -> idle    |
| Giao tiếp      | Không có         | message + broadcast        |

## Thử Nghiệm

```sh
cd learn-claude-code
python agents/s15_agent_teams.py
```

1. `Spawn alice (coder) and bob (tester). Have alice send bob a message.`
2. `Broadcast "status update: phase 1 complete" to all teammates`
3. `Check the lead inbox for any messages`
4. Gõ `/team` để xem danh sách nhóm với các trạng thái
5. Gõ `/inbox` để kiểm tra thủ công hộp thư đến của lead

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Tạo ra các thành viên nhóm bền vững, mỗi người chạy Agent Loop độc lập riêng của họ
- Gửi tin nhắn giữa các Agent thông qua các hộp thư đến JSONL lâu dài
- Theo dõi trạng thái thành viên nhóm thông qua file cấu hình dùng chung
- Phối hợp nhiều Agent mà không cần dồn tất cả vào một cuộc hội thoại duy nhất

## Tiếp Theo Là Gì

Các thành viên nhóm của bạn hiện có thể giao tiếp tự do, nhưng chúng thiếu các quy tắc phối hợp. Điều gì xảy ra khi bạn cần tắt một thành viên nhóm gọn gàng, hoặc xem xét một kế hoạch rủi ro trước khi nó thực thi? Trong s16, bạn sẽ thêm các giao thức có cấu trúc -- các handshake yêu cầu-phản hồi mang lại trật tự cho việc đàm phán đa Agent.

## Điểm Mấu Chốt

> Các thành viên nhóm tồn tại vượt ra ngoài một lần prompt, mỗi người có danh tính, vòng đời và hộp thư lâu dài -- phối hợp không còn bị giới hạn trong một vòng lặp cha duy nhất.
