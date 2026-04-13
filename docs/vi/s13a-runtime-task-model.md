# s13a: Mô Hình Runtime Task

> **Deep Dive** -- Nên đọc giữa s12 và s13. Nó ngăn chặn sự nhầm lẫn phổ biến nhất trong Stage 3.

### Khi Nào Nên Đọc Tài Liệu Này

Ngay sau s12 (Hệ thống Tác vụ), trước khi bạn bắt đầu s13 (Tác vụ Nền). Ghi chú này phân tách hai nghĩa của "tác vụ" mà người mới thường gộp thành một.

---

> Ghi chú cầu nối này giải quyết một sự nhầm lẫn nhanh chóng trở nên tốn kém:
>
> **tác vụ trong Work-graph không phải là điều tương tự với tác vụ đang chạy hiện tại**

## Cách Đọc Tài Liệu Này Cùng Với Luồng Chính

Ghi chú này hoạt động tốt nhất giữa các tài liệu sau:

- đọc [`s12-task-system.md`](./s12-task-system.md) trước để cố định Work-graph bền vững
- sau đó đọc [`s13-background-tasks.md`](./s13-background-tasks.md) để thấy thực thi nền
- nếu các thuật ngữ bắt đầu mờ đi, bạn có thể thấy hữu ích khi xem lại [`glossary.md`](./glossary.md)
- nếu bạn muốn các trường khớp chính xác, bạn có thể thấy hữu ích khi xem lại [`data-structures.md`](./data-structures.md) và [`entity-map.md`](./entity-map.md)

## Tại Sao Điều Này Cần Một Ghi Chú Cầu Nối Riêng

Luồng chính vẫn đúng:

- `s12` dạy hệ thống tác vụ
- `s13` dạy các tác vụ nền

Nhưng nếu không có thêm một tầng cầu nối, bạn có thể dễ dàng bắt đầu gộp hai nghĩa khác nhau của "tác vụ" vào cùng một nhóm.

Ví dụ:

- một tác vụ Work-graph như "triển khai module xác thực"
- một thực thi nền như "chạy pytest"
- một thực thi của thành viên nhóm như "alice đang chỉnh sửa tệp"

Cả ba đều có thể được gọi là tác vụ một cách thông thường, nhưng chúng không tồn tại ở cùng một tầng.

## Hai Loại Tác Vụ Rất Khác Nhau

### 1. Tác vụ Work-graph

Đây là nút bền vững được giới thiệu trong `s12`.

Nó trả lời:

- cần làm gì
- công việc nào phụ thuộc vào công việc nào khác
- ai sở hữu nó
- trạng thái tiến độ là gì

Hiểu tốt nhất là:

> một đơn vị công việc đã lên kế hoạch bền vững

### 2. Runtime task

Tầng này trả lời:

- đơn vị thực thi nào đang hoạt động ngay bây giờ
- loại thực thi là gì
- liệu nó đang chạy, đã hoàn thành, thất bại, hay bị dừng
- kết quả của nó nằm ở đâu

Hiểu tốt nhất là:

> một slot thực thi trực tiếp bên trong Runtime

## Mô Hình Tư Duy Tối Thiểu

Hãy xem chúng như hai bảng riêng biệt:

```text
work-graph task
  - bền vững
  - hướng mục tiêu và phụ thuộc
  - vòng đời dài hơn

runtime task
  - hướng thực thi
  - hướng đầu ra và trạng thái
  - vòng đời ngắn hơn
```

Mối quan hệ của chúng không phải là "chọn một."

Mà là:

```text
một work-graph task
  có thể sinh ra
một hoặc nhiều runtime task
```

Ví dụ:

```text
work-graph task:
  "Implement auth module"

runtime tasks:
  1. run tests in the background
  2. launch a coder teammate
  3. monitor an external service
```

## Tại Sao Sự Phân Biệt Này Quan Trọng

Nếu bạn không giữ các tầng này tách biệt, các chương sau bắt đầu rối lẫn vào nhau:

- thực thi nền của `s13` mờ vào bảng tác vụ của `s12`
- công việc của thành viên nhóm `s15-s17` không có chỗ sạch để gắn vào
- Worktree của `s18` trở nên không rõ ràng vì bạn không còn biết chúng thuộc tầng nào

Tóm tắt đúng ngắn nhất là:

**work-graph task quản lý mục tiêu; runtime task quản lý thực thi**

## Các Bản Ghi Cốt Lõi

### 1. `WorkGraphTaskRecord`

Đây là tác vụ bền vững từ `s12`.

```python
task = {
    "id": 12,
    "subject": "Implement auth module",
    "status": "in_progress",
    "blockedBy": [],
    "blocks": [13],
    "owner": "alice",
    "worktree": "auth-refactor",
}
```

### 2. `RuntimeTaskState`

Một cấu trúc dạy học tối thiểu có thể trông như thế này:

```python
runtime_task = {
    "id": "b8k2m1qz",
    "type": "local_bash",
    "status": "running",
    "description": "Run pytest",
    "start_time": 1710000000.0,
    "end_time": None,
    "output_file": ".task_outputs/b8k2m1qz.txt",
    "notified": False,
}
```

Các trường chính là:

- `type`: đơn vị thực thi này là gì
- `status`: liệu nó đang hoạt động hay đã kết thúc
- `output_file`: kết quả được lưu ở đâu
- `notified`: liệu hệ thống đã hiển thị kết quả chưa

### 3. `RuntimeTaskType`

Bạn không cần triển khai mọi loại trong repo dạy học ngay lập tức.

Nhưng bạn vẫn nên biết rằng runtime task là một họ, không chỉ là một loại lệnh shell.

Một bảng tối thiểu:

```text
local_bash
local_agent
remote_agent
in_process_teammate
monitor
workflow
```

## Các Bước Triển Khai Tối Thiểu

### Bước 1: giữ nguyên bảng tác vụ `s12`

Đừng làm nó quá tải.

### Bước 2: thêm một runtime task manager riêng biệt

```python
class RuntimeTaskManager:
    def __init__(self):
        self.tasks = {}
```

### Bước 3: tạo runtime task khi công việc nền bắt đầu

```python
def spawn_bash_task(command: str):
    task_id = new_runtime_id()
    runtime_tasks[task_id] = {
        "id": task_id,
        "type": "local_bash",
        "status": "running",
        "description": command,
    }
```

### Bước 4: tùy chọn liên kết thực thi Runtime trở lại Work-graph

```python
runtime_tasks[task_id]["work_graph_task_id"] = 12
```

Bạn không cần trường đó trong ngày đầu, nhưng nó ngày càng trở nên quan trọng khi hệ thống mở rộng đến các nhóm và Worktree.

## Bức Tranh Bạn Nên Giữ Trong Đầu

```text
Work Graph
  task #12: Implement auth module
        |
        +-- runtime task A: local_bash (pytest)
        +-- runtime task B: local_agent (coder worker)
        +-- runtime task C: monitor (watch service status)

Runtime Task Layer
  A/B/C each have:
  - their own runtime ID
  - their own status
  - their own output
  - their own lifecycle
```

## Cách Điều Này Kết Nối Với Các Chương Sau

Khi tầng này rõ ràng, phần còn lại của các chương Runtime và nền tảng trở nên dễ dàng hơn nhiều:

- các lệnh nền của `s13` là runtime task
- các thành viên nhóm `s15-s17` cũng có thể được hiểu là các biến thể runtime task
- Worktree `s18` chủ yếu ràng buộc với công việc bền vững, nhưng vẫn ảnh hưởng đến thực thi Runtime
- `s19` một số công việc bên ngoài giám sát hoặc bất đồng bộ cũng có thể nằm trong tầng Runtime

Bất cứ khi nào bạn thấy "có gì đó đang hoạt động trong nền và thúc đẩy công việc," hãy đặt hai câu hỏi:

- đây có phải là mục tiêu bền vững từ Work-graph không?
- hay đây là một slot thực thi trực tiếp trong Runtime?

## Những Lỗi Phổ Biến Của Người Mới

### 1. Đặt trạng thái shell nền trực tiếp vào bảng tác vụ

Điều đó trộn lẫn trạng thái tác vụ bền vững và trạng thái thực thi Runtime.

### 2. Giả định một work-graph task chỉ có thể có một runtime task

Trong các hệ thống thực, một mục tiêu thường sinh ra nhiều đơn vị thực thi.

### 3. Tái sử dụng cùng từ vựng trạng thái cho cả hai tầng

Ví dụ:

- tác vụ bền vững: `pending / in_progress / completed`
- runtime task: `running / completed / failed / killed`

Những điều đó nên được giữ riêng biệt khi có thể.

### 4. Bỏ qua các trường chỉ dành cho Runtime như `output_file` và `notified`

Bảng tác vụ bền vững không quan tâm nhiều đến chúng.
Tầng Runtime quan tâm rất nhiều.

## Điểm Mấu Chốt

**"Tác vụ" có hai nghĩa khác nhau: một mục tiêu bền vững trong Work-graph (cần làm gì) và một slot thực thi trực tiếp trong Runtime (đang chạy gì ngay bây giờ). Hãy giữ chúng trên các tầng riêng biệt.**
