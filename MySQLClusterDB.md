# MySQL Cluster DB 

## 1. Mô hình Replication Master - Master 
- Ở bài này, ta vẫn sẽ sử dụng kỹ thuật mysql replication đã được đề cập ở lần trước.  Tuy nhiên, mô hình sử dụng sẽ là mô hình master - master với 3 node server.
- Mô hình hoạt động master - master là khi toàn bộ data được lưu trên 1 nhóm server ( cluster ), được cập nhật  ở bất kỳ thành phần nào trên nhóm. Tất cả Master server có nhiệm vụ phản hồi lại các truy vấn dữ liệu của người dùng và đồng bộ, cập nhật những thay đổi về dữ liệu từ các Master server khác trong nhóm.

### Mô hình thực hiện  :
- OS : Ubuntu 20.04 LTS 
- DB1 : 10.0.20.4 
- DB2 : 10.0.20.5
- DB3 : 10.0.20.6


## 2. Cấu hình Replication Cluster : 

### 2.1. Cài đặt và khởi động MySQL trên cả 3 node : 
```
sudo apt update
sudo apt -y install mysql-server mysql-client
sudo mysql_secure_installation
```

### 2.2. Chỉnh sửa file config MySQL
- chỉnh sửa file config tại `/etc/mysql/mysql.conf.d/mysqld.cnf` 

#### Tại db1 : 
```
server_id           = 1
log_bin             = /var/log/mysql/mysql-bin.log
log_bin_index       = /var/log/mysql/mysql-bin.log.index
relay_log           = /var/log/mysql/mysql-relay-bin
relay_log_index     = /var/log/mysql/mysql-relay-bin.index
expire_logs_days    = 10
max_binlog_size     = 100M
log_slave_updates   = 1
auto-increment-offset = 3
bind-address            = 10.0.20.4
```

#### Tại db2 : 
```
server_id           = 2
log_bin             = /var/log/mysql/mysql-bin.log
log_bin_index       = /var/log/mysql/mysql-bin.log.index
relay_log           = /var/log/mysql/mysql-relay-bin
relay_log_index     = /var/log/mysql/mysql-relay-bin.index
expire_logs_days    = 10
max_binlog_size     = 100M
log_slave_updates   = 1
auto-increment-offset = 3
bind-address            = 10.0.20.5
```
#### Tại db3 : 
```
server_id           = 3
log_bin             = /var/log/mysql/mysql-bin.log
log_bin_index       = /var/log/mysql/mysql-bin.log.index
relay_log           = /var/log/mysql/mysql-relay-bin
relay_log_index     = /var/log/mysql/mysql-relay-bin.index
expire_logs_days    = 10
max_binlog_size     = 100M
log_slave_updates   = 1
auto-increment-offset = 3
bind-address            = 10.0.20.6
```

- `systemctl restart mysql` - restart lại daemon để nhận cấu hình mới.

### 2.3. Tạo user replication

#### Trên db1 :
```
sudo mysql -u root -p
CREATE USER 'replica'@'10.0.20.5' IDENTIFIED WITH mysql_native_password BY 'p@ssPASS1';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.5';
CREATE USER 'replica'@'10.0.20.6' IDENTIFIED WITH mysql_native_password BY 'p@ssPASS1';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.6';
FLUSH PRIVILEGES;
exit;
```

#### Trên db2 :
```
sudo mysql -u root -p
CREATE USER 'replica'@'10.0.20.4' IDENTIFIED WITH mysql_native_password BY 'p@ssPASS1';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.4';
CREATE USER 'replica'@'10.0.20.6' IDENTIFIED WITH mysql_native_password BY 'p@ssPASS1';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.6';
FLUSH PRIVILEGES;
exit;
```
#### Trên db3 : 
```
sudo mysql -u root -p
CREATE USER 'replica'@'10.0.20.5' IDENTIFIED WITH mysql_native_password BY 'p@ssPASS1';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.5';
CREATE USER 'replica'@'10.0.20.4' IDENTIFIED WITH mysql_native_password BY 'p@ssPASS1';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.4';
FLUSH PRIVILEGES;
exit;
```
### 2.4. Config Replication
- Ở step này, ta sẽ cấu hình như sau : 
 	- db1 slave db3
 	- db2 slave db1
 	- db3 slave db2

#### lấy trạng thái master trên db3 : 
```
sudo mysql -u root -p
show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |     156  |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

```
#### Cấu hình slave trên db1 : 
```
sudo mysql -u root -p

CHANGE MASTER TO MASTER_HOST='10.0.20.6',
MASTER_USER='replica', 
MASTER_PASSWORD='p@ssPASS1', 
MASTER_LOG_FILE='mysql-bin.000003', 
MASTER_LOG_POS=156;
```
#### Cấu hình slave trên db2 :
- check first file trên db1 : 
```
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      156 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
- cấu hình : 
```
CHANGE MASTER TO MASTER_HOST='10.0.20.4',
MASTER_USER='replica', 
MASTER_PASSWORD='p@ssPASS1', 
MASTER_LOG_FILE='mysql-bin.000002', 
MASTER_LOG_POS=156;
```

#### Cấu hình slave trên db3 : 
- check first file trên db2 ( tương tự như ở trên ) 
- cấu hình 
```
CHANGE MASTER TO MASTER_HOST='10.0.20.5',
MASTER_USER='replica', 
MASTER_PASSWORD='p@ssPASS1', 
MASTER_LOG_FILE='mysql-bin.000002', 
MASTER_LOG_POS=156;
```
#### Start Slave và kiểm tra lỗi :
- Làm lần lượt từ db1->3 : 
```
*mysql shell nhé* 
Start slave;
SHOW SLAVE STATUS\G;
```
### 2.5. Test Replication :
- Cách test cũng tương tự bài trước, có điều ở đây có thể tạo db mới, table mới trên bất kỳ server nào, các server con lại sẽ auto replicate lại.
- Ở đây mình không set `binlog_do_DB` nên toàn bộ các DB của 3 node sẽ được auto đồng bộ với nhau.










