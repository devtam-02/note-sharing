# Business rules và edge cases

## 1. Rule về thời gian

- Start time phải trước end time.
- Campaign/Voucher chỉ có hiệu lực trong time window đã cấu hình.
- Voucher có thể có hạn dùng riêng sau thời điểm publish.
- Campaign hết hạn thì Voucher chuyển inactive.
- Cần quy định timezone thống nhất cho mọi phép so sánh.

## 2. Rule về khách hàng

- Customer phải có định danh hợp lệ và mapping với hệ thống nguồn.
- Có thể yêu cầu Customer thuộc Segment cụ thể.
- Customer Metadata có thể là điều kiện Validation.
- Cần xác định hiệu lực khi Segment Membership thay đổi giữa lúc phân phối và sử dụng.

## 3. Rule về Voucher

- Voucher phải thuộc Campaign hợp lệ.
- Voucher phải được publish/gán đúng Customer nếu là mã cá nhân.
- Voucher phải active, chưa hết hạn và còn lượt sử dụng.
- Voucher đang Hold không được dùng bởi request khác.
- Bulk Code và Generic Code có chính sách chống trùng/giới hạn khác nhau.

## 4. Rule về Budget và Limit

- Không được vượt tổng Budget của Campaign.
- Có thể có limit theo thời gian, Customer, Voucher hoặc Campaign.
- Validate chỉ cho kết quả tạm thời; Redeem phải kiểm tra/giữ Budget lại để tránh race condition.
- Reserve, Confirm và Release Budget phải có audit.

## 5. Rule về giá trị ưu đãi

- Hỗ trợ amount, percentage, unit hoặc fixed tùy cấu hình.
- Formula có thể có min/max.
- Tổng Discount không nên vượt giá trị Order, nhưng cần xác nhận chính sách.
- Thứ tự cộng dồn có thể thay đổi giá trị cuối nếu mỗi Voucher tính trên số tiền còn lại.

## 6. Rule về Stacking

- ALL: tất cả Voucher phải hợp lệ theo cách hiểu hiện tại.
- PARTIAL: chỉ áp dụng Voucher hợp lệ.
- Category hierarchy hoặc request order quyết định thứ tự.
- Có giới hạn tổng số Voucher và số Voucher theo Category.
- Voucher exclusive có thể loại trừ các Voucher khác.

## 7. Rule về idempotency

- Một transaction Cashback không được trả tiền hai lần.
- Một event Distribution không được gán Voucher hai lần ngoài ý muốn.
- Một Redemption request khi retry phải trả cùng kết quả hoặc tiếp tục tiến trình cũ.
- Idempotency key phải trace được giữa App, Payment, Promotion và hệ thống trả tiền.

## 8. Ma trận edge case quan trọng

| Tình huống | Rủi ro | Kết quả mong đợi cần thống nhất |
|---|---|---|
| Event giao dịch trùng | Cashback hai lần | Deduplicate/idempotent |
| Customer chưa đồng bộ | Rule đánh giá sai | Retry hoặc từ chối có lý do |
| Segment đổi sau khi Voucher được gán | Khách hàng không còn đủ điều kiện | Xác định check lại ở Validation |
| Campaign bị tắt giữa Validate và Redeem | Dùng ưu đãi khi Campaign inactive | Revalidate ở Redeem |
| Voucher Hold hết hạn | Hai giao dịch cùng dùng mã | Redeem fail/re-hold theo policy |
| Budget còn ở Validate nhưng hết ở Redeem | Overspend | Reserve/atomic check ở Redeem |
| Payment timeout | Không biết có thanh toán không | Query trạng thái trước rollback |
| Redemption thành công nhưng App timeout | App retry tạo giao dịch trùng | Trả kết quả cũ theo idempotency key |
| Confirm Budget thành công, Voucher update lỗi | Lệch Budget/Voucher | Alert và manual reconciliation |
| Voucher update thành công, ghi Redemption lỗi | Không truy vết được | Recovery job/manual repair |
| Notification lỗi | Khách không biết kết quả | Retry message, không đổi kết quả tài chính |
| Campaign hết hạn đúng thời điểm request | Kết quả phụ thuộc clock | Quy định thời điểm chốt và timezone |
| Generic Code bị dùng đồng thời | Vượt limit | Atomic counter/limit check |
| Refund/reversal giao dịch gốc | Ưu đãi đã cấp nhưng giao dịch bị hủy | Clawback/reversal flow |

## 9. Câu hỏi test nghiệp vụ

Khi review acceptance criteria, luôn hỏi:

1. Điều gì xảy ra nếu cùng request được gửi hai lần?
2. Điều gì xảy ra nếu trạng thái thay đổi giữa hai bước?
3. Rule dùng dữ liệu tại thời điểm nào?
4. Bước nào là quyết định cuối cùng?
5. Nếu timeout thì ai query trạng thái?
6. Có thể rollback tự động đến đâu?
7. Case nào phải đưa Operation xử lý?
8. Khách hàng và CSKH nhìn thấy reason gì?

