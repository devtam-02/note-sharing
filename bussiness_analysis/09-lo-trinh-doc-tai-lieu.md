# Lộ trình tìm hiểu nghiệp vụ trong 10 ngày

## Ngày 1 - Bức tranh tổng thể

- Đọc `02-tong-quan-du-an.md`.
- Tự giải thích khác nhau giữa Cashback và Coupon.
- Xác nhận phạm vi hiện tại của team.

Đầu ra: một đoạn 5 câu mô tả sản phẩm.

## Ngày 2 - Ngôn ngữ nghiệp vụ

- Đọc `03-thuat-ngu-va-doi-tuong-nghiep-vu.md`.
- Vẽ quan hệ Campaign - Voucher - Rule - Redemption.
- Ghi lại thuật ngữ nội bộ team đang dùng khác tài liệu.

Đầu ra: glossary cá nhân.

## Ngày 3 - Cashback

- Đọc hai file Phase 1 trong `flows/`.
- Trace một transaction mẫu từ Transaction Manager đến Core Transfer.
- Hỏi rule eligibility, formula và duplicate handling.

Đầu ra: sequence Cashback có happy path và ba nhánh lỗi.

## Ngày 4 - Coupon Campaign

- Đọc `flows/03-phase2-chien-dich-coupon.md`.
- Mở CMS/UAT nếu có và đối chiếu wizard 6 bước.
- Ghi lại trạng thái và thao tác được phép.

Đầu ra: lifecycle Campaign.

## Ngày 5 - Distribution

- Đọc `flows/04-phan-phoi-uu-dai.md`.
- Chọn một event ví dụ và mô tả Trigger - Condition - Action - Channel.
- Hỏi cách chống phát trùng.

Đầu ra: một Distribution Rule mẫu.

## Ngày 6 - Validation và Stacking

- Đọc `flows/05-kiem-tra-hop-le-va-cong-don.md`.
- Tạo ba ví dụ: một Voucher pass, một fail, một skipped.
- So sánh ALL và PARTIAL.

Đầu ra: bảng input/rule/result.

## Ngày 7 - Redemption và Rollback

- Đọc `flows/06-doi-thuong-va-hoan-tac.md`.
- Xác định điểm commit và các bước có thể hoàn tác.
- Hỏi quy trình manual repair.

Đầu ra: sơ đồ Redemption và rollback matrix.

## Ngày 8 - Mobile và CSKH

- Đọc hai file Mobile/CSKH.
- Trace một case khách hàng chọn Voucher nhưng Validation fail.
- Kiểm tra reason mà App và CSKH nhìn thấy.

Đầu ra: customer journey và support journey.

## Ngày 9 - Rule và edge case

- Đọc `07-business-rules-va-edge-cases.md`.
- Chọn năm edge case có rủi ro cao nhất đối với task/team.
- Tìm ticket/incident tương ứng nếu có.

Đầu ra: risk list.

## Ngày 10 - Làm rõ và kiểm chứng

- Đọc `08-diem-chua-thong-nhat-va-cau-hoi-can-lam-ro.md`.
- Họp ngắn với BA/PO hoặc người vận hành.
- Cập nhật checklist và ghi nguồn xác nhận.

Đầu ra: bản đồ nghiệp vụ đã được người phụ trách xác nhận.

## Tiêu chí hoàn thành

Bạn có thể:

- Giải thích hệ thống cho người mới trong 10 phút.
- Trace được Cashback và Coupon end-to-end.
- Phân biệt Qualification, Validation và Redemption.
- Nêu lifecycle của Campaign, Voucher và Rule.
- Chỉ ra các điểm dễ lỗi về Budget, Limit, duplicate và rollback.
- Biết câu hỏi nào cần hỏi ai.

