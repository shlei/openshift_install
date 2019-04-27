openshift 安装

参考：

https://docs.openshift.com/container-platform/3.11/install/disconnected_install.html
http://ksoong.org/docs/content/openshift/install/

安装版本3.11.69过程：

1. 操作系统安装。安装版本CentOS7-1804

2. 设备规划 
192.168.39.124  master   master.mec.cmcc
192.168.39.125  infra       infra.mec.cmcc
192.168.39.126  node1    node1.mec.cmcc
192.168.39.127  node2    node2.mec.cmcc
#192.168.39.128  storage  storage.mec.cmcc dns

192.168.39.135 yum （已有）
192.168.39.129 registry.redhat.ren （已有）

3. 修改yum源 [master]
[ftp]
baseurl=ftp://192.168.39.135/data
gpgcheck=0
name=ftp
3.1 清除yum缓存及检查yum源
yum clean all
yum repolist

4. linux hosts文件配置(/etc/host)  [master]
添加：
192.168.39.124  master.mec.cmcc   master
192.168.39.125  infra.mec.cmcc       infra
192.168.39.126  node1.mec.cmcc    node1
192.168.39.127  node2.mec.cmcc    node2
#192.168.39.128  storage.mec.cmcc  storage
192.168.39.129  registry.redhat.ren  registry

5. 配置hostname [all] (hostname 与 hostname -f返回一致)
192.168.39.124  master.mec.cmcc
hostnamectl set-hostname master.mec.cmcc

192.168.39.125  infra.mec.cmcc
hostnamectl set-hostname infra.mec.cmcc

192.168.39.126  node1.mec.cmcc
hostnamectl set-hostname node1.mec.cmcc

192.168.39.127  node2.mec.cmcc
hostnamectl set-hostname node2.mec.cmcc

#192.168.39.128  storage.mec.cmcc
#hostnamectl set-hostname storage.mec.cmcc

6.1 安装openshift-ansible [master]  使用ansible 2.6
yum -y install ansible-2.6.13-1.el7ae
yum -y install openshift-ansible

6. 2 配置ansible hosts文件 [master]
/etc/ansible/hosts添加：
[nodes]
192.168.39.124
192.168.39.125
192.168.39.126
192.168.39.127
#192.168.39.128

7. 配置ssh免密登录 (root/cmcc@123) [master]
ssh-keygen
ssh-copy-id   master.mec.cmcc
ssh-copy-id   infra.mec.cmcc
ssh-copy-id   node1.mec.cmcc
ssh-copy-id   node2.mec.cmcc
#ssh-copy-id   storage.mec.cmcc

(ssh-copy-id  192.168.39.124
ssh-copy-id  192.168.39.125
ssh-copy-id  192.168.39.126
ssh-copy-id  192.168.39.127
ssh-copy-id  192.168.39.128)

8.1 将修改的yum源文件通过ansible拷贝到所有设备 [master]
ansible nodes -m copy -a "src=/etc/yum.repos.d/ftp.repo dest=/etc/yum.repos.d/"
8.2 将原有yum源备份移动备份到/etc/yum.repos.d/bak中：
ansible nodes -m raw -a "mkdir /etc/yum.repos.d/bak"
ansible nodes -m raw -a "mv C* /etc/yum.repos.d/bak/"
8.3 执行命令检查yum源是否正确
ansible nodes -m raw -a "yum clean all"
ansible nodes -m raw -a "yum repolist"

9. 将etc/hosts文件拷贝到其他节点上 【master】
ansible nodes -m copy -a "src=/etc/hosts dest=/etc/ backup=yes"

10.1 升级 yum update [master]
ansible nodes -m raw -a "yum update -y"

10.2 安装基础包 
基础包：wget git net-tools bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct yum-utils vim tree lrzsz unzip docker htop

ansible nodes -m raw -a "yum -y install wget git net-tools bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct yum-utils vim tree lrzsz unzip docker htop"

11.1  docker 配置：[master]
在/etc/sysconfig/docker中添加
INSECURE_REGISTRY='--insecure-registry registry.redhat.ren'
INSECURE_REGISTRY='--insecure-registry 192.168.39.129'

11.2 通过ansible 将docker配置文件拷贝到其他节点
ansible nodes -m copy -a "src=/etc/sysconfig/docker dest=/etc/sysconfig/ backup=yes"

11.3  docker启动
ansible nodes -m raw -a "systemctl enable docker"
ansible nodes -m raw -a "systemctl start docker"
测试docker：
ansible nodes -m raw -a "docker pull registry.redhat.ren/shilei/nginx:latest"
ansible nodes -m raw -a "docker images"

12.  重启系统
ansible nodes -m raw -a "docker images"

13.1 安装DNS： dnsmasq [dns]
yum -y install dnsmasq

13.2 配置dnsmasq [dns]
在/etc/dnsmasq.d/openshift-cluster.conf下添加：
local=/mec.cmcc/
address=/.apps.mec.cmcc/192.168.39.125
address=/master.mec.cmcc/192.168.39.124
address=/infra.mec.cmcc/192.168.39.125
address=/node1.mec.cmcc/192.168.39.126
address=/node2.mec.cmcc/192.168.39.127

systemctl enable dnsmasq
systemctl start dnsmasq

firewall-cmd --permanent --add-service=dns
firewall-cmd --reload

13.3 配置dns 【nodes】
ansible nodes -m raw -a "nmcli connection modify eno2 ipv4.dns 192.168.39.128"

13. 新建ansible hosts配置文件。

14.1 安装openshift
ansible-playbook -v -i hosts-v1 /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml

ansible-playbook -v -i hosts-v1 /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml

14.2 卸载openshift
ansible-playbook -i hosts-v1 /usr/share/ansible/openshift-ansible/playbooks/adhoc/uninstall.yml
