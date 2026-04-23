# BÀI KIỂM TRA SỐ 2
### PHẦN MỞ ĐẦU
- Họ và tên: Nguyễn Minh Hạnh
- Mã sv: K235480106023
- Lớp: K59KMT
- YÊU CẦU CỦA ĐỀ: 

     Thiết kế và khởi tạo một Database mới.

     Tạo ít nhất 3 bảng có mối quan hệ với nhau.

     Sử dụng đa dạng các kiểu dữ liệu (INT, FLOAT, NVARCHAR, DATE, MONEY...).

     Áp dụng đúng quy tắc đặt tên Bướu Lạc Đà (CamelCase/PascalCase) và bọc tên bằng ngoặc vuông [ ].

     Thiết lập và giải thích rõ các Khóa chính (PK), Khóa ngoại (FK) và Ràng buộc kiểm tra (CK).
### PHẦN 1: KHỞI TẠO CƠ SỞ DỮ LIỆU VÀ CÁC BẢNG
1. Khởi tạo cơ sở dữ liệu

<img width="1920" height="1080" alt="Screenshot (197)" src="https://github.com/user-attachments/assets/b05c9657-0565-4c0c-9649-9b95c42b700a" />

<p align="center">Tạo Database</p>

Lựa chọn đề tài: Quản lí thư viện

2. Tạo các bảng dữ liệu

- Bảng [DocGia] (Quản lý người dùng): lưu trữ thông tin định danh của người mượn sách.


Khóa chính [MaDocGia] được cài đặt tự động tăng (IDENTITY) để tối ưu việc đánh chỉ mục và tạo liên kết.

Trường [HoTenDocGia] sử dụng kiểu NVARCHAR để lưu trữ chuỗi văn bản tiếng Việt có dấu một cách chính xác.

Trường [TienDatCoc] (kiểu MONEY, giá trị mặc định bằng 0). Đây là cơ sở để thực hiện các logic nghiệp vụ sau này (ví dụ: yêu cầu độc giả phải có số dư tiền cọc nhất định mới được phép mượn sách có giá trị cao, hoặc dùng để khấu trừ khi phạt trễ hạn).

```sql
CREATE TABLE [DocGia] (
    [MaDocGia] INT IDENTITY(1,1) NOT NULL, 
    [HoTenDocGia] NVARCHAR(100) NOT NULL,
    [NgaySinh] DATE NULL,
    [TienDatCoc] MONEY DEFAULT 0,
    CONSTRAINT [PK_DocGia] PRIMARY KEY ([MaDocGia])
);
```



<img width="1920" height="1080" alt="Screenshot (199)" src="https://github.com/user-attachments/assets/d4ea2f60-801c-46fb-96cb-b96bd3fcfbdd" />
<p align="center">Tạo bảng DocGia</p>


- Bảng [Sach] (Quản lý kho tài liệu): * Lưu trữ danh mục các cuốn sách có trong thư viện. [MaSach] là khóa chính tự tăng.

Để đảm bảo dữ liệu hệ thống luôn "sạch" và tránh lỗi do người thao tác nhập liệu sai (Data entry errors), em đã thiết lập 2 Ràng buộc kiểm tra cứng (Check Constraint):

[CK_DiemDanhGia]: Ràng buộc điểm đánh giá của sách bắt buộc phải nằm trong thang điểm chuẩn từ 0.0 đến 10.0 (kiểu FLOAT).

[CK_NamXuatBan]: Đảm bảo tính logic về thời gian bằng cách sử dụng hàm YEAR(GETDATE()). Sách không thể có năm xuất bản lớn hơn thời điểm hệ thống hiện tại.
```sql
CREATE TABLE [Sach] (
    [MaSach] INT IDENTITY(1,1) NOT NULL,
    [TenSach] NVARCHAR(200) NOT NULL,
    [GiaBan] MONEY NOT NULL,
    [DiemDanhGia] FLOAT NULL,
    [NamXuatBan] INT NULL,
    CONSTRAINT [PK_Sach] PRIMARY KEY ([MaSach]),
    CONSTRAINT [CK_DiemDanhGia] CHECK ([DiemDanhGia] >= 0 AND [DiemDanhGia] <= 10),
    CONSTRAINT [CK_NamXuatBan] CHECK ([NamXuatBan] <= YEAR(GETDATE())) 
);
```
<img width="1920" height="1080" alt="Screenshot (200)" src="https://github.com/user-attachments/assets/3cf6662a-440f-487c-b9ad-2c0ec1a63460" />
<p align="center">Tạo bảng Sach</p>


- Bảng [PhieuMuon] (Quản lý giao dịch): Để xử lý mối quan hệ này trong cơ sở dữ liệu quan hệ, cần xây dựng bảng trung gian PhieuMuon. Bảng này có nhiệm vụ liên kết giữa độc giả và sách, đồng thời lưu trữ thông tin về từng lần mượn.

Khóa chính [MaPhieu] Định danh duy nhất mỗi phiếu mượn.

Bảng sử dụng hai Khóa ngoại (Foreign Key) là [MaDocGia] và [MaSach] Liên kết với bảng DocGia và Sach → đảm bảo dữ liệu hợp lệ.

Trường [NgayMuon] Đảm bảo ngày trả đúng logic (không trước ngày mượn).

Ràng buộc [CK_NgayTraHopLe] được thêm vào để khóa chặt logic thực tế: thời hạn trả dự kiến ([NgayTraDuKien]) không bao giờ được phép xảy ra trước thời điểm mượn sách.
```sql
CREATE TABLE [PhieuMuon] (
    [MaPhieu] INT IDENTITY(1,1) NOT NULL,
    [MaDocGia] INT NOT NULL,
    [MaSach] INT NOT NULL,
    [NgayMuon] DATETIME NOT NULL DEFAULT GETDATE(),
    [NgayTraDuKien] DATETIME NOT NULL,
    CONSTRAINT [PK_PhieuMuon] PRIMARY KEY ([MaPhieu]),
    CONSTRAINT [FK_PhieuMuon_DocGia] FOREIGN KEY ([MaDocGia]) REFERENCES [DocGia]([MaDocGia]),
    CONSTRAINT [FK_PhieuMuon_Sach] FOREIGN KEY ([MaSach]) REFERENCES [Sach]([MaSach]),
    CONSTRAINT [CK_NgayTraHopLe] CHECK ([NgayTraDuKien] >= [NgayMuon])
);
```


<img width="1920" height="1080" alt="Screenshot (201)" src="https://github.com/user-attachments/assets/3fc4ce2e-fd27-4fee-8d0a-1efef3799344" />
<p align="center">Tạo bảng PhieuMuon</p>

3. Chèn dữ liệu vào các bảng.
   
<img width="1920" height="1080" alt="Screenshot (203)" src="https://github.com/user-attachments/assets/8b4c2f82-eba7-4622-a166-149ab3682085" />

<p align="center">Dữ liệu đã tạo</p>

###PHẦN 2:

1. Các hàm có sẵn (Built-in Functions) trong SQL Server

Trong SQL Server, các hàm có sẵn (built-in functions) được chia thành nhiều nhóm khác nhau dựa trên mục đích sử dụng, bao gồm:

Hàm xử lý chuỗi (String Functions): LEN(), SUBSTRING(), REPLACE(), UPPER(), LOWER(), CONCAT()...

Hàm toán học (Math Functions): ROUND(), ABS(), POWER(), CEILING(), FLOOR()...

Hàm ngày tháng (Date/Time Functions): GETDATE(), DATEDIFF(), DATEADD(), YEAR(), MONTH(), DAY()...

Hàm tập hợp (Aggregate Functions): Dùng chung với GROUP BY như SUM(), COUNT(), MAX(), MIN(), AVG().

Hàm hệ thống và logic (System & Logical Functions): CAST(), CONVERT(), ISNULL(), COALESCE(), IIF(), NEWID()...


Một vài System / Built-in Function đặc sắc:

NEWID(): Hàm tạo ra một chuỗi định danh duy nhất (GUID). Điểm đặc sắc là khi kết hợp với mệnh đề ORDER BY NEWID(), ta có thể lấy ngẫu nhiên các bản ghi từ một bảng (rất hữu ích để làm tính năng "Gợi ý sách ngẫu nhiên").




IIF(): Hàm logic hoạt động giống hệt toán tử 3 ngôi (điều kiện ? đúng : sai) trong lập trình. Giúp viết các câu lệnh rẽ nhánh ngắn gọn hơn rất nhiều so với dùng CASE WHEN.

FORMAT(): Hàm định dạng dữ liệu (số, ngày tháng) dựa trên văn hóa vùng miền (Culture). Rất mạnh mẽ khi muốn format tiền tệ thành chuẩn Việt Nam (VD: 150.000 ₫).
