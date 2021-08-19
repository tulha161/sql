# MySQL Cluster DB 

## 1. Mô hình Replication Master - Master 
- Ở bài này, ta vẫn sẽ sử dụng kỹ thuật mysql replication đã được đề cập ở lần trước.  Tuy nhiên, mô hình sử dụng sẽ là mô hình master - master với 3 node server.
- Mô hình hoạt động master - master là khi toàn bộ data được lưu trên 1 nhóm server ( cluster ), được cập nhật  ở bất kỳ thành phần nào trên nhóm. Tất cả Master server có nhiệm vụ phản hồi lại các truy vấn dữ liệu của người dùng và đồng bộ, cập nhật những thay đổi về dữ liệu từ các Master server khác trong nhóm.

### Mô hình thực hiện  :
- OS : Ubuntu 18.04 LTS 
- DB4 : 10.0.20.14 
- DB5 : 10.0.20.15
- DB6 : 10.0.20.16


## 2. Cấu hình Replication Cluster : 

### 2.1. Cài đặt và khởi động MySQL trên cả 3 node : 
```
sudo apt update
sudo apt -y install mysql-server mysql-client
sudo mysql_secure_installation
```

### 2.2. Chỉnh sửa file config MySQL
- chỉnh sửa file config tại `sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf` 

#### Tại db4 : 
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
bind-address            = 10.0.20.14
```

#### Tại db5 : 
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
bind-address            = 10.0.20.15
```
#### Tại db6 : 
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
bind-address            = 10.0.20.16
```

- `systemctl restart mysql` - restart lại daemon để nhận cấu hình mới.

### 2.3. Tạo user replication

#### Trên db4 :
```
sudo mysql -u root -p
CREATE USER 'replica'@'10.0.20.15' IDENTIFIED BY 'p@sstoWord';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.15';
CREATE USER 'replica'@'10.0.20.16' IDENTIFIED BY 'p@sstoWord';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.16';
FLUSH PRIVILEGES;
exit;
```

#### Trên db5 :
```
sudo mysql -u root -p
CREATE USER 'replica'@'10.0.20.14' IDENTIFIED BY 'p@sstoWord';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.14';
CREATE USER 'replica'@'10.0.20.16' IDENTIFIED BY 'p@sstoWord';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.16';
FLUSH PRIVILEGES;
exit;
```
#### Trên db6 : 
```
sudo mysql -u root -p
CREATE USER 'replica'@'10.0.20.15' IDENTIFIED BY 'p@sstoWord';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.15';
CREATE USER 'replica'@'10.0.20.14' IDENTIFIED BY 'p@sstoWord';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.20.14';
FLUSH PRIVILEGES;
exit;
```
p
- Có thể test kết nối từ xa đến các remote db với cmd : 
`mysql -u [user] -p -h [remoteip] -P 3306`
p@sstoWord
### 2.4. Config Replication
- Ở step này, ta sẽ cấu hình như sau : 
 	- db4 slave db6
 	- db5 slave db4
 	- db6 slave db5

#### lấy trạng thái master trên db3 : 
```
sudo mysql -u root -p
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |     1261 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.01 sec)



```
#### Cấu hình slave trên db4 : 
```
sudo mysql -u root -p

CHANGE MASTER TO MASTER_HOST='10.0.20.16',
MASTER_USER='replica', 
MASTER_PASSWORD='p@sstoWord', 
MASTER_LOG_FILE='mysql-bin.000001', 
MASTER_LOG_POS=1261;
```
#### Cấu hình slave trên db5 :
- cấu hình : 
```
CHANGE MASTER TO MASTER_HOST='10.0.20.14',
MASTER_USER='replica', 
MASTER_PASSWORD='p@sstoWord', 
MASTER_LOG_FILE='mysql-bin.000001', 
MASTER_LOG_POS=1261;
```

#### Cấu hình slave trên db6 : 
- cấu hình :
```
CHANGE MASTER TO MASTER_HOST='10.0.20.15',
MASTER_USER='replica', 
MASTER_PASSWORD='p@sstoWord', 
MASTER_LOG_FILE='mysql-bin.000001', 
MASTER_LOG_POS=1261;
```
#### Start Slave và kiểm tra lỗi :
- Làm lần lượt từ db4->6 : 
```
*mysql shell nhé* 
Start slave;
SHOW SLAVE STATUS\G;
```
### 2.5. Test Replication :
- Cách test cũng tương tự bài trước, có điều ở đây có thể tạo db mới, table mới trên bất kỳ server nào, các server con lại sẽ auto replicate lại.
- Ở đây mình không set `binlog_do_DB` nên toàn bộ các DB của 3 node sẽ được auto đồng bộ với nhau.


## 3. Cấu hình HAProxy và Heartbeat
- Như vậy là dựng cluster MySQL đã xong, các db đã được đồng bộ thành 1 cụm đúng như mong muốn, nhưng ta cần có một hệ thống có tính đáp ứng cao, khả năng cân bằng tải tốt. Vì vậy, mình sẽ tiến hành cài đặt thêm **HAProxy và Heartbeat**
### 3.1. Cài đặt, cấu hình HAProxy : 

#### Cài đặt HAProxy : 
- Triển khai trên cả 3 server : `sudo apt -y install haproxy`

#### Chỉnh sửa file config :
- Tại phần này, HAProxy hoạt động khi các node được gán `virtual IP` chia sẻ chung với nhau, ở đây mình set virtual IP = `10.0.20.20` port 7000 
- edit file `nano /etc/haproxy/haproxy.cfg`
- Nội dung : 
```
listen stats

    bind 10.0.20.20:7000
    
    stats enable

    stats hide-version

    stats uri /
    
    stats auth admin:P@ssw0rd

listen mysql-cluster

    bind 10.0.20.20:3306
    
    mode tcp
    option tcplog
    option mysql-check user haproxy_check

    balance roundrobin

    server db4 10.0.20.14:3306 check
    server db5 10.0.20.15:3306 check
    server db6 10.0.20.16:3306 check
```
- Làm tương tự ở db2 và db3. 

#### Tạo user Mysql cho haproxy
- Tạo user `haproxy_check` không password để HAProxy check health status các node.
```
sudo mysql -u root -p
create user 'haproxy_check'@'%';
flush privileges;
```
- Tạo user `haproxy_root` có password và gán quyền cho user này để test load balacing.
```
create user 'haproxy_root'@'%' identified by '123.abc';
grant all privileges on *.* to 'haproxy_root'@'%';
```
- Kiểm tra trên 3 db xem đã đầy đủ các user đã tạo chưa, do 3 db replicate lẫn nhau nên chỉ cần thao tác `create user` tại một con là được. 
```
mysql> select user, Host FROM mysql.user;
+------------------+-----------+
| user             | Host      |
+------------------+-----------+
| haproxy_check    | %         |
| haproxy_root     | %         |
| replica          | 10.0.20.14|
| replica          | 10.0.20.15|
| debian-sys-maint | localhost |
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
+------------------+-----------+
9 rows in set (0.00 sec)
```

#### Chỉnh sửa config để các node apply virtual IP
- Edit file `/etc/sysctl.conf` trên 3 db, add thêm dòng sau : 
```
net.ipv4.ip_nonlocal_bind=1
```
- Apply config : `sudo sysctl -p`
- Kiểm tra hoạt động của haproxy : `haproxy -f /etc/haproxy/haproxy.cfg -db`
- Restart dịch vụ : `sudo systemctl restart haproxy`
- Kiểm tra haproxy listening stat trên 3 node : `sudo netstat -ntlp`

<img src="https://github.com/tulha161/sql/blob/main/pic/2.1.png"> 

### 3.2. Cài đặt và config Heartbeat
- Ở step này, chúng ta sẽ cài đặt và config **Heartbeat** trên mỗi node để làm IP ảo hoạt động. Nếu có node chết, nó sẽ tự động chuyển hoạt động sang node còn lại đang sống. 

#### Cài đặt trên 3 node : 
- cài heartbeat : `sudo apt -y install heartbeat`
- tạo file password mới tại : `/etc/ha.d/authkeys`
```
auth 1
1 md5 123.abc
```
- cấp quyền 600 cho file mới này : `sudo chmod 600 /etc/ha.d/authkeys`

- File `/etc/ha.d/authkeys` sẽ chứa thông tin để authen giữa các node .

#### Tạo file main config trên các node : 
- `ifconfig` để kiểm tra interface mạng, của mình là `ens33`.
- tạo file config `etc/ha.d/ha.cf` tại từng node : 
##### Tại db4 : 
```
keepalive 2
deadtime 10

udpport        694
bcast  ens33
mcast ens33 225.0.0.1 694 1 0
ucast ens33 10.0.20.15   
ucast ens33 10.0.20.16   
udp     ens33

logfacility     local0

node    db4      
node    db5      
node    db6
```

##### Tại db5 : 

```
keepalive 2
deadtime 10

udpport        694
bcast  ens33
mcast ens33 225.0.0.1 694 1 0
ucast ens33 10.0.20.14  
ucast ens33 10.0.20.16   
udp     ens33

logfacility     local0

node    db4     
node    db5     
node    db6
```

##### Tại db6 : 

```
keepalive 2
deadtime 10

udpport        694
bcast  ens33
mcast ens33 225.0.0.1 694 1 0
ucast ens33 10.0.20.14
ucast ens33 10.0.20.15
udp     ens33

logfacility     local0

node    db4
node    db5
node    db6
```

#### Tạo file haresources trên cả 3 node : 
- Tạo file `/etc/ha.d/haresources` chứa config về shared IP và master node : 
- Nội dung :
```
db4 10.0.20.20
```
- restart service tại các node : `sudo systemctl restart heartbeat`

- Show ip tại **db4** xem đã xuất hiện virtual IP chưa : 

<img src="https://github.com/tulha161/sql/blob/main/pic/2.2.png">


### 3.3. Kiểm tra hoạt động

- Như vậy, mình đã hoàn thành cài đặt MySQL Cluster Master - Master kết hợp với HAProxy/Heartbeat để loadblancing và quản lý failover. Sau bước này, chúng ta có thể truy cập vào `http://10.0.20.20:7000/` - địa chỉ virtual IP để xem Statics report bằng id và password đã config ở step 3.1

<img src="https://github.com/tulha161/sql/blob/main/pic/2.3.png">

- Kiểm tra truy vấn biến `server_id` bằng lệnh sau : 
`mysql -h 10.0.20.20 -u haproxy_root -p -e "show variables like 'server_id'"`

- Kết quả thu được : 

<img src="https://github.com/tulha161/sql/blob/main/pic/2.4.png">

- Nhận thấy kết quả trả về theo 4 lần truy vấn tuần tự là 1-2-3-1. Như vậy, ta có thể thấy các server đang hoạt động loadblancing theo kiểu robinround như đã config. 

- Giả định trường hợp die db4, kiểm tra hoạt động của loadbalancing : 

<img src="https://github.com/tulha161/sql/blob/main/pic/2.6.png">

<img src="https://github.com/tulha161/sql/blob/main/pic/2.5.png">

<img src="https://github.com/tulha161/sql/blob/main/pic/2.7.png">

- Có thể thấy master node và ip ảo giờ chuyển cho db5, db 5 và 6 trong cụm vẫn sẽ hoạt động loadbalancing robinround. 

- Tương tự, trong trường hợp db5 cũng chết, lúc này db6 sẽ đứng lên nhận nhiệm vụ là master node duy nhất gánh vác hoạt động của toàn cụm. Không nên để trường hợp này xảy ra :D 


## 4. Backup and Restore: 
- Mô hình cluster node với HA như ở trên đã phục vụ được mục đích backup lẫn nhau giữa các server trong cụm mà không phải thay đổi cứng bằng tay như Master - Slave. Tuy nhiên, giả định trường hợp toàn bộ node trong cụm vì lý do nào đó đều chết, khi đó ta sẽ phải thực hiện khôi phục lại bằng data đã được sao lưu lại từ trước, từ đó có thể phần nào khôi phục nhanh nhất hoạt động của cụm, giảm downtime xuống tối thiểu.
- Thao tác backup bằng mysqldump : `mysqldump -u [user] -p [database_name] > [file]`
	- Trong trường hợp backup toàn bộ DB dưới user root. ta có thể sử dụng : `mysqldump -u root -p --all-databases > /backup/mysql/alldatabase.sql`
- Thao tác để restore lại data khi đã có backup file: `mysql --one-database database_name < alldatabases.sql`
- Phần cốt lỗi của DB ta Databases, vì vậy việc backup databases là rất quan trọng, quản trị viên thực hiện nó
hàng ngày, thậm chí hàng giờ với những db tuyệt đối quan trọng. 

## 5. Monitoring với Icinga2 : 
- Cài đặt check command `check_mysql_health` trên server icinga2
- Tạo user `monitor` trên cụm 
- Setup theo bài tham khảo : https://icinga.com/blog/2017/06/12/how-to-monitor-your-mysql-servers-with-icinga-2

<img src="https://github.com/tulha161/sql/blob/main/pic/2.9.png">





















