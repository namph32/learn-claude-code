# s00e: Bản Đồ Module Tham Chiếu

> **Deep Dive** -- Đọc tài liệu này khi bạn muốn xác minh cách các chương giảng dạy ánh xạ vào codebase sản xuất thực tế.

Đây là một ghi chú hiệu chỉnh cho người duy trì và người học nghiêm túc. Nó không biến mã nguồn đã được kỹ thuật đảo ngược thành tài liệu đọc bắt buộc. Thay vào đó, nó trả lời một câu hỏi hẹp nhưng quan trọng: nếu bạn so sánh các cụm module có tín hiệu cao trong repo tham chiếu với repo giảng dạy này, thứ tự chương hiện tại có thực sự hợp lý không?

## Kết Luận Trước

Có.

Thứ tự `s01 -> s19` hiện tại nhìn chung là đúng, và nó gần với xương sống thiết kế thực tế hơn bất kỳ thứ tự "theo cây nguồn" ngây thơ nào.

Lý do đơn giản:

- repo tham chiếu chứa nhiều thư mục cấp bề mặt
- nhưng trọng lượng thiết kế thực sự tập trung trong một tập nhỏ hơn các module control, state, task, team, worktree và capability
- các module đó phù hợp với con đường giảng dạy bốn giai đoạn hiện tại

Vì vậy bước đi đúng **không phải** là làm phẳng repo giảng dạy thành thứ tự cây nguồn.

Bước đi đúng là:

- giữ thứ tự theo phụ thuộc hiện tại
- làm rõ ràng ánh xạ đến repo tham chiếu
- tiếp tục loại bỏ chi tiết sản phẩm có giá trị thấp khỏi nhánh chính

## Cách So Sánh Này Được Thực Hiện

So sánh dựa trên các cụm tín hiệu cao hơn của repo tham chiếu, đặc biệt là các module xung quanh:

- `Tool.ts`
- `state/AppStateStore.ts`
- `coordinator/coordinatorMode.ts`
- `memdir/*`
- `services/SessionMemory/*`
- `services/toolUseSummary/*`
- `constants/prompts.ts`
- `tasks/*`
- `tools/TodoWriteTool/*`
- `tools/AgentTool/*`
- `tools/ScheduleCronTool/*`
- `tools/EnterWorktreeTool/*`
- `tools/ExitWorktreeTool/*`
- `tools/MCPTool/*`
- `services/mcp/*`
- `plugins/*`
- `hooks/toolPermission/*`

Điều này đủ để đánh giá xương sống mà không kéo bạn qua mọi lệnh hướng tới sản phẩm, nhánh tương thích, hay chi tiết UI.

## Ánh Xạ Thực Sự

| Cụm repo tham chiếu | Ví dụ điển hình | Chương giảng dạy | Tại sao vị trí này đúng |
|---|---|---|---|
| Query loop + control state | `Tool.ts`, `AppStateStore.ts`, query/coordinator state | `s00`, `s00a`, `s00b`, `s01`, `s11` | Hệ thống thực không chỉ là `messages[] + while True`. Repo giảng dạy đúng khi bắt đầu với vòng lặp nhỏ trước, sau đó thêm control plane sau. |
| Tầng định tuyến và thực thi Tool | `Tool.ts`, native tools, tool context, execution helpers | `s02`, `s02a`, `s02b` | Mã nguồn rõ ràng xử lý Tool như một bề mặt thực thi chung, không phải một bảng dispatch đồ chơi. Việc chia tách trong giảng dạy là đúng. |
| Lập kế hoạch phiên | `TodoWriteTool` | `s03` | Lập kế hoạch phiên là một tầng nhỏ nhưng trung tâm. Nó thuộc về sớm, trước tác vụ lâu bền. |
| Ủy quyền một lần | `AgentTool` ở dạng đơn giản nhất | `s04` | Máy móc tạo Agent trong repo tham chiếu là lớn, nhưng repo giảng dạy đúng khi dạy Subagent sạch nhỏ nhất trước: Context mới, tác vụ có giới hạn, trả về tóm tắt. |
| Khám phá và tải kỹ năng | `DiscoverSkillsTool`, `skills/*`, prompt sections | `s05` | Kỹ năng không phải là những thứ bổ sung ngẫu nhiên. Chúng là một tầng tải kiến thức có chọn lọc, vì vậy chúng thuộc trước khi áp lực System Prompt và Context trở nên nghiêm trọng. |
| Áp lực Context và sụp đổ | `services/toolUseSummary/*`, `services/contextCollapse/*`, compact logic | `s06` | Repo tham chiếu rõ ràng có máy móc Compaction rõ ràng. Dạy điều này trước các tính năng nền tảng sau là đúng. |
| Cổng quyền | `types/permissions.ts`, `hooks/toolPermission/*`, approval handlers | `s07` | An toàn thực thi là một cổng riêng biệt, không phải "chỉ là một Hook khác". Giữ nó trước Hook là lựa chọn giảng dạy đúng. |
| Hook và tác dụng phụ | `types/hooks.ts`, hook runners, lifecycle integrations | `s08` | Mã nguồn tách các điểm mở rộng khỏi cổng chính. Dạy chúng sau quyền bảo tồn ranh giới đó. |
| Lựa chọn Memory lâu bền | `memdir/*`, `services/SessionMemory/*`, extract/select memory helpers | `s09` | Mã nguồn làm cho Memory thành một tầng chéo phiên có chọn lọc, không phải một sổ tay chung. Dạy điều này trước tập hợp System Prompt là đúng. |
| Tập hợp System Prompt | `constants/prompts.ts`, prompt sections, memory prompt loading | `s10`, `s10a` | Mã nguồn xây dựng đầu vào từ nhiều phần. Repo giảng dạy đúng khi trình bày tập hợp System Prompt như một pipeline thay vì một chuỗi khổng lồ. |
| Phục hồi và tiếp tục | query transition reasons, retry branches, compaction retry, token recovery | `s11`, `s00c` | Repo tham chiếu có logic tiếp tục rõ ràng. Điều này thuộc sau khi vòng lặp, Tool, Compaction, quyền, Memory và tập hợp System Prompt đã tồn tại. |
| Đồ thị công việc lâu bền | task records, task board concepts, dependency unlocks | `s12` | Repo giảng dạy đúng khi tách mục tiêu công việc lâu bền khỏi lập kế hoạch phiên tạm thời. |
| Tác vụ Runtime trực tiếp | `tasks/types.ts`, `LocalShellTask`, `LocalAgentTask`, `RemoteAgentTask`, `MonitorMcpTask` | `s13`, `s13a` | Mã nguồn có một union tác vụ Runtime rõ ràng. Điều này xác nhận mạnh mẽ việc chia tách giảng dạy giữa `TaskRecord` và `RuntimeTaskState`. |
| Kích hoạt lập lịch | `ScheduleCronTool/*`, `useScheduledTasks` | `s14` | Lập lịch xuất hiện sau khi công việc Runtime tồn tại, đây chính xác là thứ tự phụ thuộc đúng. |
| Thành viên nhóm lâu bền | `InProcessTeammateTask`, team tools, agent registries | `s15` | Mã nguồn rõ ràng phát triển từ Subagent một lần sang actor lâu bền. Dạy thành viên nhóm sau là đúng. |
| Phối hợp nhóm có cấu trúc | message envelopes, send-message flows, request tracking, coordinator mode | `s16` | Giao thức có ý nghĩa chỉ sau khi actor lâu bền tồn tại. Thứ tự hiện tại phù hợp với phụ thuộc thực sự. |
| Tự động nhận và tiếp tục | coordinator mode, task claiming, async worker lifecycle, resume logic | `s17` | Tính tự chủ trong mã nguồn không phải là ma thuật. Nó được xếp lớp trên actor, tác vụ và quy tắc phối hợp. Vị trí hiện tại là đúng. |
| Luồng thực thi Worktree | `EnterWorktreeTool`, `ExitWorktreeTool`, agent worktree helpers | `s18` | Repo tham chiếu xử lý Worktree như một cơ chế ranh giới luồng thực thi với logic đóng. Dạy nó sau tác vụ và thành viên nhóm ngăn khái niệm sụp đổ. |
| Bus năng lực bên ngoài | `MCPTool`, `services/mcp/*`, `plugins/*`, MCP resources/prompts/tools | `s19`, `s19a` | Mã nguồn rõ ràng đặt MCP và plugin ở ranh giới nền tảng bên ngoài. Giữ điều này cuối cùng là lựa chọn giảng dạy đúng. |

## Các Điểm Xác Nhận Quan Trọng Nhất

Repo tham chiếu xác nhận mạnh mẽ năm lựa chọn giảng dạy.

### 1. `s03` nên đứng trước `s12`

Mã nguồn chứa cả:

- lập kế hoạch phiên nhỏ
- máy móc tác vụ/Runtime lâu bền lớn hơn

Đây không phải là cùng một thứ.

Repo giảng dạy đúng khi dạy:

`lập kế hoạch phiên trước -> tác vụ lâu bền sau`

### 2. `s09` nên đứng trước `s10`

Mã nguồn xây dựng đầu vào mô hình từ nhiều nguồn, bao gồm Memory.

Điều đó có nghĩa là:

- Memory là một nguồn đầu vào
- tập hợp System Prompt là pipeline kết hợp các nguồn

Vì vậy Memory nên được giải thích trước tập hợp System Prompt.

### 3. `s12` phải đứng trước `s13`

Union tác vụ Runtime trong repo tham chiếu là một trong những bằng chứng mạnh nhất trong toàn bộ so sánh.

Nó cho thấy rằng:

- định nghĩa công việc lâu bền
- thực thi đang chạy trực tiếp

phải giữ sự tách biệt về mặt khái niệm.

Nếu `s13` đến trước, bạn gần như chắc chắn sẽ gộp hai tầng đó.

### 4. `s15 -> s16 -> s17` là thứ tự đúng

Mã nguồn có:

- actor lâu bền
- phối hợp có cấu trúc
- hành vi resume / nhận tự chủ

Tính tự chủ phụ thuộc vào hai cái đầu. Vì vậy thứ tự hiện tại là đúng.

### 5. `s18` nên đứng trước `s19`

Repo tham chiếu xử lý sự cô lập Worktree như một cơ chế ranh giới thực thi cục bộ.

Điều đó nên được hiểu trước khi bạn được yêu cầu lý luận về:

- nhà cung cấp năng lực bên ngoài
- MCP server
- bề mặt được cài đặt bởi plugin

Nếu không, năng lực bên ngoài trông trung tâm hơn thực tế.

## Những Gì Repo Giảng Dạy Này Vẫn Nên Tránh Sao Chép

Repo tham chiếu chứa nhiều thứ thực, nhưng vẫn không nên thống trị nhánh chính giảng dạy:

- diện tích bề mặt lệnh CLI
- chi tiết render UI
- các nhánh đo lường từ xa và phân tích
- tích hợp sản phẩm
- kết nối từ xa và doanh nghiệp
- mã tương thích theo nền tảng cụ thể
- trivia đặt tên từng dòng

Đây là các chi tiết triển khai hợp lệ.

Chúng không phải là trung tâm phù hợp của một con đường giảng dạy từ 0 đến 1.

## Nơi Repo Giảng Dạy Phải Cẩn Thận Hơn

Ánh xạ cũng tiết lộ một số nơi dễ bị nhầm lẫn.

### 1. Đừng gộp Subagent và thành viên nhóm thành một khái niệm mơ hồ

`AgentTool` của repo tham chiếu bao gồm:

- ủy quyền một lần
- worker async/nền tảng
- worker lâu bền giống thành viên nhóm
- worker cô lập Worktree

Đó chính xác là lý do tại sao repo giảng dạy nên chia câu chuyện qua:

- `s04`
- `s15`
- `s17`
- `s18`

### 2. Đừng dạy Worktree như "chỉ là một thủ thuật git"

Mã nguồn hiển thị đóng, resume, dọn dẹp và trạng thái cô lập xung quanh Worktree.

Vì vậy `s18` nên tiếp tục dạy:

- danh tính luồng
- ràng buộc tác vụ
- đóng keep/remove
- các mối quan tâm resume và dọn dẹp

không chỉ `git worktree add`.

### 3. Đừng thu gọn MCP thành "Tool từ xa"

Mã nguồn bao gồm:

- Tool
- resources
- prompts
- elicitation / trạng thái kết nối
- plugin mediation

Vì vậy `s19` nên giữ một con đường giảng dạy Tool-first, nhưng vẫn giải thích ranh giới bus năng lực rộng hơn.

## Đánh Giá Cuối Cùng

So sánh với các cụm module tín hiệu cao trong repo tham chiếu, thứ tự chương hiện tại là vững chắc.

Những cải tiến chất lượng còn lại lớn nhất **không** đến từ một sắp xếp lại lớn khác.

Chúng đến từ:

- tài liệu cầu nối sạch hơn
- giải thích ranh giới thực thể mạnh hơn
- tính nhất quán đa ngôn ngữ chặt chẽ hơn
- trang web hiển thị cùng một bản đồ học tập rõ ràng

## Điểm Mấu Chốt

**Thứ tự giảng dạy tốt nhất không phải là thứ tự các tệp xuất hiện trong một repo. Đó là thứ tự mà các phụ thuộc trở nên có thể hiểu được đối với người học muốn tái xây dựng hệ thống.**
