# Promotion Management Platform - Business Guide

Bộ tài liệu này giúp người mới hiểu dự án theo nghiệp vụ, được tổng hợp từ tài liệu Promotion Phase 1 - Tanzania và Promotion Phase 2.

## Cách đọc nhanh

Nếu chỉ có 60 phút, đọc theo thứ tự:

1. [Tổng quan dự án](02-tong-quan-du-an.md)
2. [Bản đồ các luồng nghiệp vụ](05-ban-do-luong-nghiep-vu.md)
3. [Luồng xử lý Cashback](flows/02-phase1-xu-ly-cashback.md)
4. [Luồng chiến dịch Coupon](flows/03-phase2-chien-dich-coupon.md)
5. [Validation và Stacking](flows/05-kiem-tra-hop-le-va-cong-don.md)
6. [Redemption và Rollback](flows/06-doi-thuong-va-hoan-tac.md)
7. [Điểm chưa thống nhất cần làm rõ](08-diem-chua-thong-nhat-va-cau-hoi-can-lam-ro.md)

## Danh mục tài liệu

| File | Nội dung |
|---|---|
| [01-checklist-tim-hieu-nghiep-vu.md](01-checklist-tim-hieu-nghiep-vu.md) | Checklist onboarding đã điền trước theo dự án |
| [02-tong-quan-du-an.md](02-tong-quan-du-an.md) | Mục tiêu, phạm vi Phase 1 và Phase 2 |
| [03-thuat-ngu-va-doi-tuong-nghiep-vu.md](03-thuat-ngu-va-doi-tuong-nghiep-vu.md) | Glossary và quan hệ giữa các business object |
| [04-actor-va-he-thong-lien-quan.md](04-actor-va-he-thong-lien-quan.md) | Ai làm gì, hệ thống nào cung cấp/nhận dữ liệu |
| [05-ban-do-luong-nghiep-vu.md](05-ban-do-luong-nghiep-vu.md) | Danh sách và mức ưu tiên các luồng chính |
| [06-vong-doi-va-trang-thai.md](06-vong-doi-va-trang-thai.md) | Lifecycle Campaign, Voucher, Rule, Redemption |
| [07-business-rules-va-edge-cases.md](07-business-rules-va-edge-cases.md) | Quy tắc cốt lõi và trường hợp ngoại lệ |
| [08-diem-chua-thong-nhat-va-cau-hoi-can-lam-ro.md](08-diem-chua-thong-nhat-va-cau-hoi-can-lam-ro.md) | Các điểm mâu thuẫn hoặc còn thiếu trong tài liệu |
| [09-lo-trinh-doc-tai-lieu.md](09-lo-trinh-doc-tai-lieu.md) | Kế hoạch học nghiệp vụ trong 10 ngày |
| [flows/](flows/) | Mô tả chi tiết từng luồng end-to-end |
| [99-nguon-tham-chieu.md](99-nguon-tham-chieu.md) | Vị trí nội dung trong hai PDF nguồn |

## Các luồng chi tiết

| Luồng | File |
|---|---|
| Cấu hình nền cho Cashback | [flows/01-phase1-cau-hinh-cashback.md](flows/01-phase1-cau-hinh-cashback.md) |
| Xử lý Cashback từ giao dịch | [flows/02-phase1-xu-ly-cashback.md](flows/02-phase1-xu-ly-cashback.md) |
| Tạo và vận hành Coupon Campaign | [flows/03-phase2-chien-dich-coupon.md](flows/03-phase2-chien-dich-coupon.md) |
| Phân phối ưu đãi tự động | [flows/04-phan-phoi-uu-dai.md](flows/04-phan-phoi-uu-dai.md) |
| Kiểm tra hợp lệ và cộng dồn | [flows/05-kiem-tra-hop-le-va-cong-don.md](flows/05-kiem-tra-hop-le-va-cong-don.md) |
| Đổi thưởng và hoàn tác | [flows/06-doi-thuong-va-hoan-tac.md](flows/06-doi-thuong-va-hoan-tac.md) |
| Trải nghiệm trên app Viettel Money | [flows/07-mobile-viettel-money.md](flows/07-mobile-viettel-money.md) |
| Tra cứu khiếu nại cho CSKH | [flows/08-tra-cuu-cskh.md](flows/08-tra-cuu-cskh.md) |

## Nguyên tắc sử dụng

- Tài liệu này ưu tiên ý nghĩa nghiệp vụ, không mô tả chi tiết code hoặc hạ tầng.
- Khi tài liệu nguồn có nhiều cách gọi/trạng thái khác nhau, phần cần xác nhận được đánh dấu rõ.
- Với task mới, hãy xác định task thuộc luồng nào, object nào và rule nào trước khi đọc code.

