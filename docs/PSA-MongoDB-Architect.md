# Kiến trúc và Luồng hoạt động của Mô hình MongoDB Replica Set

## 1. Các thành phần trong kiến trúc (Components)

![architecture](/imgs/PSA-Model-Architecture.png)

Hệ thống bao gồm các thành phần cốt lõi sau:

- **Client Application**: Ứng dụng máy khách thao tác với cơ sở dữ liệu thông qua MongoDB Driver. Driver chịu trách nhiệm định tuyến các yêu cầu theo thực tế cấu hình (Read/Write Concern và Read Preference).
- **Primary Node (Nút chính)**: Là trung tâm của cụm Replica Set. Đây là nút duy nhất có quyền chấp nhận các thao tác **Ghi (Writes)**. Dữ liệu thay đổi trên Primary sẽ được lưu vào hệ thống nhật ký `oplog` (Operations Log).
- **Secondary Nodes (Nút phụ)**: Chứa bản sao dữ liệu y hệt như Primary. Chúng duy trì `oplog` riêng và liên tục đồng bộ từ Primary để cập nhật CSDL. Các nút này có thể phục vụ luồng **Đọc (Reads)** để giảm tải cho Primary nếu ứng dụng cấu hình Read Preference. _(Lưu ý thực tế: Mô hình PSA tiêu chuẩn chỉ gồm 3 node - 1 Primary, 1 Secondary, 1 Arbiter để tối ưu chi phí. Hình ảnh thể hiện kiến trúc mở rộng PSSA thực tế ít dùng vì nguyên tắc Replica Set cần số lượng node mang phiếu bầu là số lẻ để tránh split-brain)._
- **Arbiter Node (Nút phân xử)**: Nút đặc biệt **không lưu trữ dữ liệu** (không có `oplog` và quá trình replication). Nhiệm vụ cốt lõi duy nhất là tham gia bỏ phiếu (Election) giúp cụm đạt được số phiếu lẻ khi cần bầu Primary mới.

---

## 2. Luồng hoạt động trong thực tế (Workflow)

### 2.1. Luồng Giao tiếp Client - Database (Read/Write Flow)

Dựa trên nguyên tắc thực tế của MongoDB chứ không chỉ theo sơ đồ:

- **Writes (Ghi dữ liệu)**: Bắt buộc Client phải định tuyến các luồng ghi đến nút **Primary**. Primary thực thi và có thể cần chờ các Secondary phản hồi đã ghi vào oplog rồi mới báo hoàn tất (phụ thuộc vào cấu hình an toàn dữ liệu `Write Concern`).
- **Reads (Đọc dữ liệu)**:
  - **Mặc định**: Đọc trực tiếp từ **Primary** nhằm đảm bảo luôn đọc được dữ liệu mới nhất (Strong Consistency).
  - **Phân tán tải**: Theo như hình vẽ, Client cũng gửi thao tác Reads đến các nút **Secondary**. Thực tế điều này xảy ra khi Driver cấu hình `Read Preference` thành `secondary` hoặc `secondaryPreferred`. Tuy tăng lượng truy vấn phản hồi nhưng có thể gặp trường hợp dữ liệu cũ do độ trễ sao chép (Eventual Consistency).

### 2.2. Luồng Sao chép dữ liệu Đồng bộ (Replication)

- **Luồng hoạt động dưới nền**: Sau khi xử lý yêu cầu Ghi, **Primary** tự động lưu dấu vết thay đổi vào nhật ký `oplog`.
- **Chủ động từ Secondary**: Thực tế các nút **Secondary** sẽ liên tục truy vấn và "kéo" (pull) dữ liệu từ `oplog` của Primary về `oplog` nội bộ của chúng.
- Secondary sau đó áp dụng (apply) các thao tác tái hiện lại trên cơ sở dữ liệu để đồng nhất trạng thái hệ thống.

### 2.3. Cơ chế Giám sát và Chuyển đổi dự phòng (HeartBeat & Failover)

- **HeartBeat**: Quá trình diễn ra liên tục, tất cả các nút (kể cả Arbiter) trao đổi gói tin HeartBeat định kỳ (mặc định 2 giây/lần) để báo cáo trạng thái "sống" cho nhau.
- **Thực thi Failover & Election (Bầu cử)**:
  - Khi **Primary** gặp sự cố hoặc rớt mạng không phản hồi HeartBeat sau một khung thời gian cụ thể (mặc định `electionTimeoutMillis` là 10 giây).
  - Bất kỳ nút Secondary dữ liệu gần nhất nào phát hiện ra mất kết nối đều có thể tự kích hoạt quy trình Bầu cử (Election) và ứng cử bản thân.
  - Các nút còn lại sẽ bỏ phiếu. Do Arbiter tồn tại, cụm luôn đảm bảo ứng viên được bầu nếu có được đa số phiếu.
  - Secondary thắng cử tự động thăng cấp thành Primary mới, Client Driver cũng sẽ tự động định tuyến lại các Write vào nút mới này mà không tạo Downtime lớn (Sẵn sàng cao).
- **Phục hồi tự động**: Thực tế nếu nút Primary bị hỏng trước đó sống lại, nó sẽ tự động phát hiện Primary mới, sau đó giáng cấp (step-down) thành Secondary và tự đối soát lại `oplog` để sẵn sàng theo kịp dữ liệu trở lại.
