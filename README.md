# BÀI KIỂM TRA SỐ 2
### PHẦN MỞ ĐẦU
- Họ và tên: Nguyễn Minh Hạnh
- Mã sv: K235480106023
- Lớp: K59KMT
  
**Yêu cầu đề bài**

**ĐỀ TÀI: QUẢN LÝ THƯ VIỆN**

  Thực hiện xây dựng một hệ thống quản lý thư viện hoàn chỉnh trên SQL Server, đáp ứng đầy đủ các yêu cầu của bài kiểm tra.

Toàn bộ quá trình thực hiện phải được ghi lại bằng các screenshot minh họa. Mỗi hình ảnh cần đi kèm câu lệnh SQL tương ứng và phần chú thích rõ ràng về chức năng, mục đích xử lý cũng như kết quả đạt được.

Bài tập được nộp dưới dạng GitHub Repository (public) gồm hai thành phần chính:

README.md: chứa toàn bộ nội dung báo cáo, hình ảnh minh họa và giải thích chi tiết  
baikiemtra2.sql: chứa toàn bộ mã SQL sử dụng trong bài làm  
  
**Giới thiệu về hệ thống quản lý thư viện**
- Hệ thống tập trung xây dựng Cơ sở dữ liệu "Quản lý Thư viện" trên nền tảng SQL Server. Mục tiêu của hệ thống là quản lý chặt chẽ thông tin Độc giả, Kho sách và quy trình Mượn/Trả tài liệu.
 Không chỉ dừng lại ở việc lưu trữ dữ liệu cơ bản, hệ thống còn được thiết kế như một phần mềm quản lý thu nhỏ, tích hợp trực tiếp các nghiệp vụ tự động hóa vào tầng cơ sở dữ liệu như: tính tiền phạt trễ hạn, kiểm duyệt hạn mức mượn sách, tự động trừ tồn kho và thanh toán nợ luân phiên. Điều này giúp giảm thiểu tối đa sai sót từ con người và tối ưu hóa công tác quản trị.

- Hệ thống được phát triển theo hướng Mô-đun hóa và Bảo vệ toàn vẹn dữ liệu từ gốc, trải qua 5 bước thiết kế logic:

1. Thiết kế Schema & Ràng buộc: Xây dựng 3 bảng cốt lõi (DocGia, Sach, PhieuMuon) kết hợp hệ thống khóa (PK, FK) và ràng buộc (CHECK) để chặn dữ liệu rác ngay từ lúc nhập liệu.

2. Đóng gói logic tính toán (Functions): Viết các hàm tự định nghĩa (UDF) để xử lý các phép tính nghiệp vụ đặc thù (VD: tính toán tiền phạt), giúp mã nguồn gọn gàng và dễ tái sử dụng.

3. Kiểm duyệt giao dịch (Stored Procedures): Thiết lập các SP đóng vai trò như "cổng kiểm duyệt" để ràng buộc điều kiện (kiểm tra tiền cọc, nợ xấu) trước khi cho phép lập phiếu mượn mới.

4. Tự động hóa ngầm (Triggers): Ứng dụng Trigger để tự động hóa các tác vụ nền như cập nhật số lượng tồn kho sách; đồng thời thực nghiệm và kiểm soát rủi ro vòng lặp đệ quy.

5. Xử lý dữ liệu tuần tự (Cursors): Sử dụng vòng lặp Cursor để giải quyết triệt để các bài toán phân bổ tài nguyên phức tạp như "Thanh toán nợ luân phiên" (FIFO) mà lệnh SQL tập hợp (Set-based) thông thường rất khó xử lý.

 Mỗi đối tượng trong SQL Server (Table, Function, SP, Trigger, Cursor) đều được khai thác đúng thế mạnh đặc thù, tạo thành một hệ thống mượt mà, bảo mật và mang tính tự động hóa cao.
### PHẦN 1: KHỞI TẠO CƠ SỞ DỮ LIỆU VÀ CÁC BẢNG
**1. Khởi tạo cơ sở dữ liệu**

<img width="1920" height="1080" alt="Screenshot (197)" src="https://github.com/user-attachments/assets/b05c9657-0565-4c0c-9649-9b95c42b700a" />

<p align="center">Tạo Database</p>

**2. Tạo các bảng dữ liệu**

a) Bảng [DocGia] (Quản lý người dùng)

- lưu trữ thông tin định danh của người mượn sách.

- Khóa chính [MaDocGia] được cài đặt tự động tăng (IDENTITY) để tối ưu việc đánh chỉ mục và tạo liên kết.

- Trường [HoTenDocGia] sử dụng kiểu NVARCHAR để lưu trữ chuỗi văn bản tiếng Việt có dấu một cách chính xác.

- Trường [TienDatCoc] (kiểu MONEY, giá trị mặc định bằng 0). Đây là cơ sở để thực hiện các logic nghiệp vụ sau này (ví dụ: yêu cầu độc giả phải có số dư tiền cọc nhất định mới được phép mượn sách có giá trị cao, hoặc dùng để khấu trừ khi phạt trễ hạn).

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
<p align="center">Tạo bảng [DocGia]</p>

---

b)  Bảng [Sach] (Quản lý kho tài liệu)

- Lưu trữ danh mục các cuốn sách có trong thư viện. [MaSach] là khóa chính tự tăng.

- Để đảm bảo dữ liệu hệ thống luôn "sạch" và tránh lỗi do người thao tác nhập liệu sai (Data entry errors), em đã thiết lập 2 Ràng buộc kiểm tra cứng (Check Constraint):

- [CK_DiemDanhGia]: Ràng buộc điểm đánh giá của sách bắt buộc phải nằm trong thang điểm chuẩn từ 0.0 đến 10.0 (kiểu FLOAT).

- [CK_NamXuatBan]: Đảm bảo tính logic về thời gian bằng cách sử dụng hàm YEAR(GETDATE()). Sách không thể có năm xuất bản lớn hơn thời điểm hệ thống hiện tại.
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
<p align="center">Tạo bảng [Sach]</p>

---

c) Bảng [PhieuMuon] (Quản lý giao dịch)

- Để xử lý mối quan hệ này trong cơ sở dữ liệu quan hệ, cần xây dựng bảng trung gian PhieuMuon. Bảng này có nhiệm vụ liên kết giữa độc giả và sách, đồng thời lưu trữ thông tin về từng lần mượn.

 Khóa chính [MaPhieu] Định danh duy nhất mỗi phiếu mượn.

- Bảng sử dụng hai Khóa ngoại (Foreign Key) là [MaDocGia] và [MaSach] Liên kết với bảng DocGia và Sach → đảm bảo dữ liệu hợp lệ.

- Trường [NgayMuon] Đảm bảo ngày trả đúng logic (không trước ngày mượn).

- Ràng buộc [CK_NgayTraHopLe] được thêm vào để khóa chặt logic thực tế: thời hạn trả dự kiến ([NgayTraDuKien]) không bao giờ được phép xảy ra trước thời điểm mượn sách.
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
<p align="center">Tạo bảng [PhieuMuon]</p>

**3. Chèn dữ liệu vào các bảng.**


```sql
-- ---------------------------------------------------------------------------------
-- 1. Thêm 20 dòng cho bảng [DocGia]
-- ---------------------------------------------------------------------------------
INSERT INTO [DocGia] ([HoTenDocGia], [NgaySinh], [TienDatCoc]) VALUES
(N'Nguyễn Văn An', '2000-01-15', 50000),
(N'Trần Thị Bích', '1999-05-20', 100000),
(N'Lê Hoàng Cường', '2001-11-02', 0),
(N'Phạm Thu Dung', '1998-03-12', 50000),
(N'Hoàng Văn Em', '2002-07-25', 100000),
(N'Đặng Thị Phương', '1995-12-10', 0),
(N'Bùi Minh Trí', '2003-02-28', 150000),
(N'Vũ Thị Hà', '1997-09-05', 50000),
(N'Ngô Tuấn Anh', '2000-06-18', 0),
(N'Lý Mỹ Ngọc', '1996-08-22', 50000),
(N'Trần Trọng Đạt', '1999-04-14', 100000),
(N'Nguyễn Thị Hoa', '2001-10-30', 50000),
(N'Lê Đức Tài', '1998-01-08', 0),
(N'Phan Thanh Tú', '2002-11-15', 100000),
(N'Dương Tấn Phát', '1994-05-09', 150000),
(N'Vương Cẩm Ly', '2003-07-11', 50000),
(N'Hồ Bích Liên', '1997-03-27', 0),
(N'Đỗ Bá Kiên', '1995-09-19', 50000),
(N'Mai Xuân Lộc', '2000-12-05', 100000),
(N'Trịnh Quốc Vượng', '1999-08-01', 50000);
GO

-- ---------------------------------------------------------------------------------
-- 2. Thêm 20 dòng cho bảng [Sach]
-- ---------------------------------------------------------------------------------
INSERT INTO [Sach] ([TenSach], [GiaBan], [DiemDanhGia], [NamXuatBan]) VALUES
(N'Đắc Nhân Tâm', 85000, 9.5, 2018),
(N'Nhà Giả Kim', 75000, 9.0, 2019),
(N'Tuổi Trẻ Đáng Giá Bao Nhiêu', 90000, 8.5, 2020),
(N'Lược Sử Loài Người', 150000, 9.8, 2017),
(N'Tội Ác Và Hình Phạt', 120000, 9.2, 2015),
(N'Số Đỏ', 60000, 8.8, 2016),
(N'Chí Phèo', 55000, 8.7, 2014),
(N'Tôi Thấy Hoa Vàng Trên Cỏ Xanh', 100000, 9.1, 2018),
(N'Mắt Biếc', 110000, 9.3, 2019),
(N'Harry Potter và Hòn Đá Phù Thủy', 200000, 9.7, 2021),
(N'Kỹ Năng Sống Dành Cho Sinh Viên', 65000, 7.5, 2022),
(N'Lập Trình C Căn Bản', 180000, 8.0, 2023),
(N'Cơ Sở Dữ Liệu SQL Server', 220000, 8.9, 2023),
(N'Tư Duy Nhanh Và Chậm', 190000, 9.6, 2018),
(N'Rừng Na Uy', 140000, 8.4, 2016),
(N'Không Gia Đình', 135000, 9.0, 2017),
(N'Suối Nguồn', 250000, 9.4, 2015),
(N'Bố Già', 160000, 9.5, 2019),
(N'Đại Gia Gatsby', 95000, 8.6, 2020),
(N'Những Người Khốn Khổ', 300000, 9.8, 2014);
GO

-- ---------------------------------------------------------------------------------
-- 3. Thêm 20 dòng cho bảng [PhieuMuon]
-- Dữ liệu giả lập có các phiếu mượn trong quá khứ, hiện tại và có phiếu bị trễ hạn.
-- Chú ý: MaDocGia (1-20), MaSach (1-20). NgayTraDuKien luôn >= NgayMuon
-- ---------------------------------------------------------------------------------
INSERT INTO [PhieuMuon] ([MaDocGia], [MaSach], [NgayMuon], [NgayTraDuKien]) VALUES
(1, 3, '2023-10-01', '2023-10-15'), -- Phiếu trong quá khứ (sẽ bị coi là quá hạn nếu chưa trả)
(2, 5, '2023-11-10', '2023-11-20'), 
(3, 1, '2024-01-05', '2024-01-15'),
(1, 10, '2024-02-01', '2024-02-10'),
(4, 13, '2024-02-15', '2024-02-28'),
(5, 7, '2024-03-01', '2024-03-10'),
(6, 8, '2024-03-15', '2024-03-25'),
(7, 2, '2024-03-20', '2024-03-30'),
(8, 15, '2024-04-01', '2024-04-15'),
(9, 12, '2024-04-05', '2024-04-20'),
(10, 4, '2024-04-10', '2024-04-25'),
(11, 18, '2024-04-12', '2024-04-26'),
(12, 19, '2024-04-15', '2024-04-25'),
(13, 20, '2024-04-18', '2024-04-30'),
(1, 6, '2024-04-20', '2024-05-05'),  -- Phiếu mới mượn (Trong hạn)
(14, 9, '2024-04-21', '2024-05-10'),
(15, 11, '2024-04-22', '2024-05-06'),
(16, 14, '2024-04-23', '2024-05-07'),
(17, 16, '2024-04-23', '2024-05-07'),
(18, 17, '2024-04-23', '2024-05-15');
GO
```
   
<img width="1920" height="1080" alt="Screenshot (203)" src="https://github.com/user-attachments/assets/8b4c2f82-eba7-4622-a166-149ab3682085" />

<p align="center">Dữ liệu đã tạo</p>

### PHẦN 2: XÂY DỰNG FUNCTION 

**1. Các hàm có sẵn (Built-in Functions) trong SQL Server**

- Trong SQL Server, các hàm có sẵn (built-in functions) được chia thành nhiều nhóm khác nhau dựa trên mục đích sử dụng, bao gồm:

-Hàm xử lý chuỗi (String Functions): LEN(), SUBSTRING(), REPLACE(), UPPER(), LOWER(), CONCAT()...

-Hàm toán học (Math Functions): ROUND(), ABS(), POWER(), CEILING(), FLOOR()...

-Hàm ngày tháng (Date/Time Functions): GETDATE(), DATEDIFF(), DATEADD(), YEAR(), MONTH(), DAY()...

-Hàm tập hợp (Aggregate Functions): Dùng chung với GROUP BY như SUM(), COUNT(), MAX(), MIN(), AVG().

-Hàm hệ thống và logic (System & Logical Functions): CAST(), CONVERT(), ISNULL(), COALESCE(), IIF(), NEWID()...


- Một vài System / Built-in Function đặc sắc:

-NEWID(): Hàm tạo ra một chuỗi định danh duy nhất (GUID). Điểm đặc sắc là khi kết hợp với mệnh đề ORDER BY NEWID(), ta có thể lấy ngẫu nhiên các bản ghi từ một bảng (rất hữu ích để làm tính năng "Gợi ý sách ngẫu nhiên").

-IIF(): Hàm logic hoạt động giống hệt toán tử 3 ngôi (điều kiện ? đúng : sai) trong lập trình. Giúp viết các câu lệnh rẽ nhánh ngắn gọn hơn rất nhiều so với dùng CASE WHEN.

-FORMAT(): Hàm định dạng dữ liệu (số, ngày tháng) dựa trên văn hóa vùng miền (Culture). Rất mạnh mẽ khi muốn format tiền tệ thành chuẩn Việt Nam (VD: 150.000 ₫).

- Lệnh SQL khai thác các hàm trên:

```sql
-- Lấy ngẫu nhiên 3 cuốn sách, định dạng tiền tệ kiểu VN và đánh giá phân loại nhanh:
SELECT TOP 3 
    MaSach,
    TenSach, 
    FORMAT(GiaBan, 'C0', 'vi-VN') AS GiaBanVND,
    IIF(DiemDanhGia >= 8.0, N'Sách HOT', N'Sách Thường') AS PhanLoai
FROM [Sach]
ORDER BY NEWID();
```

**2. Hàm do người dùng tự viết (User-Defined Functions - UDF)**

a) Mục đích: UDF được tạo ra để đóng gói (encapsulate) các đoạn logic tính toán phức tạp hoặc các câu truy vấn được sử dụng lặp đi lặp lại. Việc này giúp mã SQL gọn gàng, tăng tính tái sử dụng và dễ dàng bảo trì.

b) Phân loại và thời điểm sử dụng:

- Scalar Function (Hàm vô hướng): 

  Đặc điểm: Nhận vào tham số và trả về đúng 1 giá trị đơn lẻ (INT, VARCHAR, MONEY,...).

  Khi nào dùng: Khi cần thực hiện một phép tính toán học, xử lý chuỗi hoặc tính toán dựa trên ngày tháng (VD: Tính tuổi từ ngày sinh, tính tiền phạt trễ hạn).

-Inline Table-Valued Function - iTVF (Hàm nội tuyến trả về bảng):

  Đặc điểm: Trả về 1 bảng (Table). Bên trong hàm chỉ chứa duy nhất một câu lệnh SELECT.

  Khi nào dùng: Dùng như một VIEW nhưng ưu việt hơn vì có thể truyền được tham số vào để lọc dữ liệu. Hiệu năng của iTVF rất tốt.

- Multi-statement Table-Valued Function - mTVF (Hàm đa câu lệnh trả về bảng):

  Đặc điểm: Trả về 1 bảng (Table). Bên trong hàm có thể chứa nhiều câu lệnh xử lý phức tạp (IF/ELSE, WHILE, biến bảng tạm DECLARE @Table TABLE).

  Khi nào dùng: Khi cần phải qua nhiều bước xử lý trung gian (thêm, sửa, xóa trên bảng tạm) mới ra được tập kết quả cuối cùng.

c) Tại sao có nhiều hàm Built-in rồi mà vẫn cần tự viết function riêng?

- Bởi vì các hàm có sẵn chỉ thực hiện các thao tác nền tảng (cộng trừ, đếm ngày, cắt chuỗi...). Chúng không thể hiểu được nghiệp vụ (Business Logic) của bài toán.

- Ví dụ: Hàm DATEDIFF() có sẵn chỉ đếm được khoảng cách giữa 2 ngày. Nhưng hệ thống thư viện yêu cầu: "Trễ hạn bị phạt 5.000đ/ngày". Lúc này bắt buộc người lập trình phải kết hợp DATEDIFF với các phép toán nhân/chia để viết thành một UDF tên là fn_TinhTienPhat().

**3. Thực hành viết Function cho Database Quản lý Thư Viện**

a) Scalar Function (Hàm trả về một giá trị)

- Yêu cầu: Hệ thống thư viện quy định nếu độc giả trả sách quá hạn sẽ bị phạt 5,000 VND / 1 ngày. Viết hàm truyền vào ngày trả dự kiến và ngày thực trả để tính ra số tiền phạt.

- Luồng xử lý dữ :

Bước 1. Hàm nhận vào ngày trả dự kiến và ngày trả thực tế làm tham số đầu vào.

Bước 2. Hệ thống tính toán khoảng thời gian chênh lệch (số ngày trễ) giữa hai mốc thời gian này bằng hàm DATEDIFF.

Bước 3. Kiểm tra điều kiện logic: Nếu số ngày trễ lớn hơn 0 (tức là khách hàng trả sách quá hạn).

Bước 4. Thực hiện phép tính nhân số ngày trễ với mức phạt quy định của thư viện (5.000 VNĐ/ngày) để ra tổng tiền phạt.

Bước 5. Trả về kết quả là một giá trị duy nhất (kiểu MONEY) đại diện cho tổng số tiền phạt phải đóng.

- CODE TẠO HÀM

```sql 
CREATE FUNCTION [dbo].[fn_TinhTienPhatQuaHan] (
    @NgayTraDuKien DATETIME,
    @NgayTraThucTe DATETIME
)
RETURNS MONEY
AS
BEGIN
    DECLARE @TienPhat MONEY = 0;
    -- Tính số ngày trễ (nếu trả trước hoặc đúng hạn thì kết quả <= 0)
    DECLARE @SoNgayTre INT = DATEDIFF(DAY, @NgayTraDuKien, @NgayTraThucTe);
    
    IF @SoNgayTre > 0
        SET @TienPhat = @SoNgayTre * 5000; -- Phạt 5000đ cho mỗi ngày trễ
        
    RETURN @TienPhat;
END;
GO
```

<img width="1920" height="1080" alt="Screenshot (202)" src="https://github.com/user-attachments/assets/4163c61e-c971-4bb2-a6c5-5c944338912c" />

<p align="center">Hàm tính tiền phạt quá hạn </p>
- CODE KHAI THÁC 

```sql
-- Hiển thị danh sách phiếu mượn và số tiền phạt (giả sử nếu họ trả sách vào ngày hôm nay)
SELECT 
    MaPhieu, 
    MaDocGia, 
    NgayTraDuKien, 
    GETDATE() AS NgayTraHomNay,
    [dbo].[fn_TinhTienPhatQuaHan](NgayTraDuKien, GETDATE()) AS TienPhatDuKien
FROM [PhieuMuon];
GO
```

<img width="1920" height="1080" alt="Screenshot (205)" src="https://github.com/user-attachments/assets/a18c05c7-47ea-4886-8384-e7e4d5303883" />
<p align="center">Hàm hiển thị danh sách phiếu mượn và số tiền </p>

---

 b) Inline Table-Valued Function

- Yêu cầu: Cần một hàm để lấy nhanh "Lịch sử mượn sách" của một Độc giả cụ thể. Tham số truyền vào là MaDocGia. Hàm cần nối (JOIN) bảng Phiếu Mượn và Sách để lấy tên sách.

- Luồng xử lý dữ liệu:

Bước 1. Hàm nhận vào mã độc giả làm tham số đầu vào.

Bước 2. Hệ thống truy xuất dữ liệu từ bảng phiếu mượn (PhieuMuon) để lấy các thông tin liên quan đến các lần giao dịch mượn sách của độc giả.

Bước 3. Đồng thời, hệ thống kết nối với bảng sách (Sach) để lấy thêm thông tin về tên sách tương ứng.

Bước 4. Thực hiện liên kết giữa hai bảng thông qua mã sách (MaSach).

Bước 5. Lọc dữ liệu theo mã độc giả đã truyền vào để chỉ lấy các giao dịch mượn/trả của độc giả đó.

Bước 6. Trả về danh sách kết quả bao gồm:

Mã phiếu mượn

Tên sách

Ngày mượn

Ngày trả dự kiến

- CODE TẠO HÀM 

```sql
CREATE FUNCTION [dbo].[fn_LichSuMuonSachCuaDocGia] (
    @MaDocGia INT
)
RETURNS TABLE
AS
RETURN (
    SELECT 
        pm.MaPhieu, 
        s.TenSach, 
        pm.NgayMuon, 
        pm.NgayTraDuKien
    FROM [PhieuMuon] pm
    INNER JOIN [Sach] s ON pm.MaSach = s.MaSach
    WHERE pm.MaDocGia = @MaDocGia
);
GO
```
<img width="1920" height="1080" alt="Screenshot (207)" src="https://github.com/user-attachments/assets/46f0df93-4c0f-47ab-8b23-b27bbaa45fde" />

<p align="center">Hàm tạo lịch sử mượn sách </p>

- CODE KHAI THÁC 

```sql
-- Lấy lịch sử mượn sách của Độc giả có MaDocGia = 1
SELECT * FROM [dbo].[fn_LichSuMuonSachCuaDocGia](1);
GO
```

<img width="1920" height="1080" alt="Screenshot (208)" src="https://github.com/user-attachments/assets/567e6bc9-0f7b-4c1f-a772-0764a5c0bc84" />

<p align="center">Lịch sử mượn sách của Độc giả </p>

---

c) Multi-statement Table-Valued Function

- Yêu cầu: Cần xuất một "Báo cáo tổng hợp tình trạng Phiếu mượn". Báo cáo gồm: MaPhieu, MaDocGia, Tình trạng ("Trong hạn" hoặc "Đã quá hạn"), Số tiền phạt dự kiến. Do cần thiết lập trạng thái mặc định rồi mới kiểm tra và cập nhật trạng thái quá hạn, ta sử dụng biến bảng để xử lý nhiều bước.

- Luồng xử lý dữ liệu:

Bước 1. Hệ thống khởi tạo một biến bảng (bảng tạm trong bộ nhớ) để định hình cấu trúc dữ liệu báo cáo trả về.

Bước 2. Truy xuất toàn bộ dữ liệu giao dịch từ bảng phiếu mượn và chèn vào biến bảng, đồng thời thiết lập trạng thái mặc định ban đầu là "Trong hạn" và tiền phạt bằng 0.

Bước 3. Hệ thống quét lại tập dữ liệu trong biến bảng và thực hiện so sánh ngày trả dự kiến của từng phiếu với ngày giờ hiện tại của hệ thống (GETDATE()).

Bước 4. Lọc ra các phiếu có ngày trả dự kiến nhỏ hơn ngày hiện tại (phiếu vi phạm), thực hiện cập nhật (UPDATE) trạng thái của chúng thành "Đã quá hạn".

Bước 5. Đồng thời, gọi ngược lại hàm Scalar Function (ở phần A) để tính toán tự động và cập nhật chính xác số tiền phạt cho các phiếu vi phạm đó.

Bước 6. Trả về danh sách kết quả báo cáo cuối cùng bao gồm:

Mã phiếu

Mã độc giả

Tình trạng (Trong hạn / Đã quá hạn)

Tiền phạt

- CODE TẠO HÀM

```sql
CREATE FUNCTION [dbo].[fn_BaoCaoTinhTrangCacPhieuMuon] ()
RETURNS @BaoCao TABLE (
    MaPhieu INT,
    MaDocGia INT,
    TinhTrang NVARCHAR(50),
    TienPhat MONEY
)
AS
BEGIN
    -- Bước 1: Đổ dữ liệu ban đầu vào biến bảng, mặc định giả định là "Trong hạn" và tiền phạt = 0
    INSERT INTO @BaoCao (MaPhieu, MaDocGia, TinhTrang, TienPhat)
    SELECT MaPhieu, MaDocGia, N'Trong hạn', 0
    FROM [PhieuMuon];

    -- Bước 2: Dùng UPDATE để sửa lại tình trạng của những phiếu đã bị lố ngày (quá hạn)
    -- Tận dụng lại hàm Scalar đã tạo ở phần A để tính tiền phạt
    UPDATE @BaoCao
    SET 
        TinhTrang = N'Đã quá hạn',
        TienPhat = [dbo].[fn_TinhTienPhatQuaHan](pm.NgayTraDuKien, GETDATE())
    FROM @BaoCao b
    INNER JOIN [PhieuMuon] pm ON b.MaPhieu = pm.MaPhieu
    WHERE pm.NgayTraDuKien < GETDATE(); 

    -- Trả về bảng @BaoCao cuối cùng
    RETURN;
END;
GO
```
<img width="1920" height="1080" alt="Screenshot (211)" src="https://github.com/user-attachments/assets/df95857a-64ba-46af-b504-1597d5b8f634" />

 - CODE KHAI THÁC HÀM

 ```sql
-- Hiển thị toàn bộ báo cáo tình trạng phiếu mượn
SELECT * FROM [dbo].[fn_BaoCaoTinhTrangCacPhieuMuon]();

-- Hoặc chỉ lọc ra những người đang bị phạt tiền
SELECT * FROM [dbo].[fn_BaoCaoTinhTrangCacPhieuMuon]() WHERE TienPhat > 0;
GO
```

<img width="1920" height="1080" alt="Screenshot (209)" src="https://github.com/user-attachments/assets/ac23e419-d7dd-4966-a285-de669e7bf358" />

### PHẦN 3: XÂY DỰNG STORE PROCEDURE

**1. Tìm hiểu về System Stored Procedures**

Trong SQL Server, System Stored Procedures (tiền tố sp_) là những thủ tục được viết sẵn và lưu trong database master. Chúng đóng vai trò như các "công cụ quản trị" giúp DBA (Database Administrator) hoặc Developer kiểm tra hệ thống, cấu hình và lấy metadata một cách nhanh chóng thay vì phải viết các câu truy vấn phức tạp vào các bảng hệ thống.

Một số System SP mang tính ứng dụng cao (Góc nhìn Developer):

- sp_helptext

Cách dùng: EXEC sp_helptext 'Ten_SP_Hoac_View'

Đặc sắc: Đây là lệnh "soi mã nguồn". Khi bạn tiếp nhận một database cũ và muốn biết một View hay Stored Procedure hoạt động ra sao, lệnh này sẽ in ra toàn bộ source code ban đầu của đối tượng đó.

- sp_spaceused

Cách dùng: EXEC sp_spaceused 'DocGia'

Đặc sắc: Báo cáo cực nhanh dung lượng ổ cứng mà một bảng đang chiếm dụng (có bao nhiêu row, tốn bao nhiêu KB data, bao nhiêu KB index). Rất hữu ích khi tối ưu hóa database.

- sp_depends

Cách dùng: EXEC sp_depends 'Sach'

Đặc sắc: "Tra cứu gia phả". Nó cho biết bảng Sach đang được sử dụng bởi những View, Function hay SP nào. Khi bạn muốn sửa bảng Sach (ví dụ đổi tên cột), bạn chạy lệnh này để biết mình cần phải sửa code ở những SP nào khác để không bị lỗi dây chuyền.

**2. Viết Store Procedure kiểm tra điều kiện logic(INSERT/UPDATE)**

- Ý tưởng: "Kiểm duyệt Mượn Sách Chặt Chẽ"

- Quy trình xử lý: Khi tạo phiếu mượn mới (INSERT), hệ thống sẽ không cho mượn vô điều kiện. Nó phải kiểm tra 2 lớp bảo mật:

-Độc giả có đang chứa sách quá hạn nào không? (Nếu có -> Cấm mượn thêm).

-Cuốn sách định mượn có giá trị lớn hơn Tiền Đặt Cọc của độc giả không? (Nếu Giá bán > Tiền cọc -> Báo lỗi rủi ro cao, không cho mượn).

- Code SQL

 ```sql
CREATE PROCEDURE [dbo].[sp_TaoPhieuMuonKiemDuyet]
    @MaDocGia INT,
    @MaSach INT,
    @SoNgayMuon INT
AS
BEGIN
    DECLARE @TienCoc MONEY;
    DECLARE @GiaSach MONEY;
    DECLARE @DangNoQuaHan INT;

    -- 1. Lấy thông tin tiền cọc và giá sách
    SELECT @TienCoc = TienDatCoc FROM [DocGia] WHERE MaDocGia = @MaDocGia;
    SELECT @GiaSach = GiaBan FROM [Sach] WHERE MaSach = @MaSach;

    -- 2. Kiểm tra xem độc giả có đang giữ sách nào quá hạn không
    SELECT @DangNoQuaHan = COUNT(*) 
    FROM [PhieuMuon] 
    WHERE MaDocGia = @MaDocGia AND NgayTraDuKien < GETDATE();

    -- KIỂM TRA LOGIC
    IF @DangNoQuaHan > 0
    BEGIN
        PRINT N'TỪ CHỐI: Độc giả đang có sách quá hạn chưa trả. Vui lòng trả sách trước khi mượn mới!';
        RETURN;
    END

    IF @GiaSach > @TienCoc
    BEGIN
        PRINT N'TỪ CHỐI: Sách này có giá trị (' + CAST(@GiaSach AS NVARCHAR) + N') cao hơn tiền đặt cọc (' + CAST(@TienCoc AS NVARCHAR) + N'). Yêu cầu nạp thêm cọc!';
        RETURN;
    END

    -- Nếu qua hết bài test, tiến hành INSERT
    DECLARE @NgayTra DATETIME = DATEADD(DAY, @SoNgayMuon, GETDATE());
    INSERT INTO [PhieuMuon] (MaDocGia, MaSach, NgayMuon, NgayTraDuKien)
    VALUES (@MaDocGia, @MaSach, GETDATE(), @NgayTra);

    PRINT N'THÀNH CÔNG: Đã duyệt và tạo phiếu mượn!';
END;
GO
 ```

<img width="1920" height="1080" alt="Screenshot (212)" src="https://github.com/user-attachments/assets/d10a1e14-d133-4c2b-a0c5-110531c4592a" />
<p align="center">Kiểm tra và đối chiếu thông tin mượn sách của độc giả</p>

- Kiểm thử chương 
```sql
-- Kịch bản 1: Thất bại do Độc giả đang có sách mượn quá hạn (Nguyễn Văn An - MaDocGia 1)
PRINT N'--- KỊCH BẢN 1: ---';
EXEC [dbo].[sp_TaoPhieuMuonKiemDuyet] @MaDocGia = 1, @MaSach = 2, @SoNgayMuon = 7;

-- Kịch bản 2: Thất bại do Sách đắt hơn Tiền đặt cọc (Vũ Thị Hà mã 8 có cọc 50k, mượn Harry Potter 200k)
PRINT N'--- KỊCH BẢN 2: ---';
EXEC [dbo].[sp_TaoPhieuMuonKiemDuyet] @MaDocGia = 8, @MaSach = 10, @SoNgayMuon = 7;

-- Kịch bản 3: Thành công (Mai Xuân Lộc mã 19 có cọc 100k, chưa nợ sách nào, mượn Số Đỏ giá 60k)
PRINT N'--- KỊCH BẢN 3: ---';
EXEC [dbo].[sp_TaoPhieuMuonKiemDuyet] @MaDocGia = 19, @MaSach = 6, @SoNgayMuon = 7;
GO
```
<img width="1920" height="1080" alt="Screenshot (227)" src="https://github.com/user-attachments/assets/b1fbb00a-3801-45f1-b0c8-9bbafc09861c" />
<p align="center">Kiểm thử điều kiện mượn sách của 3 độc giả</p>

**3. Viết Store Procedure có sử dụng tham số OUTPUT để trả về một giá trị tính toán**

- Ý tưởng : "Tính toán Hạn mức (Quota) mượn sách còn lại"

- Quy trình xử lý: Thư viện quy định cứ 50.000 VNĐ tiền cọc thì được phép mượn 1 cuốn sách cùng lúc. SP này sẽ đếm số sách độc giả đang cầm, đối chiếu với tiền cọc, từ đó tính toán xem họ còn được phép mượn tối đa bao nhiêu cuốn nữa và xuất ra biến OUTPUT. (Rất hay dùng để hiển thị trên app/web cho người dùng xem).
- Code SQL
```sql
CREATE PROCEDURE [dbo].[sp_TinhHanMucMuonConLai]
    @MaDocGia INT,
    @SoSachDuocMuonThem INT OUTPUT -- Biến đầu ra chứa kết quả
AS
BEGIN
    DECLARE @TienCoc MONEY = 0;
    DECLARE @SoSachDangMuon INT = 0;
    DECLARE @TongHanMuc INT = 0;

    -- Lấy tiền cọc
    SELECT @TienCoc = ISNULL(TienDatCoc, 0) FROM [DocGia] WHERE MaDocGia = @MaDocGia;
    
    -- Đếm số sách đang mượn (giả sử các phiếu trong DB là chưa trả)
    SELECT @SoSachDangMuon = COUNT(*) FROM [PhieuMuon] WHERE MaDocGia = @MaDocGia;

    -- Tính tổng hạn mức (Cứ 50k = 1 cuốn)
    SET @TongHanMuc = CAST((@TienCoc / 50000) AS INT);

    -- Tính số lượng còn lại
    SET @SoSachDuocMuonThem = @TongHanMuc - @SoSachDangMuon;

    -- Nếu mượn lố hoặc cọc không đủ, trả về 0 (không bị âm)
    IF @SoSachDuocMuonThem < 0 
        SET @SoSachDuocMuonThem = 0;
END;
GO
```

<img width="1920" height="1080" alt="Screenshot (213)" src="https://github.com/user-attachments/assets/4d49f524-aabf-4a48-aa7f-9bbf85a4b608" />
<p align="center">Đối chiếu hạn mức của độc giả</p>

- Kiểm thử chương trình
```sql
DECLARE @KetQua INT;
-- Gọi thực thi SP cho Độc giả 2 (Trần Thị Bích - Cọc 100k -> Mức đa 2 cuốn. Đang mượn 1 cuốn)
EXEC [dbo].[sp_TinhHanMucMuonConLai] 
    @MaDocGia = 2, 
    @SoSachDuocMuonThem = @KetQua OUTPUT;
```
<img width="1920" height="1080" alt="Screenshot (229)" src="https://github.com/user-attachments/assets/d2e01f14-6732-434a-9785-6b6a93d7db69" />
<p align="center">Đối chiếu hạn mức của độc giả Trần Thị Bích</p>

**4. Viết 01 Store Procedure trả về một tập kết quả từ lệnh SELECT**

- Ý tưởng : "Báo cáo Truy Tìm Phạt Nguội & Ước tính thiệt hại"

- Quy trình xử lý: Hệ thống cần cung cấp cho nhân viên thư viện một chức năng cho phép chỉ với một thao tác, có thể liệt kê danh sách các độc giả trả sách trễ nhiều nhất.

Chức năng này được xây dựng dưới dạng Stored Procedure, thực hiện:

-Kết nối dữ liệu từ 3 bảng liên quan (JOIN)

-Sử dụng lệnh SELECT để:  
    
Tính số ngày trả trễ.
    
Dự báo số tiền phạt (5.000 VNĐ/ngày)

Kết quả trả về giúp nhân viên thư viện nhanh chóng xác định các trường hợp quá hạn nghiêm trọng và chủ động liên hệ nhắc nhở hoặc xử lý.

- Code SQL
```sql
CREATE PROCEDURE [dbo].[sp_BaoCaoTruyTimSachQuaHan]
AS
BEGIN
    -- Trả về 1 Result Set chi tiết
    SELECT 
        pm.MaPhieu,
        dg.HoTenDocGia,
        s.TenSach,
        pm.NgayTraDuKien,
        -- Tính số ngày đã trễ hạn tính tới hôm nay
        DATEDIFF(DAY, pm.NgayTraDuKien, GETDATE()) AS SoNgayTreHan,
        -- Tính tiền phạt (5000đ/ngày)
        DATEDIFF(DAY, pm.NgayTraDuKien, GETDATE()) * 5000 AS UocTinhTienPhat,
        -- Đánh giá rủi ro
        IIF(DATEDIFF(DAY, pm.NgayTraDuKien, GETDATE()) > 30, N'Rủi ro mất sách', N'Gọi điện nhắc nhở') AS HanhDongDeXuat
    FROM [PhieuMuon] pm
    INNER JOIN [DocGia] dg ON pm.MaDocGia = dg.MaDocGia
    INNER JOIN [Sach] s ON pm.MaSach = s.MaSach
    -- Chỉ lọc những phiếu đã quá hạn
    WHERE pm.NgayTraDuKien < GETDATE()
    -- Sắp xếp: Ai trễ lâu nhất (tiền phạt cao nhất) đẩy lên đầu tiên
    ORDER BY SoNgayTreHan DESC;
END;
GO
```
<img width="1920" height="1080" alt="Screenshot (214)" src="https://github.com/user-attachments/assets/8bf04cf9-f077-4c27-8e95-baedb6324eb9" />
<p align="center"></p>

- Kiểm thử SQL
```sql
EXEC [dbo].[sp_BaoCaoTruyTimSachQuaHan];
GO
```
<img width="1920" height="1080" alt="Screenshot (230)" src="https://github.com/user-attachments/assets/1cb9fb54-1050-4874-8fb0-53fc4842de75" />
<p align="center">Kiểm tra các sách quá hạn</p>

 ### Phần 4: Trigger và Xử lý logic nghiệp vụ ###
 **1. Viết 01 Trigger xử lý logic thực tế**
- Ý tưởng logic : "Quản lý Tồn kho Sách tự động"

- Thực tế: Trong thư viện, mỗi cuốn sách sẽ có số lượng nhất định. Khi một thủ thư lập Phiếu Mượn mới (Bảng A: PhieuMuon), hệ thống phải tự động vào kho sách (Bảng B: Sach) để trừ đi 1 cuốn. Nếu kho đã hết sách (Số lượng = 0) mà thủ thư vẫn cố tình tạo phiếu mượn, hệ thống phải báo lỗi và hủy thao tác ngay lập tức.

- Code SQL
```sql
ALTER TABLE [Sach] ADD SoLuongTon INT DEFAULT 5;
GO
UPDATE [Sach] SET SoLuongTon = 5;
GO

CREATE TRIGGER [dbo].[trg_MuonSach_TruTonKho]
ON [PhieuMuon]
AFTER INSERT 
AS
BEGIN
    DECLARE @MaSachVuaMuon INT;
    DECLARE @SoLuongHienTai INT;

    SELECT @MaSachVuaMuon = MaSach FROM inserted;

    SELECT @SoLuongHienTai = SoLuongTon FROM [Sach] WHERE MaSach = @MaSachVuaMuon;

    IF @SoLuongHienTai <= 0
    BEGIN
        RAISERROR(N'LỖI: Cuốn sách này đã hết trong kho. Không thể cho mượn!', 16, 1);
        ROLLBACK TRANSACTION;
        RETURN;
    END
    ELSE
    BEGIN
        UPDATE [Sach]
        SET SoLuongTon = SoLuongTon - 1
        WHERE MaSach = @MaSachVuaMuon;
        
        PRINT N'HỆ THỐNG: Đã tự động trừ 1 cuốn sách trong kho!';
    END
END;
GO
```
<img width="1920" height="1080" alt="Screenshot (220)" src="https://github.com/user-attachments/assets/ac07db9a-6757-4e8d-b221-df4714ec6251" />
<p align="center">Tạo Trigger Quản Lý Tồn Kho Sách tự động</p>

- Kiểm thử Trigger đã tạo
```sql
INSERT INTO [PhieuMuon] (MaDocGia, MaSach, NgayMuon, NgayTraDuKien)
VALUES (1, 1, GETDATE(), GETDATE() + 7);
```
<img width="1920" height="1080" alt="Screenshot (233)" src="https://github.com/user-attachments/assets/03c6a9d4-4558-4344-8a71-bf59154e1ccb" />
<p align="center">Kiểm thử MaSach= 1</p>
<img width="1920" height="1080" alt="Screenshot (233)" src="https://github.com/user-attachments/assets/46497f0f-3056-4034-a7fe-6c5590e5beed" />
<p align="center">Kiểm tra lại bảng Sách, sẽ thấy cột SoLuongTon của MaSach = 1 bị giảm xuống còn 1.</p>

**2.Thực nghiệm Trigger chéo (A cập nhật B, B cập nhật lại A) trên dữ liệu thực tế**
- Ta sẽ sử dụng 2 bảng thực tế là [PhieuMuon] (Bảng A) và [DocGia] (Bảng B) với một kịch bản logic giả định như sau:

Khi Phiếu mượn của một người được gia hạn (thực hiện lệnh UPDATE trên [PhieuMuon]), hệ thống ưu đãi tự động cộng thêm tiền cọc cho Độc giả đó (thực hiện lệnh UPDATE trên [DocGia]).

Mặt khác, khi tiền cọc của một người thay đổi (thực hiện lệnh UPDATE trên [DocGia]), hệ thống lại tự động gia hạn thêm 1 ngày cho tất cả Phiếu mượn của họ (thực hiện lệnh UPDATE trên [PhieuMuon]).

- Code SQL
```sql
-- Trigger 1: Bảng Phiếu Mượn cập nhật Bảng Độc Giả
CREATE TRIGGER trg_PhieuMuon_CapNhat_DocGia
ON [PhieuMuon]
AFTER UPDATE
AS
BEGIN
    PRINT N'Trigger trên PhieuMuon đang chạy... tiến hành cập nhật DocGia';
    
    -- Thưởng 100đ tiền cọc cho độc giả có phiếu mượn vừa được cập nhật
    UPDATE d
    SET d.TienDatCoc = d.TienDatCoc + 100
    FROM [DocGia] d
    INNER JOIN inserted i ON d.MaDocGia = i.MaDocGia;
END;
GO

-- Trigger 2: Bảng Độc Giả cập nhật lại Bảng Phiếu Mượn
CREATE TRIGGER trg_DocGia_CapNhat_PhieuMuon
ON [DocGia]
AFTER UPDATE
AS
BEGIN
    PRINT N'Trigger trên DocGia đang chạy... tiến hành cập nhật PhieuMuon';
    
    -- Gia hạn thêm 1 ngày cho phiếu mượn của độc giả vừa được cập nhật tiền cọc
    UPDATE p
    SET p.NgayTraDuKien = DATEADD(DAY, 1, p.NgayTraDuKien)
    FROM [PhieuMuon] p
    INNER JOIN inserted i ON p.MaDocGia = i.MaDocGia;
END;
GO
```
<img width="1920" height="1080" alt="Screenshot (223)" src="https://github.com/user-attachments/assets/5dc99ff0-03a1-421e-81ed-ef2f1b72f60d" />
<p align="center">Tạo Trigger chéo nhau</p>

- Khởi động vòng lặp
```sql
UPDATE [PhieuMuon] 
SET NgayTraDuKien = '2024-12-31' 
WHERE MaPhieu = 1;
```
<img width="1920" height="1080" alt="Screenshot (234)" src="https://github.com/user-attachments/assets/61daa516-d25b-4ee5-98c9-3768d2c8791a" />
<p align="center">Gia hạn thử 1 phiếu mượn (MaPhieu=1)</p>

a) Quan sát thông báo của hệ thống (Result / Messages):

- Ngay khi chạy lệnh UPDATE ở trên, hệ thống không kết thúc ngay mà tab Messages trong SQL Server sẽ in ra các dòng chữ chạy liên tục như đánh bóng bàn:

Trigger trên PhieuMuon đang chạy... tiến hành cập nhật DocGia

Trigger trên DocGia đang chạy... tiến hành cập nhật PhieuMuon

Trigger trên PhieuMuon đang chạy... tiến hành cập nhật DocGia

(Lặp lại 32 lần)

Msg 217, Level 16, State 1, Procedure trg_PhieuMuon_CapNhat_DocGia, Line ...

Maximum stored procedure, function, trigger, or view nesting level exceeded (limit 32).

b) Phân tích chi tiết và Nhận xét sự cố

- Phân tích cơ chế hoạt động của vòng lặp (Trigger Đệ Quy - Recursive Trigger):
  
Lỗi phát sinh do luồng dữ liệu bị thiết kế quay vòng tròn. Trình tự phá hoại của hệ thống diễn ra như sau:

Bước 1: Lệnh UPDATE [PhieuMuon] thủ công được thực thi. Bảng [PhieuMuon] ghi nhận có sự thay đổi dữ liệu.

Bước 2: Sự thay đổi này "đánh thức" Trigger thứ nhất (trg_PhieuMuon_CapNhat_DocGia). Nó chạy đoạn mã bên trong và tự động bắn ra một lệnh UPDATE [DocGia] (cộng 100đ tiền cọc).

Bước 3: Lúc này, Bảng [DocGia] ghi nhận có thay đổi dữ liệu. Nó lập tức "đánh thức" Trigger thứ hai (trg_DocGia_CapNhat_PhieuMuon).

Bước 4: Trigger thứ hai chạy và lại bắn ra tự động một lệnh UPDATE [PhieuMuon] (cộng thêm 1 ngày trả).

Bước 5: Bảng [PhieuMuon] lại bị thay đổi. Nó quay lại đánh thức Trigger thứ nhất (lặp lại Bước 2). Quá trình cứ thế tiếp diễn mãi mãi không có điểm dừng.

- Giải thích thông báo lỗi "limit 32": Rất may mắn, Microsoft SQL Server được trang bị một cơ chế tự vệ mặc định gọi là Nested Triggers Configuration. Engine của SQL Server nhận diện được vòng lặp vô hạn này và sẽ tự động "cắt cầu dao" ngắt toàn bộ tiến trình khi mức độ gọi lồng nhau vượt qua độ sâu tầng 32 (nesting level 32). Đồng thời, hệ thống ném ra lỗi Msg 217 và tự động ROLLBACK (hủy bỏ) toàn bộ chuỗi giao dịch để bảo vệ tài nguyên RAM và CPU của máy chủ khỏi bị treo cứng.

- Nhận xét kết luận:
1. Trong thực tế thiết kế cơ sở dữ liệu, việc để 2 bảng tự động cập nhật chéo nhau qua lại bằng Trigger bị coi là một Anti-pattern (Lỗi thiết kế tồi và nguy hiểm). Nó tiềm ẩn rủi ro sập hệ thống và làm suy giảm hiệu suất (Performance bottleneck) trầm trọng vì các transaction liên tục giữ khóa (lock) dữ liệu của nhau.
2. Cách khắc phục: * Rà soát lại Business Logic (Nghiệp vụ), đảm bảo luồng cập nhật dữ liệu tự động chỉ đi theo 1 chiều (ví dụ: A cập nhật B, B cập nhật C), tuyệt đối cấm thiết kế luồng quay ngược.

-Sử dụng hàm UPDATE(Tên_Cột) bên trong Trigger để giới hạn: Trigger chỉ được kích hoạt nếu đúng một cột cụ thể bị thay đổi, tránh việc trigger chạy lan tràn khi cập nhật các cột không liên quan.

-Dùng hàm TRIGGER_NESTLEVEL() ngay đầu Trigger. Nếu giá trị trả về lớn hơn 1 mức cho phép (VD: > 2), lập tức cho lệnh RETURN để chủ động ép dừng chuỗi phản ứng dây chuyền.

### Phần 5: Cursor và Duyệt dữ liệu ###
**1. Viết script sử dụng CURSOR giải quyết logic nghiệp vụ**

- Ý tưởng: "Chương trình Tri ân (Cashback) cho Độc giả thân thiết"

- Tình huống: Nhân dịp kỷ niệm thành lập thư viện, ban quản lý quyết định chạy một chương trình khuyến mãi: Hệ thống sẽ quét lịch sử mượn sách. Những độc giả nào đã mượn các cuốn sách có tổng giá trị cộng dồn từ 200.000 VNĐ trở lên sẽ được tặng thưởng 10% tổng giá trị đó, cộng trực tiếp vào [TienDatCoc].

- Luồng xử lý tổng quát bằng CURSOR:

Bước 1. Khai báo các biến cục bộ (@MaDocGia, @TongGiaTriSach, @TienThuong) để lưu dữ liệu khi duyệt.

Bước 2. Khai báo một Cursor lấy danh sách các Độc giả và Tổng giá trị sách họ đã mượn (Sử dụng SUM và GROUP BY kết hợp HAVING SUM >= 200000).

Bước 3. Mở Cursor (OPEN) để nạp tập dữ liệu này vào bộ nhớ.

Bước 4. Đọc dòng dữ liệu đầu tiên (FETCH NEXT).

Bước 5. Sử dụng vòng lặp WHILE duyệt từng độc giả: Tính tiền thưởng = 10% tổng giá trị sách, sau đó gọi lệnh UPDATE để cộng tiền vào bảng [DocGia]. In thông báo ra màn hình.

Bước 6. Đóng (CLOSE) và giải phóng bộ nhớ (DEALLOCATE) cho Cursor.

- Code SQL thực thi:
```sql
USE [QuanLyThuVien_K235480106023];
GO
SET STATISTICS TIME ON;
GO
PRINT N'--- BẮT ĐẦU CHẠY BẰNG CURSOR (CASHBACK) ---';
DECLARE @MaDocGia INT;
DECLARE @TongGiaTriSach MONEY;
DECLARE @TienThuong MONEY;
-- Lấy những người mượn sách tổng giá trị >= 200k
DECLARE cur_TriAn CURSOR FOR
    SELECT pm.MaDocGia, SUM(s.GiaBan) AS TongGiaTri
    FROM [PhieuMuon] pm
    INNER JOIN [Sach] s ON pm.MaSach = s.MaSach
    GROUP BY pm.MaDocGia
    HAVING SUM(s.GiaBan) >= 200000;
OPEN cur_TriAn;
FETCH NEXT FROM cur_TriAn INTO @MaDocGia, @TongGiaTriSach;
WHILE @@FETCH_STATUS = 0
BEGIN
    -- Tính 10% tiền thưởng
    SET @TienThuong = @TongGiaTriSach * 0.1;
    -- Cập nhật tiền cọc
    UPDATE [DocGia]
    SET TienDatCoc = TienDatCoc + @TienThuong
    WHERE MaDocGia = @MaDocGia;
    PRINT N'Đã thưởng ' + CAST(@TienThuong AS NVARCHAR) + N' VNĐ cho Độc giả mã số ' + CAST(@MaDocGia AS NVARCHAR);
    FETCH NEXT FROM cur_TriAn INTO @MaDocGia, @TongGiaTriSach;
END;
CLOSE cur_TriAn;
DEALLOCATE cur_TriAn;
PRINT N'--- HOÀN THÀNH CHẠY BẰNG CURSOR ---';
GO
```
<img width="1920" height="1080" alt="Screenshot (238)" src="https://github.com/user-attachments/assets/18b176ac-7330-4606-9a7d-156b0e1c1f91" />
<p align="center">Tab Messages hiển thị thông báo cộng thưởng cho độc giả </p>

**2. Giải quyết bài toán không dùng CURSOR và So sánh tốc độ**

Bài toán Tri ân cộng tiền ở trên hoàn toàn có thể được giải quyết bằng một câu lệnh UPDATE kết hợp với CTE (Common Table Expression - Bảng tạm dùng 1 lần) cực kỳ ngắn gọn mà không cần duyệt từng dòng.

- Code SQL KHÔNG dùng CURSOR
```sql
PRINT N'--- BẮT ĐẦU CHẠY BẰNG SET-BASED SQL ---';
WITH CTE_KhachVIP AS (
    SELECT pm.MaDocGia, SUM(s.GiaBan) * 0.1 AS TienThuong
    FROM [PhieuMuon] pm
    INNER JOIN [Sach] s ON pm.MaSach = s.MaSach
    GROUP BY pm.MaDocGia
    HAVING SUM(s.GiaBan) >= 200000
)
UPDATE d
SET d.TienDatCoc = d.TienDatCoc + c.TienThuong
FROM [DocGia] d
INNER JOIN CTE_KhachVIP c ON d.MaDocGia = c.MaDocGia;

PRINT N'--- HOÀN THÀNH CHẠY BẰNG SET-BASED SQL ---';
GO
```
<img width="1920" height="1080" alt="Screenshot (241)" src="https://github.com/user-attachments/assets/ba9b9df4-6eab-4b2a-b369-b453e87795f2" />
<p align="center">Thời gian thực thi khi sử dụng truy vấn SQL thuần (Set-based) - Chỉ tốn 1 lần quét bảng duy nhất</p>

**Phân tích và Nhận xét**

- Về mặt hiệu năng: Câu lệnh SQL thông thường (Set-based) chạy nhanh và tốn ít CPU hơn hẳn so với CURSOR.

- Giải thích nguyên nhân:  Khi dùng CURSOR, hệ thống làm việc theo cơ chế "RBAR" (Row By Agonizing Row - Duyệt khổ sở từng dòng). Nó gọi hàm UPDATE nhiều lần bằng với số lượng khách VIP (như ở Mục 1 là gọi 4 lần riêng biệt). Quá trình này bắt SQL Server liên tục mở/đóng Transaction và chuyển đổi ngữ cảnh, gây quá tải.

- Khi dùng Set-based (SQL thuần), Engine của SQL Server thực hiện gom nhóm (GROUP BY), liên kết (JOIN) trên RAM và tiến hành ghi đè (UPDATE) toàn bộ dữ liệu xuống ổ cứng trong đúng 1 lô duy nhất (Batch processing). Kết quả (4 rows affected) được báo cáo trong một lần chạy duy nhất. Hiệu suất được tối ưu tuyệt đối.

**3. Bắt buộc phải dùng CURSOR**
Dựa trên nguyên lý hoạt động của cơ sở dữ liệu, SQL thuần túy (Set-based) thao tác trên "Tập hợp" (cập nhật tất cả cùng lúc). Nó sẽ gần như bó tay trước các bài toán yêu cầu "Xử lý số dư luân phiên" (Running Balance / Sequential Allocation).

- Logic bài toán: "Thanh toán nợ luân phiên từ cũ đến mới"

- Tình huống nghiệp vụ: Độc giả Nguyễn Văn A có 3 phiếu mượn đều bị trễ hạn, lần lượt nợ: 40k (Phiếu 1), 50k (Phiếu 2), 30k (Phiếu 3). Tổng nợ là 120k.
Hôm nay, anh A đem 100.000 VNĐ đến thư viện nộp để khắc phục. Yêu cầu của thư viện là: Dùng số tiền 100k này để cấn trừ lần lượt cho các Phiếu từ cũ nhất đến mới nhất.

- Kịch bản mong đợi:
1. Trừ 40k cho Phiếu 1 -> Phiếu 1 hết nợ. Số dư còn: 60k.
2. Trừ 50k cho Phiếu 2 -> Phiếu 2 hết nợ. Số dư còn: 10k.
3. Trừ nốt 10k cho Phiếu 3 -> Phiếu 3 vẫn còn nợ 20k. Số dư hết (0đ). Tiến trình dừng lại.

- Tại sao SQL (Set-based) rất khó giải quyết?
Lệnh UPDATE thông thường tác động lên cả 3 phiếu cùng một lúc. Trong ngôn ngữ SQL chuẩn, không có cách nào tự nhiên để truyền một biến @SoDu = 100k vào dòng thứ nhất, làm phép trừ, rồi lấy "số dư còn lại" (carry-over) truyền tiếp xuống dòng thứ 2 trong cùng một câu lệnh UPDATE đơn lẻ. Sự phụ thuộc của dòng sau vào kết quả tính toán của dòng trước phá vỡ nguyên lý của tập hợp.

- Giải pháp hoàn hảo bằng CURSOR:
Bài toán này sinh ra là để dành cho CURSOR. Ta chỉ cần dùng biến @SoDu = 100000, mở một Cursor duyệt các phiếu quá hạn của Nguyễn Văn A theo thứ tự ORDER BY NgayMuon ASC (cũ lên trước).
Trong vòng lặp WHILE, nếu @SoDu > 0, ta lấy tiền nợ của phiếu hiện tại đem trừ đi, cập nhật trạng thái phiếu, cập nhật lại biến @SoDu và đẩy tiếp xuống dòng sau. Quá trình kiểm soát logic này diễn ra tuyến tính, chính xác và cực kỳ dễ hiểu.
