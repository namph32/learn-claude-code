# s12: Hệ thống Tác vụ

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 > [ s12 ] > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Cách nâng cấp một danh sách kiểm tra phẳng thành một đồ thị tác vụ với các phụ thuộc tường minh
- Cách các cạnh `blockedBy` và `blocks` biểu diễn thứ tự thực hiện và tính song song
- Cách các chuyển đổi trạng thái (`pending` -> `in_progress` -> `completed`) tự động gỡ chặn các tác vụ
- Cách lưu tác vụ xuống đĩa giúp chúng tồn tại qua quá trình nén và khởi động lại

Ở s03, bạn đã cung cấp cho agent một công cụ TodoWrite -- một danh sách kiểm tra phẳng theo dõi những gì đã hoàn thành và chưa hoàn thành. Cách đó hoạt động tốt cho một phiên làm việc tập trung. Nhưng công việc thực tế có cấu trúc. Tác vụ B phụ thuộc vào tác vụ A. Tác vụ C và D có thể chạy song song. Tác vụ E chờ cả C lẫn D. Một danh sách phẳng không thể biểu diễn bất kỳ điều nào trong số đó. Và vì danh sách kiểm tra chỉ tồn tại trong Memory, quá trình nén Context (s06) sẽ xóa sạch nó. Trong chương này, bạn sẽ thay thế danh sách kiểm tra bằng một đồ thị tác vụ thực sự -- hiểu được các phụ thuộc, lưu trữ trên đĩa, và trở thành xương sống điều phối cho mọi thứ tiếp theo.

## Vấn Đề

Hãy tưởng tượng bạn yêu cầu agent tái cấu trúc một codebase: phân tích AST, biến đổi các nút, xuất code mới, và chạy kiểm thử. Bước phân tích phải hoàn thành trước khi biến đổi và xuất có thể bắt đầu. Biến đổi và xuất có thể chạy song song. Kiểm thử phải chờ cả hai. Với TodoWrite phẳng của s03, agent không có cách nào biểu diễn các mối quan hệ này. Nó có thể thực hiện bước biến đổi trước khi phân tích hoàn tất, hoặc chạy kiểm thử trước khi mọi thứ sẵn sàng. Không có thứ tự, không theo dõi phụ thuộc, và không có trạng thái nào ngoài "đã xong hay chưa." Tệ hơn nữa, nếu cửa sổ Context đầy và quá trình nén bắt đầu, toàn bộ kế hoạch sẽ biến mất.

## Giải Pháp

Nâng cấp danh sách kiểm tra thành một đồ thị tác vụ được lưu trữ trên đĩa. Mỗi tác vụ là một tệp JSON với trạng thái, các phụ thuộc (`blockedBy`), và các tác vụ phụ thuộc vào nó (`blocks`). Đồ thị trả lời ba câu hỏi tại bất kỳ thời điểm nào: cái gì đang sẵn sàng, cái gì đang bị chặn, và cái gì đã hoàn thành.

```
.tasks/
  task_1.json  {"id":1, "status":"completed"}
  task_2.json  {"id":2, "blockedBy":[1], "status":"pending"}
  task_3.json  {"id":3, "blockedBy":[1], "status":"pending"}
  task_4.json  {"id":4, "blockedBy":[2,3], "status":"pending"}

Task graph (DAG):
                 +----------+
            +--> | task 2   | --+
            |    | pending  |   |
+----------+     +----------+    +--> +----------+
| task 1   |                          | task 4   |
| completed| --> +----------+    +--> | blocked  |
+----------+     | task 3   | --+     +----------+
                 | pending  |
                 +----------+

Ordering:     task 1 must finish before 2 and 3
Parallelism:  tasks 2 and 3 can run at the same time
Dependencies: task 4 waits for both 2 and 3
Status:       pending -> in_progress -> completed
```

Cấu trúc trên là một DAG -- đồ thị có hướng không có chu trình, nghĩa là các tác vụ tiến về phía trước và không bao giờ quay lại. Đồ thị tác vụ này trở thành xương sống điều phối cho các chương sau: thực thi nền (s13), nhóm agent (s15+), và cô lập Worktree (s18) đều được xây dựng trên cùng cấu trúc tác vụ bền vững này.

## Cách Thức Hoạt Động

**Bước 1.** Tạo một `TaskManager` lưu trữ một tệp JSON cho mỗi tác vụ, với các thao tác CRUD và một đồ thị phụ thuộc.

```python
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)
        self._next_id = self._max_id() + 1

    def create(self, subject, description=""):
        task = {"id": self._next_id, "subject": subject,
                "status": "pending", "blockedBy": [],
                "blocks": [], "owner": ""}
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)
```

**Bước 2.** Triển khai giải quyết phụ thuộc. Khi một tác vụ hoàn thành, xóa ID của nó khỏi danh sách `blockedBy` của mọi tác vụ khác, tự động gỡ chặn các tác vụ phụ thuộc.

```python
def _clear_dependency(self, completed_id):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

**Bước 3.** Kết nối các chuyển đổi trạng thái và các cạnh phụ thuộc trong phương thức `update`. Khi trạng thái của một tác vụ thay đổi thành `completed`, logic xóa phụ thuộc từ Bước 2 sẽ tự động kích hoạt.

```python
def update(self, task_id, status=None,
           add_blocked_by=None, add_blocks=None):
    task = self._load(task_id)
    if status:
        task["status"] = status
        if status == "completed":
            self._clear_dependency(task_id)
    self._save(task)
```

**Bước 4.** Đăng ký bốn công cụ tác vụ trong Dispatch Map, cung cấp cho agent toàn quyền kiểm soát việc tạo, cập nhật, liệt kê và kiểm tra tác vụ.

```python
TOOL_HANDLERS = {
    # ...base tools...
    "task_create": lambda **kw: TASKS.create(kw["subject"]),
    "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status")),
    "task_list":   lambda **kw: TASKS.list_all(),
    "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
}
```

Từ s12 trở đi, đồ thị tác vụ trở thành mặc định cho công việc đa bước bền vững. Todo của s03 vẫn hữu ích cho các danh sách kiểm tra nhanh trong một phiên, nhưng bất cứ điều gì cần thứ tự, tính song song, hoặc sự bền vững đều thuộc về đây.

## Đọc Cùng Nhau

- Nếu bạn đến thẳng từ s03, hãy xem lại [`data-structures.md`](./data-structures.md) để phân tách `TodoItem` / `PlanState` khỏi `TaskRecord` -- chúng trông giống nhau nhưng phục vụ các mục đích khác nhau.
- Nếu ranh giới đối tượng bắt đầu mờ đi, hãy đặt lại với [`entity-map.md`](./entity-map.md) trước khi bạn trộn lẫn tin nhắn, tác vụ, runtime task, và thành viên nhóm vào một tầng.
- Nếu bạn có kế hoạch tiếp tục vào s13, hãy giữ [`s13a-runtime-task-model.md`](./s13a-runtime-task-model.md) bên cạnh chương này vì tác vụ bền vững và runtime task là cặp dễ nhầm lẫn nhất tiếp theo.

## Những Gì Đã Thay Đổi

| Thành phần | Trước (s06) | Sau (s12) |
|---|---|---|
| Tool | 5 | 8 (`task_create/update/list/get`) |
| Mô hình lập kế hoạch | Danh sách phẳng (trong Memory) | Đồ thị tác vụ với phụ thuộc (trên đĩa) |
| Mối quan hệ | Không có | Các cạnh `blockedBy` + `blocks` |
| Theo dõi trạng thái | Xong hay chưa | `pending` -> `in_progress` -> `completed` |
| Bền vững | Mất khi nén | Tồn tại qua nén và khởi động lại |

## Thử Ngay

```sh
cd learn-claude-code
python agents/s12_task_system.py
```

1. `Create 3 tasks: "Setup project", "Write code", "Write tests". Make them depend on each other in order.`
2. `List all tasks and show the dependency graph`
3. `Complete task 1 and then list tasks to see task 2 unblocked`
4. `Create a task board for refactoring: parse -> transform -> emit -> test, where transform and emit can run in parallel after parse`

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Xây dựng một đồ thị tác vụ dựa trên tệp, trong đó mỗi tác vụ là một bản ghi JSON độc lập
- Biểu diễn thứ tự và tính song song thông qua các cạnh phụ thuộc `blockedBy` và `blocks`
- Triển khai tự động gỡ chặn khi các tác vụ thượng nguồn hoàn thành
- Lưu trữ trạng thái lập kế hoạch để nó tồn tại qua quá trình nén Context và khởi động lại tiến trình

## Tiếp Theo Là Gì

Các tác vụ giờ đây có cấu trúc và tồn tại trên đĩa. Nhưng mỗi lần gọi Tool vẫn chặn vòng lặp chính -- nếu một tác vụ liên quan đến một subprocess chậm như `npm install` hay `pytest`, agent ngồi chờ không làm gì. Trong s13, bạn sẽ thêm khả năng thực thi nền để công việc chậm chạy song song trong khi agent tiếp tục suy nghĩ.

## Điểm Mấu Chốt

> Một đồ thị tác vụ với các phụ thuộc tường minh biến một danh sách kiểm tra phẳng thành một cấu trúc điều phối biết cái gì đang sẵn sàng, cái gì đang bị chặn, và cái gì có thể chạy song song.
