# Hướng Dẫn Cài Đặt MongoDB Replica Set Mô Hình PSA (1P, 1S, 1A)

Tài liệu này hướng dẫn thiết lập MongoDB Replica Set theo mô hình chuẩn PSA: **1 Primary, 1 Secondary và 1 Arbiter**.

## 1. Tổng Quan về Mô Hình 3 Node (PSA)
- **Primary**: Nhận tất cả thao tác ghi.
- **Secondary (1 node)**: Lưu trữ bản sao dữ liệu.
- **Arbiter (1 node)**: **Không lưu dữ liệu**, chỉ tham gia bầu chọn để quyết định Primary khi có sự cố.

> [!NOTE]
> **Lưu ý về bầu chọn**: Mô hình PSA có 3 node tham gia bầu chọn (số lẻ), đây là mô hình tối ưu chi phí nhằm đảm bảo luôn có đa số phiếu (majority = 2) để bầu cử Primary mới. Khả năng chịu lỗi của hệ thống là **1 node**. Không nên dùng mô hình 4 node (1P, 2S, 1A) vì số lượng node bầu chọn là số chẵn dễ dẫn đến hiện tượng bế tắc bầu cử (split-brain) mà không tăng thêm mức độ chịu lỗi.
> *Nguồn tham khảo: [Replica Set Architectures](https://www.mongodb.com/docs/manual/core/replica-set-architectures/)*

---

## 2. Chuẩn Bị
- 2 node cài đặt MongoDB đầy đủ (để chứa dữ liệu gồm Primary và Secondary).
- 1 node Arbiter (có thể dùng cấu hình thấp hơn vì không lưu dữ liệu).
- Cấu hình file `mongod.conf` trên cả 3 node với cùng `replSetName`.

---

## 3. Các Bước Cài Đặt Chi Tiết

### Bước 1: Cấu hình File `mongod.conf`
Bạn cần chỉnh sửa file cấu hình vật lý trên hệ điều hành tại đường dẫn: `/etc/mongod.conf`. 
- **Cách thực hiện**: Dùng quyền `root` để sửa (Ví dụ: `sudo nano /etc/mongod.conf`).

> [!NOTE]
> *Nguồn tham khảo: [Configuration File Options](https://www.mongodb.com/docs/manual/reference/configuration-options/)*

**Mẫu cấu hình đầy đủ (Sử dụng cho cả 3 node):**
```yaml
# FILE PATH: /etc/mongod.conf

# 1. Cấu hình log hệ thống
systemLog:
  destination: file              # Ghi log ra file thay vì xuất ra màn hình Terminal
  path: "/var/log/mongodb/mongod.log" # Đường dẫn lưu trữ file log
  logAppend: true                # Lưu tiếp nội dung vào file cũ khi khởi động lại (không xóa log cũ)

# 2. Quản lý tiến trình
processManagement:
  fork: true                    # Chạy MongoDB dưới dạng tiến trình ngầm (background)

# 3. Cấu hình lưu trữ dữ liệu
storage:
  dbPath: "/var/lib/mongodb"    # Thư mục vật lý chứa dữ liệu trên ổ cứng

# 4. Cấu hình mạng
net:
  port: 27017                   # Cổng kết nối mặc định của MongoDB
  bindIp: "127.0.0.1,<IP_NODE>" # Đây là danh sách mà MongoDB cho phép kết nối tới. (127.0.0.1 là cho phép kết nối nội bộ, <IP_NODE> là cho phép kết nối tới IP đó với cổng 27017)

# 5. Cấu hình Replica Set (Bắt buộc cho cụm)
replication:
  replSetName: "rs0"            # Tên của Replica Set (phải giống hệt nhau trên cả 3 máy)
```

Sau khi sửa xong, hãy chạy lệnh này tại **Terminal của Linux** (quyền root/sudo):
```bash
sudo systemctl restart mongod
```

### Bước 2: Khởi Tạo Replica Set (Thực hiện bên trong MongoDB Shell)
Để gọi được các lệnh `rs.xxx`, bạn phải truy cập vào bộ điều khiển của MongoDB.

1. Tại **Terminal của Linux**, gõ:
```bash
mongosh
```

2. Sau khi đã vào trong giao diện của `mongosh` (dấu nhắc lệnh sẽ thay đổi), hãy copy đoạn mã sau để khởi tạo:

```javascript
// THỰC HIỆN TRONG MONGOSH SHELL
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-primary:27017", priority: 2 },
    { _id: 1, host: "mongodb-secondary:27017", priority: 1 },
    { _id: 2, host: "mongodb-arbiter:27017", arbiterOnly: true }
  ]
})
```

---

## 4. Bảo Mật (Internal Authentication)

### 1. Tạo Keyfile (Thực hiện tại Linux Terminal)
Chạy các lệnh này ngoài màn hình đen của Linux:
```bash
openssl rand -base64 756 > /var/lib/mongodb/keyfile
chmod 400 /var/lib/mongodb/keyfile
chown mongodb:mongodb /var/lib/mongodb/keyfile
```

### 2. Khai báo vào `mongod.conf`
Bạn quay lại file `/etc/mongod.conf` ở Bước 1 và bổ sung thêm đoạn sau:
```yaml
# SỬA TRONG FILE: /etc/mongod.conf
security:
  keyFile: /var/lib/mongodb/keyfile
  authorization: enabled
```

---

## 5. Vận Hành và Bảo Trì (Thực hiện bên trong MongoDB Shell)
Tất cả các lệnh này đều phải được gõ sau khi bạn đã vào `mongosh`.

- **Xem trạng thái chi tiết**: `rs.status()`
- **Xem cấu hình**: `rs.conf()`
- **Thêm node khác**: `rs.add("newnode:27017")`
- **Xóa node**: `rs.remove("oldnode:27017")`

> [!IMPORTANT]
> Luôn kiểm tra file log tại `/var/log/mongodb/mongod.log` nếu gặp lỗi trong quá trình khởi chạy hoặc kết nối giữa các node.
