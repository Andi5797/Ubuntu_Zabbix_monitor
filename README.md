# Ubuntu_Zabbix_monitor

Mô hình:

3 node DB: MariaDB Galera Cluster + 1 VIP (Pacemaker/Corosync).

2 node Zabbix Server + Frontend (Nginx + PHP-FPM), HA native (HANodeName/NodeAddress).

Agent cài riêng trên host cần monitor.

🟢 0. Quy hoạch (ví dụ)
Node	IP	Hostname
DB1	172.16.32.11	zabbix-db-01
DB2	172.16.32.12	zabbix-db-02
DB3	172.16.32.13	zabbix-db-03
DB VIP	172.16.32.20	zabbix-db-vip
ZBX1	172.16.32.14	zabbix-srv-01
ZBX2	172.16.32.15	zabbix-srv-02
🟢 1. Chuẩn bị (trên tất cả 5 server)
sudo timedatectl set-timezone Asia/Bangkok
sudo apt update && sudo apt -y upgrade
sudo apt -y install chrony wget curl gnupg lsb-release net-tools
sudo systemctl enable --now chrony


Cập nhật /etc/hosts:

172.16.32.11  zabbix-db-01
172.16.32.12  zabbix-db-02
172.16.32.13  zabbix-db-03
172.16.32.20  zabbix-db-vip
172.16.32.14  zabbix-srv-01
172.16.32.15  zabbix-srv-02

🟢 2. MariaDB Galera Cluster (3 node DB)
2.1 Cài gói
sudo apt -y install mariadb-server galera-4 rsync

2.2 Cấu hình /etc/mysql/mariadb.conf.d/60-galera.cnf

Ví dụ DB1:

[galera]
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
wsrep_cluster_name="zabbix_cluster"
wsrep_cluster_address="gcomm://172.16.32.11,172.16.32.12,172.16.32.13"

wsrep_node_address="172.16.32.11"
wsrep_node_name="zabbix-db-01"

binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
wsrep_sst_method=rsync
bind-address=0.0.0.0


Trên DB2/DB3 chỉ sửa wsrep_node_address và wsrep_node_name.

2.3 Khởi tạo cluster

DB1:

sudo systemctl stop mariadb
sudo galera_new_cluster


DB2 & DB3:

sudo systemctl start mariadb


Kiểm tra:

mysql -uroot -e "SHOW STATUS LIKE 'wsrep_cluster_size';"

2.4 Tạo DB cho Zabbix (chạy 1 lần trên DB1)
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'%' IDENTIFIED BY 'MatKhauManh!123';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'%';
FLUSH PRIVILEGES;
SET GLOBAL log_bin_trust_function_creators=1;

🟢 3. Tạo VIP cho DB bằng Pacemaker/Corosync

Cài trên cả 3 node DB:

sudo apt -y install pacemaker corosync pcs
sudo systemctl enable --now pcsd
sudo passwd hacluster


Auth + setup cluster (trên DB1):

pcs host auth zabbix-db-01 zabbix-db-02 zabbix-db-03 -u hacluster -p <matkhau>
pcs cluster setup --name zabbix_db_cluster zabbix-db-01 zabbix-db-02 zabbix-db-03 --force
pcs cluster start --all
pcs cluster enable --all
pcs property set no-quorum-policy=ignore
pcs property set stonith-enabled=false


Tạo resource VIP (ví dụ NIC ens160):

pcs resource create db_vip ocf:heartbeat:IPaddr2 ip=172.16.32.20 cidr_netmask=24 nic=ens160 op monitor interval=5s
pcs status

🟢 4. Zabbix Server + Frontend (2 node, dùng Nginx)
4.1 Cài repo + gói
wget https://repo.zabbix.com/zabbix/7.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb
sudo apt update

sudo apt -y install zabbix-server-mysql zabbix-frontend-php \
                    zabbix-nginx-conf php8.3-fpm \
                    zabbix-sql-scripts zabbix-agent2

4.2 Cấu hình DB trong /etc/zabbix/zabbix_server.conf

Ví dụ trên ZBX1:

DBHost=172.16.32.20
DBName=zabbix
DBUser=zabbix
DBPassword=MatKhauManh!123
DBPort=3306

HANodeName=zabbix-srv-01
NodeAddress=172.16.32.14:10051


Trên ZBX2 chỉ đổi HANodeName=zabbix-srv-02, NodeAddress=172.16.32.15:10051.

4.3 Import schema (chỉ chạy 1 lần trên ZBX1)
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
  mysql --default-character-set=utf8mb4 -h 172.16.32.20 -uzabbix -p zabbix


Xong thì:

mysql -h 172.16.32.20 -uroot -p -e "SET GLOBAL log_bin_trust_function_creators=0;"

4.4 Cấu hình Nginx cho frontend

File: /etc/zabbix/nginx.conf

server {
    listen          80;
    server_name     zabbix-srv-01;   # hoặc VIP nếu muốn load balancer
    root            /usr/share/zabbix;

    index index.php;
    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}


→ enable bằng cách symlink:

sudo ln -s /etc/zabbix/nginx.conf /etc/nginx/sites-enabled/zabbix.conf
sudo nginx -t
sudo systemctl restart nginx php8.3-fpm

4.5 Bật dịch vụ
sudo systemctl enable --now zabbix-server zabbix-agent2

🟢 5. Truy cập & hoàn tất wizard

Mở:

http://172.16.32.14/ hoặc

http://172.16.32.15/

Đăng nhập: Admin / zabbix, đổi mật khẩu.

🟢 6. Agent trên host
sudo apt -y install zabbix-agent2
sudo sed -i 's/^Server=.*/Server=172.16.32.14,172.16.32.15/' /etc/zabbix/zabbix_agent2.conf
sudo sed -i 's/^ServerActive=.*/ServerActive=172.16.32.14,172.16.32.15/' /etc/zabbix/zabbix_agent2.conf
sudo sed -i 's/^Hostname=.*/Hostname=<TênHost>/' /etc/zabbix/zabbix_agent2.conf
sudo systemctl enable --now zabbix-agent2


✅ Vậy là xong:

DB Galera 3 node với 1 VIP cluster.

Zabbix 2 node chạy HA native, frontend bằng Nginx.
