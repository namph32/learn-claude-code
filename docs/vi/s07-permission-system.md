# s07: Hệ Thống Quyền

`s01 > s02 > s03 > s04 > s05 > s06 > [ s07 ] > s08 > s09 > s10 > s11 > s12 > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Một quy trình quyền bốn giai đoạn mà mọi lệnh gọi Tool đều phải vượt qua trước khi thực thi
- Ba chế độ quyền kiểm soát mức độ Agent tự động phê duyệt các hành động
- Cách quy tắc từ chối và cho phép sử dụng khớp mẫu để tạo ra chính sách ưu tiên kết quả đầu tiên
- Phê duyệt tương tác với tùy chọn "always" ghi các quy tắc cho phép vĩnh viễn vào Runtime

Agent của bạn từ s06 đã có năng lực cao và hoạt động lâu dài. Nó đọc tệp, viết code, chạy lệnh shell, ủy thác tác vụ con, và nén Context của chính nó để tiếp tục hoạt động. Nhưng vẫn chưa có cơ chế bảo vệ an toàn. Mọi lệnh gọi Tool mà mô hình đề xuất đều đi thẳng vào thực thi. Yêu cầu nó xóa một thư mục và nó sẽ làm -- không hỏi thêm. Trước khi bạn cấp cho Agent này quyền truy cập vào bất kỳ thứ gì quan trọng, bạn cần một cổng kiểm soát giữa "mô hình muốn làm X" và "hệ thống thực sự làm X."

## Vấn Đề

Hãy tưởng tượng Agent của bạn đang giúp tái cấu trúc một codebase. Nó đọc một vài tệp, đề xuất một số chỉnh sửa, rồi quyết định chạy `rm -rf /tmp/old_build` để dọn dẹp. Ngoại trừ mô hình đã ảo giác về đường dẫn -- thư mục thực tế là thư mục home của bạn. Hoặc nó quyết định `sudo` một thứ gì đó vì mô hình đã thấy mẫu đó trong dữ liệu huấn luyện. Nếu không có tầng quyền, ý định sẽ ngay lập tức trở thành thực thi. Không có khoảnh khắc nào để hệ thống có thể nói "chờ đã, điều đó có vẻ nguy hiểm" hay để bạn có thể nói "không, đừng làm vậy." Agent cần một điểm kiểm tra -- một quy trình (chuỗi các giai đoạn mà mọi yêu cầu đều phải đi qua) giữa những gì mô hình yêu cầu và những gì thực sự xảy ra.

## Giải Pháp

Mọi lệnh gọi Tool giờ đây phải đi qua quy trình quyền bốn giai đoạn trước khi thực thi. Các giai đoạn chạy theo thứ tự, và giai đoạn đầu tiên tạo ra câu trả lời dứt khoát sẽ thắng.

```
tool_call from LLM
     |
     v
[1. Deny rules]     -- danh sách đen: luôn chặn những yêu cầu này
     |
     v
[2. Mode check]     -- chế độ plan? chế độ auto? mặc định?
     |
     v
[3. Allow rules]    -- danh sách trắng: luôn cho phép những yêu cầu này
     |
     v
[4. Ask user]       -- prompt tương tác y/n/always
     |
     v
execute (or reject)
```

## Đọc Cùng Nhau

- Nếu bạn bắt đầu nhầm lẫn "mô hình đề xuất một hành động" với "hệ thống thực sự thực thi một hành động," bạn có thể thấy hữu ích khi xem lại [`s00a-query-control-plane.md`](./s00a-query-control-plane.md).
- Nếu bạn chưa rõ tại sao các yêu cầu Tool không nên đi thẳng vào các handler, hãy giữ [`s02a-tool-control-plane.md`](./s02a-tool-control-plane.md) bên cạnh chương này.
- Nếu `PermissionRule`, `PermissionDecision`, và `tool_result` bắt đầu hợp nhất thành một ý tưởng mơ hồ, [`data-structures.md`](./data-structures.md) có thể giúp bạn phân tách chúng.

## Cách Hoạt Động

**Bước 1.** Định nghĩa ba chế độ quyền. Mỗi chế độ thay đổi cách quy trình xử lý các lệnh gọi Tool không khớp với bất kỳ quy tắc rõ ràng nào. Chế độ "default" là an toàn nhất -- nó hỏi bạn về mọi thứ. Chế độ "plan" chặn tất cả thao tác ghi hoàn toàn, hữu ích khi bạn muốn Agent khám phá mà không chạm vào bất kỳ thứ gì. Chế độ "auto" cho phép thao tác đọc qua im lặng và chỉ hỏi về thao tác ghi, phù hợp cho khám phá nhanh.

| Chế độ | Hành vi | Trường hợp sử dụng |
|--------|---------|-------------------|
| `default` | Hỏi người dùng cho mọi lệnh gọi Tool không khớp | Sử dụng tương tác thông thường |
| `plan` | Chặn tất cả thao tác ghi, cho phép đọc | Chế độ lập kế hoạch/xem xét |
| `auto` | Tự động cho phép đọc, hỏi về thao tác ghi | Chế độ khám phá nhanh |

**Bước 2.** Thiết lập quy tắc từ chối và cho phép với khớp mẫu. Các quy tắc được kiểm tra theo thứ tự -- kết quả khớp đầu tiên thắng. Quy tắc từ chối bắt các mẫu nguy hiểm không bao giờ được thực thi, bất kể chế độ nào. Quy tắc cho phép để các thao tác an toàn đã biết qua mà không cần hỏi.

```python
rules = [
    # Always deny dangerous patterns
    {"tool": "bash", "content": "rm -rf /", "behavior": "deny"},
    {"tool": "bash", "content": "sudo *",   "behavior": "deny"},
    # Allow reading anything
    {"tool": "read_file", "path": "*", "behavior": "allow"},
]
```

Khi người dùng trả lời "always" tại prompt tương tác, một quy tắc cho phép vĩnh viễn được thêm vào trong Runtime.

**Bước 3.** Triển khai kiểm tra bốn giai đoạn. Đây là cốt lõi của hệ thống quyền. Lưu ý rằng quy tắc từ chối chạy trước và không thể bị vượt qua -- điều này là có chủ ý. Bất kể bạn đang ở chế độ nào hay quy tắc cho phép nào tồn tại, quy tắc từ chối luôn thắng.

```python
def check(self, tool_name, tool_input):
    # Step 1: Deny rules (bypass-immune, always checked first)
    for rule in self.rules:
        if rule["behavior"] == "deny" and self._matches(rule, ...):
            return {"behavior": "deny", "reason": "..."}

    # Step 2: Mode-based decisions
    if self.mode == "plan" and tool_name in WRITE_TOOLS:
        return {"behavior": "deny", "reason": "Plan mode: writes blocked"}
    if self.mode == "auto" and tool_name in READ_ONLY_TOOLS:
        return {"behavior": "allow", "reason": "Auto: read-only approved"}

    # Step 3: Allow rules
    for rule in self.rules:
        if rule["behavior"] == "allow" and self._matches(rule, ...):
            return {"behavior": "allow", "reason": "..."}

    # Step 4: Fall through to ask user
    return {"behavior": "ask", "reason": "..."}
```

**Bước 4.** Tích hợp kiểm tra quyền vào Agent Loop. Mọi lệnh gọi Tool giờ đây đi qua quy trình trước khi thực thi. Kết quả là một trong ba kết quả: bị từ chối (kèm lý do), được cho phép (im lặng), hoặc được hỏi (tương tác).

```python
for block in response.content:
    if block.type == "tool_use":
        decision = perms.check(block.name, block.input)

        if decision["behavior"] == "deny":
            output = f"Permission denied: {decision['reason']}"
        elif decision["behavior"] == "ask":
            if perms.ask_user(block.name, block.input):
                output = handler(**block.input)
            else:
                output = "Permission denied by user"
        else:  # allow
            output = handler(**block.input)

        results.append({"type": "tool_result", ...})
```

**Bước 5.** Thêm theo dõi từ chối như một bộ ngắt mạch đơn giản. `PermissionManager` theo dõi các lần từ chối liên tiếp. Sau 3 lần liên tiếp, nó gợi ý chuyển sang chế độ plan -- điều này ngăn Agent liên tục va vào cùng một bức tường và lãng phí lượt.

## Những Thay Đổi So Với s06

| Thành phần | Trước (s06) | Sau (s07) |
|------------|------------|----------|
| An toàn | Không có | Quy trình quyền 4 giai đoạn |
| Chế độ | Không có | 3 chế độ: default, plan, auto |
| Quy tắc | Không có | Quy tắc từ chối/cho phép với khớp mẫu |
| Kiểm soát người dùng | Không có | Phê duyệt tương tác với tùy chọn "always" |
| Theo dõi từ chối | Không có | Bộ ngắt mạch sau 3 lần từ chối liên tiếp |

## Thử Nghiệm

```sh
cd learn-claude-code
python agents/s07_permission_system.py
```

1. Bắt đầu ở chế độ `default` -- mọi Tool ghi đều yêu cầu phê duyệt
2. Thử chế độ `plan` -- tất cả thao tác ghi bị chặn, thao tác đọc đi qua
3. Thử chế độ `auto` -- thao tác đọc được tự động phê duyệt, thao tác ghi vẫn hỏi
4. Trả lời "always" để cho phép vĩnh viễn một Tool
5. Gõ `/mode plan` để chuyển chế độ trong Runtime
6. Gõ `/rules` để kiểm tra tập quy tắc hiện tại

## Những Gì Bạn Đã Thành Thạo

Tại thời điểm này, bạn có thể:

- Giải thích tại sao ý định của mô hình phải đi qua quy trình quyết định trước khi trở thành thực thi
- Xây dựng kiểm tra quyền bốn giai đoạn: từ chối, chế độ, cho phép, hỏi
- Cấu hình ba chế độ quyền cho bạn sự đánh đổi khác nhau giữa an toàn và tốc độ
- Thêm quy tắc động trong Runtime khi người dùng trả lời "always"
- Triển khai bộ ngắt mạch đơn giản phát hiện các vòng lặp từ chối lặp đi lặp lại

## Tiếp Theo

Hệ thống quyền của bạn kiểm soát những gì Agent được phép làm, nhưng nó hoàn toàn nằm trong code của Agent. Nếu bạn muốn mở rộng hành vi -- thêm logging, kiểm toán, hoặc xác thực tùy chỉnh -- mà không sửa đổi Agent Loop chút nào? Đó là điều s08 giới thiệu: một Hook system cho phép các shell script bên ngoài quan sát và ảnh hưởng đến mọi lệnh gọi Tool.

## Điểm Mấu Chốt

> An toàn là một quy trình, không phải một boolean -- từ chối trước, sau đó xem xét chế độ, rồi kiểm tra quy tắc cho phép, cuối cùng hỏi người dùng.
