# Điểm chưa thống nhất và câu hỏi cần làm rõ

File này không kết luận tài liệu nguồn sai. Đây là danh sách các điểm cần BA/PO/Dev xác nhận trước khi dùng làm requirement hoặc test case.

## 1. Cách gọi Phase

BRD tích hợp Mobile trong tài liệu Promotion Phase 2 lại gọi phạm vi SDK hiện tại là "Phase 1".

Cần xác nhận:

- "Phase 1 Mobile" có phải là sub-phase của dự án Promotion Phase 2 không?
- Những chức năng nào chính thức thuộc từng release?

## 2. Một Voucher hay nhiều Voucher

- BRD Mobile hiện tại: chỉ cho phép một Voucher/giao dịch.
- Tổng quan backend Phase 2: có Stacking Rule và luồng Validate/Redeem nhiều Voucher.

Cần xác nhận:

- Backend hỗ trợ nhiều Voucher nhưng Mobile chưa mở UI?
- API nào App hiện tại được phép gọi?
- Khi nào Stacking được bật cho end user?

## 3. Trạng thái Campaign

- DDD: DRAFT, ACTIVE, PAUSED, COMPLETED.
- SRS chi tiết: INITIALIZING, ACTIVE, RUNNING, DISABLED, EXPIRED, ERROR, UPDATING, DELETING, DELETED.

Cần có một bảng mapping tên business, enum hệ thống và nhãn hiển thị.

## 4. Trạng thái Voucher

Các phần tài liệu sử dụng:

- ACTIVE/INACTIVE theo Campaign.
- CREATED/PUBLISHED/REDEEMED/EXPIRED trong glossary DDD.
- Đã gán/Chưa gán và Thành công/Thất bại trong công cụ CSKH.

Cần xác nhận đây là các field trạng thái riêng hay một state machine duy nhất.

Ngoài ra, trạng thái Redemption được ghi là `SUCCESS` trong phần DDD nhưng là `SUCCEEDED` trong luồng chi tiết. Cần thống nhất tên business, enum API và nhãn hiển thị.

## 5. Trình tự Payment và Redemption

BRD Mobile có đoạn mô tả khách hàng xác nhận, lock Voucher, App thanh toán rồi Redemption ghi nhận. Luồng backend lại mô tả Redemption tự thực hiện Hold, Budget, Limit và đổi trạng thái Voucher.

Cần thống nhất:

1. Qualification xảy ra khi nào?
2. Validation xảy ra trước hay sau khách hàng xác nhận?
3. Hold/Reserve xảy ra trước thanh toán hay trong Redemption?
4. Redemption gọi trước hay sau kết quả Payment?
5. Payment fail thì ai yêu cầu Rollback?

## 6. Rollback

Tài liệu nêu Confirm Budget và đổi Voucher có thể không rollback tự động.

Cần xác nhận:

- Trạng thái trung gian nào dùng cho case này?
- Alert gửi cho ai?
- Công cụ và quyền manual repair?
- SLA xử lý và cách thông báo khách hàng?
- Sau manual repair có phát event/audit không?

## 7. Cashback reversal/refund

Tài liệu tổng quan Phase 1 chưa mô tả rõ khi giao dịch gốc bị reversal/refund sau khi Cashback đã chuyển.

Cần xác nhận:

- Có thu hồi Cashback không?
- Tạo giao dịch âm hay dùng Core Transfer reversal?
- Nếu khách hàng không đủ số dư thì xử lý thế nào?

## 8. Source of truth

Cần chốt owner cho:

- Customer và Segment.
- Transaction/Order.
- Payment status.
- Budget.
- Voucher assignment/usage.

## 9. Approval và chỉnh sửa Campaign

Tài liệu mô tả tạo/kích hoạt nhưng chưa nổi bật quy trình maker-checker/phê duyệt.

Cần xác nhận:

- Ai được tạo, sửa, bật, tắt và xóa Campaign?
- Có cần approve trước khi chạy không?
- Sửa Campaign đang chạy tạo version mới hay cập nhật trực tiếp?
- Voucher đã publish chịu ảnh hưởng thế nào?

## 10. Reason code

Cần một danh mục chuẩn để App và CSKH giải thích thống nhất:

- Hết hạn/chưa hiệu lực.
- Không thuộc Customer/Segment.
- Không đúng Product/Amount.
- Hết Budget/vượt Limit.
- Đã sử dụng/đang Hold.
- Bị loại bởi Stacking Rule.
- Lỗi hệ thống/timeout.

## 11. Mẫu câu hỏi khi gặp BA/PO

> Trong trường hợp [trạng thái đầu vào], khi [event] xảy ra đồng thời với [ngoại lệ], object nào là source of truth, trạng thái cuối mong muốn là gì, khách hàng thấy gì và có cần Operation xử lý không?
