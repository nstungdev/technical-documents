# Locks Set by Different SQL Statements in InnoDB

Trong tài liệu này chúng ta sẽ tìm hiểu về "Các khóa được đặt theo các câu lệnh SQL khác nhau trong InnoDB"

## Nội dung
Locking read, một câu lệnh **UPDATE** hoặc **DELETE** thường thiết lập khóa bản ghi trên mọi chỉ mục (index record) được quét trong quá trình xử lý một câu lệnh SQL. 
Điều này không phụ thuộc vào việc câu lệnh có điều kiện WHERE để loại trừ hàng (row) hay không. InnoDB không ghi nhớ chính xác điều kiện WHERE, mà chỉ biết phạm vi chỉ mục nào đã được quét. Các khóa này thường là khóa next-key (khóa kế tiếp), cũng chặn các thao tác chèn (insert) vào “gap” ngay trước bản ghi. Tuy nhiên, việc gap locking có thể được vô hiệu hóa một cách rõ ràng, điều này khiến next-key locking không được sử dụng. Mức độ cô lập giao dịch (transaction isolation level) cũng có thể ảnh hưởng đến loại khóa được thiết lập.

Nếu một chỉ mục phụ (secondary index) được sử dụng trong một truy vấn và các khóa bản ghi trên chỉ mục đó là khóa độc quyền (exclusive locks), InnoDB cũng sẽ truy xuất các bản ghi trên chỉ mục cụm (clustered index) tương ứng và thiết lập khóa trên chúng. 

Nếu không có chỉ mục phù hợp cho câu lệnh và MySQL phải quét toàn bộ bảng để xử lý câu lệnh, thì mọi hàng trong bảng sẽ bị khóa. Điều này sẽ chặn tất cả các thao tác chèn (insert) của người dùng khác vào bảng. Do đó, việc tạo các chỉ mục tốt là rất quan trọng để các truy vấn của bạn không quét nhiều hàng hơn mức cần thiết.

InnoDB thiết lập các loại khóa cụ thể như sau.
* Lệnh **SELECT ... FROM** là một lần đọc nhất quán (consistent read), đọc từ một ảnh chụp (snapshot) của cơ sở dữ liệu mà không thiết lập bất kỳ khóa nào, trừ khi mức độ cô lập giao dịch (transaction isolation level) được đặt là **SERIALIZABLE**. Với mức **SERIALIZABLE**, truy vấn sẽ thiết lập các khóa chia sẻ next-key (shared next-key locks) trên các bản ghi chỉ mục (index records) mà nó gặp phải. Tuy nhiên, chỉ cần khóa bản ghi chỉ mục (index record lock) đối với các câu lệnh sử dụng chỉ mục duy nhất (unique index) để tìm kiếm một hàng duy nhất.
* Các câu lệnh **SELECT ... FOR UPDATE** và **SELECT ... FOR SHARE** sử dụng chỉ mục duy nhất (unique index) sẽ khóa các hàng được quét và giải phóng khóa cho các hàng không đủ điều kiện để đưa vào tập kết quả (ví dụ: nếu chúng không đáp ứng các tiêu chí được đưa ra trong mệnh đề **WHERE**). Tuy nhiên, trong một số trường hợp, các hàng có thể không được mở khóa ngay lập tức vì mối liên hệ giữa một hàng trong kết quả và nguồn gốc ban đầu của nó bị mất trong quá trình thực thi truy vấn. Ví dụ, trong một truy vấn **UNION**, các hàng được quét (và bị khóa) từ một bảng có thể được chèn vào một bảng tạm thời trước khi đánh giá xem chúng có đủ điều kiện để nằm trong tập kết quả hay không. Trong trường hợp này, mối liên hệ giữa các hàng trong bảng tạm thời và các hàng trong bảng gốc bị mất, và các hàng trong bảng gốc sẽ không được mở khóa cho đến khi quá trình thực thi truy vấn hoàn tất.
* Đối với các lần đọc có khóa (SELECT với FOR UPDATE hoặc FOR SHARE), các câu lệnh UPDATE và DELETE, loại khóa được thiết lập phụ thuộc vào việc câu lệnh sử dụng chỉ mục duy nhất (unique index) với điều kiện tìm kiếm duy nhất (unique search condition) hay điều kiện tìm kiếm dạng phạm vi (range-type search condition).
    * Đối với một chỉ mục duy nhất (unique index) với điều kiện tìm kiếm duy nhất (unique search condition), InnoDB chỉ khóa bản ghi chỉ mục được tìm thấy mà không khóa khoảng trống (gap) phía trước nó.
    * Đối với các điều kiện tìm kiếm khác và các chỉ mục không duy nhất (non-unique indexes), InnoDB sẽ khóa phạm vi chỉ mục được quét, sử dụng khóa khoảng trống (gap locks) hoặc khóa kế tiếp (next-key locks) để ngăn các phiên khác chèn dữ liệu vào các khoảng trống trong phạm vi đã được khóa.
* Đối với các bản ghi chỉ mục mà truy vấn gặp phải, câu lệnh SELECT ... FOR UPDATE sẽ chặn các phiên khác thực hiện SELECT ... FOR SHARE hoặc đọc dữ liệu trong một số mức độ cô lập giao dịch nhất định. Các lần đọc nhất quán (consistent reads) bỏ qua mọi khóa được thiết lập trên các bản ghi tồn tại trong phạm vi nhìn đọc (read view).
* Câu lệnh **UPDATE ... WHERE ...** thiết lập một khóa độc quyền next-key trên mọi bản ghi mà truy vấn gặp phải. Tuy nhiên, chỉ cần khóa bản ghi chỉ mục (index record lock) đối với các câu lệnh khóa hàng sử dụng chỉ mục duy nhất (unique index) để tìm kiếm một hàng duy nhất.
* Khi **UPDATE** sửa đổi một bản ghi trong chỉ mục cụm (clustered index), các khóa ngầm (implicit locks) được thiết lập trên các bản ghi chỉ mục phụ (secondary index) bị ảnh hưởng. Ngoài ra, thao tác UPDATE cũng thiết lập các khóa chia sẻ (shared locks) trên các bản ghi chỉ mục phụ bị ảnh hưởng khi thực hiện quét kiểm tra trùng lặp trước khi chèn các bản ghi chỉ mục phụ mới, và khi chèn các bản ghi chỉ mục phụ mới.
* **DELETE FROM ... WHERE ...** thiết lập một khóa độc quyền **next-key** trên mọi bản ghi mà truy vấn gặp phải. Tuy nhiên, chỉ cần khóa bản ghi chỉ mục (index record lock) đối với các câu lệnh khóa hàng sử dụng chỉ mục duy nhất (unique index) để tìm kiếm một hàng duy nhất.
* **INSERT** thiết lập một khóa độc quyền (exclusive lock) trên hàng được chèn vào. Khóa này là khóa bản ghi chỉ mục (index-record lock), không phải là khóa next-key (tức là không có khóa khoảng trống - gap lock) và không ngăn các phiên khác chèn vào khoảng trống trước hàng vừa được chèn.


Trước khi chèn một hàng, một loại khóa khoảng trống gọi là insert intention gap lock (khóa khoảng trống có ý định chèn) sẽ được thiết lập. Khóa này báo hiệu ý định chèn theo cách mà nhiều giao dịch chèn vào cùng một khoảng trống chỉ mục (index gap) sẽ không phải chờ đợi nhau nếu chúng không chèn vào cùng một vị trí trong khoảng trống đó. Giả sử có các bản ghi chỉ mục với giá trị 4 và 7. Các giao dịch riêng biệt cố gắng chèn các giá trị 5 và 6, mỗi giao dịch sẽ khóa khoảng trống giữa 4 và 7 bằng insert intention locks trước khi lấy khóa độc quyền (exclusive lock) trên hàng sẽ được chèn, nhưng chúng không chặn nhau vì các hàng không xung đột.

Nếu xảy ra lỗi duplicate-key, một khóa chia sẻ (shared lock) sẽ được thiết lập trên bản ghi chỉ mục bị trùng lặp. Việc sử dụng khóa chia sẻ này có thể dẫn đến tình trạng deadlock nếu có nhiều phiên cố gắng chèn cùng một hàng, trong khi một phiên khác đã có khóa độc quyền. Điều này có thể xảy ra nếu một phiên khác xóa hàng đó. Giả sử một bảng InnoDB t1 có cấu trúc như sau:
```sql
CREATE TABLE t1 (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;
```
Bây giờ giả sử rằng ba phiên thực hiện các thao tác sau theo thứ tự:\
Session 1:
```sql
START TRANSACTION;
INSERT INTO t1 VALUES(1);
```

Session 2:
```sql
START TRANSACTION;
INSERT INTO t1 VALUES(1);
```

Session 3:
```sql
START TRANSACTION;
INSERT INTO t1 VALUES(1);
```

Session 1:
```sql
ROLLBACK;
```

Quá trình xảy ra như sau:
1. Session 1 thực hiện thao tác đầu tiên và lấy khóa độc quyền (exclusive lock) cho hàng.
2. Session 2 và Session 3 thực hiện các thao tác và đều gặp phải lỗi duplicate-key. Cả hai phiên này đều yêu cầu một khóa chia sẻ (shared lock) cho hàng.
3. Khi Session 1 rollback (hủy giao dịch), khóa độc quyền của nó trên hàng sẽ được giải phóng, và các yêu cầu khóa chia sẻ trong hàng đợi của Session 2 và Session 3 sẽ được cấp phép.
4. Tuy nhiên, tại thời điểm này, Session 2 và Session 3 rơi vào tình trạng deadlock: Không phiên nào có thể lấy khóa độc quyền (exclusive lock) cho hàng vì khóa chia sẻ (shared lock) đang được giữ bởi phiên kia.\
Tình huống này xảy ra vì Session 2 và Session 3 đều yêu cầu khóa độc quyền nhưng lại không thể có được khóa này do khóa chia sẻ mà mỗi phiên nắm giữ. Đây là một ví dụ điển hình của deadlock trong các giao dịch của cơ sở dữ liệu.

Tình huống tương tự xảy ra nếu bảng đã chứa một hàng với giá trị khóa là 1 và ba phiên thực hiện các thao tác sau theo thứ tự

Session 1:
```sql
START TRANSACTION;
DELETE FROM t1 WHERE i = 1;
```

Session 2:
```sql
START TRANSACTION;
INSERT INTO t1 VALUES(1);
```

Session 3:
```sql
START TRANSACTION;
INSERT INTO t1 VALUES(1);
```

Session 1:
```sql
COMMIT;
```
Thao tác đầu tiên của Session 1 lấy khóa độc quyền (exclusive lock) cho hàng. Các thao tác của Session 2 và Session 3 đều gặp lỗi duplicate-key và chúng đều yêu cầu khóa chia sẻ (shared lock) cho hàng. Khi Session 1 commit (cam kết giao dịch), nó giải phóng khóa độc quyền của mình trên hàng và các yêu cầu khóa chia sẻ trong hàng đợi của Session 2 và Session 3 được cấp phép. Tuy nhiên, tại thời điểm này, Session 2 và Session 3 rơi vào tình trạng deadlock: Không phiên nào có thể lấy khóa độc quyền (exclusive lock) cho hàng vì khóa chia sẻ (shared lock) đang được giữ bởi phiên kia.

* Câu lệnh **INSERT ... ON DUPLICATE KEY UPDATE** khác với một câu lệnh INSERT đơn giản ở chỗ một khóa độc quyền (exclusive lock) thay vì khóa chia sẻ (shared lock) được đặt lên hàng sẽ được cập nhật khi xảy ra lỗi duplicate-key. Một khóa bản ghi chỉ mục độc quyền (exclusive index-record lock) sẽ được thiết lập cho một giá trị khóa chính trùng lặp. Một khóa next-key độc quyền (exclusive next-key lock) sẽ được thiết lập cho một giá trị khóa duy nhất trùng lặp.
* Câu lệnh **REPLACE** thực hiện giống như một câu lệnh **INSERT** nếu không có sự trùng lặp trên một khóa duy nhất. Ngược lại, một khóa next-key độc quyền (exclusive next-key lock) sẽ được đặt lên hàng sẽ bị thay thế.
* Câu lệnh **INSERT INTO T SELECT ... FROM S WHERE ...** thiết lập một khóa bản ghi chỉ mục độc quyền (exclusive index record lock) (không có khóa khoảng trống - gap lock) cho mỗi hàng được chèn vào T. Nếu mức độ cô lập giao dịch là **READ COMMITTED**, InnoDB thực hiện tìm kiếm trên S như một lần đọc nhất quán (consistent read) (không có khóa). Ngược lại, InnoDB thiết lập các khóa next-key chia sẻ (shared next-key locks) trên các hàng từ S. InnoDB phải thiết lập các khóa trong trường hợp sau: Trong quá trình phục hồi roll-forward khi sử dụng nhật ký nhị phân dựa trên câu lệnh (statement-based binary log), mỗi câu lệnh SQL phải được thực thi chính xác theo cách nó đã được thực hiện ban đầu.
* Câu lệnh **CREATE TABLE ... SELECT ...** thực hiện truy vấn SELECT với các khóa next-key chia sẻ (shared next-key locks) hoặc như một lần đọc nhất quán (consistent read), giống như trong trường hợp **INSERT ... SELECT**.

Khi một câu lệnh **SELECT** được sử dụng trong các cấu trúc **REPLACE INTO t SELECT ... FROM s WHERE ...** hoặc **UPDATE t ... WHERE col IN (SELECT ... FROM s ...)**, InnoDB thiết lập các khóa next-key chia sẻ (shared next-key locks) trên các hàng từ bảng s.
* InnoDB thiết lập một khóa độc quyền (exclusive lock) ở cuối chỉ mục liên kết với cột AUTO_INCREMENT khi khởi tạo một cột AUTO_INCREMENT đã được chỉ định trước trong bảng.

Với innodb_autoinc_lock_mode=0, InnoDB sử dụng một chế độ khóa bảng AUTO-INC đặc biệt, trong đó khóa được lấy và giữ đến cuối câu lệnh SQL hiện tại (không phải đến cuối giao dịch toàn bộ) khi truy cập bộ đếm auto-increment. Các client khác không thể chèn dữ liệu vào bảng khi khóa AUTO-INC bảng đang được giữ. Hành vi tương tự xảy ra đối với các bulk inserts khi innodb_autoinc_lock_mode=1. Các khóa AUTO-INC cấp bảng không được sử dụng khi innodb_autoinc_lock_mode=2.

InnoDB lấy giá trị của cột AUTO_INCREMENT đã được khởi tạo trước mà không thiết lập bất kỳ khóa nào.

Nếu một ràng buộc FOREIGN KEY được định nghĩa trên bảng, bất kỳ thao tác insert, update, hoặc delete nào yêu cầu kiểm tra điều kiện ràng buộc đều sẽ thiết lập các khóa chia sẻ (shared record-level locks) trên các bản ghi mà nó kiểm tra để kiểm tra ràng buộc. InnoDB cũng thiết lập các khóa này trong trường hợp ràng buộc không thành công.

Câu lệnh LOCK TABLES thiết lập khóa bảng, nhưng chính lớp MySQL cao hơn lớp InnoDB mới thiết lập các khóa này. InnoDB nhận thức được các khóa bảng nếu innodb_table_locks = 1 (mặc định) và autocommit = 0, và lớp MySQL cao hơn InnoDB biết về các khóa cấp bản ghi (row-level locks).

Trong trường hợp khác, cơ chế phát hiện deadlock tự động của InnoDB không thể phát hiện deadlock khi các khóa bảng như vậy được sử dụng. Ngoài ra, vì trong trường hợp này lớp MySQL cao hơn không biết về các khóa cấp bản ghi, nên có thể xảy ra tình huống lấy khóa bảng trên một bảng mà phiên khác hiện đang giữ các khóa cấp bản ghi. Tuy nhiên, điều này không ảnh hưởng đến tính toàn vẹn của giao dịch.

Câu lệnh LOCK TABLES sẽ lấy hai khóa trên mỗi bảng nếu innodb_table_locks=1 (mặc định). Ngoài khóa bảng ở lớp MySQL, nó còn lấy thêm một khóa bảng InnoDB. Để tránh việc lấy khóa bảng InnoDB, bạn có thể thiết lập innodb_table_locks=0. Nếu không có khóa bảng InnoDB nào được lấy, câu lệnh LOCK TABLES vẫn hoàn tất, ngay cả khi một số bản ghi của bảng đang bị khóa bởi các giao dịch khác.

Trong MySQL 8.4, innodb_table_locks=0 không có tác dụng đối với các bảng bị khóa rõ ràng với LOCK TABLES ... WRITE. Tuy nhiên, nó có tác dụng đối với các bảng bị khóa để đọc hoặc ghi bởi LOCK TABLES ... WRITE một cách ngầm định (ví dụ, thông qua triggers) hoặc bởi LOCK TABLES ... READ.

Tất cả các khóa InnoDB mà một giao dịch đang giữ sẽ được giải phóng khi giao dịch được cam kết (commit) hoặc hủy bỏ (abort). Do đó, việc gọi LOCK TABLES trên các bảng InnoDB trong chế độ autocommit=1 không có nhiều ý nghĩa vì các khóa bảng InnoDB sẽ được giải phóng ngay lập tức.

Bạn không thể khóa thêm bảng trong giữa một giao dịch vì LOCK TABLES thực hiện một COMMIT ngầm và UNLOCK TABLES.

[See more](https://dev.mysql.com/doc/refman/8.4/en/innodb-locks-set.html)