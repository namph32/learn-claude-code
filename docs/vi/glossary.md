# Bảng Thuật Ngữ

> **Tài liệu tham khảo** -- Đánh dấu trang này. Quay lại bất cứ khi nào bạn gặp một thuật ngữ chưa quen.

Bảng thuật ngữ này tập hợp các thuật ngữ quan trọng nhất trong lộ trình giảng dạy chính -- những thuật ngữ thường gây khó khăn cho người mới bắt đầu. Nếu bạn đang đọc giữa chương mà bắt gặp một từ và nghĩ "khoan, từ này nghĩa là gì nhỉ?", đây là trang để bạn quay lại.

## Tài Liệu Bổ Sung Được Đề Xuất

- [`entity-map.md`](./entity-map.md) để xem ranh giới các tầng
- [`data-structures.md`](./data-structures.md) để xem hình dạng các bản ghi
- [`s13a-runtime-task-model.md`](./s13a-runtime-task-model.md) nếu bạn hay nhầm lẫn giữa các loại "tác vụ" khác nhau

## Agent

Một mô hình có khả năng suy luận trên đầu vào và gọi Tool để hoàn thành công việc. (Hãy nghĩ về nó như "bộ não" quyết định phải làm gì tiếp theo.)

## Harness

Môi trường làm việc được chuẩn bị xung quanh mô hình -- tất cả những gì mô hình cần nhưng không thể tự cung cấp:

- Tool
- hệ thống tệp
- quyền
- lắp ráp prompt
- Memory
- Runtime tác vụ

## Agent Loop

Chu kỳ lõi lặp đi lặp lại điều khiển mọi phiên Agent. Mỗi lần lặp trông như sau:

1. gửi đầu vào hiện tại cho mô hình
2. kiểm tra xem mô hình đã trả lời hay yêu cầu Tool
3. thực thi Tool nếu cần
4. Write-back kết quả
5. tiếp tục hoặc dừng lại

## Message / `messages[]`

Lịch sử cuộc hội thoại và kết quả Tool hiển thị, được dùng làm Context làm việc. (Đây là bản ghi nhật ký cuộn mà mô hình thấy ở mỗi lượt.)

## Tool

Một hành động mà mô hình có thể yêu cầu, chẳng hạn như đọc tệp, ghi tệp, chỉnh sửa nội dung hoặc chạy lệnh shell.

## Tool Schema

Mô tả được hiển thị cho mô hình:

- tên
- mục đích
- tham số đầu vào
- kiểu dữ liệu đầu vào

## Dispatch Map

Bảng định tuyến từ tên Tool đến các handler. (Giống như tổng đài điện thoại: tên gọi đến, và bảng này kết nối nó với đúng hàm xử lý.)

## Stop Reason

Lý do kết thúc lượt mô hình hiện tại. Các giá trị phổ biến:

- `end_turn`
- `tool_use`
- `max_tokens`

## Context

Toàn bộ thông tin hiện đang hiển thị với mô hình. (Tất cả những gì nằm trong "cửa sổ" của mô hình tại một lượt nhất định.)

## Compaction

Quá trình thu nhỏ Context đang hoạt động trong khi vẫn giữ lại cốt truyện quan trọng và thông tin bước tiếp theo. (Giống như tóm tắt biên bản cuộc họp để giữ lại các hành động cần thực hiện nhưng bỏ qua những cuộc trò chuyện nhỏ.)

## Subagent

Một worker được ủy quyền dùng một lần, chạy trong Context riêng và thường trả về một bản tóm tắt. (Một trợ lý tạm thời được khởi tạo cho một công việc, sau đó bị loại bỏ.)

## Permission

Tầng quyết định xác định liệu một hành động được yêu cầu có được phép thực thi hay không.

## Hook

Điểm mở rộng cho phép hệ thống quan sát hoặc thêm hiệu ứng phụ xung quanh vòng lặp mà không cần viết lại vòng lặp. (Giống như trình lắng nghe sự kiện -- vòng lặp phát tín hiệu, và các Hook phản hồi.)

## Memory

Thông tin xuyên phiên đáng được lưu giữ vì nó vẫn có giá trị sau này và không dễ tái tạo.

## System Prompt

Bề mặt hướng dẫn cấp hệ thống ổn định xác định danh tính, quy tắc và các ràng buộc tồn tại lâu dài.

## Query

Toàn bộ quá trình đa lượt được dùng để hoàn thành một yêu cầu của người dùng. (Một Query có thể trải qua nhiều lượt lặp trước khi câu trả lời sẵn sàng.)

## Transition Reason

Lý do hệ thống tiếp tục sang lượt tiếp theo.

## Task

Một nút mục tiêu công việc bền vững trong đồ thị công việc. (Khác với một mục việc cần làm biến mất khi phiên kết thúc, một tác vụ tồn tại lâu dài.)

## Runtime Task / Runtime Slot

Một slot thực thi trực tiếp đại diện cho thứ gì đó đang chạy. (Tác vụ cho biết "điều gì sẽ xảy ra"; Runtime Slot cho biết "nó đang xảy ra ngay bây giờ.")

## Teammate

Một cộng tác viên bền vững trong hệ thống đa Agent. (Khác với Subagent được triển khai một lần và bỏ, một Teammate tồn tại lâu dài.)

## Protocol Request

Một yêu cầu có cấu trúc với danh tính, trạng thái và theo dõi rõ ràng, thường được hỗ trợ bởi `request_id`. (Một phong bì chính thức thay vì một tin nhắn thông thường.)

## Worktree

Một lane thư mục thực thi riêng biệt được dùng để tránh xung đột khi làm việc song song. (Mỗi lane có bản sao riêng của không gian làm việc, giống như các bàn làm việc riêng cho các tác vụ riêng biệt.)

## MCP

Model Context Protocol. Trong repo này, nó đại diện cho một bề mặt tích hợp khả năng bên ngoài, không chỉ là một danh sách Tool. (Cầu nối cho phép Agent của bạn giao tiếp với các dịch vụ bên ngoài.)

## DAG

Directed Acyclic Graph (Đồ thị có hướng không có chu trình). Một tập hợp các nút được kết nối bởi các cạnh một chiều không có chu trình. (Nếu bạn vẽ mũi tên giữa các tác vụ cho thấy "A phải hoàn thành trước B", và không có đường mũi tên nào quay lại điểm xuất phát, bạn có một DAG.) Được dùng trong repo này cho đồ thị phụ thuộc tác vụ.

## FSM / State Machine

Finite State Machine (Máy trạng thái hữu hạn). Một hệ thống luôn ở đúng một trạng thái trong một tập hợp đã biết, và chuyển đổi giữa các trạng thái dựa trên các sự kiện đã định nghĩa. (Hãy nghĩ về đèn giao thông chạy qua đỏ, xanh và vàng.) Logic lượt trong Agent Loop được mô hình hóa như một State Machine.

## Control Plane

Tầng quyết định điều gì sẽ xảy ra tiếp theo, khác với tầng thực sự thực hiện công việc. (Kiểm soát không lưu so với máy bay.) Trong repo này, Query engine và tool dispatch đóng vai trò Control Plane.

## Token

Các đơn vị nguyên tử mà mô hình ngôn ngữ đọc và ghi. Một Token khoảng 3/4 một từ tiếng Anh. Giới hạn Context và ngưỡng Compaction được đo bằng Token.
