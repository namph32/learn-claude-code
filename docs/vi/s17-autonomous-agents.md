# s17: Agent Tự Chủ

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 > s12 > s13 > s14 > s15 > s16 > [ s17 ] > s18 > s19`

## Những Gì Bạn Sẽ Học
- Cách polling nhàn rỗi cho phép thành viên nhóm tìm việc mới mà không cần được báo
- Cách auto-claim biến bảng tác vụ thành hàng đợi công việc tự phục vụ
- Cách tái nạp danh tính khôi phục ý thức về bản thân của thành viên nhóm sau khi nén Context
- Cách tắt dựa trên timeout ngăn các Agent nhàn rỗi chạy mãi mãi

Việc giao tác vụ thủ công không thể mở rộng. Với mười tác vụ chưa được giao trên bảng, lead phải chọn một cái, tìm một thành viên nhóm nhàn rỗi, soạn prompt và chuyển giao -- mười lần. Lead trở thành nút thắt cổ chai, dành nhiều thời gian để điều phối hơn là suy nghĩ. Trong chương này bạn sẽ loại bỏ nút thắt cổ chai đó bằng cách làm cho các thành viên nhóm tự chủ: chúng tự quét bảng tác vụ, nhận các công việc chưa được giao, và tắt gọn gàng khi không còn gì để làm.

## Vấn Đề

Trong s15-s16, các thành viên nhóm chỉ làm việc khi được yêu cầu rõ ràng. Lead phải tạo ra mỗi người với một prompt cụ thể. Nếu mười tác vụ chưa được giao trên bảng, lead giao từng cái thủ công. Điều này tạo ra một nút thắt cổ chai phối hợp ngày càng tệ hơn khi nhóm phát triển.

Tính tự chủ thực sự có nghĩa là các thành viên nhóm tự quét bảng tác vụ, nhận các tác vụ chưa được giao, làm việc với chúng, rồi tìm thêm -- tất cả mà không cần lead can thiệp.

Một sự tinh tế làm điều này khó hơn nghe: sau khi nén Context (mà bạn đã xây dựng trong s06), lịch sử cuộc hội thoại của Agent bị cắt ngắn. Agent có thể quên mình là ai. Tái nạp danh tính sửa điều này bằng cách khôi phục tên và vai trò của Agent khi Context của nó quá ngắn.

## Giải Pháp

Mỗi thành viên nhóm luân phiên giữa hai giai đoạn: WORK (gọi LLM và thực thi Tool) và IDLE (polling tin nhắn mới hoặc tác vụ chưa được giao). Nếu giai đoạn nhàn rỗi hết thời gian mà không có gì để làm, thành viên nhóm tự tắt.

```
Vòng đời thành viên nhóm với chu kỳ nhàn rỗi:

+-------+
| spawn |
+---+---+
    |
    v
+-------+   tool_use     +-------+
| WORK  | <------------- |  LLM  |
+---+---+                +-------+
    |
    | stop_reason != tool_use (hoặc gọi idle tool)
    v
+--------+
|  IDLE  |  polling mỗi 5 giây tối đa 60 giây
+---+----+
    |
    +---> kiểm tra inbox --> có tin nhắn? -------> WORK
    |
    +---> quét .tasks/ --> có unclaimed? --------> claim -> WORK
    |
    +---> timeout 60 giây --------------------- > SHUTDOWN

Tái nạp danh tính sau khi nén:
  if len(messages) <= 3:
    messages.insert(0, identity_block)
```

## Cách Hoạt Động

**Bước 1.** Vòng lặp thành viên nhóm có hai giai đoạn: WORK và IDLE. Trong giai đoạn làm việc, thành viên nhóm gọi LLM lặp đi lặp lại và thực thi các Tool. Khi LLM ngừng gọi Tool (hoặc thành viên nhóm gọi rõ ràng Tool `idle`), nó chuyển sang giai đoạn nhàn rỗi.

```python
def _loop(self, name, role, prompt):
    while True:
        # -- GIAI ĐOẠN WORK --
        messages = [{"role": "user", "content": prompt}]
        for _ in range(50):
            response = client.messages.create(...)
            if response.stop_reason != "tool_use":
                break
            # execute tools...
            if idle_requested:
                break

        # -- GIAI ĐOẠN IDLE --
        self._set_status(name, "idle")
        resume = self._idle_poll(name, messages)
        if not resume:
            self._set_status(name, "shutdown")
            return
        self._set_status(name, "working")
```

**Bước 2.** Giai đoạn nhàn rỗi polling hai thứ trong một vòng lặp: tin nhắn hộp thư đến và các tác vụ chưa được giao. Nó kiểm tra mỗi 5 giây tối đa 60 giây. Nếu một tin nhắn đến, thành viên nhóm thức dậy. Nếu một tác vụ chưa được giao xuất hiện trên bảng, thành viên nhóm nhận nó và quay lại làm việc. Nếu cả hai không xảy ra trong cửa sổ timeout, thành viên nhóm tự tắt.

```python
def _idle_poll(self, name, messages):
    for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):  # 60s / 5s = 12
        time.sleep(POLL_INTERVAL)
        inbox = BUS.read_inbox(name)
        if inbox:
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            return True
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            claim_task(unclaimed[0]["id"], name)
            messages.append({"role": "user",
                "content": f"<auto-claimed>Task #{unclaimed[0]['id']}: "
                           f"{unclaimed[0]['subject']}</auto-claimed>"})
            return True
    return False  # timeout -> shutdown
```

**Bước 3.** Quét bảng tác vụ tìm các tác vụ pending, chưa có chủ và chưa bị chặn. Lần quét đọc các file tác vụ từ đĩa và lọc các tác vụ có thể nhận -- không có chủ, không có phụ thuộc chặn và vẫn ở trạng thái `pending`.

```python
def scan_unclaimed_tasks() -> list:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            unclaimed.append(task)
    return unclaimed
```

**Bước 4.** Tái nạp danh tính xử lý một vấn đề tinh tế. Sau khi nén Context (s06), lịch sử cuộc hội thoại có thể thu nhỏ xuống chỉ còn vài tin nhắn -- và Agent quên mình là ai. Khi danh sách tin nhắn quá ngắn một cách đáng ngờ (3 tin nhắn hoặc ít hơn), harness chèn một khối danh tính vào đầu để Agent biết tên, vai trò và nhóm của mình.

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}, "
                   f"team: {team_name}. Continue your work.</identity>"})
    messages.insert(1, {"role": "assistant",
        "content": f"I am {name}. Continuing."})
```

## Đọc Cùng Nhau

- Nếu thành viên nhóm, tác vụ và Runtime slot đang bắt đầu bị nhòe thành một tầng, hãy xem lại [`team-task-lane-model.md`](./team-task-lane-model.md) để phân tách chúng rõ ràng.
- Nếu auto-claim khiến bạn tự hỏi slot thực thi đang hoạt động thực sự nằm ở đâu, hãy giữ [`s13a-runtime-task-model.md`](./s13a-runtime-task-model.md) gần đó.
- Nếu bạn bắt đầu quên sự khác biệt cốt lõi giữa một thành viên nhóm bền vững và một Subagent một lần, hãy xem lại [`entity-map.md`](./entity-map.md).

## Những Gì Thay Đổi Từ s16

| Thành phần     | Trước (s16)      | Sau (s17)                  |
|----------------|------------------|----------------------------|
| Tool           | 12               | 14 (+idle, +claim_task)    |
| Tính tự chủ    | Do lead điều hướng | Tự tổ chức               |
| Giai đoạn nhàn rỗi | Không có     | Poll inbox + bảng tác vụ  |
| Nhận tác vụ    | Chỉ thủ công    | Auto-claim tác vụ chưa giao |
| Danh tính      | System Prompt    | + tái nạp sau khi nén      |
| Timeout        | Không có         | 60 giây nhàn rỗi -> tự tắt |

## Thử Nghiệm

```sh
cd learn-claude-code
python agents/s17_autonomous_agents.py
```

1. `Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim.`
2. `Spawn a coder teammate and let it find work from the task board itself`
3. `Create tasks with dependencies. Watch teammates respect the blocked order.`
4. Gõ `/tasks` để xem bảng tác vụ với chủ sở hữu
5. Gõ `/team` để theo dõi ai đang làm việc so với nhàn rỗi

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Xây dựng các thành viên nhóm tự tìm và nhận công việc từ bảng tác vụ dùng chung mà không cần lead can thiệp
- Triển khai vòng lặp polling nhàn rỗi cân bằng giữa khả năng phản hồi và hiệu quả tài nguyên
- Khôi phục danh tính Agent sau khi nén Context để các thành viên nhóm chạy dài hạn vẫn nhất quán
- Sử dụng tắt dựa trên timeout để ngăn các Agent bị bỏ rơi chạy vô thời hạn

## Tiếp Theo Là Gì

Các thành viên nhóm của bạn bây giờ tự tổ chức, nhưng tất cả chúng chia sẻ cùng một thư mục làm việc. Khi hai Agent cùng chỉnh sửa một file tại cùng một thời điểm, mọi thứ sẽ hỏng. Trong s18, bạn sẽ cung cấp cho mỗi thành viên nhóm Worktree cô lập riêng -- một bản sao riêng của codebase nơi nó có thể làm việc mà không giẫm chân lên các thay đổi của bất kỳ ai khác.

## Điểm Mấu Chốt

> Các Agent tự chủ quét bảng tác vụ, nhận các công việc chưa được giao, và tắt khi nhàn rỗi -- loại bỏ lead như một nút thắt cổ chai phối hợp.
