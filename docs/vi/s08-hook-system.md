# s08: Hook System

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > [ s08 ] > s09 > s10 > s11 > s12 > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Ba sự kiện vòng đời cho phép code bên ngoài quan sát và ảnh hưởng đến Agent Loop
- Cách các Hook dạng shell chạy như các tiến trình con với đầy đủ Context về lệnh gọi Tool hiện tại
- Giao thức exit code: 0 nghĩa là tiếp tục, 1 nghĩa là chặn, 2 nghĩa là chèn vào một tin nhắn
- Cách cấu hình Hook trong một tệp JSON bên ngoài để bạn không bao giờ cần chạm vào code của vòng lặp chính

Agent của bạn từ s07 có hệ thống quyền kiểm soát những gì nó được phép làm. Nhưng quyền là một cổng có/không -- chúng không cho phép bạn thêm hành vi mới. Giả sử bạn muốn mọi lệnh bash được ghi vào tệp kiểm toán, hoặc bạn muốn một linter chạy tự động sau mỗi lần ghi tệp, hoặc bạn muốn một trình quét bảo mật tùy chỉnh kiểm tra đầu vào Tool trước khi chúng thực thi. Bạn có thể thêm các nhánh if/else bên trong vòng lặp chính cho mỗi trường hợp này, nhưng điều đó biến vòng lặp sạch sẽ của bạn thành một mớ các trường hợp đặc biệt. Điều bạn thực sự muốn là một cách mở rộng hành vi của Agent từ bên ngoài, mà không sửa đổi vòng lặp.

## Vấn Đề

Bạn đang chạy Agent của mình trong môi trường nhóm. Các nhóm khác nhau muốn các hành vi khác nhau: nhóm bảo mật muốn quét mọi lệnh bash, nhóm QA muốn tự động chạy kiểm thử sau khi chỉnh sửa tệp, và nhóm ops muốn có audit trail của mọi lệnh gọi Tool. Nếu mỗi yêu cầu này đòi hỏi thay đổi code trong Agent Loop, bạn sẽ có một mớ hỗn độn các điều kiện mà không ai có thể duy trì. Tệ hơn nữa, mọi yêu cầu mới đồng nghĩa với việc triển khai lại Agent. Bạn cần một cách để các nhóm có thể cắm logic của riêng họ vào những thời điểm được xác định rõ ràng -- mà không cần chạm vào code cốt lõi.

## Giải Pháp

Agent Loop công khai ba điểm mở rộng cố định (sự kiện vòng đời). Tại mỗi điểm, nó chạy các lệnh shell bên ngoài được gọi là Hook. Mỗi Hook truyền đạt ý định của nó thông qua exit code: tiếp tục im lặng, chặn thao tác, hoặc chèn vào một tin nhắn vào cuộc hội thoại.

```
tool_call from LLM
     |
     v
[PreToolUse hooks]
     |  exit 0 -> tiếp tục
     |  exit 1 -> chặn tool, trả về stderr như lỗi
     |  exit 2 -> chèn stderr vào cuộc hội thoại, tiếp tục
     |
     v
[execute tool]
     |
     v
[PostToolUse hooks]
     |  exit 0 -> tiếp tục
     |  exit 2 -> gắn thêm stderr vào kết quả
     |
     v
return result
```

## Đọc Cùng Nhau

- Nếu bạn vẫn hình dung Hook như "thêm các nhánh if/else bên trong vòng lặp chính," bạn có thể thấy hữu ích khi xem lại [`s02a-tool-control-plane.md`](./s02a-tool-control-plane.md) trước.
- Nếu vòng lặp chính, tool handler, và các tác dụng phụ của Hook bắt đầu mờ đi với nhau, [`entity-map.md`](./entity-map.md) có thể giúp bạn phân tách ai tiến hành trạng thái cốt lõi và ai chỉ quan sát từ phía.
- Nếu bạn định tiếp tục vào lắp ráp prompt, khôi phục, hoặc các nhóm, hãy giữ [`s00e-reference-module-map.md`](./s00e-reference-module-map.md) ở gần vì mẫu "vòng lặp cốt lõi cộng với mở rộng sidecar" này xuất hiện lặp đi lặp lại.

## Cách Hoạt Động

**Bước 1.** Định nghĩa ba sự kiện vòng đời. `SessionStart` kích hoạt một lần khi Agent khởi động -- hữu ích cho khởi tạo, logging, hoặc kiểm tra môi trường. `PreToolUse` kích hoạt trước mỗi lệnh gọi Tool và là sự kiện duy nhất có thể chặn thực thi. `PostToolUse` kích hoạt sau mỗi lệnh gọi Tool và có thể chú thích kết quả nhưng không thể hoàn tác nó.

| Sự kiện | Khi nào | Có thể chặn? |
|---------|---------|-------------|
| `SessionStart` | Một lần khi bắt đầu phiên | Không |
| `PreToolUse` | Trước mỗi lệnh gọi Tool | Có (exit 1) |
| `PostToolUse` | Sau mỗi lệnh gọi Tool | Không |

**Bước 2.** Cấu hình Hook trong tệp `.hooks.json` bên ngoài ở thư mục gốc workspace. Mỗi Hook chỉ định một lệnh shell để chạy. Trường `matcher` tùy chọn lọc theo tên Tool -- nếu không có matcher, Hook kích hoạt cho mọi Tool.

```json
{
  "hooks": {
    "PreToolUse": [
      {"matcher": "bash", "command": "echo 'Checking bash command...'"},
      {"matcher": "write_file", "command": "/path/to/lint-check.sh"}
    ],
    "PostToolUse": [
      {"command": "echo 'Tool finished'"}
    ],
    "SessionStart": [
      {"command": "echo 'Session started at $(date)'"}
    ]
  }
}
```

**Bước 3.** Triển khai giao thức exit code. Đây là trái tim của Hook system -- ba exit code, ba ý nghĩa. Giao thức này được thiết kế đơn giản có chủ ý để bất kỳ ngôn ngữ hay script nào cũng có thể tham gia. Viết Hook của bạn bằng bash, Python, Ruby, bất cứ thứ gì -- miễn là nó thoát với code đúng.

| Exit Code | Ý nghĩa | PreToolUse | PostToolUse |
|-----------|---------|-----------|------------|
| 0 | Thành công | Tiếp tục thực thi Tool | Tiếp tục bình thường |
| 1 | Chặn | Tool KHÔNG được thực thi, stderr được trả về như lỗi | Cảnh báo được ghi lại |
| 2 | Chèn vào | stderr được chèn vào như tin nhắn, Tool vẫn thực thi | stderr được gắn thêm vào kết quả |

**Bước 4.** Truyền Context cho Hook qua biến môi trường. Hook cần biết điều gì đang xảy ra -- sự kiện nào kích hoạt chúng, Tool nào đang được gọi, và đầu vào trông như thế nào. Đối với Hook `PostToolUse`, đầu ra của Tool cũng có sẵn.

```
HOOK_EVENT=PreToolUse
HOOK_TOOL_NAME=bash
HOOK_TOOL_INPUT={"command": "npm test"}
HOOK_TOOL_OUTPUT=...  (PostToolUse only)
```

**Bước 5.** Tích hợp Hook vào Agent Loop. Việc tích hợp gọn gàng: chạy pre-hook trước khi thực thi, kiểm tra nếu có cái nào chặn, thực thi Tool, chạy post-hook, và thu thập các tin nhắn được chèn vào. Vòng lặp vẫn sở hữu luồng kiểm soát -- Hook chỉ quan sát, chặn, hoặc chú thích tại các thời điểm được đặt tên.

```python
# Before tool execution
pre_result = hooks.run_hooks("PreToolUse", ctx)
if pre_result["blocked"]:
    output = f"Blocked by hook: {pre_result['block_reason']}"
    continue

# Execute tool
output = handler(**tool_input)

# After tool execution
post_result = hooks.run_hooks("PostToolUse", ctx)
for msg in post_result["messages"]:
    output += f"\n[Hook note]: {msg}"
```

## Những Thay Đổi So Với s07

| Thành phần | Trước (s07) | Sau (s08) |
|------------|------------|----------|
| Khả năng mở rộng | Không có | Hook system dựa trên shell |
| Sự kiện | Không có | PreToolUse, PostToolUse, SessionStart |
| Luồng kiểm soát | Chỉ quy trình quyền | Quyền + Hook |
| Cấu hình | Quy tắc trong code | Tệp `.hooks.json` bên ngoài |

## Thử Nghiệm

```sh
cd learn-claude-code
# Create a hook config
cat > .hooks.json << 'EOF'
{
  "hooks": {
    "PreToolUse": [
      {"matcher": "bash", "command": "echo 'Auditing bash command' >&2; exit 0"}
    ],
    "SessionStart": [
      {"command": "echo 'Agent session started'"}
    ]
  }
}
EOF
python agents/s08_hook_system.py
```

1. Xem Hook SessionStart kích hoạt khi khởi động
2. Yêu cầu Agent chạy lệnh bash -- xem Hook PreToolUse kích hoạt
3. Tạo Hook chặn (exit 1) và xem nó ngăn thực thi Tool
4. Tạo Hook chèn vào (exit 2) và xem nó thêm tin nhắn vào cuộc hội thoại

## Những Gì Bạn Đã Thành Thạo

Tại thời điểm này, bạn có thể:

- Giải thích tại sao các điểm mở rộng tốt hơn các điều kiện trong vòng lặp để thêm hành vi mới
- Định nghĩa các sự kiện vòng đời tại các thời điểm đúng trong Agent Loop
- Viết Hook shell truyền đạt ý định thông qua giao thức exit ba code
- Cấu hình Hook bên ngoài để các nhóm khác nhau có thể tùy chỉnh hành vi mà không cần chạm vào code Agent
- Duy trì ranh giới: vòng lặp sở hữu luồng kiểm soát, handler sở hữu thực thi, Hook chỉ quan sát, chặn, hoặc chú thích

## Tiếp Theo

Agent của bạn giờ đây có thể thực thi Tool một cách an toàn (s07) và được mở rộng mà không cần thay đổi code (s08). Nhưng nó vẫn mắc chứng mất trí nhớ -- mỗi phiên mới bắt đầu từ con số không. Sở thích của người dùng, các sửa chỉnh, và Context dự án bị quên ngay khi phiên kết thúc. Trong s09, bạn sẽ xây dựng một Memory system cho phép Agent mang các sự kiện bền vững qua các phiên.

## Điểm Mấu Chốt

> Vòng lặp chính có thể công khai các điểm mở rộng cố định mà không từ bỏ quyền sở hữu luồng kiểm soát -- Hook quan sát, chặn, hoặc chú thích, nhưng vòng lặp vẫn quyết định điều gì xảy ra tiếp theo.
