# Hệ thống MongoDB Replica Set - VNPost

Dự án này chứa các tài liệu hướng dẫn cấu hình và kiến trúc chi tiết để triển khai hệ thống cơ sở dữ liệu MongoDB theo mô hình có tính sẵn sàng cao (High Availability) bằng cơ chế Replica Set dành riêng cho VNPost.

## Nội dung Tài liệu (`docs/`)

Trong thư mục `docs/`, bạn sẽ xem được các tài liệu hướng dẫn sau:

1. **[Hướng dẫn Cài đặt MongoDB và Khái niệm cơ bản](docs/Mongo-install.md)**
   - Các khái niệm cốt lõi về MongoDB, JSON/BSON và cách thiết kế dữ liệu.
   - Hướng dẫn cài đặt MongoDB (Community Edition) trên môi trường Linux (Debian/Ubuntu).
   - Thiết lập cấu hình cơ bản: kết nối từ xa (`bindIp`) và bật bảo mật danh tính (`authorization`).

2. **[Kiến trúc Mô hình Replica Set PSA](docs/PSA-MongoDB-Architect.md)**
   - Giải thích kiến trúc theo mô hình chuẩn PSA (1 Primary, 1 Secondary, 1 Arbiter).
   - Chi tiết luồng giao tiếp Đọc/Ghi dữ liệu (Read/Write Flow) từ Client Application đến Database.
   - Cơ chế Đồng bộ dữ liệu (Replication) tự động và quá trình chuyển đổi dự phòng (HeartBeat, Failover & Election).

3. **[Hướng dẫn Thiết lập Replica Set (Mô hình PSA)](docs/Mongo-setup-replica.md)**
   - Hướng dẫn cấu hình chi tiết cho 3 node (1 Primary nhận Ghi/Đọc, 1 Secondary đồng bộ dữ liệu và 1 Arbiter để bầu chọn lúc cần thiết).
   - Các bước khởi tạo Cluster qua lệnh `rs.initiate` trên MongoDB Shell.
   - Khởi tạo bảo mật kết nối nội bộ giữa các Node (Internal Authentication / Keyfile) và các lệnh vận hành.

## Kiến trúc Tổng quan
Hệ thống sử dụng mô hình tối ưu **PSA** nhằm giảm thiểu chi phí đầu tư phần cứng bằng cách sử dụng một Server (node Arbiter) cấu hình thấp hơn nhưng vẫn mang lại khả năng chịu lỗi tối đa và chống Downtime hiệu quả cho cơ sở dữ liệu lúc hệ thống (Primary/Secondary) gặp sự cố.

> [!NOTE]  
> Mọi thay đổi về cấu hình thực tế trên máy chủ cần thông qua người quản trị viên Hệ thống (System Administrator) / DBA để luôn đảm bảo tính an toàn dữ liệu.
