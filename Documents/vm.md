# 软件安装

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
# 上传jdk-8u251-linux-x64.tar.gz
tar xzf jdk-8u251-linux-x64.tar.gz -C /usr/local/
vim /etc/profile.d/java.sh

#!/bin/bash
JAVA_HOME=/usr/local/jdk1.8.0_251
PATH=$JAVA_HOME/bin:$PATH
export JAVA_HOME PATH

source /etc/profile
```

## Maven

```shell
wget https://mirrors.bfsu.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
tar zxvf apache-maven-3.6.3-bin.tar.gz -C /usr/local/
vim /etc/profile.d/maven.sh

#!/bin/bash
M2_HOME=/usr/local/apache-maven-3.6.3
PATH=$M2_HOME/bin:$PATH
export M2_HOME PATH

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

```

## Nginx

```shell
yum -y install gcc gcc-c++ pcre-devel openssl-devel
./configure
make && make install

cat <<EOF | sudo tee /lib/systemd/system/nginx.service
[Unit]
Description=nginx service
After=network.target 
   
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

## Keepalived

```shell
yum -y gcc openssl-devel libnl3-devel net-snmp-devel
wget https://www.keepalived.org/software/keepalived-2.0.20.tar.gz
tar zxf keepalived-2.0.20.tar.gz
cd keepalived-2.0.20
./configure --prefix=/usr/local/keepalived
make && make install

mkdir /etc/keepalived
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/

cp /usr/local/keepalived/sbin/keepalived /usr/sbin/
cp keepalived/etc/init.d/keepalived /etc/init.d/

#以下是官方文档方法
yum install gcc openssl-devel libnl3-devel net-snmp-devel
curl --progress http://keepalived.org/software/keepalived-1.2.15.tar.gz | tar xz
cd keepalived-1.2.15
./configure --prefix=/usr/local/keepalived-1.2.15
make
sudo make install
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
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart Docker
systemctl daemon-reload
systemctl restart docker
sudo systemctl enable docker

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

sudo systemctl enable --now kubelet

# 6. Creating a single control-plane cluster with kubeadm (这一步只在master执行)
# Initializing your control-plane node
kubeadm init \
--apiserver-advertise-address 192.168.96.10 \
--pod-network-cidr 10.244.0.0/16 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.19.1


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
```

### ingress-nginx

```json
# https://github.com/kubernetes/ingress-nginx/blob/nginx-0.30.0/docs/deploy/index.md 参考文档
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml
```







## ElasticSearch

```shell
# 注意不能是root用户
tar -xzf elasticsearch-7.9.0-linux-x86_64.tar.gz
cd elasticsearch-7.9.0/
./bin/elasticsearch -d

[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

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
2. It turns out that the shell does not perform filename substitution when assigning values to variables. Therefore, **x=\***assigns the single character * to x. 
3. This order of evaluation is important. Remember, first the shell does ***variable substitution***, then does ***filename substitution***, and then parses the line into arguments.
4. There are also quite a few functions applicable to variables within this curly brace notation, including extracting subsets, assigning values if the variable is currently unassigned, and more. 
5. But because filename substitution is *not* done inside double quotes, * is then safely passed to echo as the value to be displayed
6. When a backslash is the last character of a line of input, the shell treats it as a line continuation character
7. back quote: its purpose is to tell the shell to replace the enclosed command with its output
8. preferred $(...) construct for ***command substitution***
9. These special variables—more formally known as ***positional parameters*** because they’re based on the position of the value in the command line—are assigned after the shell has done its normal command-line processing (that is, I/O redirection, variable substitution, filename substitution, and so on).
10. you can use the exec command to *replace the* *current program* (db) *with the new one* (/bin/sh). This means that it would now be pointless to have any commands follow the exec because they’ll never be executed.
11. That is, you can use the same special characters for specifying the patterns in a case as you can with filename substitution. For example, ? can be used to specify any single character; * can be used to specify zero or more occurrences of any character; and [...] can be used to specify any single character enclosed between the brackets
12. the shell treats loops as if they were mini-programs of their own, so whatever appears after block closing statements like done, fi and esac can have redirects, be put into background with the ampersand or even form part of a pipeline of commands
13. The notation >& specifies output redirection to a file associated with the *file descriptor* that follows. 

# MySQL实战45讲

```shell
show processlist
```

# 学习书籍

## Docker in Action

10.96.0.0/12

00001010.0011



eyJhbGciOiJIUzUxMiJ9.eyJsb2dpbl91c2VyX2tleSI6IjNlNmM2ZTRhLWQyNzQtNGY1ZC05ODRlLWRkNjYyN2I2YTA3NiJ9.ZSwFhbW_UuMDRaIudwP7WWkHkF4tXPmoJhbR54AbjeDXMdYAzRFcY7sPNiNws1hoSQxuWzgrAndA1sbAN4s85Q

# MD 例子

啊手动阀
asdfasdf
asdfasdf

asdfadsf

# 1

## 2

### 3

#### 4

##### 5

###### 6

> ads

* asf
* asdf
* asdf

1. asdf
2. 2

- [ ] bbb
- [x] ccc

```yam
---

```

```sequence
participant Alice
participant Bob

Alice->Bob: Hello Bob, how are you?
Note right of Bob: Bob think
Bob-->Alice: I am good thanks!
Alice->>Bob: aaa


```

```sequence
participant TM
participant RM1
# participant TC

Note left of TM: 发起全局事务(@GlobalTransactional)
Note left of TM: 执行本地事务(可选)
TM->RM1: 调用其他微服务
Note right of RM1: 执行业务
RM1-->TM: 返回执行结果
Note left of TM: 如果返回结果显示执行错误，则抛异常，回滚全局事务
Note left of TM: 执行本地事务(可选)
Note left of TM: 如果本地事务错误，则抛异常，回滚全局事务

```

```flow
st=>start: index
op=>operation: 申请
op2=>operation: 结果页
op3=>operation: 查询本地
i1=>inputoutput: bid入库
i2=>inputoutput: 填写个人信息
c1=>condition: 检查登录
c2=>condition: 登录
c3=>condition: 查询本地记录
c4=>condition: 检测状态
c5=>operation: 风控审核
e=>end

st->op->c1()
c1(no)->c2(yes)->op()
c1(yes)->c3(no)->i1(right)->i2(right)->c5()->op2->e
c1(yes)->c3(yes)->c4(no)->i2
c1(yes)->c3(yes)->c4(yes)->op3->op2
c3()->e
```



