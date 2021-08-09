# MySQL - Master Slave

## 1. MySQL Replication là gì ?
- MySQL Replication là một quá trình cho phép duy trì nhiều bản sao của Database bằng cách replica chúng dưới mô hình master - slave. Việc này giúp hệ thống giữ được sự toàn về về mặt database, có khả năng backup khi gặp sự cố cũng như tối ưu hiệu năng nhờ việc isolate việc ghi và cập nhật data trên master - đọc trên các slave. 
- Server Master lữu trử phiên bản Database chính, Server Slave lưu trữ phiên bản Database được nhân bản từ Master. Quá trình nhân bản từ Master -> Slave được gọi là Replication.
- Dưới đây là ví dụ mô hình hoạt động :

 <img src="https://github.com/tulha161/sql/blob/main/pic/1.png">

## 2. Cách thức hoạt động : 

### Trên Master :
 - Các kết nối từ web app tới Master DB sẽ mở một `Session_Thread` khi có nhu cầu ghi dữ liệu. `Session_Thread` sẽ ghi các statement SQL vào một file binlog (ví dụ với format của binlog là statement-based hoặc mix). Binlog được lưu trữ trong `data_dir` (cấu hình my.cnf) và có thể được cấu hình các thông số như kích thước tối đa bao nhiêu, lưu lại trên server bao nhiêu ngày.
- Master DB sẽ mở một `Dump_Thread` và gửi binlog tới cho `I/O_Thread` mỗi khi `I/O_Thread` từ Slave DB yêu cầu dữ liệu.

### Trên Slave : 
- Trên mỗi Slave DB sẽ mở một `I/O_Thread` kết nối tới Master DB thông qua network, giao thức TCP để yêu cầu binlog.
- Sau khi `Dumb_Thread` gửi binlog tới `I/O_Thread`, `I/O_Thread` sẽ có nhiệm vụ đọc binlog này và ghi vào relaylog
- Đồng thời trên Slave sẽ mở một `SQL_Thread`, nó có nhiệm vụ đọc các event từ relaylog và apply các thay đổi đso vào Database trên slave -> Hoàn thành quá trình Replication.

 <img src="https://github.com/tulha161/sql/blob/main/pic/2.png">

## 3. Cấu hình MySQL Replication 

### Mô Hình : 
- OS : Ubuntu 20.04 LTS
- DB1 : 10.0.20.4 
- DB2 : 10.0.20.5

- Với DB1 là node master,  DB2 node Slave.
### Cài đặt mysql trên cả 2 server :
- Update và cài đặt MySQL
	````
	sudo apt update
	sudo apt -y install mysql-server mysql-client
	````
- Cài đặt mysql password `sudo mysql_secure_installation`
- Khởi động dịch vụ mysql `sudo systemctl enable --now mysql`
- alow port 3306 trên slave : `sudo ufw allow from 10.0.20.5 to any port 3306`

### Config trên Master 
- edit file `/etc/mysql/mysql.conf.d/mysqld.cnf` , tìm và chỉnh sửa các config sau : 
```
bind-address            = 10.0.20.4 #set về IP của master
server-id 		= 1	    #bỏ comment và set về 1 
log_bin 		= /var/log/mysql/mysql-bin.log # bỏ comment để sử dụng config
binlog_do_db		= db 	    #define tên db muốn replicate, ở đây mình đặt là **db** 

```
	
- tạo User replication trên mysql shell : 
```
CREATE USER 'replica'@'10.0.20.5' IDENTIFIED WITH mysql_native_password BY 'p@ssPASS1';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.5';
FLUSH PRIVILEGES;
```
- Tạo **db**, dữ liệu để test :
```
CREATE database db;
USE db;
CREATE TABLE tb1 (
example_column varchar(30)
);
```

### Config trên Slave : 
- edit file `/etc/mysql/mysql.conf.d/mysqld.cnf`
```
bind-address            = 10.0.20.5 #set về IP của Slave
server-id 		= 2	    #bỏ comment và set về 2 
log_bin 		= /var/log/mysql/mysql-bin.log # bỏ comment để sử dụng config
binlog_do_db		= db 	    #define tên db muốn replicate, ở đây mình đặt là **db** 

```

### Tạo Replication
### Trên Master:
- Nguyên tắc khi tạo replication là phải LOCK tất cả các table trên Master DB, để dữ liệu không thay đổi, sau đó xác định binlog và position, 2 thông số dùng để cấu hình trên Slave xác định đoạn dữ liệu bắt đầu đồng bộ.
- Trên DB Master : 
```
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |     1027 | db           |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

- Lấy được 2 giá trị là : 
	- File : mysql-bin.000001
	- Position : 1027
- Sau đó ta sẽ dump dữ liệu từ master DB và đẩy qua Slave DB ( sau khi dump thì `UNLOCK tables;` trở lại tại master ) 
```
mysqldump -u root db > db.sql
scp db.sql ubuntu@10.0.20.5:/tmp/
```

### Trên Slave : 
- quay sang slave, tạo database **db** mysql sau đó cấu hình đồng bộ :
```
mysql
create database db;
exit;
```
- cấu hình đồng bộ với master sử dụng user đã lập từ step trước : 
```
sudo mysql db < /tmp/db.sql
```
```
mysql

CHANGE MASTER TO MASTER_HOST='10.0.20.4',
MASTER_USER='user', 
MASTER_PASSWORD='p@ssPASS1', 
MASTER_LOG_FILE='mysql-bin.000001', 
MASTER_LOG_POS=1027;
START SLAVE;
SHOW SLAVE STATUS\G;
```
- Đã thiết lập thành công : 
 <img src="https://github.com/tulha161/sql/blob/main/pic/5.png">
 
### Test replication 

- Quay sang master, lập một bảng mới **tb2**, insert dữ vào bảng cũ **tb1** : 
```
mysql
CREATE TABLE tb2 (
example_column varchar(30)
);

insert into tb1 values 
 ('Hello'),
 ('How are you'),
  ('I am Fine');
```
- Sang slave, kiểm tra xem data thay đổi đã được đồng bộ sang chưa : 

```
mysql> use db;
Database changed
mysql> show tables;
+--------------+
| Tables_in_db |
+--------------+
| tb1          |
| tb2          |
+--------------+
2 rows in set (0.00 sec)

mysql> SELECT * FROM tb1;
+----------------+
| example_column |
+----------------+
| Hello          |
| How are you    |
| I am Fine      |
+----------------+
3 rows in set (0.00 sec)

mysql> exit;
```
- Như vậy, có thể thấy data đã được đồng bộ ngay, quá trình replication đã ổn. 

## 4. Failover và cách khắc phục :
- Với giải pháp Replication Master - Slave, các node được phân chia nhiệm vụ rõ ràng, node master đóng vai trò ghi, cập nhật dữ liệu mới, node slave đóng vai trò tạo ra bản sau của các dữ liệu đó, có thể được cấu hình làm source truy xuất dữ liệu từ client. Vì vậy khi failover xảy ra ( master hoặc slave chết ), các node có vai trò khác nhau nên sẽ không có tính backup trực tiếp cho nhau như hệ thống HA - master - master . Vì vậy khi node chết thì hầu như sẽ có downtime và việc xử lí cấu hình thủ công cũng không tránh khỏi. 
### 4.1. Trường hợp node Slave die : 
- Với trường hợp này, ta cần có node Slave mới thay thế cho node Slave đã chết. Cấu hình lại node Slave mới ngay khi có sự cố, các bước cấu hình như ở trên. 
- Khuyến nghị nên có ít nhất 2 node Slave để khi 1 node chết quá trình replication vẫn được đảm bảo.
### 4.2 Trường hợp node Master die : 
- Trường hợp này sẽ phức tạp hơn do phải promote một node Slave lên làm Master thay thế, quá trình này được thực hiện thủ công.
- Sẽ có downtime hệ thống, nhưng do Slave vẫn duy trì nên sẽ hạn chế được downtime tối thiểu.
- Cụ thể, với mô hình 1 master ( db1 ) và 2 slave ( db2 và db 3 ) :
	- cấu hình `--log-slave-updates=OFF` ở các slave, điều này sẽ skip việc ghi binlog tại Slave đối với các hành vi đồng bộ dữ liệu với Master.
	- Khi db1 die, chọn db2 trở thành Master mới, sau đó cấu hình tại db3 replicate db2. Tuy nhiên như ở trên ta sẽ cần chỉ định file binlog và position của file log, và do db2 sau khi đc đổi thành master thì mới bắt đầu sinh ra binlog, nên trên db3 ta chỉ cần trỏ về file binlog và position đầu tiên của db2 là đủ. => chuyện này đảm bảo rằng db3 sẽ đồng bộ dữ liệu với db2.
	- Cập nhật lại phía client địa chỉ của master mới.
- Tham khảo thêm tại source : https://dev.mysql.com/doc/refman/8.0/en/replication-solutions-switch.html
















