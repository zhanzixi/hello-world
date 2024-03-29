etcd --data-dir=data.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}软件安装



## Docker

```shell
# 以下步骤参考官网

# Uninstall old versions
yum remove docker \
    docker-client \
    docker-client-latest \
    docker-common \
    docker-latest \
    docker-latest-logrotate \
    docker-logrotate \
    docker-engine
# Install using the repository
# SET UP THE REPOSITORY
yum install -y yum-utils bash-completion
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
# INSTALL DOCKER ENGINE
yum install docker-ce docker-ce-cli containerd.io
systemctl start docker
docker run hello-world

# 国内镜像
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "registry-mirrors": ["http://hub-mirror.c.163.com"]
}
EOF

# example mysql5.7
docker run --name mysql5.7 -e MYSQL_ROOT_PASSWORD=MyNewPass4! -d -p 3306:3306 mysql:5.7.31 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --max_connections=1000
```



## Docker Compose

```shell
curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose

docker-compose --version
```



## MySQL

```shell
# 参考官网

# Installing MySQL on Linux Using the MySQL Yum Repository
# 下载并上传mysql80-community-release-el7-3.noarch.rpm
yum localinstall mysql80-community-release-el7-3.noarch.rpm
yum repolist all | grep mysql
yum-config-manager --disable mysql80-community
yum-config-manager --enable mysql57-community
yum install mysql-community-server
service mysqld start
service mysqld status
grep 'temporary password' /var/log/mysqld.log
# smdEy/&vy9?D 
mysql -uroot -p
ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'MyNewPass4!' WITH GRANT OPTION;
# update mysql.user set host='%' where user='root'; flush privileges;

SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED
```

### 冷备份

```shell
# 停机
SET GLOBAL innodb_fast_shutdown=0
shutdown

# 拷贝整个/var/lib/mysql目录
scp -r /var/lib/mysql root@192.168.96.14:/root
# 改变用户
chown -R mysql:mysql mysql
cp -r mysql /var/lib/mysql
systemctl start mysqld
```

### 热备份

```shell
Group: mysql, Title: innobackupex 备份 8.0 的mysql, Password: ********, Creation Time: 2021/9/30 16:21:34, Last Modification Time: 2021/10/9 17:13:51

1.下载 innobackupex
https://downloads.percona.com/downloads/Percona-XtraBackup-LATEST/Percona-XtraBackup-8.0.26-18/binary/redhat/7/x86_64/percona-xtrabackup-80-8.0.26-18.1.el7.x86_64.rpm

2.安装 innobackupex
yum -y install percona-xtrabackup-80-8.0.26-18.1.el7.x86_64.rpm

安装 XtraBackup 中出现"libev.so.4()(64bit) is needed"问题
wget ftp://rpmfind.net/linux/atrpms/el6-x86_64/atrpms/stable/libev-4.04-2.el6.x86_64.rpm


rpm -ivh libev-4.04-2.el6.x86_64.rpm

3.备份
innobackupex --defaults-file=/etc/mysql/my-3466.cnf --user=root --password=M@3LRhnfKc6U%A --socket=/var/lib/mysql/mysql-3466.sock --no-timestamp --backup --target-dir=/data/tmp/3466

注：
--backup         指明是备份
--target-dir     备份目标目录

4. 恢复
## MySQL 5.6/5.7
innobackupex --defaults-file=/home/mysql/data/backup-my.cnf --apply-log /home/mysql/data

## MySQL 8.0
xtrabackup --prepare --target-dir=/data/tmp/3466

xtrabackup --datadir=/data/mysql-3466 --copy-back --target-dir=/data/tmp/3466
```

### 逻辑备份

```shell
mysqldump --databases --single-transaction --flush-logs --master-data=2 test1 -p > test1.sql
# master-data可能过时 source-data替换

scp test1.sql root@192.168.96.14:/root

mysql -uroot -p < test1.sql
```

### 时间点恢复

```mysql
show variables like '%log_bin%';
show variables like 'binlog_expire_logs_seconds';
SHOW BINARY LOGS;
SHOW MASTER STATUS;
# binlog.000004 |      156 

# binlog.000005 |     2049

# 查看binlog，找出需要恢复的时间段，需要跳过的时间段
mysqlbinlog /var/lib/mysql/binlog.000005 --base64-output=decode-rows -v
857
1425
--start-position=156 --stop-position=857
--start-position=1425 --stop-position=2049

mysqlbinlog /var/lib/mysql/binlog.000005 --start-position=156 --stop-position=857 | mysql -uroot -p
mysqlbinlog /var/lib/mysql/binlog.000005 --start-position=1425 --stop-position=2049 | mysql -uroot -p
```

### 复制

```mysql
# https://dev.mysql.com/doc/refman/8.0/en/replication-howto.html
# Setting the Replication Source Configuration
server_id=4
# Setting the Replica Configuration
server_id=14
# Creating a User for Replication
GRANT REPLICATION SLAVE ON *.* TO 'root'@'%';
#Obtaining the Replication Source Binary Log Coordinates
SHOW MASTER STATUS;
# binlog.000287 |      368

mysqldump --all-databases --single-transaction --flush-logs --master-data=2 -p > mysql4.sql

CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='192.168.96.4',
    SOURCE_USER='root',
    SOURCE_PASSWORD='MyNewPass4!',
    SOURCE_LOG_FILE='binlog.000288',
    SOURCE_LOG_POS=156;
    
START REPLICA;
SHOW REPLICA STATUS\G


################################
FLUSH TABLES WITH READ LOCK;
UNLOCK TABLES;
select * from mysql.slave_master_info;
select * from mysql.slave_relay_log_info;
```

### GTID复制

```mysql
# 配置文件source replica都配置如下
gtid_mode=ON
enforce-gtid-consistency=ON

CHANGE REPLICATION SOURCE TO
	SOURCE_HOST = '192.168.96.4',
    SOURCE_USER='root',
    SOURCE_PASSWORD='MyNewPass4!',
    SOURCE_AUTO_POSITION = 1;

```



## Redis

```shell
yum install -y gcc-c++ wget

wget http://download.redis.io/releases/redis-5.0.9.tar.gz
tar -zxf redis-5.0.9.tar.gz

cd redis-5.0.9
make

make install PREFIX=/usr/local/redis

cp redis.conf /usr/local/redis/bin/
# 修改/usr/local/redis/bin/redis.conf

## 服务化
cat > /etc/systemd/system/redis.service <<EOF
[Unit]
Description=redis service
After=network.target 
Wants=network.target
   
[Service] 
Type=forking 
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/bin/redis.conf
PrivateTmp=true 
   
[Install] 
WantedBy=multi-user.target
EOF
```



## Git

```shell
# yum安装
yum install -y git
yum erase -y git
# 如果需要安装指定版本，则从官网下载tar.gz安装，例如：
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.26.2.tar.gz
yum -y install curl-devel expat-devel gettext-devel openssl-devel zlib-devel 
yum -y install gcc-c++ perl-ExtUtils-MakeMaker
tar zxf git-2.26.2.tar.gz
cd git-2.26.2
./configure --prefix=/usr/local/git-2.26.2
make && make install
ln -s /usr/local/git-2.26.2/bin/* /usr/bin/

```



```shell
# Beginning Git and Github.pdf

# The main feature of Git is its “Three States” system. The states are the working directory, the staging area, and the git directory. So, a file goes from “modified” to “staged” to “committed.”  

git checkout v2.0  # or
git checkout master^^
# Notice that regardless of which checkout command we use, HEAD now refers directly to commit b. This is known as being in detached HEAD state. It means simply that HEAD refers to a specific commit, as opposed to referring to a named branch.

git config --global user.name "Mariot Tsitoara"
git config --global user.email "mariot.tsitoara@gmail.com"
# ~/.gitconfig ~/.bash_history

# , you can unstage a file using the Git command “git rm” with the option “--cached.”
git rm --cached README.md
git restore --staged <file>...

# In addition to the snapshot, a commit also contains information about the “author” of the content and the “commiter” or who put the changeset into the repository.

# Remember: staging concerns only changed files you choose, while committing concerns the entire project. You stage a file; then commit the project.

# To check a snapshot of the project, we use the “git checkout” command and pass the commit name as a parameter.
git checkout <name>
# Be careful not to change anything when checking out previous commits. Just like in the movies, changing the past is a very bad idea!

#  “head” is just a reference to a commit. Instead of saying “name,” when talking about commits, we say “head.”
# And that’s it! A head is a reference to a commit (there can be multiple heads in a repository), and the head pointing to the currently checked-out commit is called HEAD.
git checkout master



# Whatever changes committed must stay so, for the sake of history; changing what has happened in the past is very dangerous and counterintuitive. Instead, you will use git revert to create a new commit that contains the exact opposite of the commit you are trying to undo.
git revert <commit name>
git revert HEAD~3
git revert -n master~5..master~2

 git commit --amend
 
 git reset --soft HEAD~1
 git reset HEAD filenottocommit.txt
 
 # Be careful when you use the reset command. it’s very dangerous. make sure to double check what you write.
 
```







## GitLab

```shell
# recommend at least 4GB of free RAM 
# 参考官方文档
sudo yum install -y curl policycoreutils-python openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo systemctl reload firewalld

sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix

curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash # 注意ee和ce
sudo yum install gitlab-ce
# EXTERNAL_URL="https://gitlab.example.com" yum install -y gitlab-ee
# 根据提示编辑配置文件
vim /etc/gitlab/gitlab.rb # 修改external_url，也可选修改端口nginx['listen_port'] = 8888
sudo gitlab-ctl reconfigure
#sudo gitlab-ctl restart
# 第一次访问要设置root用户的密码minimum is 8 characters 19810422

# 配置邮箱服务
# 1.开通qq邮箱的smtp服务,得到授权码，第2步要用
# 2.修改gitlab配置 （搜索smtp_enable,git_user_email,gitlab_email_from）
# 3.测试邮件服务是否正常 gitlab-rails console;Notify.test_email('接收方','标题','邮件类容').deliver_now
```



## Java

```shell
# 上传jdk-8u261-linux-x64.tar.gz
tar xzf jdk-8u261-linux-x64.tar.gz -C /usr/local/
# 注意这里的<<\EOF
cat > /etc/profile.d/java.sh <<\EOF
#!/bin/bash
JAVA_HOME=/usr/local/jdk1.8.0_261
PATH=$JAVA_HOME/bin:$PATH
export JAVA_HOME PATH
EOF

source /etc/profile
```



## Hadoop

```shell
tar xzf hadoop-3.3.1.tar.gz -C /usr/local
# 注意这里的<<\EOF
cat > /etc/profile.d/hadoop.sh <<\EOF
#!/bin/bash
HADOOP_HOME=/usr/local/hadoop-3.3.1
PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
export HADOOP_HOME PATH
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_YARN_HOME=$HADOOP_HOME
EOF

source /etc/profile

# https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html
```





## Maven

```shell
wget https://mirrors.bfsu.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
tar zxvf apache-maven-3.6.3-bin.tar.gz -C /usr/local/

cat > /etc/profile.d/maven.sh <<\EOF
#!/bin/bash
M2_HOME=/usr/local/apache-maven-3.6.3
PATH=$M2_HOME/bin:$PATH
export M2_HOME PATH
EOF

source /etc/profile
```



## Sonatype Nexus

```shell
# 官方网站访问不了oss下载页面
```



##  Jenkins

```shell
# 安装tomcat，创建tomcat用户，将解压后的文件夹chown给tomcat
useradd tomcat
passwd tomcat
tar zxvf apache-tomcat-9.0.8 -C /usr/local/
chown -R tomcat:tomcat /usr/local/apache-tomcat-9.0.8

# 安装插件如果提示离线，jenkins/pluginManager/advanced 把https改成http
# 安装插件 Maven Integration / Sonarqube Scanner for Jenkins / Publish Over SSH / GitLab
# 配置JDK / Maven / Sonar
# 配置邮件
# 配置gitlab授权 / 配置免密登录

#####################
# 1. https://www.jenkins.io/download/ 下载jenkins.war
# 2. Unlocking Jenkins
# 3. 安装插件，如果报错添加相应的插件到/root/.jenkins/war/WEB-INF/detached-plugins，重启，如报错强制刷新页面

```



## Nginx

```shell
# https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/
yum -y install gcc gcc-c++ pcre-devel openssl-devel
wget http://nginx.org/download/nginx-1.20.1.tar.gz
tar -xf nginx-1.20.1.tar.gz
./configure --with-http_ssl_module
make && make install

# https://www.nginx.com/resources/wiki/start/topics/examples/systemd/

cat <<EOF | sudo tee /etc/systemd/system/nginx.service
[Unit]
Description=nginx service
After=network.target 
Wants=network.target
   
[Service] 
Type=forking 
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true 
   
[Install] 
WantedBy=multi-user.target
EOF

systemctl enable --now nginx
```



## Haproxy

```shell
wget https://www.haproxy.org/download/2.4/src/haproxy-2.4.0.tar.gz
tar -xf haproxy-2.4.0.tar.gz
make TARGET=linux-glibc PREFIX=/usr/local/haproxy
make install PREFIX=/usr/local/haproxy

mkdir /usr/local/haproxy/conf
cp examples/option-http_proxy.cfg /usr/local/haproxy/conf/haproxy.cfg
ln -s /usr/local/haproxy/sbin/haproxy /usr/sbin/haproxy

################################ 配置参考 ################################
# http://cbonte.github.io/haproxy-dconv/2.4/configuration.html#2.6

###################### 以下根据官方文档整理 haproxy-2.4.0/INSTALL ######################
yum groupinstall -y "Development Tools"
wget https://www.haproxy.org/download/2.4/src/haproxy-2.4.0.tar.gz
tar -xf haproxy-2.4.0.tar.gz
cd haproxy-2.4.0
make clean

cd ..
# USE_OPENSSL=1 USE_PCRE=1 USE_SYSTEMD=1
yum install -y openssl-devel pcre-devel systemd-devel
# USE_LUA=1
curl -R -O https://www.lua.org/ftp/lua-5.3.6.tar.gz
tar zxf lua-5.3.6.tar.gz
cd lua-5.3.6
yum install -y libtermcap-devel ncurses-devel libevent-devel readline-devel
make linux
make install

cd ../haproxy-2.4.0
make -j $(nproc) TARGET=linux-glibc USE_OPENSSL=1 USE_LUA=1 USE_PCRE=1 USE_SYSTEMD=1
make install PREFIX=/usr/local/haproxy

mkdir /usr/local/haproxy/conf
touch /usr/local/haproxy/conf/haproxy.cfg

# systemd服务化
cat <<EOF | sudo tee /etc/systemd/system/haproxy.service
[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target

[Service]
ExecStartPre=/usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/conf/haproxy.cfg -c
ExecStart=/usr/local/haproxy/sbin/haproxy -Ws -f /usr/local/haproxy/conf/haproxy.cfg

[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now haproxy


# 该文件是yum install -y haproxy 自动生成的
cat <<EOF | sudo tee /etc/systemd/system/haproxy.service
[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target

[Service]
EnvironmentFile=/etc/sysconfig/haproxy
ExecStart=/usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid $OPTIONS
ExecReload=/bin/kill -USR2 $MAINPID
KillMode=mixed

[Install]
WantedBy=multi-user.target
EOF
```

例子haproxy.cfg

```groovy
    global
        daemon
        maxconn 256
         
    defaults
        mode http
        timeout connect 5000ms
        timeout client 50000ms
        timeout server 50000ms
		stats enable

    frontend http-in
        bind *:8080
        default_backend servers

    backend servers
        option httpchk GET /
        server server1 192.168.96.14:80 maxconn 32 check inter 1000
        server server2 192.168.96.15:80 maxconn 32 check inter 1000
```



## Keepalived

```shell
yum -y install gcc openssl-devel libnl3-devel net-snmp-devel
wget https://www.keepalived.org/software/keepalived-2.2.2.tar.gz
tar zxf keepalived-2.2.2.tar.gz
cd keepalived-2.2.2
./configure --prefix=/usr/local/keepalived
make && make install

# ln -s /usr/local/keepalived-2.2.2 /usr/local/keepalived

mkdir /etc/keepalived
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/

cp /usr/local/keepalived/sbin/keepalived /usr/sbin/
cp keepalived/etc/init.d/keepalived /etc/init.d/

################ 以下是官方文档方法 ################
yum -y install curl gcc openssl-devel libnl3-devel net-snmp-devel
curl --progress http://keepalived.org/software/keepalived-2.2.2.tar.gz | tar xz
cd keepalived-2.2.2
./configure --prefix=/usr/local/keepalived-2.2.2
make
sudo make install
# install后就有keepalived.service了，可以进行查看 /usr/lib/systemd/system/keepalived.service
mkdir /etc/keepalived
cp /usr/local/keepalived-2.2.2/etc/keepalived/keepalived.conf /etc/keepalived/

systemctl enable --now keepalived

# yum 安装
yum -y install keepalived
# 

[Unit]
Description=LVS and VRRP High Availability Monitor
After=syslog.target network-online.target

[Service]
Type=forking
PIDFile=/var/run/keepalived.pid
KillMode=process
EnvironmentFile=-/etc/sysconfig/keepalived
ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target


```

vim /etc/keepalived/keepalived.conf

```perl
#检测脚本
vrrp_script chk_http_port {
    script "/usr/local/src/check_nginx_pid.sh" #心跳执行的脚本，检测nginx是否启动
    interval 2                          #（检测脚本执行的间隔，单位是秒）
    weight 2                            #权重
}
#vrrp 实例定义部分
vrrp_instance VI_1 {
    state MASTER            # 指定keepalived的角色，MASTER为主，BACKUP为备
    interface ens33         # 当前进行vrrp通讯的网络接口卡(当前centos的网卡) 用ifconfig查看你具体的网卡
    virtual_router_id 66    # 虚拟路由编号，主从要一直
    priority 100            # 优先级，数值越大，获取处理请求的优先级越高
    advert_int 1            # 检查间隔，默认为1s(vrrp组播周期秒数)
    #授权访问
    authentication {
        auth_type PASS #设置验证类型和密码，MASTER和BACKUP必须使用相同的密码才能正常通信
        auth_pass 1111
    }
    track_script {
        chk_http_port            #（调用检测脚本）
    }
    virtual_ipaddress {
        192.168.96.130            # 定义虚拟ip(VIP)，可多设，每行一个
    }
}
```

```perl
#检测脚本
vrrp_script chk_http_port {
    script "/usr/local/src/check_nginx_pid.sh" #心跳执行的脚本，检测nginx是否启动
    interval 2                          #（检测脚本执行的间隔）
    weight 2                            #权重
}
#vrrp 实例定义部分
vrrp_instance VI_1 {
    state BACKUP                        # 指定keepalived的角色，MASTER为主，BACKUP为备
    interface ens33                      # 当前进行vrrp通讯的网络接口卡(当前centos的网卡) 用ifconfig查看你具体的网卡
    virtual_router_id 66                # 虚拟路由编号，主从要一直
    priority 99                         # 优先级，数值越大，获取处理请求的优先级越高
    advert_int 1                        # 检查间隔，默认为1s(vrrp组播周期秒数)
    #授权访问
    authentication {
        auth_type PASS #设置验证类型和密码，MASTER和BACKUP必须使用相同的密码才能正常通信
        auth_pass 1111
    }
    track_script {
        chk_http_port                   #（调用检测脚本）
    }
    virtual_ipaddress {
        192.168.96.130                   # 定义虚拟ip(VIP)，可多设，每行一个
    }
}

```



vim /usr/local/src/check_nginx_pid.sh

```shell
#!/bin/bash
if [ `ps -C nginx --no-header | wc -l` -eq 0 ]; then    #如果nginx没有启动就启动nginx                        
      systemctl start nginx                #重启nginx
      if [ `ps -C nginx --no-header | wc -l` -eq 0 ]; then    #nginx重启失败，则停掉keepalived服务，进行VIP转移
              killall keepalived
      fi
fi
```

```perl
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 101
    authentication {
        auth_type PASS
        auth_pass 42
    }
    virtual_ipaddress {
        192.168.96.99
    }
    track_script {
        check_apiserver
    }
}
```

```bash
#!/bin/bash

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

curl --silent --max-time 2 http://localhost:8080/ -o /dev/null || errorExit "Error GET https://localhost:8080/"
```



## Kafka

```shell
# https://kafka.apache.org/downloads
tar -xzf kafka_2.13-2.8.0.tgz
cd kafka_2.13-2.8.0

bin/zookeeper-server-start.sh config/zookeeper.properties
bin/kafka-server-start.sh config/server.properties

# 服务化
cat <<EOF | sudo tee /etc/systemd/system/kafka.service
[Unit]
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
EOF

# 集群部署
broker.id=0
listeners=PLAINTEXT://192.168.96.14:9092
log.dir=/var/lib/kafka-logs
zookeeper.connect=192.168.96.14:2181

broker.id=1
listeners=PLAINTEXT://192.168.96.15:9092
log.dir=/var/lib/kafka-logs
zookeeper.connect=192.168.96.14:2181

broker.id=2
listeners=PLAINTEXT://192.168.96.16:9092
log.dir=/var/lib/kafka-logs
zookeeper.connect=192.168.96.14:2181
```



## Zookeeper

```shell
# https://zookeeper.apache.org/releases.html
tar -xf apache-zookeeper-3.5.9-bin.tar.gz
cd apache-zookeeper-3.5.9-bin/conf/
cp zoo_sample.cfg zoo.cfg

# Change the value of dataDir to specify an existing (empty to start with) directory
vim zoo.cfg

cd ..
bin/zkServer.sh start

# 集群部署 https://zookeeper.apache.org/doc/r3.5.9/zookeeperStarted.html
# https://zookeeper.apache.org/doc/r3.6.3/zookeeperAdmin.html#sc_zkMulitServerSetup 

tickTime=2000
dataDir=/var/lib/zookeeper/
clientPort=2181
initLimit=5
syncLimit=2
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888

echo 1 ? /var/lib/zookeeper/myid
```



## K8S

### Kubeadm

```shell
####################################################################################################
##### 一、 Swap disabled. You MUST disable swap in order for the kubelet to work properly.
swapoff -a
vim /etc/fstab
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
echo vm.swappiness=0 >> /etc/sysctl.conf

##### 二、 Make sure that the br_netfilter module is loaded
sudo modprobe br_netfilter
lsmod | grep br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
# 以下解决：重启后br_netfilter未加载
chmod +x /etc/rc.d/rc.local
echo sudo modprobe br_netfilter >> /etc/rc.d/rc.local

##### 三、 开启docker
systemctl start docker

##### 四、 Installing kubeadm, kubelet and kubectl
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
# 网络不通用以下源
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet-1.18.2 kubeadm-1.18.2 kubectl-1.18.2 --disableexcludes=kubernetes

sudo systemctl enable --now kubelet

##### 五、nitializing your control-plane node
kubeadm init \
--apiserver-advertise-address 192.168.96.4 \
--pod-network-cidr 10.244.0.0/16 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.18.2

##### 六、To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
# Alternatively, if you are the root user, you can run:
export KUBECONFIG=/etc/kubernetes/admin.conf
# 当前shell有效，重启后需要重新设置

##### 七、Installing a Pod network add-on, 公司经常连不上，可先把文件下载下来
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

##### 八、其他
# kubectl 自动补全
yum install -y bash-completion
source <(kubectl completion bash)



```

node加入k8s集群

```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.96.4:6443 --token l85o6h.v7l1tpd8mf977yb6 \
    --discovery-token-ca-cert-hash sha256:4cfad28f5becd50fae5f3467afe1b1cde154973bba5fb67709fce1360e6c9ba4 

```

安装ingress-nginx 

```shell
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
sed -i 's#quay.io/kubernetes-ingress-controller/nginx-ingress-controller#registry.aliyuncs.com/google_containers/nginx-ingress-controller#g' mandatory.yaml

wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml

kubectl apply -f mandatory.yaml
kubectl apply -f service-nodeport.yaml

# Verify installation
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --watch
```



### 高可用control plane搭建

```shell
# 一下步骤所有节点执行
# 1. 准备工作
yum install -y vim lrzsz
# 关闭防火墙
systemctl disable --now firewalld

# 2. Swap disabled
swapoff -a
sed -i 's@^/dev/mapper/centos-swap@#/dev/mapper/centos-swap@' /etc/fstab
echo vm.swappiness=0 > /etc/sysctl.d/swap.conf

# 3. Letting iptables see bridged traffic
sudo modprobe br_netfilter
chmod +x /etc/rc.d/rc.local
echo 'sudo modprobe br_netfilter' >> /etc/rc.d/rc.local

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

# 4. Installing runtime

# (Install Docker CE)
## Set up the repository
### Install required packages
yum install -y yum-utils device-mapper-persistent-data lvm2

## Add the Docker repository
yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
## ali repo  
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo 

# Install Docker CE
yum update -y && yum install -y \
  containerd.io-1.2.13 \
  docker-ce-19.03.11 \
  docker-ce-cli-19.03.11
  
## Create /etc/docker
mkdir /etc/docker

# Set up the Docker daemon
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "insecure-registries": ["172.30.37.192"]
}
EOF

# Restart Docker
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker

# 5. Installing kubeadm, kubelet and kubectl (install these packages on all of your machines)
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 网络不通用以下源
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

#sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo yum install -y kubelet-1.18.2 kubeadm-1.18.2 kubectl-1.18.2 --disableexcludes=kubernetes
#sudo yum install -y kubelet-1.19.1 kubeadm-1.19.1 kubectl-1.19.1 --disableexcludes=kubernetes
#sudo yum install -y kubelet-1.19.0 kubeadm-1.19.0 kubectl-1.19.0 --disableexcludes=kubernetes
#sudo yum install -y kubelet-1.22.0 kubeadm-1.22.0 kubectl-1.22.0 --disableexcludes=kubernetes

sudo systemctl enable --now kubelet

# 6. Creating a single control-plane cluster with kubeadm (这一步只在master执行)
# Initializing your control-plane node
kubeadm init \
--apiserver-advertise-address 192.168.96.10 \
--pod-network-cidr 10.244.0.0/16 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.19.0

kubeadm init \
--apiserver-advertise-address 192.168.96.14 \
--pod-network-cidr 10.244.0.0/16 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.22.0

# 集群安装
kubeadm config images pull \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.22.0

sysctl -w net.ipv4.ip_forward=1

kubeadm init \
--control-plane-endpoint "192.168.96.99:16443" \
--pod-network-cidr 10.244.0.0/16 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.22.0 \
--upload-certs

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# kubectl 自动补全
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' > /etc/profile.d/k8s.sh

# 7. Joining your nodes (这一步所有node机执行，加入集群)
kubeadm join 192.168.96.10:6443 --token 342ln0.gnihyw41wyhhaj9w \
    --discovery-token-ca-cert-hash sha256:70fccee1094ebd3e457b0bc66ea170b3f9f818a59634b4ab6d8246d35c637e86
    
kubeadm join 192.168.96.10:6443 --token fuomq6.wskgus1agi2axy3z \
    --discovery-token-ca-cert-hash sha256:2199944ed6ca978eddb174e991595740f22b047c07ca1a332aa8b39923b0e925
    
kubeadm join 192.168.220.16:6443 --token 5roybd.8btb6kjw9z28gnhq \
    --discovery-token-ca-cert-hash sha256:5fbf246a2a8f6323471aee1ec8d12a0ead3277710e9808f6642e83b6d1aa438a

kubeadm join 192.168.220.16:6443 --token 1n9k6j.8ritutnllll5irdh \
    --discovery-token-ca-cert-hash sha256:5fbf246a2a8f6323471aee1ec8d12a0ead3277710e9808f6642e83b6d1aa438a
    1n9k6j.8ritutnllll5irdh
    5fbf246a2a8f6323471aee1ec8d12a0ead3277710e9808f6642e83b6d1aa438a
    
  
```

### ingress-nginx

```json
# https://github.com/kubernetes/ingress-nginx/blob/nginx-0.30.0/docs/deploy/index.md 参考文档
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml
```

### Harbor

```shell
kubectl create secret docker-registry regsecret --docker-server=registry.cn-hangzhou.aliyuncs.com --docker-username=yin32167@aliyun.com --docker-password=xxxxxx --docker-email=yin32167@aliyun.com

kubectl create secret docker-registry NAME --docker-username=user --docker-password=password
--docker-email=email [--docker-server=string] [--from-literal=key1=value1]

kubectl create secret docker-registry harbor-secret --docker-server=172.30.37.192 --docker-username=cqss_harbor --docker-password=harborcqssQWE123 --docker-email=cqss_harbor@cqssmail.com
```

### Helm

```shell
# https://github.com/helm/helm/releases
wget https://get.helm.sh/helm-v3.3.4-linux-amd64.tar.gz
tar -zxvf helm-v3.3.4-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
# 自动补全
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(helm completion bash)
echo 'source <(helm completion bash)' > /etc/profile.d/helm.sh

helm repo add stable https://charts.helm.sh/stable
helm search repo stable

mvn -P test dockerfile:build dockerfile:push -Ddockerfile.tag=v2
mvn -P test dockerfile:build dockerfile:push -Ddockerfile.tag=v2
mvn -P test-oms dockerfile:build dockerfile:push

1fbe774145a6


docker run -e PARAMS="--spring.datasource.url=jdbc:mysql://127.0.0.1:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai" -p 8080:8080 -v /tmp:/data/applogs --name xxl-job-admin  -d xuxueli/xxl-job-admin:{指定版本}

docker run -e spring.datasource.url="jdbc:mysql://172.30.37.30:3326/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai" \
-e spring.datasource.username="root" \
-e spring.datasource.password="yzl8qgYaFjJ4mBlU" \
-p 8080:8080 --name xxl-job-admin -d 1fbe774145a6

```

### metrics-server

```shell
# https://github.com/kubernetes-sigs/metrics-server
wget https://github.com/kubernetes-sigs/metrics-server/releases/download/metrics-server-helm-chart-3.6.0/components.yaml
# 编辑components.yaml中的镜像registry.aliyuncs.com/google_containers/metrics-server:v0.5.1
# 增加参数--kubelet-insecure-tls
kubectl apply -f components.yaml
```



### Kubernetes in Action

- A ReplicaSet behaves exactly like a ReplicationController, but it has more expressive pod selectors. Whereas a ReplicationController’s label selector only allows matching pods that include a certain label, a ReplicaSet’s selector also allows matching pods that lack a certain label or pods that include a certain label key, regardless of its value.  
- By default, the default-token Secret is mounted into every container, but you can disable that in each pod by setting the automountServiceAccountToken field in the pod spec to false or by setting it to false on the service account the pod is using.   
- For now, all you need to know is that a service account is the account that the pod authenticates as when talking to the API server.  
- This also explains why labels and annotations can’t be exposed through environment variables. Because environment
  variable values can’t be updated afterward, if the labels or annotations of a pod were exposed through environment variables, there’s no way to expose the new values after they’re modified.  
- Using volumes to expose a container’s resource requests and/or limits is slightly more complicated than using environment variables, but the benefit is that it allows you to pass one container’s resource fields to a different container if needed (but
  both containers need to be in the same pod). With environment variables, a container can only be passed its own resource limits and requests.  

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```



## ElasticSearch

```shell
# 注意不能是root用户
tar -xzf elasticsearch-7.9.0-linux-x86_64.tar.gz
cd elasticsearch-7.9.0/
./bin/elasticsearch -d

[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

ulimit -n 65535 
# /etc/security/limits.conf
*       soft    nofile  65535
*       hard    nofile  65535
# /etc/sysctl.conf
vm.max_map_count=262144
# 使vm.max_map_count当前生效
sysctl -p
# config/elasticsearc.yml
node.name: node-1
network.host: 0.0.0.0
cluster.initial_master_nodes: ["node-1"]


./bin/elasticsearch -d -p pid
pkill -F pid

```



## Kibana

```shell
# 注意不能是root用户
tar -xzf kibana-7.9.0-linux-x86_64.tar.gz
cd kibana-7.9.0-linux-x86_64/

# config/kibana.yml
server.host: "0.0.0.0"
```



## Filebeat

```shell
tar -xzf filebeat-7.9.0-linux-x86_64.tar.gz
cd filebeat-7.9.0-linux-x86_64
```



## Metricbeat

```shell
curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.9.0-linux-x86_64.tar.gz
tar xf metricbeat-7.9.0-linux-x86_64.tar.gz
cd metricbeat-7.9.0-linux-x86_64/
```



## Gitea

```shell
CREATE USER 'gitea'@'%' IDENTIFIED BY 'Gitea01!';
CREATE DATABASE giteadb CHARACTER SET 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';
GRANT ALL PRIVILEGES ON giteadb.* TO 'gitea'@'%';
FLUSH PRIVILEGES;

wget -O gitea https://dl.gitea.io/gitea/1.12.5/gitea-1.12.5-linux-amd64
chmod +x gitea
./gitea web
# 管理员 zhanzixi/1981422 1987295689@qq.com
# james/James01!
```



## Rancher

```shell
# 确保docker环境
sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher

# https://rancher.com/docs/os/v1.x/en/networking/proxy-settings/
sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 -e NO_PROXY="localhost,127.0.0.1" rancher/rancher:v2.5.2

# https://docs.docker.com/engine/reference/run/
# unless-stopped	Always restart the container regardless of the exit status, including on daemon startup, except if the container was put into a stopped state before the Docker daemon was stopped.

# 设置密码 admin/admin

```

* You can install Rancher on a single node, or on a high-availability Kubernetes cluster.
* 

## prometheus

```shell
wget https://github.com/prometheus/prometheus/releases/download/v2.29.1/prometheus-2.29.1.linux-amd64.tar.gz

tar xvfz prometheus-2.29.1.linux-amd64.tar.gz
cd prometheus-*


helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```



## Istio

```shell
# https://github.com/istio/istio 选择版本，下载压缩包
tar xzf istio-1.7.4-linux-amd64.tar.gz
cd istio-1.7.4
cp bin/istioctl /usr/bin


istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled

# kubectl label namespace default istio-injection=disabled
# 卸载
istioctl manifest generate --set profile=demo | kubectl delete --ignore-not-found=true -f -

# Deploy the sample application
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl get services
kubectl get pods
```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews # 1
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```

- Like other Istio configuration, the API is specified using Kubernetes custom resource definitions (***CRD***s), which you can configure using YAML
- ***Virtual services***, along with ***destination rules***, are the key building blocks of Istio’s traffic routing functionality.
- Route destinations can be versions of the same service or entirely different services.
- Virtual service hosts don’t actually have to be part of the Istio service registry, they are simply virtual destinations.
- Routing rules are ***evaluated in sequential order from top to bottom***, with the first rule in the virtual service definition being given highest priority.
- ou can think of virtual services as how you route your traffic **to** a given destination, and then you use destination rules to configure what happens to traffic **for** that destination.



## kubevela

```shell
# Add helm chart repo for KubeVela
helm repo add kubevela https://kubevelacharts.oss-accelerate.aliyuncs.com/core
# Update the chart repo
helm repo update
# Install KubeVela
helm install --create-namespace -n vela-system kubevela kubevela/vela-core

# Get KubeVela CLI,网络不好,反复重试
curl -fsSl https://kubevela.io/script/install.sh | bash

# Enable Helm Support
helm install --create-namespace -n flux-system helm-flux http://oam.dev/catalog/helm-flux2-0.1.0.tgz

echo 'source <(vela completion bash)' > /etc/profile.d/vela.sh
```





## etcd

###  单机

```shell
# https://github.com/etcd-io/etcd/releases

tar xzf etcd-v3.4.0-linux-amd64.tar.gz
cd etcd-v3.4.0-linux-amd64/
./etcd -h

cp etcd /usr/bin

etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://0.0.0.0:2379

cat > /etc/systemd/system/etcd.service <<\EOF
[Unit]
Description=Etcd daemon
After=network.target 
Wants=network.target
   
[Service] 
ExecStart=/usr/bin/etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://0.0.0.0:2379
  
[Install] 
WantedBy=multi-user.target
EOF


--listen-peer-urls 'http://localhost:2380'
List of URLs to listen on for peer traffic.
--listen-client-urls 'http://localhost:2379'
List of URLs to listen on for client traffic.

./etcd --listen-client-urls http://127.0.0.1:12379 --advertise-client-urls http://127.0.0.1:12379 --listen-peer-urls http://127.0.0.1:12380 --logger=zap
./etcdctl --endpoints=127.0.0.1:12379 put foo bar
./etcdctl --endpoints=127.0.0.1:12379 get foo

# https://github.com/etcd-io/etcd/blob/main/etcd.conf.yml.sample


--initial-advertise-peer-urls 'http://localhost:2380'
    List of this member's peer URLs to advertise to the rest of the cluster.
--listen-peer-urls 'http://localhost:2380'
    List of URLs to listen on for peer traffic.
--advertise-client-urls 'http://localhost:2379'
    List of this member's client URLs to advertise to the public.
    The client URLs advertised should be accessible to machines that talk to etcd cluster. etcd client libraries parse these URLs to connect to the cluster.
--listen-client-urls 'http://localhost:2379'
    List of URLs to listen on for client traffic.
    
etcdctl \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
get / --prefix --keys-only
```

### 集群

```shell
#!/bin/bash
TOKEN=token-01
CLUSTER_STATE=new
NAME_1=machine-1
NAME_2=machine-2
NAME_3=machine-3
HOST_1=192.168.96.14
HOST_2=192.168.96.15
HOST_3=192.168.96.16
CLUSTER=${NAME_1}=http://${HOST_1}:2380,${NAME_2}=http://${HOST_2}:2380,${NAME_3}=http://${HOST_3}:2380

# For machine 1
THIS_NAME=${NAME_1}
THIS_IP=${HOST_1}
etcd --data-dir=data.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
	
# For machine 2
THIS_NAME=${NAME_2}
THIS_IP=${HOST_2}
etcd --data-dir=data.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

# For machine 3
THIS_NAME=${NAME_3}
THIS_IP=${HOST_3}
etcd --data-dir=data.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
	
## 服务化
cat > /etc/systemd/system/etcd.service <<\EOF
[Unit]
Description=Etcd daemon
After=network.target 
Wants=network.target
   
[Service] 
Type=forking
ExecStart=etcd --data-dir=data.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
  
[Install] 
WantedBy=multi-user.target
EOF
```

openssl

```shell
cp /etc/pki/tls/openssl.cnf .

# 生成ca根证书私钥
openssl genrsa -out ca.key 2048
# 生成ca根证书请求文件
openssl req -new -key ca.key -out ca.csr -subj /CN=etcd.example.com
# 自己作为ca机构签发根证书（自签发证书）
openssl x509 -req -days 365 -signkey ca.key -in ca.csr -out ca.crt

# 生成server端私钥
openssl genrsa -out server.key 2048
# 生成server端证书请求文件
openssl req -new -key server.key -out server.csr -subj /CN=etcd.example.com
# 2.3 使用ca根证书为server端签发证书
# touch /etc/pki/CA/index.txt
# echo "01" > /etc/pki/CA/serial
openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key -config openssl.cnf

# 生成client端私钥
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr -subj /CN=etcd.example.com
openssl ca -in client.csr -out client.crt -cert ca.crt -keyfile ca.key -config openssl.cnf
```



## NPM

```shell
npm config set registry https://registry.npmjs.org/
npm config set registry https://registry.npm.taobao.org/

# The npm registry contains packages, many of which are also Node modules, or contain Node modules
```

* The npm registry contains packages, many of which are also Node modules, or contain Node modules.
* A **package** is a file or directory that is described by a `package.json` file. A package must contain a `package.json` file in order to be published to the npm registry.
* Packages can be unscoped or scoped to a user or organization, and scoped packages can be private or public. 
* A **module** is any file or directory in the `node_modules` directory that can be loaded by the Node.js `require()` function.
* To be loaded by the Node.js `require()` function, a module must be one of the following:
  - A folder with a `package.json` file containing a `"main"` field.
  - A JavaScript file.
* When listed as a dependent in a `package.json` file, scoped packages are preceded by their scope name. The scope name is everything between the `@` and the slash
* semantic versioning syntax
  * Patch releases: `1.0` or `1.0.x` or `~1.0.4`
  * Minor releases: `1` or `1.x` or `^1.0.4`
  * Major releases: `*` or `x`

## openssl

```shell
# https://www.openssl.org/source/ 
wget https://www.openssl.org/source/openssl-3.0.0.tar.gz
tar xf openssl-3.0.0.tar.gz
yum install -y perl-core
cd openssl-3.0.0
./Configure --prefix=/usr/local/ssl --openssldir=/usr/local/ssl '-Wl,-rpath,$(LIBRPATH)'
make
make install

# 用 OpenSSL 创建可以用于 https 的证书
# https://blog.csdn.net/joyous/article/details/80659925
```



# 韩顺平Linux视频教程2018版

```shell
$ wc -l users
5 users
$ wc -l < users
5


```

# 新版本git gitlab Jenkins课程

```shell
# 课时3 安装git
yum install -y git
yum erase -y git
# 如果需要安装指定版本，则从官网下载tar.gz安装
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.26.2.tar.gz
yum -y install curl-devel expat-devel gettext-devel openssl-devel zlib-devel 
yum -y install gcc-c++ perl-ExtUtils-MakeMaker
tar zxf git-2.26.2.tar.gz
cd git-2.26.2
./configure --prefix=/usr/local/git-2.26.2
make && make install
ln -s /usr/local/git-2.26.2/bin/* /usr/bin/

# 课时4、5、6 git入门级命令
git
git --version
git help add
git init
git add
git commit -m '...'
git remote add origin ...
git push -u orgin master
git fetch # 拉去远程仓库的变更到本地仓库
git merge orgin/master # 将远程的变更，合并到本地仓库的master分支
git pull # 不建议使用
#  文件的几种状态
git status
git rm --cached Test.txt # --cached 表示保留工作区
# git-checkout - Switch branches or restore working tree files
git checkout Test.txt # 直接加文件名，从暂存区将文件恢复到工作区，如果工作区已经有该文件，则会选择覆盖
git checkout master Test.txt # 分支名 + 文件名，表示从分支名中拉取文件，并覆盖工作区的文件
# tortoisegit 还是 sourcetree?

# 课时8 分支
git branch 分支名
git branch
git branch -m 旧分支名 新分支名
git checkout 分支名
git checkout -f 分支名 # -f一定要小心，一般不建议使用

# 课时9
git log
git log -n # 最近n此提交
git log -p -2 # 显示提交的不同点
git log --author=
git log --oneline
git log --graph
git log -- # tab可以查看选项

# 课时10
git diff # 工作区和暂存区的差异
git diff --cached # --staged 暂存区与仓库的差异
git diff HEAD # 、
git diff 分支名
git diff 分支名1 分支名2
git diff 文件名 # 查看2个分支（已提交）差异
git diff commitid1 commitid2 # 用于列出2个历史提交的差异
git diff --stat # 用于罗列有变更的文件

# 课时11
git reset
git reset HEAD 文件名
git reset HEAD^
git reset --soft HEAD^
git reset --soft # HEAD指向指定提交，暂存区和工作区不变 
git reset --mixed # HEAD指向指定提交，暂存区改变，工作区不变 
git reset --hard # HEAD指向指定提交，暂存区改变，工作区改变

# 课时12
git merge 分支名 # 合并到当前分支
git diff --name-only --diff-filter=U # 用于查看产生冲突的文件

# 课时13
git tag # 不加任何参数，表示显示标签（以字母顺序）
git tag v1 # 打标签，默认给最近的一次提交
git tag 标签名 commitid # 
git show 标签名 # 显示该标签相关的那次提交信息
git tag -d 标签名 # 删除该标签
git push origin v1 # 推送标签推送到远程服务器上
git tag -d v1
git push origin :refs/tags/v1 # 删除远程标签
# 标签给和commit挂扣，只要commit存在于多个分支，那么这几个分支就都能看到这个标签

# 课时14 .gitignore，一般情况下在项目一开始的时候就创建，并推送到远程服务器上
git add * # 不添加.开头的
git add .gitignore
# *.log 123?.log /error.log src/main/test/* **/java/ !/error.log
git rm --cached 文件名 # 从暂存区删除某个文件
git rm -rf --cached 文件夹 # 提柜删除该文件夹的所有东西

# 课时19
# 账户注册时打开邮件验证功能
# 新建组、添加成员、创建项目
# 添加sshkey，上传文件

# 课时20
# master、test分支一般会保护起来，只有特定人员才可以进行合并
# 合并请求、处理请求合并
# tag保护

```

# Shell Programming

1. Remember, the shell performs variable substitution *before* it executes the command 
2. It turns out that the shell does not perform filename substitution when assigning values to variables. Therefore, **x=\* **assigns the single character * to x. 
3. This order of evaluation is important. Remember, first the shell does ***variable substitution***, then does ***filename substitution***, and then parses the line into arguments.
4. There are also quite a few functions applicable to variables within this curly brace notation, including extracting subsets, assigning values if the variable is currently unassigned, and more. 
5. But because filename substitution is *not* done inside double quotes, * is then safely passed to echo as the value to be displayed
6. Functionally, the backslash (used as a prefix) is equivalent to placing single quotes around a single character, though with a few minor exceptions.   
7. When a backslash is the last character of a line of input, the shell treats it as a line continuation character
8. There are two ways in the shell to perform command substitution: by enclosing the command in back quotes or surrounding it with the $(...) construct.  
9. back quote: its purpose is to tell the shell to replace the enclosed command with its output
10. preferred $(...) construct for ***command substitution***
11. These special variables—more formally known as ***positional parameters*** because they’re based on the position of the value in the command line—are assigned after the shell has done its normal command-line processing (that is, I/O redirection, variable substitution, filename substitution, and so on).
12. you can use the exec command to *replace the* *current program* (db) *with the new one* (/bin/sh). This means that it would now be pointless to have any commands follow the exec because they’ll never be executed.
13. That is, you can use the same special characters for specifying the patterns in a case as you can with filename substitution. For example, ? can be used to specify any single character; * can be used to specify zero or more occurrences of any character; and [...] can be used to specify any single character enclosed between the brackets
14. the shell treats loops as if they were mini-programs of their own, so whatever appears after block closing statements like done, fi and esac can have redirects, be put into background with the ampersand or even form part of a pipeline of commands
15. The notation >& specifies output redirection to a file associated with the *file descriptor* that follows. 
16. The Unix system recognizes only three basic types of files: ***ordinary files***, ***directory files***, and ***special files***.  
17. or you can specify the particular file by its ***pathname***  
18. The term ***filter*** is often used in Unix terminology to refer to any program that can take input from standard input, perform some operation on that input, and write the results to standard output  
19. The Unix system is logically divided into two different areas: the ***kernel*** and the ***utilities***  
20. Recall that the -o option to sort specifies where the sorted output is to be written, and that this can be the same as the input file
21. In a pipeline, the exit status reflects the last command in the pipe  
22. The shell variable ***$?*** is automatically set by the shell to the exit status of the last command executed.   
23. To avoid this sort of problem, many shell programmers write their test commands as **test X"$symbol" = X **which will be true if symbol is null, and false if it’s not. The X in front of symbol prevents test from interpreting the characters stored in symbol as an operator  
24. This is a good point to introduce the shell’s ***-x*** option.  
25. The symbol ***|*** has the effect of a logical OR when used between two patterns.  
26. This time, let’s take advantage of the fact that date with the ***+%H*** option writes a two-digit hour to standard output.  
27. With the shell’s built-in null command. The format of this command is simply **:** and the purpose of it is—you guessed it—to do nothing.  
28. The shell permits filename substitution in the list of words in the for statement.
29. The shell replaces the value of $* with $1, $2, … , but if you instead use the special shell variable "***$@***" the values will be passed with "$1", "$2", … . The key difference is the double quotes around the $@: without them this variable behaves just like $*.  
30. By default, all your processes are automatically terminated when you log off the system. If you want a program to continue running after you’ve logged off, you can run it with the ***nohup*** command, or schedule it to run with at or from cron.  
31. In these situations the command ***true*** can be used to return an exit status of zero. The command ***false*** can be used in the opposite situation too, as it returns a nonzero exit status.  
32. If the break command is used in the form ***break n*** the n innermost loops are immediately exited.
33. ***continue n*** causes the commands in the innermost n loops to be skipped, after which execution of the program continues as normal.  
34. This—and subsequent examples—work because the shell treats loops as if they were mini-programs of their own, so whatever appears after block closing statements like ***done***, ***fi*** and ***esac*** can have redirects, be put into background with the ampersand or even form part of a pipeline of commands.  
35. Individual statements can override block redirection, just as any other statements in your shell programs can explicitly read from a specified source or send output to a specified destination.  
36. To force input or output to come from or go to the terminal, utilize ***/dev/tty***, which always refers to your terminal program, whether you’re on a Mac, Linux, or Unix system.  
37. Don’t fret, though; the shell provides a built-in command called ***getopts*** that makes it easy to process command line arguments. The general format of the command is ***getopts options variable***  
38. The ***basename*** command gives the base filename of its argument with all directory components stripped away (for example, basename /usr/bin/troff produces troff; basename troff also produces troff).  
39. The seemingly weird behavior exhibited by vartest and vartest2 is due to the fact that these programs are run within ***subshells*** by your login shell.  A subshell is essentially an entirely new shell just to run the desired program.  
40. There is a way to make the value of a variable known to a subshell, and that’s by exporting it with the ***export*** command. The format of this command is simply  ***export variables***  
41. There is no way to change the value of a variable in a parent shell from within a subshell.  
42. After a variable is exported, it remains exported to all subshells subsequently executed.  
43. (pronounced “dot”) whose general format is ***. file*** and whose purpose is to execute the contents of file in the current shell.  
44. This means that it would now be pointless to have any commands follow the exec because they’ll never be executed.  
45. The format is a bit more general than that shown, actually, because you can assign several variables at once using the format
    ***variable=value variable=value ...***  
46. ***${parameter:-value}*** This construct says to use the value of parameter if it is not null, and to substitute value otherwise.  
47. ***${parameter:=value}*** This version is similar to the previous, but if parameter is null not only is value used, but it is also assigned to parameter as well (note the = in the construct).  You can’t assign values to positional parameters this way, however, so that means that parameter can’t be a number.  
48. ***${parameter:?value}*** If parameter is not null, the shell substitutes its value; otherwise, the shell writes value to standard error and then exits  
49. ***${parameter:+value}*** This one substitutes value if parameter is not null; otherwise, it substitutes nothing. It’s the opposite of :-  
50. Sometimes you may want to remove the definition of a variable from your environment. To do so, you type ***unset*** followed by the names of the variables  
51. When you put ***eval*** in front of it, however, the effect is that the shell scans the command line ***twice*** before executing it，Its format is as follows: ***eval command-line***  
52. Its general format is ***wait process-id***  where process-id is the process ID of the process you want to complete. If the process ID is omitted, the shell waits for all child processes to complete execution.  
53. Signal handling in a shell program is done with the trap command, whose general format is ***trap commands signals***  
54. If the command listed for trap is null, the specified signal will be ignored when received. For example, the command ***trap "" SIGINT***  specifies that the interrupt signal is to be ignored.  If you ignore a signal, all subshells also ignore that signal. If you specify a signal handler action, however, all subshells will automatically take the default action on receipt of that signal, not
    the new code sequence.  
55. You also know that you can redirect standard error from any command simply by writing 2> instead of just >:
    ***command 2> file***  
56. ***command >&2*** The notation >& specifies output redirection to a file associated with the file descriptor that
    follows. File descriptor 0 is standard input, descriptor 1 is standard output, and descriptor 2 is standard error. It’s important to remember that no space is permitted between the > and the &.  
57. The sequence ***>&-*** has the effect of closing standard output.  
58. One of the best uses of the ***in-line input redirection*** feature is for creating ***shell archive*** files.  
59. To define a function, you use the general format: ***name () { command; ... command; }***  
60. One key feature: arguments listed after the function on the command line are assigned to the positional parameters $1, $2, ..., within the function, just as with any other command.  

```shell
command 2> file
date; pwd
# The standard output from the command will still be directed to your terminal
sort bigdata > out & 
ps -f
cut -cchars file
paste files
sed command file
tr from-chars to-chars
grep pattern files
lines=one'
'two
echo "$lines"
echo "\ is the backslash character"
sort -o phonebook phonebook
sed "/$1/d" phonebook > /tmp/phonebook
[ -z "$EDITOR" ] && EDITOR=/bin/ed
grep "$name" phonebook || echo "Couldn't find $name"
while [ "$endofdata" -ne TRUE ]
do
...
done 2> errors
echo "Error: no file" 1>&2

x=100 y=200 z=50
echo $x $y $z
: ${PHONEBOOK:=$HOME/phonebook}
${variable%pattern}
${variable%%pattern}
${variable#pattern}
${variable##pattern}

set one two three four
echo $1:$2:$3:$4
command 2> file
command >&2
command > foo 2>> foo
command > foo 2>&1
exec < datafile
exec > /tmp/output
exec 2> /tmp/errors
ls >&-
command <<word
```

# Netty

* 104.18.49.137 www.allitebooks.com
  104.18.49.137 file.allitebooks.com
* Netty’s primary building blocks: ***Channels***, ***Callbacks***, ***Futures***, ***Events*** and ***handlers***  
  * think of a Channel as a vehicle for incoming (inbound) and outgoing (outbound) data.   
  * Netty uses callbacks internally when handling events; when a callback is triggered the event can be handled by an implementation of interface ***ChannelHandler***.  
  * This is quite cumbersome, so Netty provides its own implementation, ***ChannelFuture***, for use when an asynchronous operation is executed.  ChannelFuture provides additional methods that allow us to register one or more ***ChannelFutureListener*** instances.   
  * Netty provides an extensive set of predefined handlers that you can use out of the box, including handlers for protocols such as HTTP and SSL/TLS.   
* Under the covers, an ***EventLoop*** is assigned to each Channel to handle all of the events  
* The EventLoop itself is driven by only one thread that handles all of the I/O events for one Channel and does not change during the lifetime of the EventLoop.  
* By default, a handler will forward the invocation of a handler method to the next one in the chain. Therefore, if exceptionCaught() is not implemented somewhere along the chain, exceptions received will travel to the end of the
  ChannelPipeline and will be logged. For this reason, your application should supply at least one ChannelHandler that implements exceptionCaught().  
* Note that the message sent by the server may be received in chunks. That is, if the server sends 5 bytes, there’s no guarantee that all 5 bytes will be received at once. Even for such a small amount of data, the channelRead0() method could be called twice, first with a ByteBuf (Netty’s byte container) holding 3 bytes, and second with a ByteBuf holding 2 bytes. As a stream-oriented protocol, TCP guarantees that the bytes will be received in the order in which they were sent by the server.  
* in Netty 4 all I/O operations and events are handled by the Thread that has been assigned to the ***EventLoop***  
* We stated earlier the importance of not blocking the current I/O thread. We’ll say it again in another way: “Never put a long-running task in the execution queue, because it will block any other task from executing on the same thread.”  
* Once a Channel has been assigned an EventLoop, it will use this EventLoop (and the associated Thread) throughout its lifetime.   
* Here’s a  simple guideline: use ***ByteToMessageDecoder*** if it doesn’t introduce excessive complexity; otherwise, use ***ReplayingDecoder***.  

# Go

```powershell
go env
# GOPROXY=https://proxy.golang.org,direct
go env -w GOPROXY=https://goproxy.cn,direct
```

- Go code is grouped into packages, and packages are grouped into modules.
- Declare a `main` package. In Go, code executed as an application must go in a `main` package.
- The `replace` directive tells Go where to find the `greetings` module, because it isn't published yet.
- To reference a published module, a go.mod file would omit the `replace` directive and use a `require` directive with a tagged version number at the end.

# MySQL实战45讲

```shell
show processlist
```



## Java并发编程

```tex
01 | 可见性、原子性和有序性问题：并发编程Bug的源头
这些年，我们的 CPU、内存、I/O 设备都在不断迭代，不断朝着更快的方向努力。但是，在这个快速发展的过程中，有一个核心矛盾一直存在，就是这三者的速度差异。
源头之一：缓存导致的可见性问题
一个线程对共享变量的修改，另外一个线程能够立刻看到，我们称为可见性。
源头之二：线程切换带来的原子性问题
我们把一个或者多个操作在 CPU 执行的过程中不被中断的特性称为原子性。
源头之三：编译优化带来的有序性问题
只要我们能够深刻理解可见性、原子性、有序性在并发场景下的原理，很多并发 Bug 都是可以理解、可以诊断的。
在介绍可见性、原子性、有序性的时候，特意提到缓存导致的可见性问题，线程切换带来的原子性问题，编译优化带来的有序性问题，其实缓存、线程、编译优化的目的和我们写并发程序的目的是相同的，都是提高程序性能。


```





# 学习书籍

## Docker in Action

10.96.0.0/12

00001010.0011



eyJhbGciOiJIUzUxMiJ9.eyJsb2dpbl91c2VyX2tleSI6IjNlNmM2ZTRhLWQyNzQtNGY1ZC05ODRlLWRkNjYyN2I2YTA3NiJ9.ZSwFhbW_UuMDRaIudwP7WWkHkF4tXPmoJhbR54AbjeDXMdYAzRFcY7sPNiNws1hoSQxuWzgrAndA1sbAN4s85Q

# MD 例子

# 开课吧

```tex
开课吧晶晶 搬砖ing:

方式：登录https://www.kaikeba.com/cert/student
输入10位的证书编号，即可查询和下载；
方式：微信关注"开课吧服务号"，在导航「我的」-「学习证书」中，输入10位的证书编号，即可查询和下载。

开课吧晶晶 搬砖ing:
OJ6GS5CBDX


毕业证书编号： 4C2FP7DNLC
查询地址：https://www.kaikeba.com/cert/student
```

# Spring代码流程

1. org.springframework.context.annotation.**AnnotationConfigApplicationContext**

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    this();
    register(componentClasses);
    refresh(); // <-
}
```

2. org.springframework.context.support.**AbstractApplicationContext**

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```



# VUE

```shell
vue create ruoyi-ui-james

cd ruoyi-ui-james

vue add router
vue add vuex
npm i element-ui

npm i sass sass-loader@10 -D
npm i axios
npm i @smallwei/avue
npm i nprogress
```

