# THIẾT KẾ VÀ CÀI ĐẶT CSDL QUẢN LÝ CẦM ĐỒ

Họ và tên: Từ Văn Hải

Lớp: K59KMT
Giảng viên: Đỗ Duy Cốp

## 1. Mô tả bài toán

Hệ thống cần quản lý các hợp đồng vay tiền thế chấp tài sản với cơ chế tính lãi linh hoạt:

- Trước Deadline1: tính lãi đơn (5.000đ/1.000.000đ/ngày)

- Sau Deadline1: tính lãi kép trên (gốc + lãi đơn tích lũy)

- Một khách hàng có nhiều hợp đồng, một hợp đồng có nhiều tài sản

- Hợp đồng có các trạng thái: Đang vay, Quá hạn, Đã thanh toán, Đã thanh lý


- Ghi nhận lịch sử trả nợ chi tiết, không được ghi đè

---

## 2. Quy ước đặt tên


Toàn bộ sử dụng **CamelCase tiếng Việt không dấu**:

- Bảng: KhachHang, HopDongCamDo, TaiSanTheChap, ChiTietTaiSanCamCo, GiaoDichTraNo

- Procedure: sp_DangKyHopDongMoi, sp_XuLyTraNo, sp_GiaHanHopDong

- Function: fn_CalcMoneyContract, fn_CalcMoneyTransaction

- Trigger: trg_CapNhatHopDongQuaHan, trg_CapNhatTaiSanSanSangThanhLy, trg_CapNhatTaiSanDaBanThanhLy

---

# PHẦN 0 - TẠO CƠ SỞ DỮ LIỆU VÀ BẢNG

## 0.1. Phân tích logic

Trước khi viết procedure, cần tạo database và các bảng:

1. KhachHang - lưu thông tin khách hàng

2. HopDongCamDo - lưu thông tin hợp đồng vay

3. TaiSanTheChap - lưu thông tin tài sản

4. ChiTietTaiSanCamCo - bảng liên kết nhiều-nhiều giữa hợp đồng và tài sản

5. GiaoDichTraNo - lưu từng lần trả nợ

6. LichSuGiaHan - lưu lịch sử gia hạn

7. NhatKyHeThong - bảng audit log

Thiết kế đảm bảo chuẩn hóa 3NF.

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/b989e4e6-a167-4fee-be3a-65422c6f407c" />

Sơ đồ ERD


---

## 0.2. Tạo Database

```sql
IF DB_ID('QuanLyCamDo') IS NOT NULL                                           -- Kiểm tra nếu database QuanLyCamDo đã tồn tại
BEGIN                                                                         -- Bắt đầu khối xử lý khi database tồn tại
    ALTER DATABASE QuanLyCamDo SET SINGLE_USER WITH ROLLBACK IMMEDIATE;       -- Chuyển database về chế độ một người dùng, hủy ngay các transaction đang chạy
    DROP DATABASE QuanLyCamDo;                                                -- Xóa database cũ để tạo mới
END;                                                                          -- Kết thúc khối IF
GO                                                                            -- Kết thúc batch

CREATE DATABASE QuanLyCamDo;                                                  -- Tạo database mới tên QuanLyCamDo
GO                                                                            -- Kết thúc batch

USE QuanLyCamDo;                                                              -- Chuyển sang sử dụng database QuanLyCamDo
GO                                                                            -- Kết thúc batch
Lệnh chạy thử

SELECT DB_NAME() AS TenCoSoDuLieuHienTai;                                     -- Hiển thị tên database hiện tại đang sử dụng
GO
```
                                                                         -- Kết thúc batch
Kết quả 

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/69fea4d1-6e53-4165-8c90-20fe9fac6f37" />

Tạo database thành công

0.3. Tạo bảng KhachHang


```sql
CREATE TABLE KhachHang                                                        -- Tạo bảng KhachHang để lưu thông tin khách hàng
(                                                                             -- Bắt đầu định nghĩa các cột
    KhachHangID INT IDENTITY(1,1) PRIMARY KEY,                               -- Khóa chính, tự động tăng 1
    HoTen NVARCHAR(100) NOT NULL,                                            -- Họ tên khách hàng, bắt buộc nhập
    SoDienThoai VARCHAR(15) NOT NULL,                                        -- Số điện thoại, bắt buộc
    SoCCCD VARCHAR(20) NULL,                                                 -- Số căn cước công dân, có thể để trống
    DiaChi NVARCHAR(255) NULL,                                               -- Địa chỉ, có thể để trống
    NgayTao DATETIME NOT NULL DEFAULT GETDATE(),                             -- Ngày tạo bản ghi, mặc định là thời gian hiện tại
    TrangThaiHoatDong BIT NOT NULL DEFAULT 1                                 -- Trạng thái hoạt động: 1 = active, 0 = inactive
);                                                                            -- Kết thúc bảng
-- Kết thúc batch
GO
```                                                                          
Lệnh chạy thử
```sql
SELECT * FROM KhachHang;                                                      -- Xem tất cả dữ liệu trong bảng KhachHang
GO
                                                                           -- Kết thúc batch
```
Kết quả 

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/34feb31f-5da7-4d47-b1f9-0ab6932fabde" />

Bảng KhachHang

0.4. Tạo bảng HopDongCamDo
```sql
CREATE TABLE HopDongCamDo                                                     -- Tạo bảng HopDongCamDo lưu thông tin hợp đồng
(                                                                             -- Bắt đầu các cột
    HopDongID INT IDENTITY(1,1) PRIMARY KEY,                                 -- Khóa chính tự tăng của hợp đồng
    KhachHangID INT NOT NULL,                                                -- Khóa ngoại tham chiếu đến KhachHang
    MaHopDong VARCHAR(30) NOT NULL UNIQUE,                                    -- Mã hợp đồng duy nhất, không trùng lặp
    NgayCam DATE NOT NULL,                                                   -- Ngày bắt đầu cầm đồ
    SoTienVayGoc DECIMAL(18,2) NOT NULL,                                     -- Số tiền vay gốc
    Deadline1 DATE NOT NULL,                                                 -- Deadline 1: trước ngày này tính lãi đơn
    Deadline2 DATE NOT NULL,                                                 -- Deadline 2: sau ngày này tính lãi kép và thanh lý
    LaiSuatDonNgay DECIMAL(18,6) NOT NULL DEFAULT 0.005,                     -- Lãi suất đơn mỗi ngày (0.5% = 0.005)
    TrangThaiHopDong NVARCHAR(50) NOT NULL DEFAULT N'DangVay',               -- Trạng thái hợp đồng, mặc định Đang vay
    TongTienDaTra DECIMAL(18,2) NOT NULL DEFAULT 0,                          -- Tổng tiền đã trả (cả gốc và lãi)
    GhiChu NVARCHAR(500) NULL,                                               -- Ghi chú bổ sung

    CONSTRAINT FK_HopDongCamDo_KhachHang                                     -- Tên ràng buộc khóa ngoại
        FOREIGN KEY (KhachHangID) REFERENCES KhachHang(KhachHangID),         -- Tham chiếu đến KhachHangID
    CONSTRAINT CK_HopDongCamDo_SoTienVayGoc                                  -- Tên ràng buộc kiểm tra số tiền vay
        CHECK (SoTienVayGoc > 0),                                            -- Số tiền vay phải > 0
    CONSTRAINT CK_HopDongCamDo_Deadline                                      -- Tên ràng buộc kiểm tra deadline
        CHECK (Deadline2 >= Deadline1)                                       -- Deadline2 phải >= Deadline1
);                                                                            -- Kết thúc bảng
GO                                                                            -- Kết thúc batch
```
Lệnh chạy thử
```sql
SELECT * FROM HopDongCamDo;                                                  -- Kiểm tra bảng HopDongCamDo
GO                                                                            -- Kết thúc batch
```

Kết quả mong đợi
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/46ecdd40-f850-4d7b-82df-9646ce6957a3" />

Bảng HopDongCamDo

0.5. Tạo bảng TaiSanTheChap

```sql
CREATE TABLE TaiSanTheChap                                                    -- Tạo bảng lưu thông tin tài sản thế chấp
(                                                                             -- Bắt đầu các cột
    TaiSanID INT IDENTITY(1,1) PRIMARY KEY,                                  -- Khóa chính tự tăng của tài sản
    TenTaiSan NVARCHAR(200) NOT NULL,                                        -- Tên tài sản
    LoaiTaiSan NVARCHAR(100) NOT NULL,                                       -- Loại tài sản (xe, điện thoại, vàng...)
    ThuongHieu NVARCHAR(100) NULL,                                           -- Thương hiệu
    MoTa NVARCHAR(500) NULL,                                                 -- Mô tả chi tiết
    SoSerial NVARCHAR(100) NULL,                                             -- Số serial/mã nhận diện
    TinhTrangTaiSan NVARCHAR(200) NULL,                                      -- Tình trạng hiện tại của tài sản
    NgayNhapThongTin DATETIME NOT NULL DEFAULT GETDATE()                     -- Ngày nhập thông tin tài sản
);                                                                            -- Kết thúc bảng
GO                                                                            -- Kết thúc batch
```
Lệnh chạy thử

SELECT * FROM TaiSanTheChap;                                                 -- Xem bảng TaiSanTheChap
GO                                                                            -- Kết thúc batch

Kết quả 


<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/536f3805-6164-44eb-b057-b496b5046e8b" />

Bảng TaiSanTheChap
0.6. Tạo bảng ChiTietTaiSanCamCo

```sql
CREATE TABLE ChiTietTaiSanCamCo                                               -- Bảng liên kết hợp đồng và tài sản
(                                                                             -- Bắt đầu các cột
    ChiTietTaiSanID INT IDENTITY(1,1) PRIMARY KEY,                           -- Khóa chính tự tăng
    HopDongID INT NOT NULL,                                                  -- Khóa ngoại tham chiếu HopDongCamDo
    TaiSanID INT NOT NULL,                                                   -- Khóa ngoại tham chiếu TaiSanTheChap
    GiaTriDinhGia DECIMAL(18,2) NOT NULL,                                    -- Giá trị định giá tại thời điểm cầm
    NgayNhanCam DATE NOT NULL DEFAULT GETDATE(),                             -- Ngày nhận cầm tài sản
    TrangThaiTaiSan NVARCHAR(50) NOT NULL DEFAULT N'DangCamCo',              -- Trạng thái tài sản trong hợp đồng
    IsSanSangThanhLy BIT NOT NULL DEFAULT 0,                                 -- Cờ đánh dấu sẵn sàng thanh lý
    IsDaBanThanhLy BIT NOT NULL DEFAULT 0,                                   -- Cờ đánh dấu đã bán thanh lý
    NgayTraTaiSan DATE NULL,                                                 -- Ngày trả lại tài sản cho khách
    NgayThanhLy DATE NULL,                                                   -- Ngày tài sản bị thanh lý

    CONSTRAINT FK_ChiTietTaiSanCamCo_HopDong                                 -- Ràng buộc khóa ngoại đến HopDongCamDo
        FOREIGN KEY (HopDongID) REFERENCES HopDongCamDo(HopDongID),
    CONSTRAINT FK_ChiTietTaiSanCamCo_TaiSan                                  -- Ràng buộc khóa ngoại đến TaiSanTheChap
        FOREIGN KEY (TaiSanID) REFERENCES TaiSanTheChap(TaiSanID),
    CONSTRAINT CK_ChiTietTaiSanCamCo_GiaTriDinhGia                           -- Ràng buộc kiểm tra giá trị định giá
        CHECK (GiaTriDinhGia > 0)                                            -- Giá trị định giá phải > 0
);                                                                            -- Kết thúc bảng
GO                                                                            -- Kết thúc batch
```

Lệnh chạy thử

SELECT * FROM ChiTietTaiSanCamCo;                                            -- Xem bảng chi tiết tài sản cầm cố
GO                                                                            -- Kết thúc batch

Kết quả 

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/2b6860a9-b282-4927-b78f-c60e58e28eac" />

Bảng ChiTietTaiSanCamCo

0.7. Tạo bảng GiaoDichTraNo

```sql
CREATE TABLE GiaoDichTraNo                                                   -- Bảng lưu lịch sử giao dịch trả nợ
(                                                                             -- Bắt đầu các cột
    GiaoDichID INT IDENTITY(1,1) PRIMARY KEY,                               -- Khóa chính tự tăng của giao dịch
    HopDongID INT NOT NULL,                                                  -- Khóa ngoại tham chiếu HopDongCamDo
    NgayGiaoDich DATE NOT NULL DEFAULT GETDATE(),                            -- Ngày diễn ra giao dịch
    SoTienTra DECIMAL(18,2) NOT NULL,                                        -- Tổng số tiền khách trả trong lần này
    TienTraGoc DECIMAL(18,2) NOT NULL DEFAULT 0,                             -- Phần trả gốc
    TienTraLai DECIMAL(18,2) NOT NULL DEFAULT 0,                             -- Phần trả lãi
    NguoiThuTien NVARCHAR(100) NOT NULL,                                     -- Người thu tiền
    NoiDungGiaoDich NVARCHAR(300) NULL,                                      -- Nội dung giao dịch
    SoTienNoConLaiSauGiaoDich DECIMAL(18,2) NULL,                           -- Số nợ còn lại sau khi trả

    CONSTRAINT FK_GiaoDichTraNo_HopDong                                      -- Khóa ngoại đến HopDongCamDo
        FOREIGN KEY (HopDongID) REFERENCES HopDongCamDo(HopDongID),
    CONSTRAINT CK_GiaoDichTraNo_SoTienTra                                    -- Kiểm tra số tiền tra > 0
        CHECK (SoTienTra > 0),
    CONSTRAINT CK_GiaoDichTraNo_TienTraGoc                                   -- Kiểm tra tiền trả gốc >= 0
        CHECK (TienTraGoc >= 0),
    CONSTRAINT CK_GiaoDichTraNo_TienTraLai                                   -- Kiểm tra tiền trả lãi >= 0
        CHECK (TienTraLai >= 0)
);                                                                            -- Kết thúc bảng
GO                                                                            -- Kết thúc batch
```

Lệnh chạy thử

SELECT * FROM GiaoDichTraNo;                                                 -- Kiểm tra bảng GiaoDichTraNo
GO                                                                            -- Kết thúc batch
Kết quả mong đợi

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/17470089-b7dc-4096-a386-a32da28fd8aa" />

Bảng GiaoDichTraNo
0.8. Tạo bảng LichSuGiaHan

```sql
CREATE TABLE LichSuGiaHan                                                   -- Bảng lưu lịch sử gia hạn hợp đồng
(                                                                             -- Bắt đầu các cột
    GiaHanID INT IDENTITY(1,1) PRIMARY KEY,                                 -- Khóa chính tự tăng
    HopDongID INT NOT NULL,                                                  -- Khóa ngoại tham chiếu HopDongCamDo
    NgayGiaHan DATE NOT NULL DEFAULT GETDATE(),                              -- Ngày diễn ra gia hạn
    Deadline1Cu DATE NOT NULL,                                               -- Deadline1 cũ trước khi gia hạn
    Deadline1Moi DATE NOT NULL,                                              -- Deadline1 mới sau khi gia hạn
    Deadline2Cu DATE NOT NULL,                                               -- Deadline2 cũ
    Deadline2Moi DATE NOT NULL,                                              -- Deadline2 mới
    SoTienLaiDaDong DECIMAL(18,2) NOT NULL,                                 -- Số tiền lãi đã đóng để gia hạn
    NguoiXuLy NVARCHAR(100) NOT NULL,                                        -- Người xử lý gia hạn
    GhiChu NVARCHAR(300) NULL,                                               -- Ghi chú

    CONSTRAINT FK_LichSuGiaHan_HopDong                                      -- Khóa ngoại đến HopDongCamDo
        FOREIGN KEY (HopDongID) REFERENCES HopDongCamDo(HopDongID)
);                                                                            -- Kết thúc bảng
GO                                                                            -- Kết thúc batch
```

Lệnh chạy thử

SELECT * FROM LichSuGiaHan;                                                  -- Kiểm tra bảng LichSuGiaHan
GO                                                                            -- Kết thúc batch
Kết quả mong đợi
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/4b1001dd-58ff-4499-8e92-4db669b9d50a" />

Bảng LichSuGiaHan

0.9. Tạo bảng NhatKyHeThong (Audit Log)
```sql
CREATE TABLE NhatKyHeThong                                                   -- Bảng audit log ghi nhận thay đổi hệ thống
(                                                                             -- Bắt đầu các cột
    NhatKyID INT IDENTITY(1,1) PRIMARY KEY,                                 -- Khóa chính tự tăng
    TenBang NVARCHAR(100) NOT NULL,                                          -- Tên bảng bị thay đổi
    KhoaChinhBanGhi NVARCHAR(100) NOT NULL,                                  -- Giá trị khóa chính của bản ghi
    HanhDong NVARCHAR(50) NOT NULL,                                          -- Hành động: INSERT, UPDATE, DELETE
    NoiDungThayDoi NVARCHAR(MAX) NULL,                                       -- Nội dung chi tiết thay đổi (có thể lưu JSON)
    ThoiDiemThayDoi DATETIME NOT NULL DEFAULT GETDATE(),                     -- Thời điểm thay đổi
    NguoiThucHien NVARCHAR(100) NULL                                         -- Người thực hiện thay đổi
);                                                                            -- Kết thúc bảng
GO                                                                            -- Kết thúc batch
SELECT * FROM NhatKyHeThong;                                                 -- Kiểm tra bảng NhatKyHeThong
GO                                                                            -- Kết thúc batch
```
Kết quả mong đợi

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/073a1d99-e1d6-4a48-ad8a-be3377e85985" />

Bảng NhatKyHeThong

PHẦN 1 - EVENT 1: ĐĂNG KÝ HỢP ĐỒNG MỚI

1.1. Phân tích logic

Yêu cầu: Viết Store Procedure tiếp nhận hợp đồng mới với các bước:

Lưu thông tin khách hàng vào bảng KhachHang

Tạo hợp đồng mới trong HopDongCamDo với mã hợp đồng tự sinh

Nhận danh sách tài sản (nhiều tài sản) qua tham số Table Type

Lưu từng tài sản vào TaiSanTheChap

Tạo liên kết trong ChiTietTaiSanCamCo

Ghi log vào NhatKyHeThong

Khó khăn: SQL Server Procedure không nhận mảng trực tiếp → cần dùng Table Type.

Giải pháp:

Tạo kiểu bảng DanhSachTaiSanType

Procedure nhận tham số READONLY kiểu này

Dùng vòng lặp hoặc INSERT...SELECT để xử lý

1.2. Tạo Table Type để truyền danh sách tài sản

```sql
CREATE TYPE DanhSachTaiSanType AS TABLE                                       -- Tạo kiểu bảng để truyền danh sách tài sản
(                                                                             -- Bắt đầu các cột của kiểu
    TenTaiSan NVARCHAR(200),                                                 -- Tên tài sản
    LoaiTaiSan NVARCHAR(100),                                                -- Loại tài sản
    ThuongHieu NVARCHAR(100),                                                -- Thương hiệu
    MoTa NVARCHAR(500),                                                      -- Mô tả
    SoSerial NVARCHAR(100),                                                  -- Số serial
    TinhTrangTaiSan NVARCHAR(200),                                           -- Tình trạng tài sản
    GiaTriDinhGia DECIMAL(18,2)                                              -- Giá trị định giá
);                                                                            -- Kết thúc kiểu bảng
GO                                                                            -- Kết thúc batch

SELECT * FROM sys.types WHERE name = 'DanhSachTaiSanType';                  -- Kiểm tra kiểu đã được tạo
GO                                                                            -- Kết thúc batch
```
Kết quả 


<img width="1917" height="1077" alt="image" src="https://github.com/user-attachments/assets/cec90ac5-a144-4527-b8ba-876c13bfd3c5" />

Tạo Table Type
1.3. Viết Store Procedure sp_DangKyHopDongMoi

```sql
CREATE OR ALTER PROCEDURE sp_DangKyHopDongMoi                                -- Tạo hoặc cập nhật procedure đăng ký hợp đồng
(                                                                             -- Tham số đầu vào
    @HoTen NVARCHAR(100),                                                    -- Họ tên khách hàng
    @SoDienThoai VARCHAR(15),                                                -- Số điện thoại
    @SoCCCD VARCHAR(20),                                                     -- Số CCCD
    @DiaChi NVARCHAR(255),                                                   -- Địa chỉ
    @SoTienVayGoc DECIMAL(18,2),                                             -- Số tiền vay gốc
    @NgayCam DATE,                                                           -- Ngày cầm đồ
    @Deadline1 DATE,                                                         -- Deadline 1
    @Deadline2 DATE,                                                         -- Deadline 2
    @GhiChu NVARCHAR(500),                                                   -- Ghi chú
    @DanhSachTaiSan DanhSachTaiSanType READONLY                              -- Danh sách tài sản (kiểu table, chỉ đọc)
)
AS
BEGIN
    SET NOCOUNT ON;                                                          -- Tắt thông báo số dòng bị ảnh hưởng
    BEGIN TRY                                                                -- Bắt đầu khối xử lý lỗi
        BEGIN TRANSACTION;                                                   -- Bắt đầu transaction đảm bảo tính atomic

        DECLARE @KhachHangID INT;                                            -- Biến lưu ID khách hàng sau khi thêm
        DECLARE @HopDongID INT;                                              -- Biến lưu ID hợp đồng sau khi thêm
        DECLARE @MaHopDong VARCHAR(30);                                      -- Biến lưu mã hợp đồng

        -- Kiểm tra dữ liệu đầu vào
        IF @SoTienVayGoc <= 0                                                -- Nếu số tiền vay <= 0
        BEGIN                                                                -- Bắt đầu khối lệnh
            RAISERROR(N'SoTienVayGoc phai lon hon 0', 16, 1);                -- Ném lỗi với thông báo tiếng Việt
            ROLLBACK TRANSACTION;                                            -- Rollback transaction
            RETURN;                                                           -- Thoát procedure
        END                                                                  -- Kết thúc khối IF

        IF @Deadline2 < @Deadline1                                           -- Nếu deadline2 < deadline1
        BEGIN                                                                -- Bắt đầu khối lệnh
            RAISERROR(N'Deadline2 phai lon hon hoac bang Deadline1', 16, 1); -- Ném lỗi
            ROLLBACK TRANSACTION;                                            -- Rollback
            RETURN;                                                           -- Thoát
        END                                                                  -- Kết thúc khối IF

        -- Thêm khách hàng mới
        INSERT INTO KhachHang(HoTen, SoDienThoai, SoCCCD, DiaChi)            -- Thêm bản ghi vào KhachHang
        VALUES(@HoTen, @SoDienThoai, @SoCCCD, @DiaChi);                     -- Giá trị từ tham số

        SET @KhachHangID = SCOPE_IDENTITY();                                -- Lấy ID vừa sinh của khách hàng

        -- Sinh mã hợp đồng tự động
        SET @MaHopDong = 'HD' + FORMAT(GETDATE(), 'yyyyMMddHHmmss');         -- Tạo mã HD + timestamp

        -- Thêm hợp đồng mới
        INSERT INTO HopDongCamDo                                             -- Thêm vào bảng HopDongCamDo
        (
            KhachHangID, MaHopDong, NgayCam, SoTienVayGoc,
            Deadline1, Deadline2, LaiSuatDonNgay, TrangThaiHopDong, GhiChu
        )
        VALUES
        (
            @KhachHangID, @MaHopDong, @NgayCam, @SoTienVayGoc,
            @Deadline1, @Deadline2, 0.005, N'DangVay', @GhiChu               -- LaiSuatDonNgay = 0.005 = 0.5%/ngày
        );

        SET @HopDongID = SCOPE_IDENTITY();                                  -- Lấy ID hợp đồng vừa tạo

        -- Thêm tài sản và chi tiết cầm cố
        -- Dùng INSERT...SELECT trực tiếp thay vì vòng lặp
        INSERT INTO TaiSanTheChap                                            -- Thêm tài sản vào bảng TaiSanTheChap
        (
            TenTaiSan, LoaiTaiSan, ThuongHieu, MoTa, SoSerial, TinhTrangTaiSan
        )
        SELECT                                                              -- Chọn từ danh sách truyền vào
            TenTaiSan, LoaiTaiSan, ThuongHieu, MoTa, SoSerial, TinhTrangTaiSan
        FROM @DanhSachTaiSan;                                               -- Từ table type

        -- Lấy ID các tài sản vừa thêm (có thể dùng OUTPUT clause)
        DECLARE @TaiSanIDs TABLE (TaiSanID INT, RowNum INT);                -- Bảng tạm lưu ID tài sản

        INSERT INTO @TaiSanIDs
        SELECT TaiSanID, ROW_NUMBER() OVER (ORDER BY TaiSanID)              -- Lấy ID với số thứ tự
        FROM TaiSanTheChap
        WHERE TaiSanID >= (SELECT MIN(TaiSanID) FROM TaiSanTheChap)        -- ID mới nhất
          AND TaiSanID < (SELECT MIN(TaiSanID) FROM TaiSanTheChap) + (SELECT COUNT(*) FROM @DanhSachTaiSan);

        INSERT INTO ChiTietTaiSanCamCo                                       -- Thêm vào bảng chi tiết
        (
            HopDongID, TaiSanID, GiaTriDinhGia, NgayNhanCam, TrangThaiTaiSan
        )
        SELECT
            @HopDongID,                                                      -- HopDongID hiện tại
            t.TaiSanID,                                                      -- TaiSanID từ bảng tạm
            d.GiaTriDinhGia,                                                 -- Giá trị định giá từ danh sách
            @NgayCam,                                                        -- Ngày nhận cam
            N'DangCamCo'                                                     -- Trạng thái ban đầu
        FROM @TaiSanIDs t
        INNER JOIN @DanhSachTaiSan d ON t.RowNum = (                              -- Kết nối theo thứ tự
            SELECT ROW_NUMBER() OVER (ORDER BY (SELECT NULL))
            FROM @DanhSachTaiSan d2
            WHERE d2.TenTaiSan = (SELECT TenTaiSan FROM TaiSanTheChap WHERE TaiSanID = t.TaiSanID)
        );

        -- Ghi log hệ thống
        INSERT INTO NhatKyHeThong
        (
            TenBang, KhoaChinhBanGhi, HanhDong, NoiDungThayDoi, NguoiThucHien
        )
        VALUES
        (
            N'HopDongCamDo',                                                -- Tên bảng bị thay đổi
            CAST(@HopDongID AS NVARCHAR(100)),                              -- Khóa chính
            N'INSERT',                                                      -- Hành động thêm mới
            N'Tạo hợp đồng mới với danh sách tài sản thế chấp',             -- Nội dung
            N'HệThống'                                                      -- Người thực hiện
        );

        COMMIT TRANSACTION;                                                  -- Xác nhận transaction

        SELECT                                                              -- Trả kết quả cho người gọi
            @KhachHangID AS KhachHangID,
            @HopDongID AS HopDongID,
            @MaHopDong AS MaHopDong,
            N'Đăng ký hợp đồng thành công' AS ThongBao;
    END TRY                                                                -- Kết thúc khối thử
    BEGIN CATCH                                                            -- Bắt đầu khối bắt lỗi
        ROLLBACK TRANSACTION;                                              -- Rollback nếu có lỗi
        THROW;                                                             -- Ném lỗi tiếp tục
    END CATCH                                                              -- Kết thúc khối catch
END;
GO                                                                            -- Kết thúc batch

```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/43c03644-797a-4035-abea-720f13b737d5" />

Lệnh chạy thử
```sql
DECLARE @DanhSachTaiSan DanhSachTaiSanType;                                 -- Khai báo biến kiểu table

INSERT INTO @DanhSachTaiSan                                                 -- Thêm dữ liệu mẫu vào danh sách
(TenTaiSan, LoaiTaiSan, ThuongHieu, MoTa, SoSerial, TinhTrangTaiSan, GiaTriDinhGia)
VALUES
(N'iPhone 13 Pro Max', N'Điện thoại', N'Apple', N'Máy 256GB, màu xanh', N'IP13PM001', N'Đã qua sử dụng tốt', 18000000),
(N'Laptop XPS 13', N'Laptop', N'Dell', N'Core i7, RAM 16GB', N'DELLXPS013', N'Ngoại hình đẹp', 22000000);

EXEC sp_DangKyHopDongMoi                                                   -- Gọi procedure đăng ký hợp đồng
    @HoTen = N'Nguyễn Văn A',
    @SoDienThoai = '0901234567',
    @SoCCCD = '079999999999',
    @DiaChi = N'Hà Nội',
    @SoTienVayGoc = 20000000,
    @NgayCam = '2026-05-01',
    @Deadline1 = '2026-05-10',
    @Deadline2 = '2026-05-20',
    @GhiChu = N'Hợp đồng vay lần 1',
    @DanhSachTaiSan = @DanhSachTaiSan;
GO                                                                            -- Kết thúc batch

```
Kết quả 

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/83fcd788-0acf-4c5a-92db-fca6ec7d801a" />

 Đăng ký hợp đồng thành công
 
Kiểm tra dữ liệu đã thêm

SELECT * FROM KhachHang;                                                    -- Kiểm tra bảng KhachHang
SELECT * FROM HopDongCamDo;                                                 -- Kiểm tra bảng HopDongCamDo
SELECT * FROM TaiSanTheChap;                                                -- Kiểm tra bảng TaiSanTheChap
SELECT * FROM ChiTietTaiSanCamCo;                                           -- Kiểm tra bảng ChiTietTaiSanCamCo
SELECT * FROM NhatKyHeThong;                                                -- Kiểm tra bảng audit log
GO                                                                            -- Kết thúc batch
Kết quả kiểm tra

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/06f57f90-0d9d-4031-9c6b-10cdba4ffe91" />

 Kiểm tra KhachHang, HopDongCamDo, TaiSan, ChiTietTaiSan
 
PHẦN 2 - EVENT 2: TÍNH TOÁN CÔNG NỢ THỜI GIAN THỰC

2.1. Phân tích logic

Yêu cầu:

fn_CalcMoneyTransaction(TransactionID, TargetDate) - tính nợ theo giao dịch

`fn_CalcMoneyContract(Contract

Tiếp tục viết phần Event 2 từ đầu:

2.1. Phân tích logic
Yêu cầu:

fn_CalcMoneyTransaction(TransactionID, TargetDate) - tính nợ theo giao dịch

fn_CalcMoneyContract(ContractID, TargetDate) - tính tổng nợ hợp đồng đến ngày TargetDate
Quy tắc tính lãi:

Trước Deadline1: lãi đơn = gốc × 0.005 × số ngày

Sau Deadline1: lãi kép = (gốc + lãi đơn) × (1.005)^(số ngày sau Deadline1) - (gốc + lãi đơn)

Cần tính cả số tiền đã trả để được gốc còn lại.

2.2. Function tính nợ theo hợp đồng

```sql
CREATE OR ALTER FUNCTION fn_CalcMoneyContract                                 -- Tạo function tính tổng nợ theo hợp đồng
(                                                                              -- Tham số
    @HopDongID INT,                                                           -- ID hợp đồng cần tính
    @TargetDate DATE                                                          -- Ngày tính đến
)
RETURNS DECIMAL(18,2)                                                        -- Trả về số tiền tổng nợ
AS
BEGIN
    -- Khai báo biến
    DECLARE @SoTienVayGoc DECIMAL(18,2);                                     -- Số tiền vay gốc ban đầu
    DECLARE @NgayCam DATE;                                                   -- Ngày cầm đồ
    DECLARE @Deadline1 DATE;                                                 -- Deadline 1
    DECLARE @LaiSuatDonNgay DECIMAL(18,6);                                   -- Lãi suất đơn ngày
    DECLARE @TongDaTra DECIMAL(18,2);                                        -- Tổng tiền đã trả
    DECLARE @GocConLai DECIMAL(18,2);                                        -- Gốc còn lại
    DECLARE @SoNgayLaiDon INT;                                               -- Số ngày tính lãi đơn
    DECLARE @LaiDon DECIMAL(18,2);                                           -- Lãi đơn tích lũy
    DECLARE @SoNgayLaiKep INT;                                               -- Số ngày tính lãi kép
    DECLARE @VonCoSo DECIMAL(18,2);                                          -- Vốn gốc + lãi đơn
    DECLARE @TongSauLaiKep DECIMAL(18,2);                                    -- Tổng sau lãi kép
    DECLARE @LaiKep DECIMAL(18,2);                                           -- Lãi kép
    DECLARE @TongNo DECIMAL(18,2);                                           -- Tổng nợ cuối cùng

    -- Lấy thông tin hợp đồng
    SELECT
        @SoTienVayGoc = SoTienVayGoc,
        @NgayCam = NgayCam,
        @Deadline1 = Deadline1,
        @LaiSuatDonNgay = LaiSuatDonNgay
    FROM HopDongCamDo
    WHERE HopDongID = @HopDongID;

    -- Nếu không tìm thấy hợp đồng thì trả về 0
    IF @SoTienVayGoc IS NULL
        RETURN 0;

    -- Tính tổng tiền đã trả đến ngày TargetDate
    SELECT @TongDaTra = ISNULL(SUM(SoTienTra), 0)
    FROM GiaoDichTraNo
    WHERE HopDongID = @HopDongID
      AND NgayGiaoDich <= @TargetDate;

    -- Tính gốc còn lại (không âm)
    SET @GocConLai = @SoTienVayGoc - @TongDaTra;
    IF @GocConLai < 0 SET @GocConLai = 0;

    -- Nếu TargetDate <= Deadline1: chỉ tính lãi đơn
    IF @TargetDate <= @Deadline1
    BEGIN
        SET @SoNgayLaiDon = DATEDIFF(DAY, @NgayCam, @TargetDate);           -- Số ngày từ ngày cầm đến TargetDate
        IF @SoNgayLaiDon < 0 SET @SoNgayLaiDon = 0;                        -- Nếu âm thì = 0

        SET @LaiDon = @GocConLai * @LaiSuatDonNgay * @SoNgayLaiDon;         -- Tính lãi đơn
        SET @TongNo = @GocConLai + @LaiDon;                                -- Tổng nợ = gốc + lãi đơn

        RETURN @TongNo;                                                     -- Trả kết quả
    END

    -- Trường hợp TargetDate > Deadline1: tính cả lãi đơn và lãi kép

    -- Tính lãi đơn từ ngày cầm đến Deadline1
    SET @SoNgayLaiDon = DATEDIFF(DAY, @NgayCam, @Deadline1);
    IF @SoNgayLaiDon < 0 SET @SoNgayLaiDon = 0;

    SET @LaiDon = @GocConLai * @LaiSuatDonNgay * @SoNgayLaiDon;

    -- Tính lãi kép từ Deadline1 đến TargetDate
    SET @SoNgayLaiKep = DATEDIFF(DAY, @Deadline1, @TargetDate);
    IF @SoNgayLaiKep < 0 SET @SoNgayLaiKep = 0;

    SET @VonCoSo = @GocConLai + @LaiDon;                                   -- Vốn gốc + lãi đơn tích lũy
    SET @TongSauLaiKep = @VonCoSo * POWER(1 + @LaiSuatDonNgay, @SoNgayLaiKep); -- Áp dụng công thức lãi kép
    SET @LaiKep = @TongSauLaiKep - @VonCoSo;                              -- Lãi kép = tổng sau - vốn ban đầu

    SET @TongNo = @GocConLai + @LaiDon + @LaiKep;                         -- Tổng nợ cuối

    RETURN @TongNo;                                                        -- Trả kết quả
END;
GO                                                                         -- Kết thúc batch
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/ec5d2bae-a7ad-48b4-91f7-6a4fe9b1f9d9" />

Lệnh chạy thử
```sql
-- Tạo giao dịch mẫu để test
INSERT INTO GiaoDichTraNo
(
    HopDongID, NgayGiaoDich, SoTienTra, TienTraGoc, TienTraLai,
    NguoiThuTien, NoiDungGiaoDich, SoTienNoConLaiSauGiaoDich
)
VALUES
(1, '2026-05-05', 3000000, 3000000, 0, N'Thu ngân 1', N'Trả một phần', NULL);

-- Test function trước Deadline1 (2026-05-10)
SELECT dbo.fn_CalcMoneyContract(1, '2026-05-08') AS TongNoTruocDeadline1;

-- Test function sau Deadline1 (2026-05-15)
SELECT dbo.fn_CalcMoneyContract(1, '2026-05-15') AS TongNoSauDeadline1;
GO                                                                         -- Kết thúc batch
```
Kết quả mong đợi
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/098b3f0f-4525-4689-824b-b482f263b754" />
Tính nợ hợp đồng

2.3. Function tính nợ theo giao dịch

```sql
CREATE OR ALTER FUNCTION fn_CalcMoneyTransaction                             -- Tạo function tính nợ theo giao dịch
(                                                                              -- Tham số
    @GiaoDichID INT,                                                          -- ID giao dịch cần tra cứu
    @TargetDate DATE                                                          -- Ngày tính đến
)
RETURNS DECIMAL(18,2)                                                        -- Trả về tổng nợ của hợp đồng đó
AS
BEGIN
    DECLARE @HopDongID INT;                                                   -- Biến lưu ID hợp đồng
    DECLARE @TongNo DECIMAL(18,2);                                            -- Biến lưu kết quả

    -- Lấy HopDongID từ GiaoDichID
    SELECT @HopDongID = HopDongID
    FROM GiaoDichTraNo
    WHERE GiaoDichID = @GiaoDichID;

    -- Nếu không tìm thấy giao dịch thì trả về 0
    IF @HopDongID IS NULL
        RETURN 0;

    -- Gọi function fn_CalcMoneyContract để tính tổng nợ
    SET @TongNo = dbo.fn_CalcMoneyContract(@HopDongID, @TargetDate);

    RETURN @TongNo;                                                           -- Trả kết quả
END;
GO                                                                             -- Kết thúc batch
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/099f65af-3f92-465d-8184-67285d0fb478" />

Lệnh chạy thử
```sql
-- Test function với giao dịch có ID = 1
SELECT dbo.fn_CalcMoneyTransaction(1, '2026-05-15') AS TongNoTheoGiaoDich;
GO                                                                             -- Kết thúc batch
```

Kết quả mong đợi
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/ec3cf60e-f624-47a3-b510-3a2d20db07b3" />

Tính nợ giao dịch

PHẦN 3 - EVENT 3: XỬ LÝ TRẢ NỢ VÀ HOÀN TRẢ TÀI SẢN
3.1. Phân tích logic

Yêu cầu: Procedure xử lý khi khách đến trả tiền:


Kiểm tra nếu có tài sản đã bán thanh lý (IsDaBanThanhLy = 1) → báo lỗi, không thu tiền

Tính tổng nợ hiện tại đến ngày trả

Trừ số tiền khách trả

Nếu trả hết (dư nợ = 0):

Cập nhật trạng thái hợp đồng = Đã thanh toán đủ

Cập nhật tất cả tài sản thành Đã trả khách

Nếu chưa hết:

Cập nhật trạng thái hợp đồng = Đang trả góp

Ghi giao dịch vào GiaoDichTraNo

Đề xuất danh sách tài sản có thể trả (nếu giá trị còn lại >= dư nợ)

3.2. Store Procedure xử lý trả nợ

```sql
CREATE OR ALTER PROCEDURE sp_XuLyTraNo                                     -- Tạo procedure xử lý trả nợ
(                                                                           -- Tham số đầu vào
    @HopDongID INT,                                                        -- ID hợp đồng cần trả nợ
    @SoTienKhachTra DECIMAL(18,2),                                         -- Số tiền khách trả
    @NgayTra DATE,                                                         -- Ngày trả
    @NguoiThuTien NVARCHAR(100)                                            -- Người thu tiền
)
AS
BEGIN
    SET NOCOUNT ON;                                                        -- Tắt thông báo số dòng
    BEGIN TRY                                                              -- Bắt đầu xử lý với try-catch
        BEGIN TRANSACTION;                                                 -- Bắt đầu transaction đảm bảo atomic

        -- Khai báo biến
        DECLARE @TongNoHienTai DECIMAL(18,2);                             -- Tổng nợ hiện tại
        DECLARE @DuNoConLai DECIMAL(18,2);                                -- Dư nợ còn lại sau khi trả
        DECLARE @TongTienDaTra DECIMAL(18,2);                             -- Tổng tiền đã trả trước đó
        DECLARE @SoTienVayGoc DECIMAL(18,2);                              -- Số tiền vay gốc

        -- Kiểm tra số tiền hợp lệ
        IF @SoTienKhachTra <= 0                                            -- Nếu số tiền <= 0
        BEGIN                                                              -- Bắt đầu khối lỗi
            RAISERROR(N'Số tiền khách trả phải lớn hơn 0', 16, 1);         -- Ném lỗi
            ROLLBACK TRANSACTION;                                          -- Rollback
            RETURN;                                                         -- Thoát
        END                                                                -- Kết thúc kiểm tra

        -- Kiểm tra tài sản đã thanh lý chưa
        IF EXISTS                                                          -- Nếu tồn tại
        (
            SELECT 1
            FROM ChiTietTaiSanCamCo
            WHERE HopDongID = @HopDongID
              AND IsDaBanThanhLy = 1                                       -- Tài sản đã bán thanh lý
        )
        BEGIN                                                              -- Bắt đầu khối lỗi
            RAISERROR(N'Tài sản đã bị thanh lý, không thu tiền và không trả đồ', 16, 1);
            ROLLBACK TRANSACTION;
            RETURN;
        END

        -- Tính tổng nợ hiện tại đến ngày trả
        SET @TongNoHienTai = dbo.fn_CalcMoneyContract(@HopDongID, @NgayTra);

        -- Tính dư nợ còn lại
        SET @DuNoConLai = @TongNoHienTai - @SoTienKhachTra;
        IF @DuNoConLai < 0 SET @DuNoConLai = 0;

        -- Lấy số tiền vay gốc để phân bổ trả gốc/lãi
        SELECT @SoTienVayGoc = SoTienVayGoc
        FROM HopDongCamDo
        WHERE HopDongID = @HopDongID;

        -- Thêm giao dịch trả nợ
        INSERT INTO GiaoDichTraNo
        (
            HopDongID, NgayGiaoDich, SoTienTra, TienTraGoc, TienTraLai,
            NguoiThuTien, NoiDungGiaoDich, SoTienNoConLaiSauGiaoDich
        )
        VALUES
        (
            @HopDongID,
            @NgayTra,
            @SoTienKhachTra,
            CASE
                WHEN @SoTienKhachTra <= @SoTienVayGoc THEN @SoTienKhachTra     -- Trả gốc hết số tiền
                ELSE @SoTienVayGoc                                            -- Chỉ trả phần gốc còn lại
            END,
            CASE
                WHEN @SoTienKhachTra > @SoTienVayGoc THEN @SoTienKhachTra - @SoTienVayGoc  -- Trả lãi phần chênh lệch
                ELSE 0                                                         -- Không trả lãi
            END,
            @NguoiThuTien,
            N'Thanh toán bởi khách hàng',
            @DuNoConLai
        );

        -- Cập nhật tổng tiền đã trả trên hợp đồng
        UPDATE HopDongCamDo
        SET TongTienDaTra = ISNULL(TongTienDaTra, 0) + @SoTienKhachTra
        WHERE HopDongID = @HopDongID;

        -- Kiểm tra nếu đã trả hết nợ
        IF @DuNoConLai = 0
        BEGIN                                                              -- Trả hết nợ
            -- Cập nhật trạng thái hợp đồng
            UPDATE HopDongCamDo
            SET TrangThaiHopDong = N'DaThanhToan'
            WHERE HopDongID = @HopDongID;

            -- Trả lại tất cả tài sản
            UPDATE ChiTietTaiSanCamCo
            SET TrangThaiTaiSan = N'DaTraKhach',
                NgayTraTaiSan = @NgayTra
            WHERE HopDongID = @HopDongID
              AND IsDaBanThanhLy = 0;

            -- Ghi log hệ thống
            INSERT INTO NhatKyHeThong
            (
                TenBang, KhoaChinhBanGhi, HanhDong, NoiDungThayDoi, NguoiThucHien
            )
            VALUES
            (
                N'HopDongCamDo',
                CAST(@HopDongID AS NVARCHAR(100)),
                N'UPDATE',
                N'Khách đã thanh toán đủ nợ, trả hết tài sản',
                @NguoiThuTien
            );

            -- Trả kết quả thành công
            SELECT
                N'Khách đã thanh toán đủ nợ, trả hết tài sản' AS ThongBao,
                @TongNoHienTai AS TongNoHienTai,
                @SoTienKhachTra AS SoTienKhachTra,
                @DuNoConLai AS DuNoConLai;
        END
        ELSE                                                                -- Chưa trả hết
        BEGIN                                                              -- Xử lý trả một phần
            -- Cập nhật trạng thái hợp đồng
            UPDATE HopDongCamDo
            SET TrangThaiHopDong = N'DangTraGop'
            WHERE HopDongID = @HopDongID;

            -- Ghi log
            INSERT INTO NhatKyHeThong
            (
                TenBang, KhoaChinhBanGhi, HanhDong, NoiDungThayDoi, NguoiThucHien
            )
            VALUES
            (
                N'GiaoDichTraNo',
                CAST(@HopDongID AS NVARCHAR(100)),
                N'INSERT',
                N'Khách thanh toán một phần, còn dư nợ',
                @NguoiThuTien
            );

            -- Trả kết quả và đề xuất tài sản có thể trả
            SELECT
                N'Khách thanh toán một phần, dưới đây là tài sản có thể gợi ý trả' AS ThongBao,
                @TongNoHienTai AS TongNoHienTai,
                @SoTienKhachTra AS SoTienKhachTra,
                @DuNoConLai AS DuNoConLai;

            -- Tính và đề xuất tài sản có thể trả
            -- Điều kiện: Giá trị tài sản còn lại >= Dư nợ còn lại
            ;WITH TaiSanDangGiu AS                                           -- CTE lấy tài sản đang giữ
            (
                SELECT
                    c.ChiTietTaiSanID,
                    t.TenTaiSan,
                    c.GiaTriDinhGia
                FROM ChiTietTaiSanCamCo c
                INNER JOIN TaiSanTheChap t ON c.TaiSanID = t.TaiSanID
                WHERE c.HopDongID = @HopDongID
                  AND c.TrangThaiTaiSan = N'DangCamCo'
                  AND c.IsDaBanThanhLy = 0
            ),
            TongGiaTri AS                                                   -- CTE tính tổng giá trị tài sản đang giữ
            (
                SELECT SUM(GiaTriDinhGia) AS TongGiaTri
                FROM TaiSanDangGiu
            )
            SELECT
                ts.ChiTietTaiSanID,
                ts.TenTaiSan,
                ts.GiaTriDinhGia,
                (tg.TongGiaTri - ts.GiaTriDinhGia) AS GiaTriTaiSanConLaiNeuTraTaiSanNay,
                @DuNoConLai AS DuNoConLai
            FROM TaiSanDangGiu ts
            CROSS JOIN TongGiaTri tg
            WHERE (tg.TongGiaTri - ts.GiaTriDinhGia) >= @DuNoConLai;        -- Chỉ đề xuất nếu còn đủ bảo đảm
        END

        COMMIT TRANSACTION;                                                 -- Xác nhận transaction
    END TRY                                                                -- Kết thúc try
    BEGIN CATCH                                                            -- Bắt lỗi
        ROLLBACK TRANSACTION;                                              -- Rollback
        THROW;                                                             -- Ném lỗi tiếp
    END CATCH                                                              -- Kết thúc catch
END;
GO                                                                         -- Kết thúc batch
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/3fb80a01-6f5f-47bf-b42f-ff22a8862e48" />


Lệnh chạy thử
```sql
-- Test procedure trả nợ với hợp đồng ID = 1, trả 5 triệu
EXEC sp_XuLyTraNo
    @HopDongID = 1,
    @SoTienKhachTra = 5000000,
    @NgayTra = '2026-05-16',
    @NguoiThuTien = N'Thu ngân 2';
GO                                                                         -- Kết thúc batch
```
Kết quả mong đợi

<img width="1913" height="1078" alt="image" src="https://github.com/user-attachments/assets/18dba543-78fd-468a-a489-73b14b449890" />

Xử lý trả nợ

Kiểm tra dữ liệu sau khi trả
```sql
SELECT * FROM GiaoDichTraNo WHERE HopDongID = 1;                          -- Kiểm tra giao dịch đã thêm
SELECT * FROM HopDongCamDo WHERE HopDongID = 1;                           -- Kiểm tra trạng thái hợp đồng
SELECT * FROM ChiTietTaiSanCamCo WHERE HopDongID = 1;                     -- Kiểm tra trạng thái tài sản
SELECT * FROM NhatKyHeThong ORDER BY NhatKyID DESC;                       -- Kiểm tra audit log
GO                                                                         -- Kết thúc batch
```

Kết quả kiểm tra
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/ef9db076-8ec3-4c1d-a6de-87cc198b0850" />

 Kiểm tra giao dịch, Kiểm tra hợp đồng
 
PHẦN 4 - EVENT 4: TRUY VẤN DANH SÁCH NỢ XẤU

4.1. Phân tích logic

Yêu cầu: Liệt kê khách hàng nợ xấu với các điều kiện:

Đã quá Deadline1

Chưa thanh toán đủ (trạng thái không phải Đã thanh toán)

Các cột: Tên KH, SĐT, Số tiền vay gốc, Số ngày quá hạn, Tổng tiền phải trả hiện tại, Tổng tiền phải trả sau 1 tháng

Giải pháp: Tạo VIEW sử dụng function fn_CalcMoneyContract để tính nợ thời gian thực.

4.2. View danh sách nợ xấu

```sql
CREATE OR ALTER VIEW vw_DanhSachNoXau                                     -- Tạo view danh sách nợ xấu
AS
SELECT
    kh.HoTen AS TenKhachHang,                                             -- Họ tên khách hàng
    kh.SoDienThoai,                                                       -- Số điện thoại
    hd.SoTienVayGoc,                                                     -- Số tiền vay gốc
    DATEDIFF(DAY, hd.Deadline1, CAST(GETDATE() AS DATE)) AS SoNgayQuaHan, -- Số ngày quá hạn tính đến hôm nay
    dbo.fn_CalcMoneyContract(hd.HopDongID, CAST(GETDATE() AS DATE))      -- Tổng nợ hiện tại
        AS TongTienPhaiTraHienTai,
    dbo.fn_CalcMoneyContract(hd.HopDongID, DATEADD(MONTH, 1, CAST(GETDATE() AS DATE)))
        AS TongTienPhaiTraSau1Thang,                                      -- Tổng nợ sau 1 tháng
    hd.TrangThaiHopDong,                                                 -- Trạng thái hợp đồng
    hd.MaHopDong                                                         -- Mã hợp đồng
FROM HopDongCamDo hd
INNER JOIN KhachHang kh                                                   -- Kết nối bảng khách hàng
    ON hd.KhachHangID = kh.KhachHangID
WHERE CAST(GETDATE() AS DATE) > hd.Deadline1                              -- Đã quá Deadline1
  AND hd.TrangThaiHopDong NOT IN (N'DaThanhToan', N'DaThanhLy');          -- Chưa thanh toán đủ
GO                                                                         -- Kết thúc batch
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/39e425c9-1ad5-42ee-86b4-5a31fe4a7812" />

Lệnh chạy thử
```sql
SELECT * FROM vw_DanhSachNoXau;                                           -- Xem danh sách nợ xấu
GO                                                                         -- Kết thúc batch
Kết quả mong đợi
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/ffc8c0c3-e3ce-43ab-a76e-70930622d042" />

anh sách nợ xấu

PHẦN 5 - EVENT 5: TRIGGER QUẢN LÝ THANH LÝ TÀI SẢN

5.1. Phân tích logic

Yêu cầu 3 trigger:

Hợp đồng Đang vay quá Deadline1 → tự động chuyển sang Quá hạn

Hợp đồng Quá hạn quá Deadline2 → tài sản chuyển sang Sẵn sàng thanh lý

Hợp đồng chuyển sang Đã thanh lý → tài sản chuyển sang Đã bán thanh lý

Lưu ý: Trigger trong SQL Server chỉ chạy khi có thao tác UPDATE/INSERT. Cần dùng inserted/deleted tables.

5.2. Trigger 1: Chuyển hợp đồng qua hạn

```sql
CREATE OR ALTER TRIGGER trg_CapNhatHopDongQuaHan                           -- Tạo trigger cập nhật hợp đồng quá hạn
ON HopDongCamDo                                                           -- Kích hoạt trên bảng HopDongCamDo
AFTER INSERT, UPDATE                                                     -- Sau khi INSERT hoặc UPDATE
AS
BEGIN
    SET NOCOUNT ON;                                                      -- Tắt thông báo

    -- Cập nhật trạng thái hợp đồng thành Quá hạn nếu:
    -- - Trạng thái hiện tại là Đang vay
    -- - Ngày hiện tại > Deadline1
    UPDATE hd
    SET TrangThaiHopDong = N'QuaHan'
    FROM HopDongCamDo hd
    INNER JOIN inserted i ON hd.HopDongID = i.HopDongID                 -- Lấy bản ghi mới
    WHERE hd.TrangThaiHopDong = N'DangVay'
      AND CAST(GETDATE() AS DATE) > hd.Deadline1;
END;
GO                                                                        -- Kết thúc batch
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/d2d0a09c-4a7b-4c91-83f0-0c23f7f12e11" />

Lệnh chạy thử (Trigger chỉ chạy khi UPDATE)
```sql
-- Cập nhật hợp đồng để kích hoạt trigger
UPDATE HopDongCamDo
SET GhiChu = N'Kích hoạt trigger kiểm tra quá hạn'
WHERE HopDongID = 1;

-- Kiểm tra trạng thái sau trigger
SELECT HopDongID, MaHopDong, TrangThaiHopDong, Deadline1, Deadline2
FROM HopDongCamDo
WHERE HopDongID = 1;
GO                                                                        -- Kết thúc batch
```
Kết quả mong đợi
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/b98d46fa-88f9-4a98-935f-8bef6e723477" />

 Hợp đồng quá hạn
 
5.3. Trigger 2: Tài sản sẵn sàng thanh lý

```sql
CREATE OR ALTER TRIGGER trg_CapNhatTaiSanSanSangThanhLy                   -- Tạo trigger tài sản sẵn sàng thanh lý
ON HopDongCamDo                                                           -- Kích hoạt trên bảng HopDongCamDo
AFTER INSERT, UPDATE                                                     -- Sau khi INSERT/UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Cập nhật tài sản thành SanSangThanhLy nếu:
    -- - Hợp đồng ở trạng thái Quá hạn
    -- - Ngày hiện tại > Deadline2
    -- - Tài sản chưa bị bán
    UPDATE ct
    SET
        ct.TrangThaiTaiSan = N'SanSangThanhLy',
        ct.IsSanSangThanhLy = 1
    FROM ChiTietTaiSanCamCo ct
    INNER JOIN inserted i ON ct.HopDongID = i.HopDongID
    INNER JOIN HopDongCamDo hd ON hd.HopDongID = i.HopDongID
    WHERE hd.TrangThaiHopDong = N'QuaHan'
      AND CAST(GETDATE() AS DATE) > hd.Deadline2
      AND ct.IsDaBanThanhLy = 0
      AND ct.TrangThaiTaiSan = N'DangCamCo';
END;
GO                                                                        -- Kết thúc batch
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/1a482a8e-1ba2-422f-8c8b-fff7f6ecf77a" />

Lệnh chạy thử
```sql
-- Cập nhật hợp đồng để kích hoạt trigger (nếu đã qua Deadline2)
UPDATE HopDongCamDo
SET TrangThaiHopDong = N'QuaHan'
WHERE HopDongID = 1;

-- Kiểm tra tài sản sau trigger
SELECT *
FROM ChiTietTaiSanCamCo
WHERE HopDongID = 1;
GO                                                                        -- Kết thúc batch
```

Kết quả mong đợi
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/12e49db9-aa25-43ee-966b-9570893956ef" />

Tài sản sẵn sàng thanh lý

5.4. Trigger 3: Tài sản đã bán thanh lý

```sql
CREATE OR ALTER TRIGGER trg_CapNhatTaiSanDaBanThanhLy                    -- Tạo trigger tài sản đã bán thanh lý
ON HopDongCamDo                                                           -- Kích hoạt trên bảng HopDongCamDo
AFTER UPDATE                                                             -- Chỉ sau khi UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Khi hợp đồng chuyển sang DaThanhLy, cập nhật tài sản thành DaBanThanhLy
    UPDATE ct
    SET
        ct.TrangThaiTaiSan = N'DaBanThanhLy',
        ct.IsDaBanThanhLy = 1,
        ct.NgayThanhLy = CAST(GETDATE() AS DATE)
    FROM ChiTietTaiSanCamCo ct
    INNER JOIN inserted i ON ct.HopDongID = i.HopDongID                   -- Bản ghi mới
    INNER JOIN deleted d ON d.HopDongID = i.HopDongID                     -- Bản ghi cũ
    WHERE i.TrangThaiHopDong = N'DaThanhLy'                               -- Trạng thái mới = Đã thanh lý
      AND d.TrangThaiHopDong <> N'DaThanhLy';                            -- Trạng thái cũ khác
END;
GO                                                                        -- Kết thúc batch
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/372a9068-5109-4b36-8c22-9491562e0a2d" />

Lệnh chạy thử
```sql
-- Cập nhật hợp đồng sang trạng thái Đã thanh lý
UPDATE HopDongCamDo
SET TrangThaiHopDong = N'DaThanhLy'
WHERE HopDongID = 1;

-- Kiểm tra tài sản sau trigger
SELECT *
FROM ChiTietTaiSanCamCo
WHERE HopDongID = 1;
GO                                                                        -- Kết thúc batch
```
Kết quả mong đợi
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/2f6c41ae-baef-4358-91df-c5b37affa8e2" />

Tài sản đã bán thanh lý

PHẦN BỔ SUNG 1 - GIA HẠN HỢP ĐỒNG

6.1. Phân tích logic

Yêu cầu:

Khách đến trả toàn bộ tiền lãi tính đến thời điểm hiện tại

Dời Deadline1 và Deadline2 sang kỳ hạn mới

Tránh bị tính lãi kép tiếp tục từ mốc cũ

Phải lưu lịch sử gia hạn
Quy trình:

Tính tổng nợ hiện tại

Tính gốc còn lại = số tiền vay - tổng gốc đã trả

Tiền lãi cần đóng = tổng nợ - gốc còn lại

Ghi giao dịch đóng lãi

Lưu lịch sử gia hạn

Cập nhật deadline mới, reset trạng thái về Đang vay

6.2. Store Procedure gia hạn hợp đồng

```sql
CREATE OR ALTER PROCEDURE sp_GiaHanHopDong                               -- Tạo procedure gia hạn hợp đồng
(                                                                         -- Tham số
    @HopDongID INT,                                                      -- ID hợp đồng cần gia hạn
    @NgayGiaHan DATE,                                                    -- Ngày diễn ra gia hạn
    @Deadline1Moi DATE,                                                  -- Deadline1 mới
    @Deadline2Moi DATE,                                                  -- Deadline2 mới
    @NguoiXuLy NVARCHAR(100)                                             -- Người xử lý
)
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        -- Khai báo biến
        DECLARE @Deadline1Cu DATE;

        DECLARE @Deadline2Cu DATE;                                        -- Lưu Deadline2 cũ
        DECLARE @TongNoHienTai DECIMAL(18,2);                             -- Tổng nợ hiện tại
        DECLARE @TongGocDaTra DECIMAL(18,2);                              -- Tổng số tiền gốc đã trả
        DECLARE @SoTienVayGoc DECIMAL(18,2);                              -- Số tiền vay gốc ban đầu
        DECLARE @GocConLai DECIMAL(18,2);                                 -- Số tiền gốc còn lại
        DECLARE @TienLaiCanDong DECIMAL(18,2);                            -- Số tiền lãi cần đóng để gia hạn

        -- Lấy deadline cũ và số tiền vay gốc
        SELECT
            @Deadline1Cu = Deadline1,
            @Deadline2Cu = Deadline2,
            @SoTienVayGoc = SoTienVayGoc
        FROM HopDongCamDo
        WHERE HopDongID = @HopDongID;

        -- Tính tổng tiền gốc đã trả đến ngày gia hạn
        SELECT @TongGocDaTra = ISNULL(SUM(TienTraGoc), 0)
        FROM GiaoDichTraNo
        WHERE HopDongID = @HopDongID
          AND NgayGiaoDich <= @NgayGiaHan;

        -- Tính số gốc còn lại
        SET @GocConLai = @SoTienVayGoc - @TongGocDaTra;
        IF @GocConLai < 0 SET @GocConLai = 0;

        -- Tính tổng nợ hiện tại
        SET @TongNoHienTai = dbo.fn_CalcMoneyContract(@HopDongID, @NgayGiaHan);

        -- Tiền lãi cần đóng = tổng nợ - gốc còn lại
        SET @TienLaiCanDong = @TongNoHienTai - @GocConLai;
        IF @TienLaiCanDong < 0 SET @TienLaiCanDong = 0;

        -- Ghi giao dịch đóng lãi để gia hạn
        INSERT INTO GiaoDichTraNo
        (
            HopDongID, NgayGiaoDich, SoTienTra, TienTraGoc, TienTraLai,
            NguoiThuTien, NoiDungGiaoDich, SoTienNoConLaiSauGiaoDich
        )
        VALUES
        (
            @HopDongID,
            @NgayGiaHan,
            @TienLaiCanDong,
            0,
            @TienLaiCanDong,
            @NguoiXuLy,
            N'Đóng lãi để gia hạn hợp đồng',
            @GocConLai
        );

        -- Lưu lịch sử gia hạn
        INSERT INTO LichSuGiaHan
        (
            HopDongID, NgayGiaHan, Deadline1Cu, Deadline1Moi,
            Deadline2Cu, Deadline2Moi, SoTienLaiDaDong, NguoiXuLy, GhiChu
        )
        VALUES
        (
            @HopDongID,
            @NgayGiaHan,
            @Deadline1Cu,
            @Deadline1Moi,
            @Deadline2Cu,
            @Deadline2Moi,
            @TienLaiCanDong,
            @NguoiXuLy,
            N'Gia hạn hợp đồng sau khi khách đóng đủ tiền lãi'
        );

        -- Cập nhật hợp đồng với deadline mới và đưa trạng thái về Đang vay
        UPDATE HopDongCamDo
        SET
            Deadline1 = @Deadline1Moi,
            Deadline2 = @Deadline2Moi,
            TrangThaiHopDong = N'DangVay'
        WHERE HopDongID = @HopDongID;

        -- Ghi log hệ thống
        INSERT INTO NhatKyHeThong
        (
            TenBang, KhoaChinhBanGhi, HanhDong, NoiDungThayDoi, NguoiThucHien
        )
        VALUES
        (
            N'HopDongCamDo',
            CAST(@HopDongID AS NVARCHAR(100)),
            N'UPDATE',
            N'Gia hạn hợp đồng và cập nhật deadline mới',
            @NguoiXuLy
        );

        -- Trả kết quả
        SELECT
            @HopDongID AS HopDongID,
            @TienLaiCanDong AS SoTienLaiCanDong,
            @Deadline1Moi AS Deadline1Moi,
            @Deadline2Moi AS Deadline2Moi,
            N'Gia hạn hợp đồng thành công' AS ThongBao;

        COMMIT TRANSACTION;                                                -- Xác nhận transaction
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;                                              -- Hoàn tác nếu lỗi
        THROW;                                                             -- Ném lỗi ra ngoài
    END CATCH
END;
GO                                                                        -- Kết thúc batch
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/a680e26c-57fc-4301-b2f4-2f7d764f106c" />

Lệnh chạy thử
```sql
EXEC sp_GiaHanHopDong                                                     -- Gọi procedure gia hạn hợp đồng
    @HopDongID = 1,                                                       -- Hợp đồng ID = 1
    @NgayGiaHan = '2026-05-18',                                           -- Ngày gia hạn
    @Deadline1Moi = '2026-05-28',                                         -- Deadline1 mới
    @Deadline2Moi = '2026-06-10',                                         -- Deadline2 mới
    @NguoiXuLy = N'Thu ngân 3';                                           -- Người xử lý
GO                                                                        -- Kết thúc batch
```
Kết quả mong đợi
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/9a55fd49-313d-4d07-9f06-35850a55b56d" />

Gia hạn hợp đồng

Kiểm tra dữ liệu sau gia hạn

```sql
SELECT * FROM LichSuGiaHan WHERE HopDongID = 1;                           -- Kiểm tra lịch sử gia hạn
SELECT * FROM HopDongCamDo WHERE HopDongID = 1;                           -- Kiểm tra deadline mới của hợp đồng
SELECT * FROM GiaoDichTraNo WHERE HopDongID = 1 ORDER BY GiaoDichID DESC; -- Kiểm tra giao dịch đóng lãi
GO                                                                        -- Kết thúc batch
```
Kết quả kiểm tra

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/cace58d8-aee0-4e58-8aea-f2ed63feaeca" />

Kiểm tra lịch sử gia hạn
Kiểm tra hợp đồng sau gia hạn


PHẦN BỔ SUNG 2 - AUDIT LOG LỊCH SỬ TRẢ NỢ

7.1. Phân tích logic

Đề yêu cầu:

CSDL phải ghi lại mỗi lần khách trả một ít tiền

Tránh việc chỉ ghi đè số tổng nợ khiến mất dấu vết

Thực tế bài làm đã có:

GiaoDichTraNo: lưu chi tiết từng lần trả

NhatKyHeThong: lưu log hệ thống

Để tăng tính tự động, ta viết thêm trigger:

Mỗi lần thêm bản ghi vào GiaoDichTraNo

Tự động sinh log vào NhatKyHeThong

7.2. Trigger log giao dịch trả nợ

```sql
CREATE OR ALTER TRIGGER trg_LogGiaoDichTraNo                              -- Tạo trigger log khi có giao dịch trả nợ
ON GiaoDichTraNo                                                          -- Gắn trigger trên bảng GiaoDichTraNo
AFTER INSERT                                                              -- Chạy sau khi INSERT
AS
BEGIN
    SET NOCOUNT ON;                                                       -- Tắt thông báo số dòng

    -- Ghi log cho từng giao dịch vừa thêm
    INSERT INTO NhatKyHeThong
    (
        TenBang, KhoaChinhBanGhi, HanhDong, NoiDungThayDoi, NguoiThucHien
    )
    SELECT
        N'GiaoDichTraNo',                                                 -- Tên bảng phát sinh thay đổi
        CAST(i.GiaoDichID AS NVARCHAR(100)),                              -- Khóa chính của giao dịch
        N'INSERT',                                                        -- Hành động là thêm mới
        N'Thêm giao dịch trả nợ. SoTienTra = ' 
            + CAST(i.SoTienTra AS NVARCHAR(50))
            + N', SoTienNoConLai = '
            + ISNULL(CAST(i.SoTienNoConLaiSauGiaoDich AS NVARCHAR(50)), N'NULL'),
        i.NguoiThuTien                                                    -- Người thu tiền cũng là người thực hiện
    FROM inserted i;                                                      -- Lấy dữ liệu vừa được thêm
END;
GO                                                                        -- Kết thúc batch
```

<img width="1913" height="1078" alt="image" src="https://github.com/user-attachments/assets/3f936f09-1720-4417-84ef-76c7193ce59e" />

Lệnh chạy thử
```sql
INSERT INTO GiaoDichTraNo                                                 -- Thêm thử một giao dịch mới để kích hoạt trigger
(
    HopDongID, NgayGiaoDich, SoTienTra, TienTraGoc, TienTraLai,
    NguoiThuTien, NoiDungGiaoDich, SoTienNoConLaiSauGiaoDich
)
VALUES
(
    1,
    '2026-05-19',
    2000000,
    1500000,
    500000,
    N'Thu ngân kiểm thử',
    N'Giao dịch kiểm thử trigger log',
    10000000
);
GO                                                                        -- Kết thúc batch

```
Kết quả mong đợi
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/bb3a4d4e-070f-4241-b973-2060dcf89f9e" />

Trigger log giao dịch trả nợ

Kiểm tra log đã sinh

```sql
SELECT *                                                                  -- Xem toàn bộ log hệ thống
FROM NhatKyHeThong
ORDER BY NhatKyID DESC;                                                   -- Mới nhất ở trên cùng
GO                                                                        -- Kết thúc batch
```


Kết quả kiểm tra
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/61ccf2c3-863c-469c-8cd4-9b4db513c6e8" />
Kiểm tra NhatKyHeThong
