# s09: Memory System

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > [ s09 ] > s10 > s11 > s12 > s13 > s14 > s15 > s16 > s17 > s18 > s19`

## Bạn Sẽ Học Được Gì

- Bốn danh mục Memory bao gồm những gì đáng nhớ: sở thích người dùng, phản hồi, sự kiện dự án, và tài liệu tham khảo
- Cách các tệp YAML frontmatter cung cấp cho mỗi bản ghi Memory một tên, loại, và mô tả
- Những gì KHÔNG nên đưa vào Memory -- và tại sao việc sai ranh giới này là lỗi phổ biến nhất
- Sự khác biệt giữa Memory, tác vụ, kế hoạch, và CLAUDE.md

Agent của bạn từ s08 mạnh mẽ và có thể mở rộng. Nó có thể thực thi Tool một cách an toàn, được mở rộng thông qua Hook, và hoạt động trong các phiên dài nhờ nén Context. Nhưng nó mắc chứng mất trí nhớ. Mỗi lần bạn bắt đầu một phiên mới, Agent gặp bạn lần đầu tiên. Nó không nhớ rằng bạn thích pnpm hơn npm, rằng bạn đã nói ba lần với nó để dừng việc sửa đổi test snapshot, hoặc rằng thư mục cũ không thể xóa vì việc triển khai phụ thuộc vào nó. Bạn phải lặp lại với mình mỗi phiên. Giải pháp là một kho Memory nhỏ, bền vững -- không phải là bản ghi tất cả những gì Agent đã thấy, mà là một tập hợp các sự kiện được chọn lọc vẫn còn quan trọng lần sau.

## Vấn Đề

Không có Memory, một phiên mới bắt đầu từ con số không. Agent tiếp tục quên những thứ như sở thích người dùng lâu dài, các sửa chỉnh bạn đã lặp lại nhiều lần, các ràng buộc dự án không rõ ràng từ code, và các tài liệu tham khảo bên ngoài mà dự án phụ thuộc vào. Kết quả là một Agent luôn cảm thấy như đang gặp bạn lần đầu tiên. Bạn lãng phí thời gian thiết lập lại Context mà đáng lẽ phải được lưu một lần và tải lại tự động.

## Giải Pháp

Một kho Memory nhỏ dựa trên tệp lưu các sự kiện bền vững như các tệp markdown riêng lẻ với YAML frontmatter (một khối metadata ở đầu mỗi tệp, được phân cách bởi các dòng `---`). Vào đầu mỗi phiên, các Memory liên quan được tải và chèn vào Context của mô hình.

```text
conversation
   |
   | durable fact appears
   v
save_memory
   |
   v
.memory/
  ├── MEMORY.md
  ├── prefer_pnpm.md
  ├── ask_before_codegen.md
  └── incident_dashboard.md
   |
   v
next session loads relevant entries
```

## Đọc Cùng Nhau

- Nếu bạn vẫn nghĩ Memory chỉ là "một cửa sổ Context dài hơn," bạn có thể thấy hữu ích khi xem lại [`s06-context-compact.md`](./s06-context-compact.md) và tách biệt lại Compaction với Memory bền vững.
- Nếu `messages[]`, các khối tóm tắt, và kho Memory bắt đầu hợp nhất với nhau, hãy giữ [`data-structures.md`](./data-structures.md) mở trong khi đọc để giúp phân tách.
- Nếu bạn sắp tiếp tục vào s10, hãy đọc [`s10a-message-prompt-pipeline.md`](./s10a-message-prompt-pipeline.md) cùng với chương này vì Memory quan trọng nhất khi nó tái nhập đầu vào mô hình tiếp theo.

## Cách Hoạt Động

**Bước 1.** Định nghĩa bốn danh mục Memory. Đây là các loại sự kiện đáng lưu giữ qua các phiên. Mỗi danh mục có mục đích rõ ràng -- nếu một sự kiện không phù hợp với một trong số này, nó có lẽ không nên ở trong Memory.

### 1. `user` -- Sở thích người dùng ổn định

Ví dụ: thích `pnpm`, muốn câu trả lời ngắn gọn, không thích tái cấu trúc lớn mà không có kế hoạch.

### 2. `feedback` -- Các sửa chỉnh người dùng muốn được thực thi

Ví dụ: "đừng thay đổi test snapshot trừ khi tôi yêu cầu", "hỏi trước khi sửa đổi các tệp được tạo tự động."

### 3. `project` -- Các sự kiện dự án bền vững không rõ ràng từ repository

Ví dụ: "thư mục cũ này vẫn không thể xóa vì việc triển khai phụ thuộc vào nó", "dịch vụ này tồn tại vì yêu cầu tuân thủ, không phải sở thích kỹ thuật."

### 4. `reference` -- Con trỏ đến tài nguyên bên ngoài

Ví dụ: URL bảng theo dõi sự cố, vị trí bảng giám sát, vị trí tài liệu đặc tả.

```python
MEMORY_TYPES = ("user", "feedback", "project", "reference")
```

**Bước 2.** Lưu một bản ghi mỗi tệp sử dụng frontmatter. Mỗi Memory là một tệp markdown với YAML frontmatter cho hệ thống biết Memory đó được gọi là gì, loại nào, và nó nói về điều gì.

```md
---
name: prefer_pnpm
description: User prefers pnpm over npm
type: user
---
The user explicitly prefers pnpm for package management commands.
```

```python
def save_memory(name, description, mem_type, content):
    path = memory_dir / f"{slugify(name)}.md"
    path.write_text(render_frontmatter(name, description, mem_type) + content)
    rebuild_index()
```

**Bước 3.** Xây dựng một chỉ mục nhỏ để hệ thống biết Memory nào tồn tại mà không cần đọc mọi tệp.

```md
# Memory Index

- prefer_pnpm [user]
- ask_before_codegen [feedback]
- incident_dashboard [reference]
```

Chỉ mục không phải là Memory -- nó là bản đồ nhanh về những gì tồn tại.

**Bước 4.** Tải Memory liên quan vào đầu phiên và biến nó thành một phần của prompt. Memory chỉ hữu ích khi nó được đưa trở lại đầu vào của mô hình. Đây là lý do tại sao s09 kết nối tự nhiên vào s10.

```python
memories = memory_store.load_all()
```

**Bước 5.** Biết những gì KHÔNG nên đưa vào Memory. Ranh giới này là phần quan trọng nhất của chương, và là nơi hầu hết người mới bắt đầu mắc sai lầm.

| Đừng lưu | Lý do |
|----------|-------|
| Cấu trúc cây tệp | Có thể đọc lại từ repository |
| Tên hàm và chữ ký | Code là nguồn sự thật |
| Trạng thái tác vụ hiện tại | Thuộc về tác vụ / kế hoạch, không phải Memory |
| Tên nhánh tạm thời hoặc số PR | Trở nên lỗi thời nhanh chóng |
| Bí mật hoặc thông tin xác thực | Rủi ro bảo mật |

Quy tắc đúng là: chỉ giữ thông tin vẫn còn quan trọng qua các phiên và không thể dễ dàng tái tạo từ workspace hiện tại.

**Bước 6.** Hiểu các ranh giới so với các khái niệm lân cận. Bốn thứ này nghe có vẻ giống nhau nhưng phục vụ các mục đích khác nhau.

| Khái niệm | Mục đích | Thời gian tồn tại |
|-----------|---------|------------------|
| Memory | Các sự kiện nên tồn tại qua các phiên | Bền vững |
| Tác vụ | Hệ thống đang cố gắng hoàn thành điều gì ngay bây giờ | Một tác vụ |
| Kế hoạch | Lượt hoặc phiên này dự định tiến hành như thế nào | Một phiên |
| CLAUDE.md | Tài liệu hướng dẫn ổn định và các quy tắc thường trực cấp dự án | Bền vững |

Quy tắc ngắn gọn: chỉ hữu ích cho tác vụ này -- dùng `task` hoặc `plan`. Hữu ích cả phiên tiếp theo -- dùng `memory`. Văn bản hướng dẫn lâu dài -- dùng `CLAUDE.md`.

## Các Lỗi Thường Gặp

**Lỗi 1: Lưu những thứ mà repository có thể cho bạn biết.** Nếu code có thể trả lời được, Memory không nên sao chép nó. Bạn sẽ chỉ có các bản sao lỗi thời xung đột với thực tế.

**Lỗi 2: Lưu tiến độ tác vụ đang hoạt động.** "Hiện đang sửa auth" không phải là Memory. Điều đó thuộc về trạng thái kế hoạch hoặc tác vụ. Khi tác vụ hoàn thành, Memory đó vô nghĩa.

**Lỗi 3: Coi Memory là sự thật tuyệt đối.** Memory có thể lỗi thời. Quy tắc an toàn hơn là: Memory cho hướng đi, quan sát hiện tại cho sự thật.

## Những Thay Đổi So Với s08

| Thành phần | Trước (s08) | Sau (s09) |
|------------|------------|----------|
| Trạng thái liên phiên | Không có | Kho Memory dựa trên tệp |
| Các loại Memory | Không có | user, feedback, project, reference |
| Định dạng lưu trữ | Không có | Tệp markdown với YAML frontmatter |
| Bắt đầu phiên | Khởi động lạnh | Tải các Memory liên quan |
| Độ bền | Mọi thứ bị quên | Các sự kiện quan trọng được lưu giữ |

## Thử Nghiệm

```sh
cd learn-claude-code
python agents/s09_memory_system.py
```

Thử yêu cầu nó ghi nhớ:

- một sở thích người dùng
- một sửa chỉnh bạn muốn được thực thi sau này
- một sự kiện dự án không rõ ràng từ repository

## Những Gì Bạn Đã Thành Thạo

Tại thời điểm này, bạn có thể:

- Giải thích tại sao Memory là kho lưu trữ các sự kiện bền vững được chọn lọc, không phải bản ghi tất cả những gì Agent đã thấy
- Phân loại các sự kiện thành bốn loại: sở thích người dùng, phản hồi, kiến thức dự án, và tài liệu tham khảo
- Lưu và truy xuất Memory bằng các tệp markdown dựa trên frontmatter
- Vạch ra ranh giới rõ ràng giữa những gì thuộc về Memory và những gì thuộc về trạng thái tác vụ, kế hoạch, hoặc CLAUDE.md
- Tránh ba lỗi phổ biến nhất: sao chép repository, lưu trạng thái tạm thời, và coi Memory là sự thật tuyệt đối

## Tiếp Theo

Agent của bạn giờ đây nhớ những thứ qua các phiên, nhưng các Memory đó chỉ nằm trong tệp cho đến khi bắt đầu phiên. Trong s10, bạn sẽ xây dựng quy trình lắp ráp System Prompt -- cơ chế lấy Memory, kỹ năng, quyền, và các Context khác rồi đan xen chúng vào prompt mà mô hình thực sự nhìn thấy mỗi lượt.

## Điểm Mấu Chốt

> Memory không phải là bản ghi tất cả những gì Agent đã thấy -- nó là kho lưu trữ nhỏ các sự kiện bền vững vẫn còn quan trọng ở phiên tiếp theo.
