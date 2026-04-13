# s18: Worktree + Cô Lập Tác Vụ

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 > s12 > s13 > s14 > s15 > s16 > s17 > [ s18 ] > s19`

## Những Gì Bạn Sẽ Học
- Cách git Worktree (các bản sao cô lập của thư mục dự án, được quản lý bởi git) ngăn chặn xung đột file giữa các Agent song song
- Cách liên kết một tác vụ với một Worktree chuyên dụng để "làm gì" và "làm ở đâu" được tách biệt gọn gàng
- Cách các sự kiện vòng đời cho bạn một bản ghi có thể quan sát của mọi hành động tạo, giữ và xóa
- Cách các lane thực thi song song cho phép nhiều Agent làm việc trên các tác vụ khác nhau mà không bao giờ giẫm chân lên file của nhau

Khi hai Agent đều cần chỉnh sửa cùng một codebase cùng một lúc, bạn có vấn đề. Tất cả những gì bạn đã xây dựng cho đến nay -- bảng tác vụ, các Agent tự chủ, giao thức nhóm -- đều giả định rằng các Agent làm việc trong một thư mục dùng chung. Điều đó hoạt động tốt cho đến khi nó không. Chương này cung cấp cho mỗi tác vụ thư mục riêng của nó, để công việc song song luôn song song.

## Vấn Đề

Đến s17, các Agent của bạn có thể nhận tác vụ, phối hợp qua giao thức nhóm và hoàn thành công việc một cách tự chủ. Nhưng tất cả chúng chạy trong cùng thư mục dự án. Hãy tưởng tượng Agent A đang tái cấu trúc module xác thực, và Agent B đang xây dựng trang đăng nhập mới. Cả hai cần chạm vào `config.py`. Agent A stage các thay đổi của mình, Agent B stage các thay đổi khác vào cùng file, và bây giờ bạn có một đống lộn xộn các chỉnh sửa chưa stage mà không Agent nào có thể rollback gọn gàng.

Bảng tác vụ theo dõi *làm gì* nhưng không có ý kiến về *làm ở đâu*. Bạn cần một cách để cung cấp cho mỗi tác vụ thư mục làm việc cô lập riêng của nó, để các thao tác cấp file không bao giờ xung đột. Giải pháp rất đơn giản: ghép mỗi tác vụ với một git Worktree -- một checkout riêng biệt của cùng một kho trên nhánh riêng của nó. Tác vụ quản lý mục tiêu; Worktree quản lý Context thực thi. Liên kết chúng bằng ID tác vụ.

## Đọc Cùng Nhau

- Nếu tác vụ, Runtime slot và lane Worktree đang bị nhòe lẫn trong đầu bạn, [`team-task-lane-model.md`](./team-task-lane-model.md) phân tách chúng rõ ràng.
- Nếu bạn muốn xác nhận các trường nào thuộc về bản ghi tác vụ so với bản ghi Worktree, [`data-structures.md`](./data-structures.md) có Schema đầy đủ.
- Nếu bạn muốn xem tại sao chương này đến sau các tác vụ và nhóm trong chương trình giảng dạy tổng thể, [`s00e-reference-module-map.md`](./s00e-reference-module-map.md) có lý do sắp xếp thứ tự.

## Giải Pháp

Hệ thống chia thành hai mặt phẳng: một mặt phẳng điều khiển (`.tasks/`) theo dõi các mục tiêu, và một mặt phẳng thực thi (`.worktrees/`) quản lý các thư mục cô lập. Mỗi tác vụ trỏ đến Worktree của nó theo tên, và mỗi Worktree trỏ lại tác vụ của nó theo ID.

```
Mặt phẳng điều khiển (.tasks/)         Mặt phẳng thực thi (.worktrees/)
+------------------+                +------------------------+
| task_1.json      |                | auth-refactor/         |
|   status: in_progress  <------>   branch: wt/auth-refactor
|   worktree: "auth-refactor"   |   task_id: 1             |
+------------------+                +------------------------+
| task_2.json      |                | ui-login/              |
|   status: pending    <------>     branch: wt/ui-login
|   worktree: "ui-login"       |   task_id: 2             |
+------------------+                +------------------------+
                                    |
                          index.json (registry Worktree)
                          events.jsonl (log vòng đời)

Máy trạng thái:
  Tác vụ:    pending -> in_progress -> completed
  Worktree:  absent  -> active      -> removed | kept
```

## Cách Hoạt Động

**Bước 1.** Tạo một tác vụ. Mục tiêu được ghi lại trước, trước khi bất kỳ thư mục nào tồn tại.

```python
TASKS.create("Implement auth refactor")
# -> .tasks/task_1.json  status=pending  worktree=""
```

**Bước 2.** Tạo một Worktree và liên kết nó với tác vụ. Truyền `task_id` tự động chuyển tác vụ sang `in_progress` -- bạn không cần cập nhật trạng thái riêng biệt.

```python
WORKTREES.create("auth-refactor", task_id=1)
# -> git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
# -> index.json gets new entry, task_1.json gets worktree="auth-refactor"
```

Việc liên kết ghi trạng thái vào cả hai phía để bạn có thể duyệt qua mối quan hệ từ cả hai hướng:

```python
def bind_worktree(self, task_id, worktree):
    task = self._load(task_id)
    task["worktree"] = worktree
    if task["status"] == "pending":
        task["status"] = "in_progress"
    self._save(task)
```

**Bước 3.** Chạy các lệnh trong Worktree. Chi tiết quan trọng: `cwd` trỏ đến thư mục cô lập, không phải thư mục gốc dự án chính của bạn. Mọi thao tác file xảy ra trong một sandbox không thể xung đột với các Worktree khác.

```python
subprocess.run(command, shell=True, cwd=worktree_path,
               capture_output=True, text=True, timeout=300)
```

**Bước 4.** Đóng Worktree. Bạn có hai lựa chọn, tùy thuộc vào việc công việc đã hoàn thành chưa:

- `worktree_keep(name)` -- giữ nguyên thư mục để dùng sau (hữu ích khi tác vụ bị tạm dừng hoặc cần xem xét).
- `worktree_remove(name, complete_task=True)` -- xóa thư mục, đánh dấu tác vụ liên kết là hoàn thành và phát ra sự kiện. Một lần gọi xử lý cả việc dọn dẹp và hoàn thành cùng nhau.

```python
def remove(self, name, force=False, complete_task=False):
    self._run_git(["worktree", "remove", wt["path"]])
    if complete_task and wt.get("task_id") is not None:
        self.tasks.update(wt["task_id"], status="completed")
        self.tasks.unbind_worktree(wt["task_id"])
        self.events.emit("task.completed", ...)
```

**Bước 5.** Quan sát luồng sự kiện. Mỗi bước vòng đời phát ra một sự kiện có cấu trúc vào `.worktrees/events.jsonl`, cung cấp cho bạn toàn bộ audit trail về những gì đã xảy ra và khi nào:

```json
{
  "event": "worktree.remove.after",
  "task": {"id": 1, "status": "completed"},
  "worktree": {"name": "auth-refactor", "status": "removed"},
  "ts": 1730000000
}
```

Các sự kiện được phát ra: `worktree.create.before/after/failed`, `worktree.remove.before/after/failed`, `worktree.keep`, `task.completed`.

Trong phiên bản giảng dạy, `.tasks/` cộng với `.worktrees/index.json` là đủ để tái tạo trạng thái mặt phẳng điều khiển hiển thị sau khi crash. Bài học quan trọng không phải là mọi trường hợp biên sản xuất. Bài học quan trọng là trạng thái mục tiêu và trạng thái lane thực thi đều phải luôn đọc được trên đĩa.

## Những Gì Thay Đổi Từ s17

| Thành phần           | Trước (s17)                  | Sau (s18)                                        |
|----------------------|------------------------------|--------------------------------------------------|
| Phối hợp             | Bảng tác vụ (chủ/trạng thái) | Bảng tác vụ + liên kết Worktree rõ ràng         |
| Phạm vi thực thi     | Thư mục dùng chung           | Thư mục cô lập theo phạm vi tác vụ              |
| Khả năng phục hồi    | Chỉ trạng thái tác vụ        | Trạng thái tác vụ + index Worktree              |
| Dọn dẹp              | Hoàn thành tác vụ            | Hoàn thành tác vụ + keep/remove rõ ràng         |
| Khả năng hiển thị vòng đời | Ẩn trong logs          | Sự kiện rõ ràng trong `.worktrees/events.jsonl` |

## Thử Nghiệm

```sh
cd learn-claude-code
python agents/s18_worktree_task_isolation.py
```

1. `Create tasks for backend auth and frontend login page, then list tasks.`
2. `Create worktree "auth-refactor" for task 1, then bind task 2 to a new worktree "ui-login".`
3. `Run "git status --short" in worktree "auth-refactor".`
4. `Keep worktree "ui-login", then list worktrees and inspect events.`
5. `Remove worktree "auth-refactor" with complete_task=true, then list tasks/worktrees/events.`

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Tạo các git Worktree cô lập để các Agent song song không bao giờ tạo ra xung đột file
- Liên kết tác vụ với Worktree bằng tham chiếu hai chiều (tác vụ trỏ đến tên Worktree, Worktree trỏ đến ID tác vụ)
- Chọn giữa giữ và xóa một Worktree khi đóng, với cập nhật trạng thái tác vụ tự động
- Đọc luồng sự kiện trong `events.jsonl` để hiểu toàn bộ vòng đời của mỗi Worktree

## Tiếp Theo Là Gì

Bây giờ bạn có các Agent có thể làm việc trong cô lập hoàn toàn, mỗi Agent trong thư mục riêng của mình với nhánh riêng. Nhưng mọi khả năng chúng sử dụng -- bash, read, write, edit -- đều được hard-code vào harness Python của bạn. Trong s19, bạn sẽ tìm hiểu cách các chương trình bên ngoài có thể cung cấp các khả năng mới thông qua MCP (Model Context Protocol), để Agent của bạn có thể phát triển mà không cần thay đổi code cốt lõi của nó.

## Điểm Mấu Chốt

> Tác vụ trả lời *công việc gì đang được thực hiện*; Worktree trả lời *công việc đó chạy ở đâu*; giữ chúng tách biệt làm cho các hệ thống song song dễ lý luận và phục hồi hơn nhiều.
