# Mô Hình Nhóm - Tác Vụ - Lane

> **Tìm Hiểu Sâu** -- Nên đọc vào đầu Giai đoạn 4 (s15-s18). Tài liệu này tách biệt năm khái niệm trông giống nhau nhưng tồn tại trên các tầng khác nhau.

### Khi Nào Đọc Tài Liệu Này

Trước khi bạn bắt đầu các chương về nhóm. Giữ nó mở làm tài liệu tham khảo trong suốt s15-s18.

---

> Khi bạn đến `s15-s18`, thứ dễ bị nhòe nhất không phải là tên hàm.
>
> Mà là:
>
> **Ai đang làm việc, ai đang điều phối, cái gì ghi lại mục tiêu, và cái gì cung cấp lane thực thi.**

## Tài Liệu Bridge Này Giải Quyết Gì

Trong `s15-s18`, bạn sẽ gặp các từ sau có thể dễ dàng nhòe thành một ý niệm mơ hồ:

- teammate
- protocol request
- task
- runtime task
- worktree

Tất cả đều liên quan đến việc hoàn thành công việc, nhưng chúng **không** tồn tại trên cùng một tầng.

Nếu bạn không tách biệt chúng, các chương sau sẽ bắt đầu cảm thấy rối rắm:

- Teammate có giống task không?
- Sự khác biệt giữa `request_id` và `task_id` là gì?
- Worktree chỉ là một runtime task khác?
- Tại sao một tác vụ có thể hoàn thành trong khi worktree vẫn còn được giữ?

Tài liệu này tồn tại để tách biệt các tầng đó một cách rõ ràng.

## Thứ Tự Đọc Được Đề Xuất

1. Đọc [`s15-agent-teams.md`](./s15-agent-teams.md) để biết về các Teammate tồn tại lâu dài.
2. Đọc [`s16-team-protocols.md`](./s16-team-protocols.md) để biết về điều phối request-response có theo dõi.
3. Đọc [`s17-autonomous-agents.md`](./s17-autonomous-agents.md) để biết về các Teammate tự nhận việc.
4. Đọc [`s18-worktree-task-isolation.md`](./s18-worktree-task-isolation.md) để biết về các lane thực thi độc lập.

Nếu từ vựng bắt đầu bị nhòe, bạn có thể thấy hữu ích khi xem lại:

- [`entity-map.md`](./entity-map.md)
- [`data-structures.md`](./data-structures.md)
- [`s13a-runtime-task-model.md`](./s13a-runtime-task-model.md)

## Sự Tách Biệt Lõi

```text
teammate
  = ai tham gia theo thời gian

protocol request
  = một yêu cầu điều phối có theo dõi bên trong nhóm

task
  = điều gì cần được thực hiện

runtime task / execution slot
  = điều gì đang chạy tích cực ngay bây giờ

worktree
  = nơi công việc thực thi mà không va chạm với các lane khác
```

Sự nhầm lẫn phổ biến nhất là giữa ba cái cuối:

- `task`
- `runtime task`
- `worktree`

Hỏi ba câu hỏi riêng biệt mỗi lần:

- Đây có phải là mục tiêu không?
- Đây có phải là đơn vị thực thi đang chạy không?
- Đây có phải là thư mục thực thi độc lập không?

## Sơ Đồ Nhỏ Gọn Nhất

```text
Team Layer
  teammate: alice (frontend)

Protocol Layer
  request_id=req_01
  kind=plan_approval
  status=pending

Work Graph Layer
  task_id=12
  subject="Implement login page"
  owner="alice"
  status="in_progress"

Runtime Layer
  runtime_id=rt_01
  type=in_process_teammate
  status=running

Execution Lane Layer
  worktree=login-page
  path=.worktrees/login-page
  status=active
```

Chỉ một trong số những bản ghi đó ghi lại mục tiêu công việc bản thân:

> `task_id=12`

Các bản ghi khác hỗ trợ điều phối, thực thi hoặc cách ly xung quanh mục tiêu đó.

## 1. Teammate: Ai Đang Cộng Tác

Được giới thiệu trong `s15`.

Tầng này trả lời:

- worker tồn tại lâu dài được gọi là gì
- vai trò của nó là gì
- nó đang ở trạng thái `working`, `idle`, hay `shutdown`
- nó có hộp thư riêng không

Ví dụ:

```python
member = {
    "name": "alice",
    "role": "frontend",
    "status": "idle",
}
```

Điểm mấu chốt không phải là "một instance Agent khác."

Mà là:

> một danh tính bền vững có thể liên tục nhận công việc.

## 2. Protocol Request: Điều Gì Đang Được Điều Phối

Được giới thiệu trong `s16`.

Tầng này trả lời:

- ai đã hỏi ai
- loại yêu cầu này là gì
- nó còn đang chờ hay đã được giải quyết

Ví dụ:

```python
request = {
    "request_id": "a1b2c3d4",
    "kind": "plan_approval",
    "from": "alice",
    "to": "lead",
    "status": "pending",
}
```

Đây không phải là trò chuyện thông thường.

Mà là:

> một bản ghi điều phối có trạng thái có thể tiếp tục phát triển.

## 3. Task: Điều Gì Cần Được Thực Hiện

Đây là tác vụ đồ thị công việc bền vững từ `s12`, và là thứ các Teammate trong `s17` nhận lấy.

Nó trả lời:

- mục tiêu là gì
- ai sở hữu nó
- điều gì chặn nó
- nó đang ở trạng thái tiến trình nào

Ví dụ:

```python
task = {
    "id": 12,
    "subject": "Implement login page",
    "status": "in_progress",
    "owner": "alice",
    "blockedBy": [],
}
```

Từ khóa:

**mục tiêu**

Không phải thư mục. Không phải protocol. Không phải tiến trình.

## 4. Runtime Task / Execution Slot: Điều Gì Đang Chạy

Tầng này đã được làm rõ trong tài liệu bridge `s13a`, nhưng nó còn quan trọng hơn trong `s15-s18`.

Ví dụ:

- một lệnh shell nền
- một Teammate tồn tại lâu dài đang làm việc
- một tiến trình monitor theo dõi trạng thái bên ngoài

Những thứ này được hiểu rõ nhất là:

> các slot thực thi đang hoạt động

Ví dụ:

```python
runtime = {
    "id": "rt_01",
    "type": "in_process_teammate",
    "status": "running",
    "work_graph_task_id": 12,
}
```

Ranh giới quan trọng:

- một tác vụ đồ thị công việc có thể tạo ra nhiều runtime task
- một runtime task là một instance thực thi, không phải mục tiêu bền vững bản thân

## 5. Worktree: Nơi Công Việc Xảy Ra

Được giới thiệu trong `s18`.

Tầng này trả lời:

- thư mục độc lập nào được sử dụng
- nó gắn với tác vụ nào
- lane đó đang ở trạng thái `active`, `kept` hay `removed`

Ví dụ:

```python
worktree = {
    "name": "login-page",
    "path": ".worktrees/login-page",
    "task_id": 12,
    "status": "active",
}
```

Từ khóa:

**ranh giới thực thi**

Nó không phải là mục tiêu tác vụ bản thân. Nó là lane độc lập nơi mục tiêu đó được thực thi.

## Cách Các Tầng Kết Nối

```text
teammate
  điều phối thông qua protocol request
  nhận một tác vụ
  chạy như một execution slot
  làm việc bên trong một worktree lane
```

Trong một câu cụ thể hơn:

> `alice` nhận `task #12` và tiến hành nó bên trong worktree lane `login-page`.

Câu đó gọn gàng hơn nhiều so với việc nói:

> "alice đang làm worktree task login-page"

vì câu ngắn hơn kết hợp sai:

- teammate
- tác vụ
- worktree

## Các Lỗi Phổ Biến

### 1. Xem Teammate và Task như cùng một đối tượng

Teammate thực thi. Task thể hiện mục tiêu.

### 2. Xem `request_id` và `task_id` như có thể thay thế cho nhau

Một cái theo dõi điều phối. Cái kia theo dõi mục tiêu công việc.

### 3. Xem Runtime Slot như là tác vụ bền vững

Việc thực thi đang chạy có thể kết thúc trong khi tác vụ bền vững vẫn tồn tại.

### 4. Xem Worktree như chính tác vụ

Worktree chỉ là lane thực thi.

### 5. Nói "hệ thống hoạt động song song" mà không đặt tên các tầng

Giảng dạy tốt không dừng lại ở "có nhiều Agent."

Nó có thể nói rõ ràng:

> các Teammate cung cấp sự cộng tác lâu dài, các yêu cầu theo dõi điều phối, các tác vụ ghi lại mục tiêu, các Runtime Slot thực thi, và các Worktree cách ly thư mục thực thi.

## Những Gì Bạn Nên Có Thể Nói Sau Khi Đọc Tài Liệu Này

1. Tính tự chủ `s17` nhận các tác vụ đồ thị công việc `s12`, không phải Runtime Slot `s13`.
2. Worktree `s18` gắn kết các lane thực thi với các tác vụ; chúng không biến tác vụ thành thư mục.
3. Một Teammate có thể ở trạng thái idle trong khi tác vụ vẫn tồn tại và trong khi Worktree vẫn còn được giữ.
4. Một Protocol Request theo dõi một trao đổi điều phối, không phải một mục tiêu công việc.

## Kết Luận Chính

**Năm thứ nghe có vẻ giống nhau -- teammate, protocol request, task, runtime slot, worktree -- tồn tại trên năm tầng riêng biệt. Việc đặt tên cho tầng bạn muốn nói đến là cách bạn giữ cho các chương nhóm không sụp đổ vào sự nhầm lẫn.**
