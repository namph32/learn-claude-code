# Giải thích `README.md`

`README.md` là tài liệu định hướng chính của repo `learn-claude-code`. Nó không chỉ nói cách chạy code, mà chủ yếu giải thích repo này đang dạy điều gì, dành cho ai, và nên học theo thứ tự nào.

## 1. Repo này dùng để làm gì

Mục tiêu của repo là dạy cách tự xây một coding-agent harness từ đầu.

Repo không cố sao chép mọi chi tiết của một hệ thống production thật. Thay vào đó, nó tập trung vào các cơ chế cốt lõi quyết định một agent có hoạt động tốt hay không, ví dụ:

- agent loop
- tools
- planning
- context control
- permissions
- hooks
- memory
- prompt assembly
- tasks
- teams
- worktree isolation
- MCP và plugin

Ý chính của README là:

**hãy hiểu đủ sâu phần xương sống của hệ thống để có thể tự dựng lại nó.**

## 2. Tư tưởng trung tâm của repo

README nhấn mạnh một câu rất quan trọng:

**Model làm phần suy luận. Harness cung cấp môi trường làm việc cho model.**

Nói cách khác:

- model không tự có khả năng thao tác với máy
- harness mới là nơi cung cấp tool, context, luật chơi, và trạng thái chạy

Từ góc nhìn kiến trúc, đây là tư tưởng xuyên suốt toàn bộ repo.

## 3. Repo này thực sự đang dạy những gì

README chia hệ thống agent thành các khối cơ bản:

- `Agent Loop`: vòng lặp hỏi model, chạy tool, trả kết quả tool lại
- `Tools`: bàn tay của agent
- `Planning`: giúp công việc nhiều bước không bị trôi
- `Context Management`: giữ context đang hoạt động đủ nhỏ và mạch lạc
- `Permissions`: chặn việc model biến ý định thành hành động nguy hiểm
- `Hooks`: thêm hành vi quanh loop mà không phải viết lại loop
- `Memory`: lưu những dữ kiện cần sống qua nhiều session
- `Prompt Construction`: ghép prompt từ các quy tắc ổn định và runtime state
- `Tasks / Teams / Worktree / MCP`: mở rộng single-agent core thành một nền tảng lớn hơn

README muốn người đọc thấy rằng một coding agent không chỉ là “gọi model + chạy shell”, mà là một tập hợp cơ chế hợp tác với nhau.

## 4. Repo này cố tình không tập trung vào điều gì

README nói rõ repo không muốn để các chi tiết phụ lấn át đường học chính. Vì vậy nó tránh đặt nặng:

- packaging và release
- lớp tương thích đa nền tảng
- enterprise policy glue
- telemetry
- account wiring
- compatibility branches
- các chi tiết lịch sử hoặc naming đặc thù của sản phẩm thật

Ý nghĩa của phần này là:

- production có thể cần nhiều thứ hơn
- nhưng để học từ 0 đến 1, cần tách “xương sống” ra khỏi “phần rìa”

## 5. Đối tượng người đọc

README giả định người đọc:

- đã biết Python cơ bản
- hiểu function, class, list, dictionary
- có thể hoàn toàn mới với agent systems

Vì thế repo đặt ra một số nguyên tắc dạy học:

- giải thích khái niệm trước khi dùng
- mỗi khái niệm nên có một nơi giải thích chính
- đi từ “nó là gì” sang “vì sao cần” rồi mới đến “cách làm”
- không bắt người học tự lắp ghép hệ thống từ các mảnh rời rạc

## 6. Thứ tự đọc được khuyến nghị

README khuyên không nên mở ngẫu nhiên các chapter.

Trình tự an toàn là:

1. Đọc overview kiến trúc
2. Đọc tài liệu giải thích vì sao chapter được sắp theo thứ tự đó
3. Đọc code reading order để biết nên mở file nào trước
4. Học lần lượt theo các stage `s01-s06`, `s07-s11`, `s12-s14`, `s15-s19`
5. Sau mỗi stage, nên tự dựng lại một phiên bản nhỏ trước khi học tiếp

Điểm này rất quan trọng vì repo được thiết kế như một tuyến học có chủ đích, không phải một tập file độc lập.

## 7. Bốn giai đoạn lớn của repo

README chia repo thành 4 stage:

1. `s01-s06`: xây single-agent core hữu ích
2. `s07-s11`: thêm safety, extension points, memory, prompt assembly, error recovery
3. `s12-s14`: biến planning tạm thời thành runtime work bền hơn
4. `s15-s19`: mở rộng sang teams, protocols, autonomy, worktree isolation, external capability routing

Đây là bản đồ tiến hóa của hệ thống:

- bắt đầu từ loop tối thiểu
- sau đó thêm control plane
- rồi thêm runtime structure
- cuối cùng mới đi tới hệ đa agent và external capability

## 8. Danh sách các chapter

README liệt kê từng chapter và vai trò của nó:

- `s00`: kiến trúc tổng thể
- `s01`: agent loop
- `s02`: tool use
- `s03`: todo/planning
- `s04`: subagent
- `s05`: skills
- `s06`: context compact
- `s07`: permission system
- `s08`: hook system
- `s09`: memory system
- `s10`: system prompt
- `s11`: error recovery
- `s12`: task system
- `s13`: background tasks
- `s14`: cron scheduler
- `s15`: agent teams
- `s16`: team protocols
- `s17`: autonomous agents
- `s18`: worktree isolation
- `s19`: MCP & plugin

Nếu nhìn theo logic tăng trưởng, danh sách này cho thấy repo đang dạy cách đi từ một agent đơn giản tới một nền tảng agent tương đối hoàn chỉnh.

## 9. Quick Start

README có phần khởi động nhanh:

```sh
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code
pip install -r requirements.txt
cp .env.example .env
```

Sau đó cần cấu hình `ANTHROPIC_API_KEY` hoặc endpoint tương thích trong `.env`, rồi có thể chạy:

```sh
python agents/s01_agent_loop.py
python agents/s18_worktree_task_isolation.py
python agents/s19_mcp_plugin.py
python agents/s_full.py
```

README cũng nhấn mạnh nên chạy `s01` trước để chắc chắn đã hiểu vòng lặp tối thiểu.

## 10. Cách đọc từng chapter

README gợi ý một nhịp đọc thống nhất:

1. Nếu thiếu cơ chế này thì sẽ gặp vấn đề gì
2. Khái niệm mới có nghĩa là gì
3. Bản cài đặt nhỏ nhất đúng là gì
4. State thật sự nằm ở đâu
5. Nó gắn lại vào loop như thế nào
6. Nên dừng ở đâu trước, cái gì có thể để sau

Đây là một hướng dẫn đọc code rất thực dụng. Nó ép người học không chỉ nhìn syntax, mà phải hiểu:

- vì sao cơ chế này tồn tại
- nó sống ở đâu trong hệ thống
- nó nối vào đường mainline ra sao

## 11. Cấu trúc thư mục repo

README mô tả nhanh cấu trúc:

- `agents/`: các file Python có thể chạy được, tương ứng từng chapter
- `docs/zh/`: tài liệu tiếng Trung
- `docs/en/`: tài liệu tiếng Anh
- `docs/ja/`: tài liệu tiếng Nhật
- `skills/`: các file skill dùng trong chapter `s05`
- `web/`: nền tảng web để học trực quan hơn
- `requirements.txt`: dependency Python

Phần này giúp người mới biết nên tìm code, docs, và các phần phụ trợ ở đâu.

## 12. Trạng thái ngôn ngữ

README nói rõ:

- tiếng Trung là bản chuẩn và cập nhật nhanh nhất
- tiếng Anh và tiếng Nhật có main chapters cùng các bridge docs chính

Điều này có nghĩa là nếu cần tuyến giải thích đầy đủ nhất, bản tiếng Trung vẫn là nguồn chính.

## 13. Mục tiêu cuối cùng của repo

README kết thúc bằng một nhóm câu hỏi mà người học nên trả lời được sau khi đi hết repo:

- state tối thiểu của một coding agent là gì
- vì sao `tool_result` là trung tâm của loop
- khi nào nên dùng subagent thay vì nhồi thêm context
- mỗi cơ chế như permissions, hooks, memory, prompt assembly, tasks giải quyết vấn đề gì
- khi nào single-agent nên phát triển thành tasks, teams, worktrees, và MCP

Đây là cách README định nghĩa “học xong repo”: không phải nhớ API, mà là hiểu rõ mô hình vận hành.

## 14. Tóm tắt ngắn

`README.md` là bản đồ học tập của toàn repo.

Nó làm 4 việc chính:

1. Giải thích repo này đang dạy điều gì
2. Chỉ ra phần nào là cốt lõi, phần nào cố tình lược bỏ
3. Đưa ra thứ tự học hợp lý
4. Cho thấy cách một coding agent phát triển từ loop tối thiểu thành hệ thống lớn hơn

Nếu `s01_agent_loop.py` dạy “trái tim đang đập như thế nào”, thì `README.md` dạy “toàn cơ thể này được tổ chức ra sao và nên học từ đâu trước”.
