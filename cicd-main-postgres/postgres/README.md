Setup Postgresql, Pgpool2 on Ubuntu

Phần này sẽ trình bày cách cài đặt, cấu hình Streaming replication bằng Pgpool2. Trong ví dụ này, chúng ta sẽ sử dụng 3 máy chủ cài Pgpool2 để quản lý các máy chủ PostgreSQL nhằm tạo ra một cụm có tính sẵn sàng cao
Mô hình triển khai
Sử dụng 3 máy chỉ với OS Ubuntu và hostname của 3 máy chủ lần lượt là `dev-postgres1`, `dev-postgres2` và `dev-postgres3`. Đồng thời cài đặt PostgreSQL và Pgpool2 trên mỗi máy chủ
![](images/cluster_architect.png)
**Note:** Các vai trò `Leader`, `Standby`, `Primary` ở mô hình trên không có định và có thể thay đổi dựa trên các thao tác với cụm
Hostname and IP address

|Hostname|IP Address|Virtual IP|
|---|---|---|
|dev-postgres1|192.168.11.45|192.168.11.48|
|dev-postgres2|192.168.11.46|192.168.11.48|
|dev-postgres3|192.168.11.47|192.168.11.48|

### PostgreSQL version and Configuration

|Item|Value|Detail|
|---|---|---|
|PostgreSQL Version|15.8|-|
|port|5432|-|
|$PGDATA|/data/postgresql/main/|-|
|Archive mode|on|/data/postgresql/archivedir|
|Replication Slots|Enabled|Trong ví dụ này các Replication Slot được tự động tạo/xóa trong các script được thực thi trong quá trình `failover` hoặc `online recovery`. Xem phần dưới để biết thêm chi tiết về các tập lệnh|
|Async/Sync Replication|Async|-|

### Pgpool2 version and Configuration

|Item|Value|Detail|
|---|---|---|
|Pgpool2 Version|4.5.2|-|
|port1|9999|Pgpool2 accepts connections|
|port2|9898|Pcp process accepts connections|
|port3|9000|Watchdog accept connections|
|port4|9694|UDP port receiving Watchdog's hearbeat signal|
|Config file|/etc/pgpool2/pgpool.conf|Pgpool2 config file|
|Running mode|Streaming replication mode|-|
|Watchdog|on|Life check method: heartbeat|

### Scripts

|Feature|Script|Detail|
|---|---|---|
|Failover|[/etc/pgpool2/failover.sh](../postgres/config/failover.sh)|Chạy bởi `failover_command` để thực hiện chuyển đổi dự phòng|
|Follow Primary|[/etc/pgpool2/follow_primary.sh](../postgres/config/follow_primary.sh)|Chạy bởi `follow_primary_command` để đồng bộ hóa node `Standby` với node `Primary` mới sau khi thực hiện Failover|
|Online recovery|[/data/postgresql/main/recovery_1st_stage](../postgres/config/recovery_1st_stage)|Chạy bởi `recovery_1st_stage_command` để khôi phục Standby node|
|Remote Start|[/data/postgresql/main/pgpool_remote_start](../postgres/config/pgpool_remote_start)|Chạy **sau** `recovery_1st_stage_command` để khởi động node Standby node|
|Watchdog|[/etc/pgpool2/escalation.sh](../postgres/config/escalation.sh)|Cấu hình tùy chọn. Chạy bởi `wd_escalation_command` để chuyển đổi Leader/Standby Pgpool2 một cách an toàn.
I. Setup PostgreSQL 15.8.
1. Thực hiện cài đặt PostgreSQL 15 trên toàn bộ 3 node `dev-postgres1`, `dev-postgres2` và `dev-postgres3`.
Automated repository configuration:
```
[all server]# sudo apt install -y postgresql-common
[all server]# sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh

[all server]# sudo apt install curl ca-certificates
[all server]# sudo install -d /usr/share/postgresql-common/pgdg
[all server]# sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
[all server]# sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
[all server]# sudo apt update
[all server]# sudo apt -y install postgresql-15
```

2. Turning Kernel.
Tạo tệp `/etc/sysctl.d/90-postgresql.conf` trên tất cả các server
Thêm các cấu hình sau vào tệp `/etc/sysctl.d/90-postgresql.conf`:
```
# kernel tuning parameters for postgresql
# shared memory
kernel.shmmax=4831838208
kernel.shmall=4831838208

# swappiness
vm.swappiness=1

# memory overcommit
vm.overcommit_memory=2
vm.overcommit_ratio=50

# memory dirty pages
vm.dirty_background_ratio=15
vm.dirty_ratio=20

# backlog
net.core.somaxconn = 6553
```
Thực hiện áp dụng cấu hình:
```
[all servers]# sysctl -w --system
```
3. Mở kết nối giữa các cụm.
Thêm các rule sau vào iptables
```
[all server]# iptables -A INPUT -m iprange --src-range 192.168.11.45-192.168.11.48 -m comment --comment "postgresql" -j ACCEPT

[all server]# iptables -A OUTPUT -m iprange --dst-range 192.168.11.45-192.168.11.48 -m comment --comment "postgresql" -j ACCEPT
```

4. Drop PostgreSQL default.

Khi cài PostgreSQL từ package trên Ubuntu, mặc định service PostgreSQL sẽ được init và start luôn và nó sẽ chiếm port 5432, nếu muốn custom ta cần stop và xóa dữ liệu mặc định của nó đi
```
[all server]# sudo systemctl stop postgresql
[all server]# sudo rm -rf /var/lib/postgresql/15/main
```
Trước khi drop, ta cần show PostgreSQL cluster default bằng lệnh sau:
```
[all server]# pg_lsclusters
```
Sử dụng output của lệnh `pg_lsclusters` để drop PostgreSQL default
```
[all server]# pg_dropcluster 15 main
```
5. Thực hiện tạo các folder cần thiết.
Đường dẫn chứa dữ liệu:
```
[all server]# mkdir -p /data/postgresql/main
```
Đường dẫn lưu file wal:
```
[all server]# mkdir -p /data/postgresql/archivedir
```
Phần quyền:
```
[all server]# chown -R postgres. /data/postgresql
[all server]# chmod -R 700 /data/postgresql/main
```
Đường dẫn lưu log:
```
[all server]# mkdir -p /var/log/postgresql
[all server]# chown -R postgres. /var/log/postgresql
[all server]# mkdir -p /var/log/pgpool_log
[all server]# chown -R postgres. /var/log/pgpool_log
```

6. Init và start PosgreSQL database (Trên 1 node).
Khởi tạo cơ sở dữ liệu trên node `dev-postgres1`
```
[dev-postgres1]# su - postgres
[dev-postgres1]# /usr/lib/postgresql/15/bin/initdb -D /data/postgresql/main
```
Khởi động PostgreSQL:
```
[dev-postgres1]# su - postgres
[dev-postgres1]# /usr/lib/postgresql/15/bin/pg_ctl start -D /data/postgresql/main
```
7. Set password cho user `postgres` nếu cần thiết.
```
[all server]# passwd postgres
```
II. Setup Pgpool2.
1. Install Pgpool2 Package.
Cài đặt các package cần thiết 
```
[all server]# sudo apt install pgpool2 -y
[all server]# sudo apt-get install postgresql-15-pgpool2 -y
[all server]# sudo apt install arping -y
```
Trên Ubuntu, sau khi cài đặt Pgpool2, service sẽ được restart với cấu hình default, ta cần stop nó đi và tự custom config sau này
```
[all server]# sudo systemctl stop pgpool2
```
2. Cấu hình cụm PostgreSQL.
Thực hiện cấu hình postgresql trong tệp `/data/postgresql/main/postgresql.conf` với nội dung như sau
```
[dev-postgres1]# vi /data/postgresql/main/postgresql.conf
```
[/data/postgresqsl/main/postgresql.conf](../postgres/config/postgresql.conf)

3. Tạo các user cần thiết. 
Vì lý do bảo mật, ta cần tạo user `repl` chỉ được sử dụng cho mục đích replication, user `pgpool` cho mục đích check độ trễ Streaming Replication và health check, user `postgres` được sử dụng để `Online recovery`

**Users**

|User Name|Password|Detail|
|---|---|---|
|repl|repl|PostgreSQL replication user|
|pgpool|pgpool|Pgpool2 healcheck and replication delay check user|
|postgres|postgres|User running online recovery|

Tiến hành tạo các user trong database, thêm quyền giám sát cho user `pgpool` nhằm quan sát trạng thái `replication_state` và `replication_sync_state`

Thực hiện trên node `dev-postgres1`:
```
[dev-postgres1]# psql -U postgres -p 5432
postgres=# SET password_encryption = 'scram-sha-256';
postgres=# CREATE ROLE pgpool WITH LOGIN;
postgres=# CREATE ROLE repl WITH REPLICATION LOGIN;
postgres=# \password pgpool
postgres=# \password repl
postgres=# \password postgres
postgres=# GRANT pg_monitor TO pgpool;
```
> Note: Lưu ý, lưu lại password điền vào file `/var/lib/postgresql/.pgpass` tại bước 6.
4. Cấu hình HBA PostgreSQL.

```
[dev-postgres1]# vi /data/postgresql/main/pg_hba.conf
```

Cấu hình tập tin `/data/postgresql/main/pg_hba.conf` để mở kết nối từ client đến database server, sử dụng giải thuật `scram-sha-256` để xác thực. Thực hiện trên tất cả các node cài PostgreSQL
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust
host    all             all             192.168.11.0/24         scram-sha-256
host    replication     all             192.168.11.0/24         scram-sha-256
```
> Note: Lưu ý, thay đổi IP và subnet cho phù hợp

5. Thiết lập SSH public key authentication.
Sửa config Service SSH để cho phép sử dụng SSH thông qua Public Key. Bỏ comment dòng dưới trong file `/etc/ssh/sshd_config`
```
[all server]# vi /etc/ssh/sshd_config
```
```
PubkeyAuthentication yes
```
Và thêm dòng sau:
```
AllowUsers postgres
```
Sau khi sửa cấu hình SSH, restart service để cập nhật cấu hình mới
```
sudo systemctl restart sshd
```
Để sử dụng tính năng Automated failver và Online recovery của Pgpool cần phải cấu hình SSH xác thực thông qua SSH public key trên tất cả server, từ user `root` tới user `postgres`, từ user `postgres` tới user `postgres`. 
Thực hiện lệnh sau trên tất cả máy chủ `dev-postgres1`, `dev-postgres2`, `dev-postgres3` để thiết lập passwordless SSH, tạo key file có tên là `id_rsa_pgpool`
```
[all server]# mkdir ~/.ssh
[all server]# chmod 700 ~/.ssh
[all server]# cd ~/.ssh
[all server]# ssh-keygen -t rsa -f id_rsa_pgpool
[all server]# ssh-copy-id -i id_rsa_pgpool.pub postgres@192.168.11.45
[all server]# ssh-copy-id -i id_rsa_pgpool.pub postgres@192.168.11.46
[all server]# ssh-copy-id -i id_rsa_pgpool.pub postgres@192.168.11.47

[all server]# su - postgres
[all server]# mkdir ~/.ssh
[all server]# chmod 700 ~/.ssh
[all server]# cd ~/.ssh
[all server]# ssh-keygen -t rsa -f id_rsa_pgpool
[all server]# ssh-copy-id -i id_rsa_pgpool.pub postgres@192.168.11.45
[all server]# ssh-copy-id -i id_rsa_pgpool.pub postgres@192.168.11.46
[all server]# ssh-copy-id -i id_rsa_pgpool.pub postgres@192.168.11.47
```
6. Tạo tập tin `/var/lib/postgresql/.pgpass`.
Để cho phép user `repl`, `postgres` thực hiện Streaming replication, Online Recovery, ta cần tạo tệp tin `.pgpass` trong và thay đổi quyền 600 trên tất cả server
```
[all server]# su - postgres
[all server]$ vi /var/lib/postgresql/.pgpass
192.168.11.45:5432:replication:repl:S3tplpVQuwqbUnm
192.168.11.46:5432:replication:repl:S3tplpVQuwqbUnm
192.168.11.47:5432:replication:repl:S3tplpVQuwqbUnm
192.168.11.45:5432:postgres:postgres:S3tplpVQuwqbUnm
192.168.11.46:5432:postgres:postgres:S3tplpVQuwqbUnm
192.168.11.47:5432:postgres:postgres:S3tplpVQuwqbUnm
dev-postgres1:5432:replication:repl:S3tplpVQuwqbUnm
dev-postgres2:5432:replication:repl:S3tplpVQuwqbUnm
dev-postgres3:5432:replication:repl:S3tplpVQuwqbUnm
dev-postgres1:5432:postgres:postgres:S3tplpVQuwqbUnm
dev-postgres2:5432:postgres:postgres:S3tplpVQuwqbUnm
dev-postgres3:5432:postgres:postgres:S3tplpVQuwqbUnm

[all server]$ chmod 600 /var/lib/postgresql/.pgpass
```
7. Tạo `pgpool_node_id` định danh cho các server.

Cấu hình Pgpool2

Thực hiện trên tất cả các server, copy file cấu hình mẫu
```
File `pgpool.conf`: [/etc/pgpool2/pgpool.conf](../configs/pgpool.conf)
```
Cấu hình node id
```
[dev-postgres1]# echo 0 > /etc/pgpool2/pgpool_node_id
[dev-postgres2]# echo 1 > /etc/pgpool2/pgpool_node_id
[dev-postgres3]# echo 2 > /etc/pgpool2/pgpool_node_id
```
8. Copy script mẫu vào thư mục `/etc/pgpool2/` và phân quyền.

File `failover.sh`: [/etc/pgpool2/failover.sh](../postgres/config/failover.sh)

File `follow_primary.sh`: [/etc/pgpool2/follow_primary.sh](../postgres/config/follow_primary.sh)

Sau khi copy file 2 script `failover.sh` và `follow_primary.sh` vào đường dẫn `/etc/pgpool2`, ta cần phân quyền cho 2 script này
```
[all server]# chown -R postgres. /etc/pgpool2/{failover.sh,follow_primary.sh}
[all server]# chmod +x /etc/pgpool2/{failover.sh,follow_primary.sh}
```
9. Mã hóa password của user `pgpool` bằng tool `pg_md5`, tạo file `/var/lib/pgsql/.pcppass`.
Mã hóa password của user `pgpool` bằng tool `pg_md5` và thêm vào cuối file `/etc/pgpool-II/pcp.conf`
```
[all server]# echo 'pgpool:'`pg_md5 S3tplpVQuwqbUnm` >> /etc/pgpool2/pcp.conf
```
Tạo file `/var/lib/pgsql/.pcppass` và phân quyền
```
[all servers]# su - postgres
[all servers]$ echo 'localhost:9798:pgpool:S3tplpVQuwqbUnm' > ~/.pcppass
[all servers]$ chmod 600 ~/.pcppass
```
10. Cấu hình tính năng phục hồi trực tuyến (Online recovery).
Copy script mẫu `pgpool_remote_start` và `recovery_1st_stage` vào thư mục `/data/postgresql/main` trên node 1 (`dev-postgres1`)

File `pgpool_remote_start`: [/data/postgresql/main/pgpool_remote_start](../postgres/config/pgpool_remote_start)

File `recovery_1st_stage`: [/data/postgresql/main/recovery_1st_stage](../postgres/config/recovery_1st_stage)

Sau khi copy file 2 script `pgpool_remote_start` và `recovery_1st_stage` vào đường dẫn `/data/postgresql/main`, ta cần phân quyền cho 2 script này
```
[dev-postgres1]# chown -R postgres. /data/postgresql/main/{pgpool_remote_start,recovery_1st_stage}
[dev-postgres1]# chmod +x /data/postgresql/main/{pgpool_remote_start,recovery_1st_stage}
```
Để sử dụng tính năng khôi phục trực tuyến (Online recovery), ta cần cài đặt extension `pgpool_recovery` trên database `template1` trên node 1 (`dev-postgres1`)
```
[dev-postgres1]# su - postgres
[dev-postgres1]$ psql template1 -c "CREATE EXTENSION pgpool_recovery"
```
11. Cấu hình HBA cho Pgpool2.
Cấu hình file `/etc/pgpool2/pool_hba.conf` trên tất cả các server

File `pool_hba.conf`: [/etc/pgpool2/pool_hba.conf](../postgres/config/pool_hba.conf)

12. Tạo file `/var/lib/postgresql/.pgpoolkey` trên tất cả server và Encrypt password.
Tạo file `/var/lib/postgresql/.pgpoolkey` trên tất cả server 
```
[all server]# su - postgres
[all server]$ echo 'Cntt@123' > ~/.pgpoolkey
[all server]$ chmod 600 ~/.pgpoolkey
```
Thực hiện Encrypt password cho 2 user `postgres` và `pgpool` để ghi username và password dạng AES vào file `/etc/pgpool2/pool_password` trên tất cả các server
```
[all server]# su - postgres
[all server]$ pg_enc -m -k ~/.pgpoolkey -u pgpool -p
db password: <Nhập pass của pgpool>
[all server]$ pg_enc -m -k ~/.pgpoolkey -u postgres -p
db password: <Nhập pass của postgres>

[all server]# chmod 600 ~/.pgpoolkey
```
Kiểm tra xem pass đã được ghi chưa:
```
[all servers]# cat /etc/pgpool-II/pool_passwd
pgpool:AES57ZBWfJZDB4DaAZiiBT3Ydesl91tkv/SNl6qNrYBqI8=
postgres:AES57ZBWfJZDB4DaAZiiBT3Ydesl91tkv/SNl6qNrYBqI8=
```
13. Cấu hình watchdog.
Thêm các quyền sau cho user `postgres` trong visudo
```
postgres ALL=NOPASSWD: /sbin/ip
postgres ALL=NOPASSWD: /usr/sbin/arping
```
```
chown -R postgres. /etc/pgpool2/escalation.sh  && chmod +x /etc/pgpool2/escalation.sh
```

Copy script mẫu `escalation.sh` vào đường dẫn `/etc/pgpool2/`. Thực hiện trên tất cả server

File `escalation.sh`: [/etc/pgpool2/escalation.sh](../postgres/config/escalation.sh)

Sửa nội dung trong tệp `/etc/pgpool2/escalation.sh`
```
[all servers]# vi /etc/pgpool2/escalation.sh
...
PGPOOLS=(dev-postgres1 dev-postgres2 dev-postgres3)
VIP=192.168.10.48
DEVICE=ens160
```

14. Start pgpool.

Khởi động lại Primary Database (Thao tác trên node `dev-postgres1`)
```
[dev-postgres1]# su - postgres
[dev-postgres1]$ /usr/lib/postgresql/15/bin/pg_ctl stop -D /data/postgresql/main/
[dev-postgres1]$ /usr/lib/postgresql/15/bin/pg_ctl start -D /data/postgresql/main/
```
Khởi động Pgpool2 trên cả 3 server:
```
[all servers]# systemctl start pgpool
[all servers]# systemctl enable pgpool
```
15. Online Recovery cho các node Standby.
```
[dev-postgres1]# pcp_recovery_node -h 192.168.11.48 -p 9798 -U pgpool -n 1 -W
Password: 
pcp_recovery_node -- Command Successful

[dev-postgres1]# pcp_recovery_node -h 192.168.11.48 -p 9798 -U pgpool -n 2 -W
Password: 
pcp_recovery_node -- Command Successful
```
Kiểm tra trạng thái của cụm sau khi Online Recovery:
```
[dev-postgres1]# psql -U postgres -p 9999  -c "show pool_nodes"
```

