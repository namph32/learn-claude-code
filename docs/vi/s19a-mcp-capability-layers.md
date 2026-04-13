# s19a: Các Tầng Khả Năng MCP

> **Deep Dive** -- Nên đọc cùng với s19. Cho thấy rằng MCP không chỉ là các Tool bên ngoài.

### Khi Nào Nên Đọc Tài Liệu Này

Sau khi đọc cách tiếp cận ưu tiên Tool của s19, khi bạn sẵn sàng xem toàn bộ ngăn xếp khả năng MCP.

---

> `s19` vẫn nên giữ mainline ưu tiên Tool.
> Ghi chú cầu nối này bổ sung mô hình tư duy thứ hai:
>
> **MCP không chỉ là truy cập Tool bên ngoài. Đó là một ngăn xếp các tầng khả năng.**

## Cách Đọc Tài Liệu Này Cùng Mainline

Nếu bạn muốn học MCP mà không bị lạc khỏi mục tiêu giảng dạy:

- đọc [`s19-mcp-plugin.md`](./s19-mcp-plugin.md) trước và giữ rõ đường dẫn ưu tiên Tool
- sau đó bạn có thể thấy hữu ích khi xem lại [`s02a-tool-control-plane.md`](./s02a-tool-control-plane.md) để xem khả năng bên ngoài quay trở lại bus Tool thống nhất như thế nào
- nếu các bản ghi trạng thái bắt đầu bị nhòe, bạn có thể thấy hữu ích khi xem lại [`data-structures.md`](./data-structures.md)
- nếu các ranh giới khái niệm bị nhòe, bạn có thể thấy hữu ích khi xem lại [`glossary.md`](./glossary.md) và [`entity-map.md`](./entity-map.md)

## Tại Sao Điều Này Xứng Đáng Có Ghi Chú Cầu Nối Riêng

Đối với một repo giảng dạy, giữ mainline tập trung vào Tool bên ngoài trước là đúng.

Đó là điểm vào dễ nhất:

- kết nối một server bên ngoài
- nhận các định nghĩa Tool
- gọi một Tool
- đưa kết quả trở lại Agent

Nhưng nếu bạn muốn hình dạng hệ thống tiếp cận hành vi hoàn thành cao thực sự, bạn nhanh chóng gặp các câu hỏi sâu hơn:

- server có được kết nối qua stdio, HTTP, SSE hay WebSocket không
- tại sao một số server là `connected`, trong khi những server khác là `pending` hoặc `needs-auth`
- resources và prompts phù hợp với các Tool ở đâu
- tại sao elicitation trở thành một loại tương tác đặc biệt
- OAuth hoặc các luồng xác thực khác nên được đặt ở đâu về mặt khái niệm

Không có bản đồ tầng khả năng, MCP bắt đầu có vẻ rời rạc.

## Các Thuật Ngữ Trước

### Tầng khả năng có nghĩa là gì

Một tầng khả năng đơn giản là:

> một lát trách nhiệm trong một hệ thống lớn hơn

Mục đích là tránh trộn mọi mối quan tâm MCP vào một túi.

### Transport có nghĩa là gì

Transport là kênh kết nối giữa Agent của bạn và một server MCP:

- stdio (đầu vào/đầu ra tiêu chuẩn, tốt cho các tiến trình cục bộ)
- HTTP
- SSE (Server-Sent Events, một giao thức streaming một chiều qua HTTP)
- WebSocket

### Elicitation có nghĩa là gì

Đây là một trong những thuật ngữ ít quen thuộc hơn.

Định nghĩa giảng dạy đơn giản là:

> một tương tác trong đó server MCP yêu cầu người dùng nhập thêm thông tin trước khi có thể tiếp tục

Vì vậy hệ thống không chỉ còn là:

> Agent gọi Tool -> Tool trả về kết quả

Server cũng có thể nói:

> Tôi cần thêm thông tin trước khi có thể hoàn thành

Điều này biến một lời gọi và trả về đơn giản thành một cuộc hội thoại nhiều bước giữa Agent và server.

## Mô Hình Tư Duy Tối Thiểu

Một bức tranh sáu tầng rõ ràng:

```text
1. Tầng Config
   cấu hình server trông như thế nào

2. Tầng Transport
   kết nối server được truyền như thế nào

3. Tầng Trạng Thái Kết Nối
   connected / pending / failed / needs-auth

4. Tầng Khả Năng
   tools / resources / prompts / elicitation

5. Tầng Xác Thực
   xác thực có cần thiết không và nó ở trạng thái gì

6. Tầng Tích Hợp Router
   cách MCP định tuyến lại vào định tuyến Tool, quyền và thông báo
```

Bài học chính là:

**Tool là một tầng, không phải toàn bộ câu chuyện MCP**

## Tại Sao Mainline Vẫn Nên Ưu Tiên Tool

Điều này rất quan trọng cho việc giảng dạy.

Mặc dù MCP chứa nhiều tầng, mainline của chương vẫn nên dạy:

### Bước 1: Tool bên ngoài trước

Vì điều đó kết nối tự nhiên nhất với mọi thứ bạn đã học:

- Tool cục bộ
- Tool bên ngoài
- một router dùng chung

### Bước 2: cho thấy có nhiều tầng khả năng hơn tồn tại

Ví dụ:

- resources
- prompts
- elicitation
- auth

### Bước 3: quyết định tầng nâng cao nào repo thực sự nên triển khai

Điều đó phù hợp với mục tiêu giảng dạy:

**xây dựng hệ thống tương tự trước, sau đó thêm các tầng nền tảng nặng hơn**

## Các Bản Ghi Cốt Lõi

### 1. `ScopedMcpServerConfig`

Ngay cả một phiên bản giảng dạy tối giản cũng nên expose ý tưởng này:

```python
config = {
    "name": "postgres",
    "type": "stdio",
    "command": "npx",
    "args": ["-y", "..."],
    "scope": "project",
}
```

`scope` quan trọng vì cấu hình server có thể đến từ các nơi khác nhau (cài đặt người dùng toàn cục, cài đặt cấp dự án, hoặc thậm chí ghi đè theo workspace).

### 2. Trạng thái kết nối MCP

```python
server_state = {
    "name": "postgres",
    "status": "connected",   # pending / failed / needs-auth / disabled
    "config": {...},
}
```

### 3. `MCPToolSpec`

```python
tool = {
    "name": "mcp__postgres__query",
    "description": "...",
    "input_schema": {...},
}
```

### 4. `ElicitationRequest`

```python
request = {
    "server_name": "some-server",
    "message": "Please provide additional input",
    "requested_schema": {...},
}
```

Điểm giảng dạy không phải là bạn cần triển khai elicitation ngay lập tức.

Điểm là:

**MCP không được đảm bảo luôn là một lời gọi Tool một chiều**

## Bức Tranh Nền Tảng Gọn Gàng Hơn

```text
MCP Config
  |
  v
Transport
  |
  v
Trạng Thái Kết Nối
  |
  +-- connected
  +-- pending
  +-- needs-auth
  +-- failed
  |
  v
Khả Năng
  +-- tools
  +-- resources
  +-- prompts
  +-- elicitation
  |
  v
Tích Hợp Router / Quyền / Thông Báo
```

## Tại Sao Xác Thực Không Nên Chi Phối Mainline Của Chương

Auth là một tầng thực trong nền tảng đầy đủ.

Nhưng nếu mainline sa lào vào chi tiết OAuth hoặc luồng auth đặc thù của nhà cung cấp quá sớm, người mới học mất hình dạng hệ thống thực sự.

Thứ tự giảng dạy tốt hơn là:

- đầu tiên giải thích rằng một tầng auth tồn tại
- sau đó giải thích rằng `connected` và `needs-auth` là các trạng thái kết nối khác nhau
- chỉ sau đó, trong công việc nền tảng nâng cao, mở rộng máy trạng thái auth đầy đủ

Điều đó giữ repo trung thực mà không làm lạc đường học tập của bạn.

## Cách Điều Này Liên Quan Đến `s19` và `s02a`

- chương `s19` tiếp tục dạy đường dẫn khả năng bên ngoài ưu tiên Tool
- ghi chú này cung cấp bản đồ nền tảng rộng hơn
- `s02a` giải thích cách khả năng MCP cuối cùng kết nối lại với mặt phẳng điều khiển Tool thống nhất

Cùng nhau, chúng dạy ý tưởng thực sự:

**MCP là một nền tảng khả năng bên ngoài, và Tool chỉ là bộ mặt đầu tiên của nó đi vào mainline**

## Các Lỗi Thường Gặp Của Người Mới Học

### 1. Xem MCP chỉ là một danh mục Tool bên ngoài

Điều đó làm cho resources, prompts, auth và elicitation có vẻ đáng ngạc nhiên sau này.

### 2. Đi sâu vào chi tiết transport hoặc OAuth quá sớm

Điều đó phá vỡ mainline giảng dạy.

### 3. Cho phép MCP Tool bỏ qua kiểm tra quyền

Điều đó mở ra một cửa hậu nguy hiểm trong ranh giới hệ thống.

### 4. Trộn cấu hình server, trạng thái kết nối và các khả năng được expose vào một khối

Các tầng đó nên được giữ tách biệt về mặt khái niệm.

## Điểm Mấu Chốt

**MCP là một nền tảng khả năng sáu tầng. Tool là tầng đầu tiên bạn xây dựng, nhưng resources, prompts, elicitation, auth và tích hợp router đều là một phần của bức tranh đầy đủ.**
