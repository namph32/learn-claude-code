# s00: Tổng Quan Kiến Trúc

Chào mừng đến với bản đồ tổng thể. Trước khi bắt đầu xây dựng từng phần một, sẽ rất hữu ích nếu bạn nhìn toàn bộ bức tranh từ trên cao. Tài liệu này cho bạn thấy hệ thống đầy đủ bao gồm những gì, tại sao các chương được sắp xếp theo thứ tự này, và bạn sẽ thực sự học được gì.

## Bức Tranh Tổng Thể

Nhánh chính của repo này hợp lý vì nó phát triển hệ thống qua bốn giai đoạn theo thứ tự phụ thuộc:

1. xây dựng một vòng lặp Agent đơn thực sự hoạt động
2. củng cố vòng lặp đó với tính an toàn, Memory và khả năng phục hồi
3. chuyển đổi công việc phiên tạm thời thành công việc Runtime lâu bền
4. mở rộng từ một executor đơn thành một nền tảng đa Agent với các luồng riêng biệt và định tuyến năng lực bên ngoài

Thứ tự này tuân theo **thứ tự phụ thuộc giữa các cơ chế**, không phải thứ tự tệp và không phải sức hút của sản phẩm.

Nếu người học chưa hiểu:

`user input -> model -> tools -> write-back -> next turn`

thì quyền, hooks, memory, tác vụ, nhóm, worktrees, và MCP đều trở thành những từ ngữ rời rạc không liên kết.

## Repo Này Đang Cố Gắng Tái Tạo Điều Gì

Repo này không cố gắng phản chiếu một codebase sản xuất từng dòng một.

Nó đang cố gắng tái tạo những phần quyết định liệu một hệ thống Agent có thực sự hoạt động hay không:

- các module chính là gì
- các module đó cộng tác với nhau như thế nào
- mỗi module chịu trách nhiệm về điều gì
- trạng thái quan trọng nằm ở đâu
- một yêu cầu đi qua hệ thống như thế nào

Điều đó có nghĩa là mục tiêu là:

**độ trung thực cao với xương sống thiết kế, không phải độ trung thực 1:1 với mọi chi tiết triển khai bên ngoài.**

## Ba Lời Khuyên Trước Khi Bắt Đầu

### Lời Khuyên 1: Học phiên bản nhỏ nhất đúng đắn trước

Ví dụ, một Subagent không cần mọi khả năng nâng cao ngay từ ngày đầu.

Phiên bản nhỏ nhất đúng đắn đã dạy được bài học cốt lõi:

- phần tử cha định nghĩa tác vụ con
- phần tử con nhận một `messages[]` riêng biệt
- phần tử con trả về bản tóm tắt

Chỉ sau khi điều đó ổn định mới nên thêm:

- Context được kế thừa
- quyền riêng biệt
- Runtime nền tảng
- sự cô lập của Worktree

### Lời Khuyên 2: Các thuật ngữ mới nên được giải thích trước khi sử dụng

Repo này sử dụng các thuật ngữ như:

- state machine
- dispatch map
- dependency graph
- worktree
- protocol envelope
- MCP

Nếu một thuật ngữ chưa quen thuộc, hãy dừng lại và kiểm tra tài liệu tham khảo thay vì tiếp tục một cách mù quáng.

Tài liệu đi kèm được khuyến nghị:

- [`glossary.md`](./glossary.md)
- [`entity-map.md`](./entity-map.md)
- [`data-structures.md`](./data-structures.md)
- [`teaching-scope.md`](./teaching-scope.md)

### Lời Khuyên 3: Đừng để sự phức tạp ngoại vi giả vờ là cơ chế cốt lõi

Giảng dạy tốt không cố gắng bao gồm tất cả mọi thứ.

Nó giải thích những phần quan trọng một cách hoàn chỉnh và giữ cho sự phức tạp có giá trị thấp tránh khỏi con đường của bạn:

- đóng gói và luồng phát hành
- tích hợp doanh nghiệp
- đo lường từ xa
- các nhánh tương thích theo sản phẩm cụ thể
- trivia về tên tệp / số dòng ngược kỹ thuật

## Các Tài Liệu Cầu Nối Quan Trọng

Coi các tài liệu này như bản đồ liên chương:

| Tài liệu | Nó Làm Rõ Điều Gì |
|---|---|
| [`s00d-chapter-order-rationale.md`](./s00d-chapter-order-rationale.md) (Deep Dive) | tại sao thứ tự chương trình học là như vậy |
| [`s00e-reference-module-map.md`](./s00e-reference-module-map.md) (Deep Dive) | cách các cụm module thực trong repo tham chiếu ánh xạ vào chương trình học hiện tại |
| [`s00a-query-control-plane.md`](./s00a-query-control-plane.md) (Deep Dive) | tại sao một Agent hoàn thành tốt cần nhiều hơn `messages[] + while True` |
| [`s00b-one-request-lifecycle.md`](./s00b-one-request-lifecycle.md) (Deep Dive) | một yêu cầu di chuyển qua toàn bộ hệ thống như thế nào |
| [`s02a-tool-control-plane.md`](./s02a-tool-control-plane.md) (Deep Dive) | tại sao Tool trở thành mặt phẳng điều khiển, không chỉ là bảng hàm |
| [`s10a-message-prompt-pipeline.md`](./s10a-message-prompt-pipeline.md) (Deep Dive) | tại sao System Prompt chỉ là một bề mặt đầu vào |
| [`s13a-runtime-task-model.md`](./s13a-runtime-task-model.md) (Deep Dive) | tại sao tác vụ lâu bền và các slot Runtime trực tiếp phải được tách biệt |
| [`s19a-mcp-capability-layers.md`](./s19a-mcp-capability-layers.md) (Deep Dive) | tại sao MCP không chỉ là danh sách Tool từ xa |

## Bốn Giai Đoạn Học Tập

### Stage 1: Agent Đơn Cốt Lõi (`s01-s06`)

Mục tiêu: xây dựng một Agent đơn có thể thực sự thực hiện công việc.

| Chương | Tầng Mới |
|---|---|
| `s01` | vòng lặp và write-back |
| `s02` | Tool và dispatch |
| `s03` | lập kế hoạch phiên |
| `s04` | cô lập tác vụ con được ủy quyền |
| `s05` | khám phá và tải kỹ năng |
| `s06` | Compaction Context |

### Stage 2: Củng Cố (`s07-s11`)

Mục tiêu: làm cho vòng lặp an toàn hơn, ổn định hơn và dễ mở rộng hơn.

| Chương | Tầng Mới |
|---|---|
| `s07` | cổng quyền |
| `s08` | Hook và tác dụng phụ |
| `s09` | Memory lâu bền |
| `s10` | tập hợp System Prompt |
| `s11` | phục hồi và tiếp tục |

### Stage 3: Công Việc Runtime (`s12-s14`)

Mục tiêu: nâng cấp công việc phiên thành công việc Runtime lâu bền, nền tảng và lập lịch.

| Chương | Tầng Mới |
|---|---|
| `s12` | đồ thị tác vụ lâu bền |
| `s13` | các slot thực thi Runtime |
| `s14` | kích hoạt theo thời gian |

### Stage 4: Nền Tảng (`s15-s19`)

Mục tiêu: phát triển từ một executor thành một nền tảng lớn hơn.

| Chương | Tầng Mới |
|---|---|
| `s15` | thành viên nhóm lâu bền |
| `s16` | giao thức nhóm có cấu trúc |
| `s17` | tự động nhận và tiếp tục |
| `s18` | các luồng thực thi cô lập |
| `s19` | định tuyến năng lực bên ngoài |

## Tham Khảo Nhanh: Mỗi Chương Thêm Gì

| Chương | Cấu Trúc Cốt Lõi | Những Gì Bạn Có Thể Xây Dựng |
|---|---|---|
| `s01` | `LoopState`, write-back `tool_result` | một vòng lặp Agent hoạt động tối giản |
| `s02` | `ToolSpec`, dispatch map | định tuyến Tool ổn định |
| `s03` | `TodoItem`, `PlanState` | lập kế hoạch phiên có thể nhìn thấy |
| `s04` | Context con cô lập | các tác vụ con được ủy quyền mà không làm ô nhiễm phần cha |
| `s05` | `SkillRegistry` | khám phá nhẹ và tải sâu theo yêu cầu |
| `s06` | bản ghi Compaction | các phiên dài vẫn có thể sử dụng |
| `s07` | quyết định quyền | thực thi sau cổng kiểm tra |
| `s08` | sự kiện vòng đời | mở rộng mà không viết lại vòng lặp |
| `s09` | bản ghi Memory | Memory dài hạn có chọn lọc |
| `s10` | các phần System Prompt | tập hợp đầu vào theo giai đoạn |
| `s11` | lý do tiếp tục | các nhánh phục hồi dễ đọc |
| `s12` | `TaskRecord` | đồ thị công việc lâu bền |
| `s13` | `RuntimeTaskState` | thực thi nền tảng với write-back sau |
| `s14` | `ScheduleRecord` | công việc kích hoạt theo thời gian |
| `s15` | `TeamMember`, hộp thư đến | thành viên nhóm lâu bền |
| `s16` | protocol envelope | phối hợp yêu cầu / phản hồi có cấu trúc |
| `s17` | chính sách nhận | tự nhận và tự tiếp tục |
| `s18` | `WorktreeRecord` | các luồng thực thi cô lập |
| `s19` | định tuyến năng lực | định tuyến thống nhất native + plugin + MCP |

## Điểm Mấu Chốt

**Thứ tự chương tốt không phải là danh sách tính năng. Đó là một con đường mà mỗi cơ chế phát triển tự nhiên từ cơ chế trước đó.**
