# Phạm Vi Giảng Dạy

Tài liệu này giải thích những gì bạn sẽ học trong repo này, những gì được cố ý bỏ qua, và cách mỗi chương được căn chỉnh với mô hình tư duy của bạn khi nó phát triển.

## Mục Tiêu Của Repo Này

Đây không phải là bình luận từng dòng về một codebase sản xuất thượng nguồn nào đó.

Mục tiêu thực sự là:

**dạy bạn cách xây dựng một Harness coding-agent hoàn thiện cao từ đầu.**

Điều đó ngụ ý ba nghĩa vụ:

1. bạn thực sự có thể tái xây dựng nó
2. bạn giữ mainline rõ ràng thay vì chìm đắm trong chi tiết phụ
3. bạn không tiếp thu những cơ chế không thực sự tồn tại

## Những Gì Mỗi Chương Nên Đề Cập

Mỗi chương mainline nên làm rõ những điều sau:

- cơ chế này giải quyết vấn đề gì
- nó thuộc module hoặc tầng nào
- nó sở hữu trạng thái nào
- nó giới thiệu những cấu trúc dữ liệu nào
- nó kết nối trở lại vòng lặp như thế nào
- điều gì thay đổi trong luồng Runtime sau khi nó xuất hiện

Nếu bạn hoàn thành một chương mà vẫn không thể nói cơ chế tồn tại ở đâu hoặc sở hữu trạng thái nào, chương đó chưa hoàn chỉnh.

## Những Gì Chúng Tôi Cố Ý Giữ Đơn Giản

Các chủ đề này không bị cấm, nhưng chúng không nên chi phối lộ trình học của bạn:

- đóng gói, build và luồng phát hành
- keo tương thích đa nền tảng
- dây nối telemetry và chính sách doanh nghiệp
- các nhánh tương thích lịch sử
- các sự cố đặt tên theo sản phẩm cụ thể
- khớp code thượng nguồn từng dòng

Những thứ đó thuộc về phụ lục, ghi chú bảo trì, hoặc ghi chú sản phẩm hóa sau này, không phải ở trung tâm của lộ trình người mới bắt đầu.

## "Độ Trung Thực Cao" Thực Sự Có Nghĩa Là Gì Ở Đây

Độ trung thực cao trong một repo giảng dạy không có nghĩa là tái tạo mọi chi tiết cạnh 1:1.

Nó có nghĩa là ở gần với cột sống hệ thống thực sự:

- mô hình Runtime lõi
- ranh giới module
- các bản ghi chính
- các chuyển đổi trạng thái
- sự hợp tác giữa các hệ thống con chính

Tóm lại:

**trung thành cao với phần trunk, và có chủ đích về các đơn giản hóa giảng dạy ở các cạnh.**

## Tài Liệu Này Dành Cho Ai

Bạn không cần phải là chuyên gia về các nền tảng Agent.

Giả định tốt hơn về bạn:

- Python cơ bản đã quen thuộc
- hàm, lớp, danh sách và từ điển đã quen thuộc
- hệ thống Agent có thể hoàn toàn mới

Điều đó có nghĩa là các chương nên:

- giải thích các khái niệm mới trước khi sử dụng chúng
- giữ một khái niệm hoàn chỉnh ở một nơi chính
- di chuyển từ "nó là gì" sang "tại sao nó tồn tại" đến "cách xây dựng nó"

## Cấu Trúc Chương Được Đề Xuất

Các chương mainline nên tuân theo thứ tự này:

1. vấn đề gì xuất hiện khi không có cơ chế này
2. trước tiên giải thích các thuật ngữ mới
3. đưa ra mô hình tư duy nhỏ nhất hữu ích
4. hiển thị các bản ghi lõi / cấu trúc dữ liệu
5. hiển thị cài đặt đúng nhỏ nhất
6. hiển thị cách nó kết nối vào vòng lặp chính
7. hiển thị các lỗi phổ biến của người mới bắt đầu
8. hiển thị những gì một phiên bản hoàn thiện hơn sẽ thêm sau này

## Hướng Dẫn Thuật Ngữ

Nếu một chương giới thiệu thuật ngữ từ các danh mục này, nó nên giải thích nó:

- mẫu thiết kế
- cấu trúc dữ liệu
- thuật ngữ đồng thời
- thuật ngữ protocol / mạng
- từ vựng kỹ thuật không phổ biến

Ví dụ:

- state machine
- scheduler
- queue
- worktree
- DAG
- protocol envelope

Không được đưa ra tên mà không có giải thích.

## Nguyên Tắc Phiên Bản Đúng Tối Thiểu

Các cơ chế thực tế thường phức tạp, nhưng việc giảng dạy hoạt động tốt nhất khi không bắt đầu với mọi nhánh cùng một lúc.

Ưu tiên thứ tự này:

1. hiển thị phiên bản đúng nhỏ nhất
2. giải thích vấn đề lõi mà nó đã giải quyết
3. hiển thị những gì các lần lặp sau sẽ thêm

Ví dụ:

- hệ thống quyền: đầu tiên `deny -> mode -> allow -> ask`
- phục hồi lỗi: đầu tiên ba nhánh phục hồi chính
- hệ thống tác vụ: đầu tiên bản ghi tác vụ, phụ thuộc và mở khóa
- protocol nhóm: đầu tiên request / response cộng với `request_id`

## Danh Sách Kiểm Tra Để Viết Lại Một Chương

- Màn hình đầu tiên có giải thích tại sao cơ chế tồn tại không?
- Các thuật ngữ mới có được giải thích trước khi sử dụng không?
- Có mô hình tư duy nhỏ hoặc hình ảnh luồng không?
- Các bản ghi chính có được liệt kê rõ ràng không?
- Điểm kết nối trở lại vòng lặp có được giải thích không?
- Các cơ chế lõi có được tách khỏi chi tiết sản phẩm ngoại vi không?
- Các điểm nhầm lẫn dễ nhất có được chỉ ra không?
- Chương có tránh bịa đặt các cơ chế không được repo hỗ trợ không?

## Cách Sử Dụng Tài Liệu Nguồn Reverse-Engineered

Nguồn reverse-engineered nên được sử dụng như:

**tài liệu hiệu chỉnh của người bảo trì**

Sử dụng nó để:

- xác minh cơ chế mainline được mô tả chính xác
- xác minh các ranh giới và bản ghi quan trọng không bị thiếu
- xác minh cài đặt giảng dạy không bị trôi dạt sang hư cấu

Nó không bao giờ nên trở thành điều kiện tiên quyết để hiểu tài liệu giảng dạy.

## Kết Luận Chính

**Chất lượng của một repo giảng dạy được quyết định ít hơn bởi số lượng chi tiết nó đề cập và nhiều hơn bởi việc liệu các chi tiết quan trọng có được giải thích đầy đủ hay không và các chi tiết không quan trọng có được bỏ qua một cách an toàn hay không.**
