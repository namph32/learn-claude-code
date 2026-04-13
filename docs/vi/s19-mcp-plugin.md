# s19: MCP & Plugin

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 > s12 > s13 > s14 > s15 > s16 > s17 > s18 > [ s19 ]`

## Những Gì Bạn Sẽ Học
- Cách MCP (Model Context Protocol -- một cách tiêu chuẩn để Agent giao tiếp với các server khả năng bên ngoài) cho phép Agent của bạn có thêm Tool mới mà không cần thay đổi code cốt lõi
- Cách chuẩn hóa tên Tool với tiền tố `mcp__{server}__{tool}` ngăn các Tool bên ngoài xung đột với Tool gốc
- Cách một router thống nhất điều phối các lời gọi Tool đến các handler cục bộ hoặc server từ xa qua cùng một đường dẫn
- Cách các plugin manifest cho phép các server khả năng bên ngoài được khám phá và khởi động tự động

Cho đến nay, mọi Tool mà Agent của bạn sử dụng -- bash, read, write, edit, tasks, Worktrees -- đều nằm trong harness Python của bạn. Bạn đã viết từng cái bằng tay. Điều đó hoạt động tốt cho một codebase giảng dạy, nhưng một Agent thực cần nói chuyện với cơ sở dữ liệu, trình duyệt, dịch vụ đám mây và các Tool chưa tồn tại. Hard-code mọi khả năng có thể là không bền vững. Chương này chỉ ra cách các chương trình bên ngoài có thể tham gia Agent của bạn thông qua cùng mặt phẳng định tuyến Tool mà bạn đã xây dựng.

## Vấn Đề

Agent của bạn mạnh mẽ, nhưng các khả năng của nó bị đóng băng tại thời điểm build. Nếu bạn muốn nó truy vấn cơ sở dữ liệu Postgres, bạn viết một handler Python mới. Nếu bạn muốn nó điều khiển trình duyệt, bạn viết một handler khác. Mỗi khả năng mới có nghĩa là thay đổi harness cốt lõi, kiểm tra lại router Tool và triển khai lại. Trong khi đó, các nhóm khác đang xây dựng các server chuyên biệt đã biết cách nói chuyện với các hệ thống này. Bạn cần một giao thức tiêu chuẩn để các server bên ngoài đó có thể expose Tool của chúng cho Agent của bạn, và Agent của bạn có thể gọi chúng một cách tự nhiên như gọi Tool gốc của nó -- mà không cần viết lại vòng lặp cốt lõi mỗi lần.

## Giải Pháp

MCP cung cấp cho Agent của bạn một cách tiêu chuẩn để kết nối với các server khả năng bên ngoài qua stdio. Agent khởi động một tiến trình server, hỏi những Tool nào nó cung cấp, chuẩn hóa tên của chúng với tiền tố, và định tuyến các lời gọi đến server đó -- tất cả qua cùng pipeline Tool xử lý các Tool gốc.

```text
LLM
  |
  | yêu cầu gọi một Tool
  v
Router Tool của Agent
  |
  +-- Tool gốc  -> handler Python cục bộ
  |
  +-- MCP Tool  -> server MCP bên ngoài
                        |
                        v
                    trả về kết quả
```

## Đọc Cùng Nhau

- Nếu bạn muốn hiểu cách MCP phù hợp với bề mặt khả năng rộng hơn ngoài chỉ các Tool (resources, prompts, khám phá plugin), [`s19a-mcp-capability-layers.md`](./s19a-mcp-capability-layers.md) bao gồm ranh giới nền tảng đầy đủ.
- Nếu bạn muốn xác nhận rằng các khả năng bên ngoài vẫn trả về qua cùng bề mặt thực thi như Tool gốc, hãy kết hợp chương này với [`s02b-tool-execution-runtime.md`](./s02b-tool-execution-runtime.md).
- Nếu kiểm soát Query và định tuyến khả năng bên ngoài đang trôi dạt trong mô hình tư duy của bạn, [`s00a-query-control-plane.md`](./s00a-query-control-plane.md) kết nối chúng lại.

## Cách Hoạt Động

Có ba phần thiết yếu. Khi bạn hiểu chúng, MCP sẽ không còn bí ẩn nữa.

**Bước 1.** Xây dựng `MCPClient` quản lý kết nối đến một server bên ngoài. Nó khởi động tiến trình server qua stdio, gửi handshake và cache danh sách Tool có sẵn.

```python
class MCPClient:
    def __init__(self, server_name, command, args=None, env=None):
        self.server_name = server_name
        self.command = command
        self.args = args or []
        self.process = None
        self._tools = []

    def connect(self):
        self.process = subprocess.Popen(
            [self.command] + self.args,
            stdin=subprocess.PIPE, stdout=subprocess.PIPE,
            stderr=subprocess.PIPE, text=True,
        )
        self._send({"method": "initialize", "params": {
            "protocolVersion": "2024-11-05",
            "capabilities": {},
            "clientInfo": {"name": "teaching-agent", "version": "1.0"},
        }})
        response = self._recv()
        if response and "result" in response:
            self._send({"method": "notifications/initialized"})
            return True
        return False

    def list_tools(self):
        self._send({"method": "tools/list", "params": {}})
        response = self._recv()
        if response and "result" in response:
            self._tools = response["result"].get("tools", [])
        return self._tools

    def call_tool(self, tool_name, arguments):
        self._send({"method": "tools/call", "params": {
            "name": tool_name, "arguments": arguments,
        }})
        response = self._recv()
        if response and "result" in response:
            content = response["result"].get("content", [])
            return "\n".join(c.get("text", str(c)) for c in content)
        return "MCP Error: no response"
```

**Bước 2.** Chuẩn hóa tên Tool bên ngoài với tiền tố để chúng không bao giờ xung đột với Tool gốc. Quy ước đơn giản: `mcp__{server}__{tool}`.

```text
mcp__postgres__query
mcp__browser__open_tab
```

Tiền tố này phục vụ mục đích kép: nó ngăn xung đột tên, và nó cho router biết chính xác server nào nên xử lý lời gọi.

```python
def get_agent_tools(self):
    agent_tools = []
    for tool in self._tools:
        prefixed_name = f"mcp__{self.server_name}__{tool['name']}"
        agent_tools.append({
            "name": prefixed_name,
            "description": tool.get("description", ""),
            "input_schema": tool.get("inputSchema", {
                "type": "object", "properties": {}
            }),
        })
    return agent_tools
```

**Bước 3.** Xây dựng một router thống nhất. Router không quan tâm liệu một Tool là gốc hay bên ngoài ngoài quyết định điều phối. Nếu tên bắt đầu bằng `mcp__`, định tuyến đến server MCP; ngược lại, gọi handler cục bộ. Điều này giữ vòng lặp Agent không bị thay đổi -- nó chỉ thấy một danh sách Tool phẳng.

```python
if tool_name.startswith("mcp__"):
    return mcp_router.call(tool_name, arguments)
else:
    return native_handler(arguments)
```

**Bước 4.** Thêm khám phá plugin. Nếu MCP trả lời "Agent giao tiếp với một server khả năng bên ngoài như thế nào," thì plugin trả lời "các server đó được khám phá và cấu hình như thế nào?" Một plugin tối giản là một file manifest cho harness biết các server nào cần khởi động:

```json
{
  "name": "my-db-tools",
  "version": "1.0.0",
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"]
    }
  }
}
```

File này nằm trong `.claude-plugin/plugin.json`. `PluginLoader` quét các manifest này, trích xuất cấu hình server và chuyển chúng cho `MCPToolRouter` để kết nối.

**Bước 5.** Thực thi ranh giới an toàn. Đây là quy tắc quan trọng nhất của toàn bộ chương: các Tool bên ngoài vẫn phải đi qua cùng cổng quyền như Tool gốc. Nếu MCP Tool bỏ qua kiểm tra quyền, bạn đã tạo ra một backdoor bảo mật ở rìa hệ thống của bạn.

```python
decision = permission_gate.check(block.name, block.input or {})
# Cùng kiểm tra cho "bash", "read_file", và "mcp__postgres__query"
```

## Cách Kết Nối Với Harness Đầy Đủ

MCP trở nên khó hiểu khi nó được xem như một vũ trụ riêng biệt. Mô hình gọn gàng hơn là:

```text
khởi động
  ->
plugin loader tìm các manifest
  ->
cấu hình server được trích xuất
  ->
MCP clients kết nối và liệt kê Tool
  ->
Tool bên ngoài được chuẩn hóa vào cùng pool Tool

runtime
  ->
LLM phát ra tool_use
  ->
cổng quyền dùng chung
  ->
tuyến gốc hoặc tuyến MCP
  ->
chuẩn hóa kết quả
  ->
tool_result trả về cùng vòng lặp
```

Điểm vào khác nhau, cùng mặt phẳng điều khiển và mặt phẳng thực thi.

## Plugin vs Server vs Tool

| Tầng | Nó là gì | Nó dùng để làm gì |
|---|---|---|
| plugin manifest | một khai báo cấu hình | cho harness biết server nào cần khám phá và khởi động |
| MCP server | một tiến trình / kết nối bên ngoài | expose một tập hợp các khả năng |
| MCP tool | một khả năng có thể gọi từ server đó | thứ cụ thể mà mô hình gọi |

Ghi nhớ ngắn nhất:

- plugin = khám phá
- server = kết nối
- tool = gọi thực thi

## Cấu Trúc Dữ Liệu Chính

### Cấu hình server

```python
{
    "command": "npx",
    "args": ["-y", "..."],
    "env": {}
}
```

### Định nghĩa Tool bên ngoài đã được chuẩn hóa

```python
{
    "name": "mcp__postgres__query",
    "description": "Run a SQL query",
    "input_schema": {...}
}
```

### Registry client

```python
clients = {
    "postgres": mcp_client_instance
}
```

## Những Gì Thay Đổi Từ s18

| Thành phần           | Trước (s18)                          | Sau (s19)                                            |
|----------------------|--------------------------------------|------------------------------------------------------|
| Nguồn Tool           | Tất cả gốc (Python cục bộ)          | Gốc + server MCP bên ngoài                          |
| Đặt tên Tool         | Tên phẳng (`bash`, `read_file`)     | Có tiền tố cho Tool ngoài (`mcp__postgres__query`)  |
| Định tuyến           | Map handler đơn                      | Router thống nhất: dispatch gốc + dispatch MCP      |
| Phát triển khả năng  | Chỉnh sửa code harness mỗi Tool     | Thêm plugin manifest hoặc kết nối server            |
| Phạm vi quyền        | Chỉ Tool gốc                        | Tool gốc + Tool bên ngoài qua cùng cổng             |

## Thử Nghiệm

```sh
cd learn-claude-code
python agents/s19_mcp_plugin.py
```

1. Quan sát cách các Tool bên ngoài được khám phá từ plugin manifest khi khởi động.
2. Gõ `/tools` để xem Tool gốc và MCP được liệt kê cạnh nhau trong một pool phẳng.
3. Gõ `/mcp` để xem server MCP nào đang được kết nối và mỗi cái cung cấp bao nhiêu Tool.
4. Yêu cầu Agent sử dụng một Tool và chú ý cách kết quả trả về qua cùng vòng lặp như Tool cục bộ.

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Kết nối với các server khả năng bên ngoài sử dụng giao thức MCP stdio
- Chuẩn hóa tên Tool bên ngoài với tiền tố `mcp__{server}__{tool}` để ngăn xung đột
- Định tuyến các lời gọi Tool qua một dispatcher thống nhất xử lý cả Tool gốc và MCP
- Khám phá và khởi động server MCP tự động thông qua plugin manifest
- Thực thi cùng kiểm tra quyền trên Tool bên ngoài như trên Tool gốc

## Toàn Cảnh

Bây giờ bạn đã đi qua toàn bộ xương sống thiết kế của một Agent lập trình sản xuất, từ s01 đến s19.

Bạn bắt đầu với một Agent Loop đơn giản gọi LLM và ghi thêm kết quả Tool. Bạn đã thêm sử dụng Tool, sau đó là danh sách tác vụ bền vững, rồi Subagent, tải skill và nén Context. Bạn đã xây dựng hệ thống quyền, hệ thống Hook và hệ thống Memory. Bạn đã xây dựng pipeline System Prompt, thêm khôi phục lỗi và cung cấp cho các Agent một bảng tác vụ đầy đủ với thực thi nền và lập lịch cron. Bạn đã tổ chức các Agent thành nhóm với giao thức phối hợp, làm chúng tự chủ, cung cấp cho mỗi tác vụ Worktree cô lập riêng, và cuối cùng mở cánh cửa đến các khả năng bên ngoài thông qua MCP.

Mỗi chương thêm chính xác một ý tưởng vào hệ thống. Không có chương nào yêu cầu bạn bỏ đi những gì đã làm trước đó. Agent bạn có bây giờ không phải là đồ chơi -- đó là một mô hình làm việc của cùng các quyết định kiến trúc định hình các Agent sản xuất thực.

Nếu bạn muốn kiểm tra sự hiểu biết của mình, hãy thử xây dựng lại hệ thống hoàn chỉnh từ đầu. Bắt đầu với Agent Loop. Thêm Tool. Thêm tác vụ. Tiếp tục cho đến khi bạn đến MCP. Nếu bạn có thể làm điều đó mà không cần nhìn lại các chương, bạn hiểu thiết kế. Và nếu bạn bị kẹt ở đâu đó giữa chừng, chương bao gồm ý tưởng đó sẽ đang chờ bạn.

## Điểm Mấu Chốt

> Các khả năng bên ngoài nên đi vào cùng pipeline Tool như Tool gốc -- cùng đặt tên, cùng định tuyến, cùng quyền -- để vòng lặp Agent không bao giờ cần biết sự khác biệt.
