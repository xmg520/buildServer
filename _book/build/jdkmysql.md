# 一、环境变量
> 不懂是啥-->[点我]()

```
（1） /etc/profile： 此文件为系统的每个用户设置环境信息,当用户第一次登录时,该文件被执行. 并从/etc/profile.d目录的配置文件中搜集shell的设置。

（2） /etc/bashrc: 为每一个运行bash shell的用户执行此文件.当bash shell被打开时,该文件被读取（即每次新开一个终端，都会执行bashrc）。

（3） ~/.bash_profile: 每个用户都可使用该文件输入专用于自己使用的shell信息,当用户登录时,该文件仅仅执行一次。默认情况下,设置一些环境变量,执行用户的.bashrc文件。

（4） ~/.bashrc: 该文件包含专用于你的bash shell的bash信息,当登录时以及每次打开新的shell时,该该文件被读取。

（5） ~/.bash_logout: 当每次退出系统(退出bash shell)时,执行该文件. 另外,/etc/profile中设定的变量(全局)的可以作用于任何用户,而~/.bashrc等中设定的变量(局部)只能继承 /etc/profile中的变量,他们是"父子"关系。


（6） ~/.bash_profile: 是交互式、login 方式进入 bash 运行的~/.bashrc 是交互式 non-login 方式进入 bash 运行的通常二者设置大致相同，所以通常前者会调用后者。

执行顺序为： /etc/profile -> (~/.bash_profile | ~/.bash_login | ~/.profile) -> ~/.bashrc -> /etc/bashrc -> ~/.bash_logout

环境变量：
    想对所有用户生效 全局: /etc/profile
    想对当前用户生效 用户: ~/.bashrc

```

### 扩展
centos7默认安装jdk1.7版本，若配置需卸载原配置

> 查看内置JDK  

rpm -qa | grep jdk  

![](http://tmp.wyjsjxh.com/201906142355_391.png)

> 卸载内置JDK  

yum remove xxx

## 二、添加java和hadoop用户变量(!!查找规律)
解压命令：`tar -zxf xxx.tar.gz -C /usr/local/`
> 所有组件均需要配置 XXX_HOME 和 PATH  

`XXX_HOME:`组件解压后文件存放路径  
`PATH:`可执行文件路径`:`分割(bin/sbin)

编辑`~/.bashrc`:
```
export JAVA_HOME=/usr/local/java
export PATH=$PATH:$JAVA_HOME/bin

export HADOOP_HOME=/usr/local/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```
运行:`source ~/.bashrc`更新环境变量



### 成功截图
![](http://tmp.wyjsjxh.com/201906142359_636.png)

----------
# 三、CentOS7 离线安装mysql-5.7.16
## 1 . 安装新版mysql前，需将系统自带的mariadb-lib卸载
```linux
[root@slave mytmp]# rpm -qa|grep mariadb
mariadb-libs-5.5.44-2.el7.centos.x86_64
[root@slave mytmp]# rpm -e --nodeps mariadb-libs-5.5.44-2.el7.centos.x86_64
```
## 2 . 解压mysql
```
[root@slave mytmp]# tar -zxf mysql-5.7.16-1.el7.x86_64.rpm-bundle.tar
[root@slave mytmp]# ls
mysql-5.7.16-1.el7.x86_64.rpm-bundle.tar                 mysql-community-libs-5.7.16-1.el7.x86_64.rpm
mysql-community-client-5.7.16-1.el7.x86_64.rpm           mysql-community-libs-compat-5.7.16-1.el7.x86_64.rpm
mysql-community-common-5.7.16-1.el7.x86_64.rpm           mysql-community-minimal-debuginfo-5.7.16-1.el7.x86_64.rpm
mysql-community-devel-5.7.16-1.el7.x86_64.rpm            mysql-community-server-5.7.16-1.el7.x86_64.rpm
mysql-community-embedded-5.7.16-1.el7.x86_64.rpm         mysql-community-server-minimal-5.7.16-1.el7.x86_64.rpm
mysql-community-embedded-compat-5.7.16-1.el7.x86_64.rpm  mysql-community-test-5.7.16-1.el7.x86_64.rpm
mysql-community-embedded-devel-5.7.16-1.el7.x86_64.rpm
```
## 3 . 使用rpm -ivh命令依次进行安装
```
rpm -ivh mysql-community-common-5.7.16-1.el7.x86_64.rpm

rpm -ivh mysql-community-libs-5.7.16-1.el7.x86_64.rpm 

rpm -ivh mysql-community-client-5.7.16-1.el7.x86_64.rpm
```
```
安装mysql-community-server-5.7.16-1.el7.x86_64.rpm 前需要安装libaio-0.3.107-10.el6.x86_64.rpm

下载地址：
http://mirror.centos.org/centos/6/os/x86_64/Packages/libaio-0.3.107-10.el6.x86_64.rpm

安装libaio库：
rpm -ivh libaio-0.3.107-10.el6.x86_64.rpm（若在有网情况下可执行yum install libaio）

安装mysql-community-server：
rpm -ivh mysql-community-server-5.7.16-1.el7.x86_64.rpm
```
## 4 . 初始化数据库
```
// 指定datadir, 执行后会生成~/.mysql_secret密码文件
[root@slave mytmp]# mysql_install_db --datadir=/var/lib/m

// 初始化，执行生会在/var/log/mysqld.log生成随机密码
[root@slave mytmp]# mysqld --initialize
```
## 5 . 更改mysql数据库目录的所属用户及其所属组，并启动mysql数据库
```
[root@slave mytmp]# chown mysql:mysql /var/lib/mysql -R
[root@slave mytmp]# systemctl start mysqld.service
```

## 6 . 登录到mysql，更改root用户的密码
```
// password 通过 cat ~/.mysql_secret 命令可以查看初始密码

[root@slave mytmp]# mysql -uroot -p
Enter password: 

mysql> set password=password('1234');
```
## 7 . 创建用户，及作权限分配
```
mysql> CREATE USER 'zz'@'%' IDENTIFIED BY '1234'; 

mysql> GRANT ALL PRIVILEGES ON *.* TO 'zz'@'%';

mysql> FULSH PRIVILEGES;
```
## 8 . 远程登陆授权
```
mysql> grant all privileges on *.* to 'root'@'%' identified by '1234' with grant option;

mysql> FLUSH PRIVILEGES;
```
## 9 . 设置mysql开机启动
```
// 检查是否已经是开机启动
systemctl list-unit-files | grep mysqld

// 开机启动
systemctl enable mysqld.service
```