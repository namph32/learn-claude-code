# s10: System Prompt

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > [ s10 ] > s11 > s12 > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Cách lắp ráp System Prompt từ các phần độc lập thay vì một chuỗi được mã hóa cứng
- Ranh giới giữa nội dung ổn định (vai trò, quy tắc) và nội dung động (ngày, thư mục hiện tại, nhắc nhở mỗi lượt)
- Cách các tệp CLAUDE.md tầng chồng các hướng dẫn mà không ghi đè lên nhau
- Tại sao Memory phải được tái chèn vào thông qua quy trình prompt để thực sự hướng dẫn Agent

Khi Agent của bạn chỉ có một Tool và một công việc, một chuỗi prompt được mã hóa cứng hoạt động tốt. Nhưng hãy nhìn vào mọi thứ Harness của bạn đã tích lũy đến bây giờ: mô tả vai trò, định nghĩa Tool, các kỹ năng đã tải, Memory đã lưu, các tệp hướng dẫn CLAUDE.md, và Context Runtime mỗi lượt. Nếu bạn tiếp tục nhồi nhét tất cả vào một chuỗi lớn, không ai -- kể cả bạn -- có thể biết mỗi phần đến từ đâu, tại sao nó ở đó, hoặc làm thế nào để thay đổi nó một cách an toàn. Giải pháp là dừng xử lý prompt như một khối và bắt đầu xử lý nó như một quy trình lắp ráp.

## Vấn Đề

Hãy tưởng tượng bạn muốn thêm một Tool mới vào Agent. Bạn mở System Prompt, cuộn qua đoạn vai trò, qua các quy tắc an toàn, qua ba mô tả kỹ năng, qua khối Memory, và dán mô tả Tool ở đâu đó ở giữa. Tuần sau ai đó thêm bộ tải CLAUDE.md và gắn đầu ra của nó vào cùng chuỗi. Một tháng sau prompt dài 6.000 ký tự, một nửa trong số đó đã lỗi thời, và không ai nhớ dòng nào được cho là thay đổi mỗi lượt và dòng nào nên giữ cố định trong toàn bộ phiên.

Đây không phải là tình huống giả định -- đó là quỹ đạo tự nhiên của mọi Agent giữ prompt trong một biến duy nhất.

## Giải Pháp

Biến việc xây dựng prompt thành một quy trình. Mỗi phần có một nguồn và một trách nhiệm. Một đối tượng builder lắp ráp chúng theo thứ tự cố định, với ranh giới rõ ràng giữa các phần giữ ổn định và các phần thay đổi mỗi lượt.

```text
1. core identity and rules
2. tool catalog
3. skills
4. memory
5. CLAUDE.md instruction chain
6. dynamic runtime context
```

Sau đó lắp ráp:

```text
core
+ tools
+ skills
+ memory
+ claude_md
+ dynamic_context
= final model input
```

## Cách Hoạt Động

**Bước 1. Định nghĩa builder.** Mỗi phương thức sở hữu đúng một nguồn nội dung.

```python
class SystemPromptBuilder:
    def build(self) -> str:
        parts = []
        parts.append(self._build_core())
        parts.append(self._build_tools())
        parts.append(self._build_skills())
        parts.append(self._build_memory())
        parts.append(self._build_claude_md())
        parts.append(self._build_dynamic())
        return "\n\n".join(p for p in parts if p)
```

Đó là ý tưởng trung tâm của chương. Mỗi phương thức `_build_*` chỉ lấy từ một nguồn: `_build_tools()` đọc danh sách Tool, `_build_memory()` đọc kho Memory, và cứ thế. Nếu bạn muốn biết một dòng trong prompt đến từ đâu, bạn kiểm tra phương thức duy nhất chịu trách nhiệm về nó.

**Bước 2. Tách nội dung ổn định khỏi nội dung động.** Đây là ranh giới quan trọng nhất trong toàn bộ quy trình.

Nội dung ổn định thay đổi hiếm khi hoặc không bao giờ trong một phiên:

- mô tả vai trò
- hợp đồng Tool (danh sách các Tool và Schema của chúng)
- các quy tắc an toàn lâu dài
- chuỗi hướng dẫn dự án (các tệp CLAUDE.md)

Nội dung động thay đổi mỗi lượt hoặc vài lượt:

- ngày hiện tại
- thư mục làm việc hiện tại
- chế độ hiện tại (chế độ plan, chế độ code, v.v.)
- cảnh báo hoặc nhắc nhở mỗi lượt

Trộn lẫn chúng lại có nghĩa là mô hình phải đọc lại hàng nghìn Token văn bản ổn định chưa thay đổi, trong khi vài Token thực sự thay đổi bị chôn vùi ở đâu đó ở giữa. Một hệ thống thực tế tách chúng bằng một dấu phân cách để tiền tố ổn định có thể được cache qua các lượt để tiết kiệm Token prompt.

**Bước 3. Tầng chồng hướng dẫn CLAUDE.md.** `CLAUDE.md` không giống Memory và không giống kỹ năng. Nó là một nguồn hướng dẫn phân tầng -- có nghĩa là nhiều tệp đóng góp, và các tầng sau bổ sung vào các tầng trước thay vì thay thế chúng:

1. Tệp hướng dẫn cấp người dùng (`~/.claude/CLAUDE.md`)
2. Tệp hướng dẫn gốc dự án (`<project>/CLAUDE.md`)
3. Các tệp hướng dẫn thư mục con sâu hơn

Điểm quan trọng không phải là tên tệp. Điểm quan trọng là các nguồn hướng dẫn có thể được phân tầng thay vì bị ghi đè.

**Bước 4. Tái chèn Memory.** Lưu Memory (trong s09) chỉ là một nửa cơ chế. Nếu Memory không bao giờ tái nhập đầu vào mô hình, nó thực sự không hướng dẫn Agent. Vì vậy Memory thuộc tự nhiên vào quy trình prompt:

- lưu các sự kiện bền vững trong `s09`
- tái chèn chúng thông qua prompt builder trong `s10`

**Bước 5. Gắn nhắc nhở mỗi lượt riêng biệt.** Một số thông tin còn ngắn hạn hơn cả "Context động" -- nó chỉ quan trọng cho lượt này và không nên làm ô nhiễm System Prompt ổn định. Một tin nhắn người dùng `system-reminder` giữ các tín hiệu tạm thời này hoàn toàn bên ngoài builder:

- hướng dẫn chỉ cho lượt này
- thông báo tạm thời
- hướng dẫn khôi phục tạm thời

## Những Thay Đổi So Với s09

| Khía cạnh | s09: Memory System | s10: System Prompt |
|-----------|--------------------|--------------------|
| Mối quan tâm cốt lõi | Lưu giữ các sự kiện bền vững qua các phiên | Lắp ráp tất cả các nguồn vào đầu vào mô hình |
| Vai trò của Memory | Ghi và lưu | Đọc và chèn vào |
| Cấu trúc prompt | Được giả định nhưng không được quản lý | Quy trình rõ ràng với các phần |
| Các tệp hướng dẫn | Chưa được đề cập | Giới thiệu phân tầng CLAUDE.md |
| Context động | Chưa được đề cập | Được tách khỏi nội dung ổn định |

## Đọc Cùng Nhau

- Nếu bạn vẫn coi prompt là một khối văn bản bí ẩn, hãy xem lại [`s00a-query-control-plane.md`](./s00a-query-control-plane.md) để xem những gì đến được mô hình và qua các tầng kiểm soát nào.
- Nếu bạn muốn ổn định thứ tự lắp ráp, hãy giữ [`s10a-message-prompt-pipeline.md`](./s10a-message-prompt-pipeline.md) bên cạnh chương này -- đó là ghi chú cầu nối quan trọng cho `s10`.
- Nếu các quy tắc hệ thống, tài liệu Tool, Memory, và trạng thái Runtime bắt đầu sụp đổ thành một khối đầu vào lớn, hãy thiết lập lại bằng [`data-structures.md`](./data-structures.md).

## Các Lỗi Phổ Biến Của Người Mới

**Lỗi 1: dạy prompt như một chuỗi cố định.** Điều đó ẩn cách hệ thống thực sự phát triển. Một chuỗi cố định tốt cho demo; nó ngừng tốt ngay khi bạn thêm khả năng thứ hai.

**Lỗi 2: đặt mọi chi tiết thay đổi vào cùng một khối prompt.** Điều đó trộn lẫn các quy tắc bền vững với tiếng ồn mỗi lượt. Khi bạn cập nhật một cái, bạn có nguy cơ phá vỡ cái kia.

**Lỗi 3: coi kỹ năng, Memory, và CLAUDE.md là cùng một thứ.** Chúng có thể trở thành các phần prompt, nhưng nguồn và mục đích của chúng khác nhau:

- `skills`: các gói khả năng tùy chọn được tải theo yêu cầu
- `memory`: các sự kiện liên phiên bền vững về người dùng hoặc dự án
- `CLAUDE.md`: các tài liệu hướng dẫn thường trực phân tầng mà không ghi đè

## Thử Nghiệm

```sh
cd learn-claude-code
python agents/s10_system_prompt.py
```

Tìm kiếm ba điều này:

1. mỗi phần đến từ đâu
2. phần nào ổn định
3. phần nào được tạo động mỗi lượt

## Những Gì Bạn Đã Thành Thạo

Tại thời điểm này, bạn có thể:

- Xây dựng System Prompt từ các phần độc lập, có thể kiểm thử thay vì một chuỗi mờ đục
- Vạch ra ranh giới rõ ràng giữa nội dung ổn định và nội dung động
- Phân tầng các tệp hướng dẫn để các quy tắc cấp dự án và cấp thư mục cùng tồn tại mà không ghi đè
- Tái chèn Memory vào quy trình prompt để các sự kiện đã lưu thực sự ảnh hưởng đến mô hình
- Gắn nhắc nhở mỗi lượt riêng biệt khỏi System Prompt chính

## Tiếp Theo

Quy trình lắp ráp prompt có nghĩa là Agent của bạn giờ đây bước vào mỗi lượt với các hướng dẫn đúng, các Tool đúng, và Context đúng. Nhưng công việc thực sự tạo ra lỗi thực sự -- đầu ra bị cắt ngắn, prompt phát triển quá lớn, API hết thời gian. Trong [s11: Error Recovery](./s11-error-recovery.md), bạn sẽ dạy Harness phân loại các lỗi đó và chọn đường khôi phục thay vì bị sập.

## Điểm Mấu Chốt

> System Prompt là một quy trình lắp ráp với các phần rõ ràng và ranh giới rõ ràng, không phải một chuỗi bí ẩn lớn.
