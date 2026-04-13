# s00d: Lý Do Thứ Tự Chương

> **Deep Dive** -- Đọc tài liệu này sau khi hoàn thành Stage 1 (s01-s06) hoặc bất cứ khi nào bạn tự hỏi "tại sao khóa học được sắp xếp theo cách này?"

Ghi chú này không nói về một cơ chế cụ thể. Nó trả lời một câu hỏi giảng dạy cơ bản hơn: tại sao chương trình học này dạy hệ thống theo thứ tự hiện tại thay vì theo thứ tự tệp nguồn, sự hào hứng về tính năng, hay độ phức tạp triển khai thô?

## Kết Luận Trước

Thứ tự `s01 -> s19` hiện tại có cấu trúc vững chắc.

Sức mạnh của nó không chỉ là sự bao quát rộng. Sức mạnh của nó là nó phát triển hệ thống theo cùng thứ tự bạn nên hiểu nó:

1. Xây dựng vòng lặp Agent hoạt động nhỏ nhất.
2. Thêm tầng control-plane và củng cố xung quanh vòng lặp đó.
3. Nâng cấp lập kế hoạch phiên thành công việc lâu bền và trạng thái Runtime.
4. Chỉ sau đó mở rộng sang nhóm lâu bền, luồng thực thi cô lập và bus năng lực bên ngoài.

Đó là thứ tự giảng dạy đúng vì nó tuân theo:

**thứ tự phụ thuộc giữa các cơ chế**

không phải thứ tự tệp hay thứ tự đóng gói sản phẩm.

## Bốn Tuyến Phụ Thuộc

Chương trình học này thực sự được tổ chức theo bốn tuyến phụ thuộc:

1. `core loop dependency`
2. `control-plane dependency`
3. `work-state dependency`
4. `platform-boundary dependency`

Nói một cách đơn giản:

```text
đầu tiên làm cho Agent chạy
  -> sau đó làm cho nó chạy an toàn
  -> sau đó làm cho nó chạy lâu bền
  -> sau đó làm cho nó chạy như một nền tảng
```

## Hình Dạng Thực Sự Của Chuỗi

```text
s01-s06
  xây dựng một hệ thống Agent đơn hoạt động

s07-s11
  củng cố và kiểm soát hệ thống đó

s12-s14
  chuyển lập kế hoạch tạm thời thành công việc lâu bền + Runtime

s15-s19
  mở rộng sang thành viên nhóm, giao thức, tự chủ, luồng cô lập và năng lực bên ngoài
```

Sau mỗi giai đoạn, bạn nên có thể nói:

- sau `s06`: "Tôi có thể xây dựng một harness Agent đơn thực sự"
- sau `s11`: "Tôi có thể làm cho harness đó an toàn hơn, ổn định hơn và dễ mở rộng hơn"
- sau `s14`: "Tôi có thể quản lý công việc lâu bền, thực thi nền tảng và khởi động theo thời gian"
- sau `s19`: "Tôi hiểu ranh giới nền tảng của một hệ thống Agent hoàn thành tốt"

## Tại Sao Các Chương Đầu Phải Giữ Thứ Tự Hiện Tại

### `s01` phải đứng đầu

Vì nó thiết lập:

- điểm vào tối thiểu
- vòng lặp từng lượt
- tại sao kết quả Tool phải chảy lại vào cuộc gọi mô hình tiếp theo

Không có điều này, mọi thứ sau trở thành nói chuyện về tính năng rời rạc.

### `s02` phải ngay sau `s01`

Vì một Agent không thể định tuyến ý định vào Tool thì vẫn chỉ đang nói, không hành động.

`s02` là nơi người học lần đầu thấy harness trở nên thực sự:

- mô hình phát ra `tool_use`
- hệ thống dispatch đến handler
- Tool thực thi
- `tool_result` chảy lại vào vòng lặp

### `s03` nên đứng trước `s04`

Đây là một rào cản quan trọng.

Bạn nên hiểu trước:

- cách Agent hiện tại tổ chức công việc của chính nó

trước khi học:

- khi nào nên ủy quyền công việc vào một sub-context riêng biệt

Nếu `s04` đến quá sớm, Subagent trở thành một lối thoát thay vì một cơ chế cô lập rõ ràng.

### `s05` nên đứng trước `s06`

Hai chương này giải quyết hai nửa của cùng một vấn đề:

- `s05`: ngăn kiến thức không cần thiết vào Context
- `s06`: quản lý Context vẫn phải duy trì hoạt động

Thứ tự đó quan trọng. Một hệ thống tốt đầu tiên tránh phình to, sau đó nén những gì vẫn cần thiết.

## Tại Sao `s07-s11` Tạo Thành Một Khối Củng Cố

Các chương này đều trả lời cùng một câu hỏi lớn hơn:

**vòng lặp đã hoạt động, vậy làm thế nào nó trở nên ổn định, an toàn và dễ đọc như một hệ thống thực?**

### `s07` nên đứng trước `s08`

Quyền đến trước vì hệ thống phải trả lời trước:

- hành động này có được phép xảy ra không
- nó có nên bị từ chối không
- nó có nên hỏi người dùng trước không

Chỉ sau đó mới nên dạy Hook, trả lời:

- hành vi bổ sung nào gắn xung quanh vòng lặp

Vì vậy thứ tự giảng dạy đúng là:

**cổng trước, mở rộng sau**

### `s09` nên đứng trước `s10`

Đây là một quyết định thứ tự rất quan trọng khác.

`s09` dạy:

- thông tin lâu bền nào tồn tại
- sự kiện nào xứng đáng được lưu trữ lâu dài

`s10` dạy:

- cách nhiều nguồn thông tin được tập hợp thành đầu vào mô hình

Điều đó có nghĩa là:

- Memory định nghĩa một nguồn nội dung
- tập hợp System Prompt giải thích cách tất cả các nguồn nội dung được kết hợp

Nếu bạn đảo ngược chúng, xây dựng System Prompt bắt đầu cảm thấy tùy tiện và bí ẩn.

### `s11` là chương đóng phù hợp cho khối này

Phục hồi lỗi không phải là một tính năng riêng biệt.

Đó là nơi hệ thống cuối cùng cần giải thích:

- tại sao nó tiếp tục
- tại sao nó thử lại
- tại sao nó dừng lại

Điều đó chỉ trở nên rõ ràng sau khi đường đầu vào, đường Tool, đường trạng thái và đường điều khiển đã tồn tại.

## Tại Sao `s12-s14` Phải Giữ Thứ Tự Mục Tiêu -> Runtime -> Lịch

Đây là phần dễ dạy tồi nhất của chương trình học nếu thứ tự sai.

### `s12` phải đứng trước `s13`

`s12` dạy:

- công việc nào tồn tại
- quan hệ phụ thuộc giữa các nút công việc
- khi nào công việc hạ nguồn được mở khóa

`s13` dạy:

- thực thi trực tiếp nào đang chạy hiện tại
- kết quả nền tảng đi đâu
- trạng thái Runtime write-back như thế nào

Đó là sự phân biệt quan trọng:

- `task` là mục tiêu công việc lâu bền
- `runtime task` là slot thực thi trực tiếp

Nếu `s13` đến trước, bạn gần như chắc chắn sẽ gộp hai cái đó thành một khái niệm.

### `s14` phải đứng sau `s13`

Cron không thêm một loại tác vụ khác.

Nó thêm một điều kiện khởi động mới:

**thời gian trở thành một cách khác để khởi động công việc vào Runtime**

Vì vậy thứ tự đúng là:

`durable task graph -> runtime slot -> schedule trigger`

## Tại Sao `s15-s19` Nên Giữ Nhóm -> Giao Thức -> Tự Chủ -> Worktree -> Bus Năng Lực

### `s15` định nghĩa ai tồn tại trong hệ thống

Trước khi giao thức hoặc tính tự chủ có ý nghĩa, hệ thống cần các actor lâu bền:

- thành viên nhóm là ai
- danh tính họ mang theo
- họ tồn tại qua công việc như thế nào

### `s16` sau đó định nghĩa cách các actor đó phối hợp

Giao thức không nên đến trước actor.

Giao thức tồn tại để cấu trúc:

- ai yêu cầu
- ai phê duyệt
- ai phản hồi
- cách yêu cầu được theo dõi

### `s17` chỉ có ý nghĩa sau cả hai

Tính tự chủ dễ dạy một cách mơ hồ.

Nhưng trong một hệ thống thực nó chỉ trở nên rõ ràng sau:

- thành viên nhóm lâu bền tồn tại
- phối hợp có cấu trúc đã tồn tại

Nếu không, "tự động nhận" nghe như ma thuật thay vì cơ chế có giới hạn mà nó thực sự là.

### `s18` nên đứng trước `s19`

Sự cô lập Worktree là một vấn đề ranh giới thực thi cục bộ:

- công việc song song thực sự chạy ở đâu
- cách một luồng công việc được cô lập khỏi luồng khác

Điều đó nên trở nên rõ ràng trước khi chuyển ra ngoài sang:

- plugin
- MCP server
- định tuyến năng lực bên ngoài

Nếu không bạn có nguy cơ tập trung quá vào năng lực bên ngoài và học kém ranh giới nền tảng cục bộ.

### `s19` đúng khi đứng cuối

Đó là ranh giới nền tảng bên ngoài.

Nó chỉ trở nên rõ ràng khi bạn đã hiểu:

- actor cục bộ
- luồng công việc cục bộ
- công việc lâu bền cục bộ
- thực thi Runtime cục bộ
- sau đó là nhà cung cấp năng lực bên ngoài

## Năm Sắp Xếp Lại Sẽ Làm Khóa Học Tệ Hơn

1. Chuyển `s04` trước `s03`
   Điều này dạy ủy quyền trước lập kế hoạch cục bộ.

2. Chuyển `s10` trước `s09`
   Điều này dạy tập hợp System Prompt trước khi người học hiểu một trong các đầu vào cốt lõi của nó.

3. Chuyển `s13` trước `s12`
   Điều này gộp mục tiêu lâu bền và Runtime slot trực tiếp thành một ý tưởng nhầm lẫn.

4. Chuyển `s17` trước `s15` hoặc `s16`
   Điều này biến tính tự chủ thành ma thuật polling mơ hồ.

5. Chuyển `s19` trước `s18`
   Điều này làm cho nền tảng bên ngoài trông quan trọng hơn ranh giới thực thi cục bộ.

## Kiểm Tra Người Duy Trì Tốt Trước Khi Sắp Xếp Lại

Trước khi di chuyển các chương xung quanh, hãy hỏi:

1. Người học đã hiểu khái niệm điều kiện tiên quyết chưa?
2. Việc sắp xếp lại này có làm mờ hai khái niệm nên được giữ riêng biệt không?
3. Chương này chủ yếu về mục tiêu, trạng thái Runtime, actor hay ranh giới năng lực?
4. Nếu tôi chuyển nó sớm hơn, người đọc có vẫn có thể xây dựng phiên bản đúng tối thiểu không?
5. Tôi có đang tối ưu hóa cho sự hiểu biết, hay chỉ đơn giản là sao chép thứ tự tệp nguồn?

Nếu câu trả lời thành thật cho câu hỏi cuối cùng là "thứ tự tệp nguồn", việc sắp xếp lại có lẽ là một lỗi.

## Điểm Mấu Chốt

**Thứ tự chương tốt không chỉ là danh sách các cơ chế. Đó là một chuỗi mà mỗi chương cảm thấy như tầng tự nhiên tiếp theo được phát triển từ chương trước.**
