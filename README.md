# BÀI KIỂM TRA SỐ 2
### PHẦN MỞ ĐẦU
- Họ và tên: Nguyễn Minh Hạnh
- Mã sv: K235480106023
- Lớp: K59KMT
- YÊU CẦU CỦA ĐỀ: Quản lý thư viện

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

---

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

---

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

### PHẦN 2: XÂY DỰNG FUNCTION 

1. Các hàm có sẵn (Built-in Functions) trong SQL Server

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

2. Hàm do người dùng tự viết (User-Defined Functions - UDF)

- Mục đích: UDF được tạo ra để đóng gói (encapsulate) các đoạn logic tính toán phức tạp hoặc các câu truy vấn được sử dụng lặp đi lặp lại. Việc này giúp mã SQL gọn gàng, tăng tính tái sử dụng và dễ dàng bảo trì.

- Phân loại và thời điểm sử dụng:

-Scalar Function (Hàm vô hướng): 

  Đặc điểm: Nhận vào tham số và trả về đúng 1 giá trị đơn lẻ (INT, VARCHAR, MONEY,...).

  Khi nào dùng: Khi cần thực hiện một phép tính toán học, xử lý chuỗi hoặc tính toán dựa trên ngày tháng (VD: Tính tuổi từ ngày sinh, tính tiền phạt trễ hạn).

-Inline Table-Valued Function - iTVF (Hàm nội tuyến trả về bảng):

  Đặc điểm: Trả về 1 bảng (Table). Bên trong hàm chỉ chứa duy nhất một câu lệnh SELECT.

  Khi nào dùng: Dùng như một VIEW nhưng ưu việt hơn vì có thể truyền được tham số vào để lọc dữ liệu. Hiệu năng của iTVF rất tốt.

-Multi-statement Table-Valued Function - mTVF (Hàm đa câu lệnh trả về bảng):

  Đặc điểm: Trả về 1 bảng (Table). Bên trong hàm có thể chứa nhiều câu lệnh xử lý phức tạp (IF/ELSE, WHILE, biến bảng tạm DECLARE @Table TABLE).

  Khi nào dùng: Khi cần phải qua nhiều bước xử lý trung gian (thêm, sửa, xóa trên bảng tạm) mới ra được tập kết quả cuối cùng.

- Tại sao có nhiều hàm Built-in rồi mà vẫn cần tự viết function riêng?

Bởi vì các hàm có sẵn chỉ thực hiện các thao tác nền tảng (cộng trừ, đếm ngày, cắt chuỗi...). Chúng không thể hiểu được nghiệp vụ (Business Logic) của bài toán.

Ví dụ: Hàm DATEDIFF() có sẵn chỉ đếm được khoảng cách giữa 2 ngày. Nhưng hệ thống thư viện yêu cầu: "Trễ hạn bị phạt 5.000đ/ngày". Lúc này bắt buộc người lập trình phải kết hợp DATEDIFF với các phép toán nhân/chia để viết thành một UDF tên là fn_TinhTienPhat().

3. Thực hành viết Function cho Database Quản lý Thư Viện

-  Scalar Function (Hàm trả về một giá trị)

Yêu cầu: Hệ thống thư viện quy định nếu độc giả trả sách quá hạn sẽ bị phạt 5,000 VND / 1 ngày. Viết hàm truyền vào ngày trả dự kiến và ngày thực trả để tính ra số tiền phạt.

Luồng xử lý dữ :

Bước 1. Hàm nhận vào ngày trả dự kiến và ngày trả thực tế làm tham số đầu vào.

Bước 2. Hệ thống tính toán khoảng thời gian chênh lệch (số ngày trễ) giữa hai mốc thời gian này bằng hàm DATEDIFF.

Bước 3. Kiểm tra điều kiện logic: Nếu số ngày trễ lớn hơn 0 (tức là khách hàng trả sách quá hạn).

Bước 4. Thực hiện phép tính nhân số ngày trễ với mức phạt quy định của thư viện (5.000 VNĐ/ngày) để ra tổng tiền phạt.

Bước 5. Trả về kết quả là một giá trị duy nhất (kiểu MONEY) đại diện cho tổng số tiền phạt phải đóng.

-CODE TẠO HÀM

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
-CODE KHAI THÁC 

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

 - Inline Table-Valued Function

-Yêu cầu: Cần một hàm để lấy nhanh "Lịch sử mượn sách" của một Độc giả cụ thể. Tham số truyền vào là MaDocGia. Hàm cần nối (JOIN) bảng Phiếu Mượn và Sách để lấy tên sách.

Luồng xử lý dữ liệu:

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

-CODE TẠO HÀM 

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

-CODE KHAI THÁC 

```sql
-- Lấy lịch sử mượn sách của Độc giả có MaDocGia = 1
SELECT * FROM [dbo].[fn_LichSuMuonSachCuaDocGia](1);
GO
```

<img width="1920" height="1080" alt="Screenshot (208)" src="https://github.com/user-attachments/assets/567e6bc9-0f7b-4c1f-a772-0764a5c0bc84" />

<p align="center">Lịch sử mượn sách của Độc giả </p>

---

- C. Multi-statement Table-Valued Function

Yêu cầu: Cần xuất một "Báo cáo tổng hợp tình trạng Phiếu mượn". Báo cáo gồm: MaPhieu, MaDocGia, Tình trạng ("Trong hạn" hoặc "Đã quá hạn"), Số tiền phạt dự kiến. Do cần thiết lập trạng thái mặc định rồi mới kiểm tra và cập nhật trạng thái quá hạn, ta sử dụng biến bảng để xử lý nhiều bước.

Luồng xử lý dữ liệu:

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
-CODE TẠO HÀM

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

 -CODE KHAI THÁC HÀM

 ```sql
-- Hiển thị toàn bộ báo cáo tình trạng phiếu mượn
SELECT * FROM [dbo].[fn_BaoCaoTinhTrangCacPhieuMuon]();

-- Hoặc chỉ lọc ra những người đang bị phạt tiền
SELECT * FROM [dbo].[fn_BaoCaoTinhTrangCacPhieuMuon]() WHERE TienPhat > 0;
GO
```

<img width="1920" height="1080" alt="Screenshot (209)" src="https://github.com/user-attachments/assets/ac23e419-d7dd-4966-a285-de669e7bf358" />

### PHẦN 3: XÂY DỰNG STORE PROCEDURE

1,Tìm hiểu về System Stored Procedure
