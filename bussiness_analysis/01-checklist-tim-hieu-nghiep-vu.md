# Checklist tìm hiểu nghiệp vụ Promotion Platform

## 1. Tổng quan

- [x] Sản phẩm: nền tảng quản lý khuyến mại tập trung.
- [x] Người vận hành chính: nhân viên BackOffice/Marketing/CRM.
- [x] Người hưởng lợi: khách hàng cuối sử dụng ưu đãi.
- [x] Giá trị: cấu hình nhanh chương trình ưu đãi, cá nhân hóa, kiểm soát ngân sách và theo dõi hiệu quả.
- [ ] KPI kinh doanh và KPI vận hành chính thức cần xác nhận với PO/BA.

## 2. Phạm vi theo giai đoạn

- [x] Phase 1: MVP Tanzania, trọng tâm Cashback và Segment.
- [x] Phase 2: Coupon/Voucher, Distribution, Validation, Stacking, Category, Metadata và tích hợp Viettel Money.
- [ ] Xác nhận chức năng nào của Phase 2 đã hoàn thành, đang phát triển hoặc bị defer.

## 3. Luồng nghiệp vụ cần nắm

| Mức ưu tiên | Luồng | Trạng thái tìm hiểu |
|---|---|:---:|
| P0 | Xử lý Cashback từ giao dịch đến trả tiền | [ ] |
| P0 | Tạo, kích hoạt và vận hành Coupon Campaign | [ ] |
| P0 | Qualification/Validation ưu đãi | [ ] |
| P0 | Redemption và Rollback | [ ] |
| P1 | Phân phối voucher tự động | [ ] |
| P1 | Quản lý Segment/Customer/Product | [ ] |
| P1 | Cấu hình Validation Rule và Formula | [ ] |
| P1 | Tra cứu voucher và khiếu nại CSKH | [ ] |
| P2 | Metadata và Category | [ ] |
| P2 | Báo cáo, thống kê và audit | [ ] |

## 4. Business object và lifecycle

- [ ] Campaign Cashback.
- [ ] Coupon Campaign.
- [ ] Voucher/Coupon Code.
- [ ] Validation Rule.
- [ ] Stacking Rule.
- [ ] Distribution Rule.
- [ ] Redemption.
- [ ] Customer và Segment.
- [ ] Budget và Redemption Limit.

## 5. Rule quan trọng

- [ ] Ai đủ điều kiện nhận/sử dụng ưu đãi?
- [ ] Khi nào campaign/voucher có hiệu lực?
- [ ] Cách tính giá trị Cashback/Discount?
- [ ] Giới hạn ngân sách và số lần sử dụng được kiểm tra ở đâu?
- [ ] Khi nhiều voucher cùng áp dụng thì ưu tiên thế nào?
- [ ] Chế độ ALL và PARTIAL khác nhau thế nào?
- [ ] Khi nào voucher được hold, release, redeem hoặc rollback?

## 6. Exception case

- [ ] Giao dịch trùng lặp.
- [ ] Campaign hết hạn hoặc bị tắt giữa luồng.
- [ ] Voucher hết hạn/đã dùng/không thuộc khách hàng.
- [ ] Hết ngân sách hoặc vượt hạn mức.
- [ ] Validation pass nhưng bước thanh toán thất bại.
- [ ] Redemption thất bại sau khi đã giữ ngân sách hoặc đổi trạng thái voucher.
- [ ] Gửi thông báo thất bại nhưng Cashback/Redemption thành công.
- [ ] Dữ liệu Customer/Segment/Order chưa đồng bộ.

## 7. Người cần xác nhận

| Chủ đề | Vai trò nên hỏi |
|---|---|
| Phạm vi và ưu tiên | PO/BA Lead |
| Rule Campaign/Coupon | BA phụ trách Campaign |
| Validation/Stacking | BA phụ trách Rule/Redemption |
| Cashback Tanzania | BA/Dev đã triển khai Phase 1 |
| Tích hợp app | BA Mobile/Product thanh toán |
| Khiếu nại voucher | CSKH/Operation |
| Đối soát và ngân sách | Finance/Operation/PO |

## 8. Mục tiêu hoàn thành

- [ ] Tự vẽ lại được hai luồng P0: Cashback và Coupon Redemption.
- [ ] Giải thích được khác nhau giữa Campaign, Voucher, Distribution và Redemption.
- [ ] Mô tả được vòng đời Coupon Campaign và Voucher.
- [ ] Nêu được ít nhất 10 business rule quan trọng.
- [ ] Nêu được các bước không thể rollback tự động.
- [ ] Biết các điểm tài liệu đang chưa thống nhất để hỏi đúng người.

