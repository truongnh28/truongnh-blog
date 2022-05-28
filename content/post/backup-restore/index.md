+++
author = "truongnh3"
title = "Backup and Restore"
date = "2022-05-19"
description = "Phục hồi database tại một thời điểm chưa sao lưu"
categories = [
    "Database",
]
tags = [
    "backup-restore",
]
image = "backup-restore-mssql-database.jpg"
+++
## Lấy Lại Dữ Liệu Sau Khi Xóa

### Giới thiệu

Bạn đã bao giờ nhỡ tay xóa mất một bảng (DROP TABLE) hoặc xóa sạch dữ liệu trong bảng (DELETE) của một database đang hoạt động rất nhộn nhịp, để rồi giật mình nhận ra mình vừa phạm một lỗi tày đình? Mồ hôi vã ra đầm đìa, nghe chuông điện thoại kêu mà giật mình thon thót, đầu óc quay cuồng nghĩ cách ăn nói thế nào cho phải. Những hoàn cảnh tương tự như vậy không phải hiếm. Đối với các DBA thường xuyên phải làm việc trực tiếp trên dữ liệu, những sự cố xảy ra do nhầm lẫn là không tránh khỏi. Rất may SQL Server đã tính đến tình huống này, và cung cấp các phương tiện cần thiết để lấy lại dữ liệu, với điều kiện database phải có những thiết lập đúng đắn từ trước.
Tuy nhiên bạn có thể thấy là vấn đề đã trở nên phức tạp hơn, câu lệnh không thể ROLLBACK lại được nữa vì SQL Server để chế độ mặc định là AUTO COMMIT. Khôi phục lại từ bản backup từ ngày hôm trước cũng không giải quyết được, vì như thế dữ liệu vừa mới được cập nhật trong ngày cũng sẽ mất tiêu luôn. Vấn đề đặt ra là cần lấy lại dữ liệu giống như thời điểm ngay trước khi chạy lệnh DELETE, chứ không phải từ ngày hôm qua. Một gợi ý cho bạn: log file luôn lưu lại tất cả các hành động diễn ra đối với database, bao gồm cả lệnh DELETE vừa xong. Đây là dấu vết duy nhất và cách làm của ta cũng là dựa vào đó để lần ngược lại.

### Điều kiện để có thể phục hồi

Để có thể khôi phục lại được đòi hỏi ba điều kiện sau:

- database có chế độ RECOVERY MODE là FULL
- database đã từng được FULL BACKUP và bạn có trong tay file backup gần nhất
- log file chưa từng bị SHRINK kể từ sau lần full backup gần nhất.
Nếu một trong ba điều kiện trên bị vi phạm thì vấn đề kể như hết cách giải cứu.
Giả sử cả ba điều kiện trên được thỏa mãn và lần full backup gần đây nhất là đêm hôm trước. Bạn có thể khôi phục thông qua các bước sau (xem thêm script ở phần dưới):
    1. Đóng lại tất cả các kết nối đến database để không tiếp nhận thêm dữ liệu
    2. Ghi lại thời điểm xảy ra lệnh DELETE lỗi
    3. Thực hiện BACKUP LOG cho database
    4. Khôi phục lại database theo trình tự sau:
       - RESTORE từ bản full backup đêm hôm trước
       - RESTORE từ bản log backup với lựa chọn STOPAT = thời điểm ngay trước khi có sự cố

Và khi mọi việc đã hoàn tất, chuyển lại database sang chế độ hoạt động bình thường để các ứng dụng lại có thể kết nối vào database.

## Thực hiện

**Script**
Tạo database và bảng:

``` sql
USE master
GO
IF DB_ID('TestDB') IS NOT NULL DROP DATABASE TestDB
GO
CREATE DATABASE TestDB
GO
USE TestDB
GO
CREATE TABLE dbo.Table1(ID INT IDENTITY, Ten VARCHAR(30) )
GO
INSERT INTO dbo.Table1(Ten)
SELECT 'Nguyen Van A' UNION ALL
SELECT 'Nguyen Van B' UNION ALL
SELECT 'Nguyen Van C'
```

Thực hiện full backup:

```sql
BACKUP DATABASE TestDB TO DISK = 'D:\Backup\TestDB.bak' WITH INIT
Thêm dữ liệu mới sau khi full backup:
INSERT INTO dbo.Table1(Ten)
SELECT 'Nguyen Van D' UNION ALL
SELECT 'Nguyen Van E'
```

Ba đoạn lệnh trên mô phỏng tình huống thực tế một database đã có chứa dữ liệu, được backup full lúc nửa đêm hôm trước, và trong ngày đã có thêm dữ liệu mới. Tại một thời điểm nào đó trong ngày bạn xóa mất dữ liệu do sơ suất:

```sql
DELETE FROM dbo.Table1
```

Sau khi xảy ra sự cố, việc tiếp theo đầu tiên bạn cần làm là đóng lại database, để không ai có thể tiếp tục cập nhật dữ liệu đến khi database được khôi phục xong. Bạn làm việc này bằng cách chuyển database về trạng thái SINGLE_USER (một người dùng). Vì bạn đang kết nối vào database, không ai khác có thể kết nối được nữa:

```sql
ALTER DATABASE TestDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE
```

Sau đó ghi lại thời điểm xảy ra sự cố:

```sql
SELECT GETDATE()
```

Khi đang chạy script cho bài viết này, thời điểm hiện tại của tôi là ’2010-12-07 17:51:03.990′.
Việc tiếp theo là backup log:

```sql
BACKUP LOG TestDB TO DISK = 'D:\backup\TestDB.trn' WITH INIT
```

Sau đó khôi phục lại database theo thứ tự bản full backup trước rồi đến bản log backup:

```sql
USE master
go
RESTORE DATABASE TestDB FROM DISK = 'D:\backup\TestDB.bak' WITH NORECOVERY
 
RESTORE DATABASE TestDB FROM DISK = 'D:\backup\TestDB.trn' WITH STOPAT='2010-12-07 17:50:00'
```

Điểm mấu chốt trong đoạn lệnh trên là mệnh đề STOPAT ở lệnh RESTORE thứ hai. Mục đích của nó là khôi phục lại database từ log backup nhưng dừng lại tại thời điểm được chỉ định. Khi các hành động xảy ra đối với database được lưu vào log file, nó cũng kèm theo thời điểm xảy ra hành động đó. Khi backup log file thì bản backup cũng chứa y nguyên các thông tin này. Vì thế khi restore từ log file với mệnh đề STOPAT, bạn đã yêu cầu hệ thống “tua” lại các hành động đã được áp dụng đối với database, nhưng dừng lại trước thời điểm có sự cố. Do đó lệnh DELETE trên không được thực hiện lại và bảng đã trở về trạng thái như cũ.
Hãy để ý ở mệnh đề STOPAT, tôi đã đẩy lùi thời gian lại một chút để đảm bảo thời điểm đó là trước khi xảy ra xóa dữ liệu. Khi chạy thử nghiệm bạn cần lưu ý điều này và chọn thời điểm cho thích hợp.
Và dữ liệu đã được khôi phục:

```sql
USE TestDB
GO
SELECT * FROM dbo.Table1
 
ID          Ten
----------- ------------------------------
1           Nguyen Van A
2           Nguyen Van B
3           Nguyen Van C
4           Nguyen Van D
5           Nguyen Van E
 
(5 ROW(s) affected)
```

Bước cuối cùng là chuyển lại database sang chế độ bình thường

```sql
ALTER DATABASE TestDB SET MULTI_USER
```

Vậy là vấn đề đã được giải quyết (sau khoảng nửa tiếng đến 1 tiếng gián đoạn sử dụng). Bây giờ là một câu hỏi giành cho bạn: Trong ví dụ trên tôi giả sử database chỉ có một backup duy nhất vào nửa đêm, và đó là full backup. Nếu ngoài ra database còn có các log backup định kỳ trong ngày (ví dụ mỗi tiếng 1 lần), thì việc khôi phục cần được thực hiện như thế nào?

Các bước thực hiện:

- Thực hiện Restore Full backup từ lúc nửa đêm
- Sau đó thực hiện restore các backup định kỳ theo đúng thứ tự
- Bước cuối cùng là RESTORE với tùy chọn STOPAT như trong bài đã hướng dẫn
Vâng đúng là như vậy. Một điều nữa là khi đó lệnh backup log ở trong ví dụ trên cần bỏ lựa chọn "WITH INIT" hoặc cần backup vào một file riêng (khác với file dùng trong các backup định kỳ)

## Tham khảo

Thầy Lưu Nguyễn Kỳ Thư

Đây là đồ án mà tôi đã thực hiện để demo vấn đề này. [backup-restore](https://github.com/truongnh28/TTCS_backup_restore)

<style>
.canon { background: white; width: 100%; height: auto; }
</style>