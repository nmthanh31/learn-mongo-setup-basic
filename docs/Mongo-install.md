# Hướng dẫn Cài đặt MongoDB trên Linux Server và Các khái niệm cơ bản

## 1. Các khái niệm và Lý thuyết cơ bản về MongoDB

Trước khi tiến hành cài đặt và vận hành, dưới đây là các khái niệm cốt lõi cần nắm về MongoDB:

### 1.1. MongoDB là gì?

MongoDB là một hệ quản trị cơ sở dữ liệu **NoSQL** mã nguồn mở, hoạt động theo mô hình lưu trữ hướng tài liệu (**Document-oriented**). Khác với các cơ sở dữ liệu quan hệ (RDBMS) truyền thống như MySQL hay SQL Server lưu trữ dữ liệu theo dạng bảng (table) và cột (column), MongoDB lưu trữ dữ liệu dưới dạng các tài liệu (document) mang định dạng giống JSON.

### 1.2. JSON và BSON

- **JSON (JavaScript Object Notation):** Là định dạng dữ liệu dễ đọc với con người.
- **BSON (Binary JSON):** MongoDB thực chất lưu trữ các document dưới định dạng BSON ở dưới nền. BSON giúp tối ưu hóa không gian lưu trữ và tốc độ quét (parsing), đồng thời hỗ trợ nhiều kiểu dữ liệu phức tạp hơn JSON (ví dụ: Date, BinData, ObjectId).

### 1.3. Ánh xạ thuật ngữ (SQL vs MongoDB)

Để dễ hình dung với những người đã quen thuộc thao tác trên hệ thống cơ sở dữ liệu quan hệ (RDBMS):

| Thuật ngữ SQL (RDBMS) | Thuật ngữ MongoDB     | Ý nghĩa                                  |
| :-------------------- | :-------------------- | :--------------------------------------- |
| Database              | Database              | Cơ sở dữ liệu chứa nhiều collection      |
| Table                 | Collection            | Tập hợp các tài liệu (documents)         |
| Row                   | Document              | Một bản ghi, đại diện cho một đối tượng  |
| Column                | Field                 | Một trường dữ liệu cụ thể trong document |
| Primary Key           | `_id`                 | Khóa chính, thường là ObjectId tự sinh   |
| Index                 | Index                 | Bảng mục lục giúp truy vấn nhanh hơn     |
| JOIN                  | `$lookup` / Embedding | Kết nối hoặc lồng ghép dữ liệu liên quan |

### 1.4. Lược đồ động (Dynamic / Flexible Schema)

Một trong những điểm mạnh lớn nhất của MongoDB là tính linh hoạt của Schema (cấu trúc). Các document trong cùng một collection không bắt buộc phải có cùng số lượng các trường dữ liệu, hay cùng kiểu dữ liệu cho một trường. Điều này giúp đẩy nhanh tốc độ phát triển (Agile) khi ứng dụng thường xuyên nâng cấp cấu trúc đối tượng hiển thị.

### 1.5. Thiết kế dữ liệu: Embedding vs Referencing

- **Embedding (Nhúng):** Bao gộp các dữ liệu con/liên quan vào chung trong một document duy nhất. Rất tốt cho các dữ liệu được truy vấn cùng nhau thường xuyên (để lấy ra trong 1 lần đọc duy nhất).
- **Referencing (Tham chiếu):** Chỉ lưu ID của document này vào document khác (giống Foreign Key trong SQL) và dùng `$lookup` để lấy ra khi cần. Phù hợp với dữ liệu tĩnh, dùng chung ở nhiều chỗ hoặc mảng dữ liệu có thể phình to vô hạn.

### 1.6. Tính Sẵn sàng (High Availability) và Mở rộng (Scalability)

- **Replica Set:** Là mô hình triển khai của MongoDB giúp nhân bản (replicate) dữ liệu ra nhiều server. Hệ thống tự động bầu chọn (Election) một Primary node để phục vụ đọc/ghi và những Secondary node để đồng bộ dự phòng. Điều này giúp hệ thống không bị gián đoạn (Downtime) nếu một server gặp sự cố (Tự động Failover).
- **Sharding:** Kỹ thuật phân mảnh dữ liệu để chia nhỏ và phân tán theo chiều ngang (Horizontal Scaling) trên nhiều server khi lượng dữ liệu và lưu lượng đọc/ghi vượt quá tải trọng của một cụm Replica Set duy nhất.

---

## 2. Hướng dẫn Cài đặt MongoDB (Community Edition) trực tiếp trên Linux

_Lưu ý: Hướng dẫn dưới đây là chuẩn các bước áp dụng cho các hệ điều hành họ Debian/Ubuntu (ví dụ: Ubuntu 20.04/22.04 LTS). Đối với RHEL/CentOS, các câu lệnh sử dụng `yum/dnf` thay vì `apt`, nhưng nguyên lý các bước là tương tự._

MongoDB không khuyến khích sử dụng gói cài đặt có sẵn trong repository mặc định của hệ điều hành (apt default) do chúng thường xuyên bị out-of-date. Do đó chúng ta sẽ dùng repository chính thức của MongoDB.

### Bước 1: Import khóa công khai (Public Key) của MongoDB

Chạy lệnh sau trên terminal để tải khóa về hệ thống (giúp xác thực các gói tin bảo mật từ MongoDB):

```bash
# Đổi "7.0" thành phiên bản bạn mong muốn, ví dụ: 6.0, 5.0
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -

# Hoặc theo hướng dẫn từ MongoDB chính thức:
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor
```

_(Nếu hệ điều hành Ubuntu mới báo cảnh báo `apt-key` bị deprecate, đây là cảnh báo bình thường, thao tác tiếp theo vẫn hoạt động. Hoặc bạn có thể tham khảo việc tải khóa GPG thẳng vào thư mục `/etc/apt/keyrings/` theo tài liệu trên trang chủ)._

### Bước 2: Thêm kho lưu trữ (Repository) của MongoDB

Chạy lệnh sau để list repository vào source (Với Ubuntu 22.04 Jammy):

```bash
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

### Bước 3: Cập nhật chỉ mục gói và Cài đặt

```bash
sudo apt-get update
# Cài đặt trình mongodb bản Community bao gồm mongodb-org-server, mongos, shell mongosh, và tools
sudo apt-get install -y mongodb-org
```

### Bước 4: Khởi động bộ máy MongoDB Service

Sau khi cài đặt xong, MongoDB chạy ngầm dưới tư cách một systemd service với tên là `mongod`.

Khởi động bộ máy:

```bash
sudo systemctl start mongod
```

Cấu hình cho phép Service tự chạy mỗi khi khởi động lại máy chủ (Startup):

```bash
sudo systemctl enable mongod
```

Kiểm tra trạng thái (Nên hiển thị dòng màu xanh _Active (running)_):

```bash
sudo systemctl status mongod
```

---

## 3. Cấu hình cơ bản sau khi cài đặt

### 3.1. Tổng quan

Sau khi cài đặt, MongoDB mặc định:

- Chỉ lắng nghe tại `localhost` (`127.0.0.1`)
- Không yêu cầu xác thực

Điều này:

- An toàn trong môi trường local
- Nhưng không thể dùng cho production hoặc hệ thống phân tán

Do đó cần cấu hình lại file: `/etc/mongod.conf`

### 3.2. Cấu hình mạng (Network Configuration)

**a. Cấu hình**

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1,<IP_NODE>
```

**b. Giải thích**

- **port**
  - Xác định cổng MongoDB sử dụng (mặc định: `27017`)
  - Client sẽ kết nối qua: `mongodb://<IP>:27017`
- **bindIp**
  - Bản chất: Là danh sách địa chỉ mà MongoDB lắng nghe kết nối
  - Phân tích từng giá trị:
    - `127.0.0.1` (Localhost Loopback):
      - **Bản chất**: Đây là địa chỉ loopback, chỉ có ý nghĩa trên chính máy tính/máy chủ đang chạy MongoDB. Các gói tin mạng gửi tới địa chỉ này sẽ không bao giờ đi ra ngoài card mạng vật lý (NIC).
      - **Mục đích an toàn**: Chặn hoàn toàn mọi truy cập từ ngoài Internet hoặc mạng LAN.
      - **Dùng cho**: Quản trị viên SSH trực tiếp vào server đó và gõ lệnh `mongosh`. Hoặc khi ứng dụng (Web server/API) chạy cùng trên một máy chủ vật lý/máy ảo với MongoDB.
    - `<IP_NODE>` (IP private của máy chủ - Card mạng LAN/VLAN):
      - **Bản chất**: Đây là địa chỉ IP được gán cho một card mạng (NIC). MongoDB không bind vào card mạng, mà bind vào IP đó, và hệ điều hành sẽ tự định tuyến lưu lượng qua card mạng tương ứng.
      - **Cách hoạt động**: Khi khai báo IP này, MongoDB bắt đầu lắng nghe các yêu cầu kết nối từ các máy tính khác (có thể ping được tới `<IP_NODE>`) thông qua switch mạng ảo/thật. Khi Client gửi request đến `<IP_NODE>:27017` thì OS sẽ nhận packet tại card mạng sau check IP đích tìm đến tiến trình MongoDB và chuyển tiếp packet đến tiến trình MongoDB.
      - **Bắt buộc trong**:
        - **Kiến trúc Client-server**: Khi code ứng dụng Backend (App Server) nằm ở một máy chủ khác với máy chủ chạy Database (DB Server).
        - **Replica Set / Cluster**: Các node MongoDB (Primary, Secondary, Arbiter) nằm trên các máy ảo khác nhau bắt buộc cần giao tiếp với nhau qua IP Private mạng LAN.
  - Cách hoạt động: MongoDB sẽ mở các socket (`127.0.0.1:27017` và `<IP_NODE>:27017`). Client kết nối tới IP nào → MongoDB phải listen tại IP đó.

**c. Trường hợp đặc biệt: 0.0.0.0**

```yaml
bindIp: 0.0.0.0
```

- Ý nghĩa: Lắng nghe trên tất cả các network interface
- Rủi ro: Mở toàn bộ hệ thống ra mọi mạng. Nếu có public IP → database bị expose.
- Điều kiện bắt buộc nếu sử dụng: Phải có firewall giới hạn IP truy cập.

### 3.3. Cấu hình bảo mật (Authentication)

**a. Cấu hình**

```yaml
security:
  authorization: enabled
```

**b. Ý nghĩa**
Kích hoạt cơ chế:

- Xác thực người dùng (Authentication)
- Phân quyền (Authorization)

**c. Cách hoạt động**
Khi client kết nối:

1. MongoDB yêu cầu đăng nhập.
2. Client gửi `username/password`.
3. MongoDB kiểm tra: Tài khoản hợp lệ, Quyền truy cập (role).
4. Cho phép hoặc từ chối thao tác.

**d. Nếu không bật**
Bất kỳ ai truy cập được port đều có toàn quyền:

- Đọc/ghi dữ liệu
- Xóa database
- Tạo user admin
  → Đây là lỗi bảo mật nghiêm trọng.

### 3.4. Cơ chế hoạt động tổng thể

Quá trình kết nối tới MongoDB:

- **Bước 1: Client gửi request** (`connect` → `10.0.0.5:27017`)
- **Bước 2: Kiểm tra network**: MongoDB có bind IP này không? Firewall có cho phép không?
- **Bước 3: Kiểm tra bảo mật**: Nếu `authorization = enabled` → yêu cầu đăng nhập.
- **Bước 4: Truy cập hệ thống**: Nếu hợp lệ → cho phép thao tác. Nếu không → từ chối.

### 3.5. Áp dụng cấu hình

Sau khi chỉnh sửa file `/etc/mongod.conf`:

```bash
sudo systemctl restart mongod
```

---

## 4. Tạo người dùng Quản trị (Admin User) đầu tiên

Sau khi bật Authentication (`authorization: "enabled"`), bạn bắt buộc phải có user/password để dùng CSDL.

Tuy nhiên, MongoDB có một cơ chế ngoại lệ thông minh tên là `localhost exception`. Khi cấu hình yêu cầu xác thực ĐÃ BẬT nhưng thực tế LẠI CHƯA CÓ bất kỳ user nào được tạo trước đó trên database `admin`, MongoDB sẽ tạm cho phép bạn truy cập từ localhost mà không cần mật khẩu để bạn... tạo user :)

Vào trình shell của MongoDB (gõ trực tiếp trên máy chủ chứa cài đặt):

```bash
mongosh
```

Chạy cụm lệnh sau để chỉ định cơ sở dữ liệu hệ thống `admin` và tạo một tài khoản SuperRoot:

```javascript
use admin

db.createUser(
  {
    user: "rootDbAdmin",
    pwd: passwordPrompt(),  // Hệ thống sẽ bắt bạn nhập mật khẩu trên terminal cho an toàn thay vì gõ chữ text thường
    // Cấp đặc quyền cao cấp nhất có thể quản trị mọi Database
    roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
  }
)
```

Thoát ra và kiểm tra kết nối lại với tư cách user vừa tạo:

```bash
exit   # thoát mongosh

# Đăng nhập lại với tài khoản vừa tạo (cần nhập mật khẩu nãy bạn gõ):
mongosh -u "rootDbAdmin" -p --authenticationDatabase "admin"
```
