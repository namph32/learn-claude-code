# s16: Giao Thức Nhóm

`s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 > s12 > s13 > s14 > s15 > [ s16 ] > s17 > s18 > s19`

## Những Gì Bạn Sẽ Học
- Cách mẫu yêu cầu-phản hồi với ID theo dõi cấu trúc việc đàm phán đa Agent
- Cách giao thức tắt cho phép lead dừng một thành viên nhóm một cách gọn gàng
- Cách cổng phê duyệt kế hoạch chặn công việc rủi ro sau một bước xem xét
- Cách một FSM có thể tái sử dụng (một bộ theo dõi trạng thái đơn giản với các chuyển tiếp được xác định) bao gồm cả hai giao thức

Trong s15 các thành viên nhóm của bạn có thể gửi tin nhắn tự do, nhưng sự tự do đó đi kèm với hỗn loạn. Một Agent bảo Agent khác "xin hãy dừng lại," và Agent kia bỏ qua. Một thành viên nhóm bắt đầu di chuyển cơ sở dữ liệu rủi ro mà không hỏi trước. Vấn đề không phải là bản thân giao tiếp -- bạn đã giải quyết điều đó với các hộp thư đến -- mà là thiếu các quy tắc phối hợp. Trong chương này bạn sẽ thêm các giao thức có cấu trúc: một wrapper tin nhắn tiêu chuẩn hóa với ID theo dõi biến các tin nhắn lỏng lẻo thành các handshake đáng tin cậy.

## Vấn Đề

Hai khoảng trống phối hợp trở nên rõ ràng khi nhóm của bạn phát triển vượt qua các ví dụ đơn giản:

**Tắt.** Giết luồng của thành viên nhóm để lại các file viết dở dang và danh sách config cũ. Bạn cần một handshake: lead yêu cầu tắt, và thành viên nhóm chấp thuận (hoàn thành công việc hiện tại và thoát gọn gàng) hoặc từ chối (tiếp tục làm việc vì nó có nghĩa vụ chưa hoàn thành).

**Phê duyệt kế hoạch.** Khi lead nói "tái cấu trúc module xác thực," thành viên nhóm bắt đầu ngay lập tức. Nhưng đối với các thay đổi có rủi ro cao, lead nên xem xét kế hoạch trước khi bất kỳ code nào được viết.

Cả hai tình huống đều chia sẻ một cấu trúc giống hệt nhau: một phía gửi yêu cầu mang một ID duy nhất, phía kia phản hồi tham chiếu đến cùng ID đó. Một mẫu duy nhất đó là đủ để xây dựng bất kỳ giao thức phối hợp nào bạn cần.

## Giải Pháp

Cả tắt và phê duyệt kế hoạch đều theo một hình dạng: gửi yêu cầu với `request_id`, nhận phản hồi tham chiếu cùng `request_id` đó, và theo dõi kết quả qua một máy trạng thái đơn giản (`pending -> approved` hoặc `pending -> rejected`).

```
Giao thức tắt            Giao thức phê duyệt kế hoạch
==================           ======================

Lead             Thành viên  Thành viên         Lead
  |                 |           |                 |
  |--shutdown_req-->|           |--plan_req------>|
  | {req_id:"abc"}  |           | {req_id:"xyz"}  |
  |                 |           |                 |
  |<--shutdown_resp-|           |<--plan_resp-----|
  | {req_id:"abc",  |           | {req_id:"xyz",  |
  |  approve:true}  |           |  approve:true}  |

FSM dùng chung:
  [pending] --approve--> [approved]
  [pending] --reject---> [rejected]

Bộ theo dõi:
  shutdown_requests = {req_id: {target, status}}
  plan_requests     = {req_id: {from, plan, status}}
```

## Cách Hoạt Động

**Bước 1.** Lead khởi tạo tắt bằng cách tạo `request_id` duy nhất và gửi yêu cầu qua hộp thư đến của thành viên nhóm. Yêu cầu được theo dõi trong một dictionary để lead có thể kiểm tra trạng thái của nó sau.

```python
shutdown_requests = {}

def handle_shutdown_request(teammate: str) -> str:
    req_id = str(uuid.uuid4())[:8]
    shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
    BUS.send("lead", teammate, "Please shut down gracefully.",
             "shutdown_request", {"request_id": req_id})
    return f"Shutdown request {req_id} sent (status: pending)"
```

**Bước 2.** Thành viên nhóm nhận yêu cầu trong hộp thư đến và phản hồi bằng chấp thuận hoặc từ chối. Phản hồi mang cùng `request_id` để lead có thể khớp với yêu cầu gốc -- đây là sự tương quan làm cho giao thức đáng tin cậy.

```python
if tool_name == "shutdown_response":
    req_id = args["request_id"]
    approve = args["approve"]
    shutdown_requests[req_id]["status"] = "approved" if approve else "rejected"
    BUS.send(sender, "lead", args.get("reason", ""),
             "shutdown_response",
             {"request_id": req_id, "approve": approve})
```

**Bước 3.** Phê duyệt kế hoạch theo mẫu giống hệt nhau nhưng theo hướng ngược lại. Thành viên nhóm gửi kế hoạch (tạo `request_id`), và lead xem xét nó (tham chiếu cùng `request_id` để chấp thuận hoặc từ chối).

```python
plan_requests = {}

def handle_plan_review(request_id, approve, feedback=""):
    req = plan_requests[request_id]
    req["status"] = "approved" if approve else "rejected"
    BUS.send("lead", req["from"], feedback,
             "plan_approval_response",
             {"request_id": request_id, "approve": approve})
```

Trong bản demo giảng dạy này, một hình dạng FSM bao gồm cả hai giao thức. Một hệ thống sản xuất có thể xử lý các họ giao thức khác nhau một cách khác biệt, nhưng phiên bản giảng dạy cố ý giữ một mẫu có thể tái sử dụng để bạn có thể thấy rõ cấu trúc dùng chung.

## Đọc Cùng Nhau

- Nếu các tin nhắn thông thường và các yêu cầu giao thức bắt đầu bị nhòe lẫn nhau, hãy xem lại [`glossary.md`](./glossary.md) và [`entity-map.md`](./entity-map.md) để thấy sự khác biệt.
- Nếu bạn dự định tiếp tục sang s17 và s18, hãy đọc [`team-task-lane-model.md`](./team-task-lane-model.md) trước để tính tự chủ và các lane Worktree không bị gộp thành một ý tưởng.
- Nếu bạn muốn theo dõi cách một yêu cầu giao thức trả về hệ thống chính, hãy kết hợp chương này với [`s00b-one-request-lifecycle.md`](./s00b-one-request-lifecycle.md).

## Cách Kết Nối Với Hệ Thống Nhóm

Nâng cấp thực sự trong s16 không phải là "hai loại tin nhắn mới." Đó là một đường dẫn phối hợp lâu dài:

```text
người yêu cầu bắt đầu một hành động giao thức
  ->
ghi RequestRecord
  ->
gửi ProtocolEnvelope qua hộp thư đến
  ->
người nhận đọc hết hộp thư đến trong vòng lặp tiếp theo
  ->
cập nhật trạng thái yêu cầu theo request_id
  ->
gửi phản hồi có cấu trúc
  ->
người yêu cầu tiếp tục dựa trên approved / rejected
```

Đó là tầng còn thiếu giữa "các Agent có thể trò chuyện" và "các Agent có thể phối hợp đáng tin cậy."

## Tin Nhắn vs Giao Thức vs Yêu Cầu vs Tác Vụ

| Đối tượng | Câu hỏi nào nó trả lời | Các trường điển hình |
|---|---|---|
| `MessageEnvelope` | ai nói gì với ai | `from`, `to`, `content` |
| `ProtocolEnvelope` | đây có phải là yêu cầu / phản hồi có cấu trúc không | `type`, `request_id`, `payload` |
| `RequestRecord` | luồng phối hợp này đang ở đâu | `kind`, `status`, `from`, `to` |
| `TaskRecord` | mục công việc thực tế nào đang được thực hiện | `subject`, `status`, `blockedBy`, `owner` |

Đừng gộp chúng lại:

- một yêu cầu giao thức không phải là bản thân tác vụ
- kho yêu cầu không phải là bảng tác vụ
- các giao thức theo dõi luồng phối hợp
- các tác vụ theo dõi tiến trình công việc

## Những Gì Thay Đổi Từ s15

| Thành phần     | Trước (s15)      | Sau (s16)                    |
|----------------|------------------|------------------------------|
| Tool           | 9                | 12 (+shutdown_req/resp +plan)|
| Tắt            | Chỉ thoát tự nhiên | Handshake yêu cầu-phản hồi |
| Chặn kế hoạch  | Không có         | Gửi/xem xét với phê duyệt   |
| Tương quan      | Không có         | request_id cho mỗi yêu cầu  |
| FSM            | Không có         | pending -> approved/rejected |

## Thử Nghiệm

```sh
cd learn-claude-code
python agents/s16_team_protocols.py
```

1. `Spawn alice as a coder. Then request her shutdown.`
2. `List teammates to see alice's status after shutdown approval`
3. `Spawn bob with a risky refactoring task. Review and reject his plan.`
4. `Spawn charlie, have him submit a plan, then approve it.`
5. Gõ `/team` để theo dõi các trạng thái

## Những Gì Bạn Đã Nắm Vững

Tại thời điểm này, bạn có thể:

- Xây dựng các giao thức yêu cầu-phản hồi sử dụng ID duy nhất để tương quan
- Triển khai tắt gọn gàng thông qua handshake hai bước
- Chặn công việc rủi ro sau một bước phê duyệt kế hoạch
- Tái sử dụng một mẫu FSM duy nhất (`pending -> approved/rejected`) cho bất kỳ giao thức mới nào bạn sáng tạo

## Tiếp Theo Là Gì

Nhóm của bạn bây giờ có cấu trúc và quy tắc, nhưng lead vẫn phải trông chừng mọi thành viên nhóm -- giao tác vụ từng cái một, thúc đẩy các worker nhàn rỗi. Trong s17, bạn sẽ làm cho các thành viên nhóm tự chủ: chúng tự quét bảng tác vụ, nhận công việc chưa được giao, và tiếp tục sau khi nén Context mà không mất danh tính.

## Điểm Mấu Chốt

> Một yêu cầu giao thức là một tin nhắn có cấu trúc với ID theo dõi, và phản hồi phải tham chiếu đến cùng ID đó -- mẫu duy nhất đó là đủ để xây dựng bất kỳ handshake phối hợp nào.
