# 一，前期准备	
## 1.各种账号
*注意：如果是真实环境，用户名密码等会有复杂度要求，请酌情建立。
不用统一建立这些账号，后面教程会一一提到

账号 | 用户名 | 密码| 说明
---|---|---|---
centos |root | *****隐去 |安装OS时自带的用户
centos |jw | *****隐去|安装OS时自带的用户
centos |vmail | vmail|专门用于虚拟用户邮件收发的用户
centos |jw1 | jwmail|cyrus-sasl的smtp用户，可以收发邮件，虚拟用户建立后，失效
mysql |root | 123456 |mysql根用户
mysql |postfix| postfix|专门为 postfix准备的用户，数据库也叫  postfix
mysql |webmail | webmail|专门为 webmail 准备的用户，数据库也叫 webmail
postfixadmin |setup用户 | postfix2018|---
postfixadmin |jw1@test.net   | postfix2018|不一定非得是jw1@test.net， 只要是一个邮箱就行
虚拟用户 |user1@test.net | user2018|postfixadmin创建的
虚拟用户 |user2@test.net | user2018|postfixadmin创建的

**HOSTNAME:**
mail.test.net  
*注意：虽然dns对后缀没要求，但是postfix会自动校验后缀合法性。请使用主流后缀.net .com 等等*

**DNS:**
在本机上构建了一套DNS解析的系统，否则会在投递 xxx@test.net 的过程中出现无法解析domain的情况
当然，如果自己已经购买了DNS服务器，直接在把域名挂载DNS上即可，注意要一条A记录，一条MX记录。

## 2.系统安装
CentOS 6.5 安装过程略。  

###网络
实验环境-192.168.2.234

###防火墙

```
# 永久关闭 
chkconfig iptables off
```
在生产环境中，基于安全策略可能并不允许关闭防火墙。但是要保证下列端口畅通。
```
25, 465, 587, 110, 995, 143, 993
```

###SELINUX
```
# 永久关闭selinux 
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

## 3.JDK
JDK环境用于后面测试 发送邮件的jar是否可用。不安装也OK。
```
yum search java|grep jdk
yum install java-1.7.0-openjdk.x86_64
java -v
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-b3626c6c6659160b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4.LAMP环境
用于postfixadmin  和 web端的邮件收发。
*PHP 和 APACHE的安全类设置本文档未提及，但是在真实环境里，需要考虑安全策略。*

### LAMP的准备工作
CentOS 6.5 有时候会找不到一些源，造成安装失败。
```
# 安装下载工具
yum install wget
# 下载
wget http://www.atomicorp.com/installers/atomic
# 安装
sh ./atomic
# 更新yum源
yum check-update
```

### 安装apache
```
yum install httpd
# 安装完成后，启动apache
/etc/init.d/httpd start
# 设置为开机启动
chkconfig httpd on
```
### 安装mysql
```
yum install mysql mysql-server
#安装完成后，启动mysql
/etc/init.d/mysqld start
#设置为开机启动
chkconfig mysqld on
```
### 安装php
```
#检查当前安装的PHP包
yum list installed | grep php
#如果有安装的PHP包，先删除他们, 如:
yum remove php.x86_64 php-cli.x86_64 php-common.x86_64
#配置安装包源:
rpm -Uvh http://mirror.webtatic.com/yum/el6/latest.rpm
#执行安装，分下面几部
yum -y install php56w.x86_64
yum -y --enablerepo=webtatic install php56w-devel
yum -y install php56w-gd.x86_64 php56w-ldap.x86_64 php56w-mbstring.x86_64 php56w-mcrypt.x86_64 php56w-mysql.x86_64 php56w-pdo.x86_64 php56w-opcache.x86_64
yum -y install php56w-imap
yum install php-dom
```

制作一个首页
```
cd /var/www/html	
vi index.php	
<?php	
phpinfo();	
?>	
```

### 安装telnet 并激活服务
```
yum install xinetd
yum install telnet
yum install telnet -server

cd /etc/xinetd.d  
cp telnet telnet.20180327
vi telnet
#修改 disable = yes 为 disable = no
#需要激活xinetd服务
service xinetd restart
```

### 重启mysql和apache
```
#重启MySql
/etc/init.d/mysqld restart
#重启Apache 
/etc/init.d/httpd restart
```


### 测试PHP首页
```
telnet 192.168.2.234 80
GET /index. php HTTP/1.1
HOST: 127.0.0.1 (连续两个回车)
```
![](https://upload-images.jianshu.io/upload_images/3611412-f207571cd1fbd37e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

或者
浏览器内地址栏输入 ：http://192.168.2.234
![image.png](https://upload-images.jianshu.io/upload_images/3611412-3e7b17fd4bf06100.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### mysql建表建用户
####强制指定mysql的ROOT用户的密码为123456
```
/etc/init.d/mysqld stop 
# 执行如下命令
mysqld_safe --user=mysql --skip-grant-tables --skip-networking
mysql -u root mysql
mysql>UPDATE user SET Password=PASSWORD('123456') where USER='root';
mysql>FLUSH PRIVILEGES;
mysql>quit
/etc/init.d/mysqld restart 
#如果失败，可以整机重启
```

####建库，建用户
```
 mysql -uroot -p
123456	
mysql>create database postfix default character set utf8 collate utf8_bin;
mysql>grant all on postfix.* to 'postfix'@'%' identified by 'postfix';
mysql>flush privileges;
mysql>quit
```

####测试新用户
```
mysql -upostfix -ppostfix
```

## 5.hostname
```
vim /etc/sysconfig/network
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-b71e6c861977fe16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
根据实际情况 酌情填写。
重启完成之后，

```
hostname
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-df7eeb72d3875422.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 6.创建邮件专用用户

为了后续的管理方便，我们使用系统的一个用户映射为对邮件服务器的用户，该用户对于postfix来说是一个虚拟用户。
所在在此之前，我们需要添加一个不能登录到系统的，并且指定用户组和用户ID的特殊用户vmail，该用户也可以自行定义。

```
groupadd -g 5000 vmail
useradd -g vmail -u 5000 -s /sbin/nologin vmail
cat /etc/passwd |grep vmail
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-faa44a797da50d67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 7.构建本机DNS

注意：如果已经在第三方上购买了域名，就不需要再在本机搭建DNS服务器了。
注意：如果是局域网内已经有DNS服务器了，把mail.test.net绑定即可，要绑一条A记录，一条MX记录
```
yum install -y bind bind-chroot
先备份
cp /etc/named.conf /etc/named.conf.20180326
vim /etc/named.conf 
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-e526b6c820ff5ee4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
cd /var/named
cp named.localhost test.net.zone
cp named.localhost 2.168.192.in-addr.arpa.zone
```
接下来修改配置文件。
```
vim test.net.zone
```
![](https://upload-images.jianshu.io/upload_images/3611412-171a8e02653b0be4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
vim 2.168.192.in-addr.arpa.zone
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-86e0015451f82ba9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
# 赋权限
chown :named test.net.zone
chown :named 2.168.192.in-addr.arpa.zone

#验证配置是否正确
named-checkconf /etc/named.conf
named-checkzone test.net test.net.zone
named-checkzone 2.168.192.in-addr.arpa 2.168.192.in-addr.arpa.zone

#重启服务
service named restart

#把本机IP的DNS指向DNS服务器（本例中就是自己）
vim /etc/sysconfig/network-scripts/ifcfg-eth0
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-3e5fcdd22c002130.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

整机重启。

地址栏中使用 mail.test.net 也是可行的。

**此处要做还原点** 

# 二，邮件服务器构建	
## 1.cyrus-sasl 
cyrus-sasl(Simple Authentication Security Layer)简单认证安全层, SASL主要是用于SMTP认证。而cyrus-sasl在OS里面，saslauthd是其守护进程。  

### 创建
```
yum -y install cyrus-sasl
查看版本
/usr/sbin/saslauthd -v
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-b8dbcda9bb1d84cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
# 配置
vim /etc/sysconfig/saslauthd
# 修改mech
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-69f129547767c92d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
# 如果没有该文件，就创建。
vim /etc/sasl2/smtpd.conf
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-fa9fc0634710143f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 测试
```
useradd jw1
echo 'jwmail' | passwd --stdin jw1
# 切换到新用户
su - jw1
mkdir -p ~/mail/.imap/INBOX

#切换回ROOT
/etc/init.d/saslauthd start
chkconfig saslauthd on

#进行认证测试
testsaslauthd -u jw1 -p 'jwmail'
```

![image.png](https://upload-images.jianshu.io/upload_images/3611412-1336f1200b4af1e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
意味着创建成功。

## 2.postfix
postfix作为发送邮件服务器。

```
yum -y install postfix

#先备份
cp /etc/postfix/main.cf /etc/postfix/main.cf.20180326

#修改配置
vim /etc/postfix/main.cf
#参见 《附件1main.cf》

#测试postfix配置
/etc/init.d/postfix start
chkconfig postfix on
netstat -tunlp
ps -ef |grep postfix

```

查看mail日志，这个功能会经常使用。
```
tail -f /var/log/maillog
```

查看postfix现在生效的配置（挑错时候用）
```
postconf -n
```


## 3.dovecot

```
#安装
yum -y install dovecot dovecot-devel dovecot-mysql pam-devel

#修改配置之前先备份
cp /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.20180326
cp /etc/dovecot/conf.d/10-mail.conf /etc/dovecot/conf.d/10-mail.conf.20180326
cp /etc/dovecot/conf.d/10-master.conf /etc/dovecot/conf.d/10-master.conf.20180326
cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.20180326
cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.20180326
cp /etc/dovecot/conf.d/10-logging.conf /etc/dovecot/conf.d/10-logging.conf.20180326

#进行配置
vim /etc/dovecot/dovecot.conf
>protocols = imap pop3
>listen = *
>!include conf.d/*.conf

vim /etc/dovecot/conf.d/10-auth.conf
>disable_plaintext_auth = no
>auth_mechanisms = plain login
>!include auth-system.conf.ext

vi /etc/dovecot/conf.d/10-mail.conf
>mail_location = maildir:~/Maildir

vi /etc/dovecot/conf.d/10-master.conf
>unix_listener /var/spool/postfix/private/auth { 	
>     mode = 0666 
>	user = postfix 
>	group = postfix
>}

vim /etc/dovecot/conf.d/10-ssl.conf
>ssl = no

#只要是一个合法邮箱即可，否则无法启动，这属于docecot的BUG
vim /etc/dovecot/conf.d/15-lda.conf
>postmaster_address = postmaster@example.com

vim /etc/dovecot/conf.d/10-logging.conf
# Log file to use for informational messages. Defaults to log_path.
>info_log_path = /var/log/dovecot_info.log
# Log file to use for debug messages. Defaults to info_log_path.
>debug_log_path = /var/log/dovecot.debug.log
# 注意：如果后续日志中提示没有写入权限的话，修改其权限即可。

#查看dovecot生效的非默认配置
doveconf -n

/etc/init.d/dovecot start
chkconfig dovecot on
/etc/init.d/portreserve stop
chkconfig portreserve off

```
*注意：上述命令中的portreserve服务相关的两行，这个如果启动的话，你会发现系统重启后dovecot会无法启动，
这是因为portreserve占用了dovecot的端口，所以在此我们禁用portreserve服务。

###简单测试
使用本章开头建立的用户 jw1/jwmail
![image.png](https://upload-images.jianshu.io/upload_images/3611412-a76e4bbbf765ea8c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/3611412-b4c6e8ac2d8e853c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/3611412-bd2e8dae7b1dff19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此处要建立一个还原点。

# 三，虚拟用户配置	

##为什么要配置虚拟用户
如果按照电子邮件的原始定义，邮件是发送到操作系统的用户上的。
但是当使用者越来越多的时候，不可能在一台操作系统上构建特别多的用户。
于是，就产生了虚拟用户的概念

###实体用户
![image.png](https://upload-images.jianshu.io/upload_images/3611412-1dab609178a0b10b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###虚拟用户
![image.png](https://upload-images.jianshu.io/upload_images/3611412-3d88934d6d19b909.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 1.postfix配置虚拟用户

修改main.cf文件
```
#先备份
cp /etc/postfix/main.cf /etc/postfix/main.cf.20180327

#修改配置，增加对虚拟用户的支持
vim /etc/postfix/main.cf
#参见 SHEET 《附件2main.cf》

#其中还需要说明的是我们在此启用的虚拟用户是《1.前期准备》中创建的vmail用户，该用户的id是5000，
#所以在postfix主配置文件会看到vmail的家目录/home/vmail/，以及vmail的id信息5000。
```
修改master.cf文件
```
#接下来会修改大量的配置文件,记得备份
#先统一进行备份
cp /etc/postfix/master.cf /etc/postfix/master.cf.20180327
vim /etc/postfix/master.cf
#添加下面两句
dovecot   unix  –       n       n       –       –       pipe
  flags=DRhu user=vmail:vmail argv=/usr/libexec/dovecot/dovecot-lda -f ${sender} -d ${recipient}
#注意 flags前面有两个半角空格。
```

数据库连接相关文件
连接数据库相关文件有7个，在创建配置文件之前，我们要在/etc/postfix/目录下建立sql目录用来存放这些配置如下：
```
mkdir /etc/postfix/sql/

vim /etc/postfix/sql/mysql_virtual_alias_maps.cf
user = postfix
password = postfix
hosts = localhost
dbname = postfix
query = SELECT goto FROM alias WHERE address='%s' AND active = '1'

vim /etc/postfix/sql/mysql_virtual_alias_domain_maps.cf
user = postfix
password = postfix
hosts = localhost
dbname = postfix
query = SELECT goto FROM alias,alias_domain WHERE alias_domain.alias_domain = '%d' and alias.address = CONCAT('%u', '@', alias_domain.target_domain) AND alias.active = 1 AND alias_domain.active='1'

vim /etc/postfix/sql/mysql_virtual_alias_domain_catchall_maps.cf
user = postfix
password = postfix
hosts = localhost
dbname = postfix
query = SELECT goto FROM alias,alias_domain WHERE alias_domain.alias_domain = '%d' and alias.address = CONCAT('@', alias_domain.target_domain) AND alias.active = 1 AND alias_domain.active='1'

vim /etc/postfix/sql/mysql_virtual_domains_maps.cf
user = postfix
password = postfix
hosts = localhost
dbname = postfix
query = SELECT domain FROM domain WHERE domain='%s' AND active = '1'

vim /etc/postfix/sql/mysql_virtual_mailbox_maps.cf
user = postfix
password = postfix
hosts = localhost
dbname = postfix
query = SELECT maildir FROM mailbox WHERE username='%s' AND active = '1'

vim /etc/postfix/sql/mysql_virtual_alias_domain_mailbox_maps.cf
user = postfix
password = postfix
hosts = localhost
dbname = postfix
query = SELECT maildir FROM mailbox,alias_domain WHERE alias_domain.alias_domain = '%d' and mailbox.username = CONCAT('%u','@',alias_domain.target_domain) AND mailbox.active = 1 AND alias_domain.active='1'

vim /etc/postfix/sql/mysql_virtual_mailbox_limit_maps.cf
user = postfix
password = postfix
hosts = localhost
dbname = postfix
query = SELECT quota FROM mailbox WHERE username='%s' AND active = '1'

```
测试sasl与postfix集成
```
telnet mail.test.net 25
ehlo test.net

#该命令需要手工输入，而如果出现250-AUTH PLAIN LOGIN和250-AUTH=PLAIN LOGIN两行，则说明postfix已经正确启用smtp认证。
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-40966a1784af23cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 2.dovecot配置虚拟用户
因为要修改的文件很多，所以先统一备份
```
cp /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.20180327
cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.20180327
cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.20180327
cp /etc/dovecot/conf.d/10-mail.conf  /etc/dovecot/conf.d/10-mail.conf.20180327
cp /etc/dovecot/conf.d/10-logging.conf /etc/dovecot/conf.d/10-logging.conf.20180327
cp /etc/dovecot/conf.d/10-master.conf /etc/dovecot/conf.d/10-master.conf.20180327
cp /etc/dovecot/conf.d/15-lda.conf /etc/dovecot/conf.d/15-lda.conf.20180327
```
修改dovecot.conf文件
```
vim /etc/dovecot/dovecot.conf
#修改
protocols = imap pop3
listen = *
!include conf.d/*.conf

#追加
passdb {
  driver = sql
  args = /etc/dovecot/dovecot-sql.conf.ext
}
userdb {
  driver = static
  args = uid=5000 gid=5000 home=/home/vmail/%d/%n
}
#log
auth_debug_passwords=yes
mail_debug=yes
auth_verbose=yes
auth_verbose_passwords=plain
```
修改10-auth.conf文件
```
vim /etc/dovecot/conf.d/10-auth.conf
#修改红色
disable_plaintext_auth = no
auth_mechanisms = plain login cram-md5
!include auth-system.conf.ext
```
修改10-mail.conf文件
```
vim /etc/dovecot/conf.d/10-mail.conf
#修改
mail_location = maildir:/home/vmail/%d/%n
mbox_write_locks = fcntl
```
修改10-master.conf文件
10-master.conf文件定义了dovecot的pop3和imap端口，以及其他的一些信息。
```
vim /etc/dovecot/conf.d/10-master.conf

  inet_listener imap {
    port = 143
  }

  inet_listener pop3 {
    port = 110
  }

service auth {
   unix_listener auth-userdb {
mode = 0600
    user = vmail
    group = vmail
  }
  # Postfix smtp-auth
unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    #group = postfix （注释掉）
  }
}
```
添加dovecot-sql.conf.ext文件
```
vim /etc/dovecot/dovecot-sql.conf.ext												
driver = mysql
connect = host=localhost dbname=postfix user=postfix password=postfix
default_pass_scheme = MD5-CRYPT
password_query = SELECT username AS user,password FROM mailbox WHERE username = '%u' AND active='1'
user_query = SELECT maildir, 5000 AS uid, 5000 AS gid, CONCAT('dict:storage=',floor(quota/1000),' proxy::quota') as quota FROM mailbox WHERE username = '%u' AND active='1'
#注意： proxy::quata 的前面有半角空格	
```

## 3.postfixadmin配置
确保LAMP环境
###安装postfixadmin
```
wget http://nchc.dl.sourceforge.net/project/postfixadmin/postfixadmin/postfixadmin-2.93/postfixadmin-2.93.tar.gz
tar -xf postfixadmin-2.93.tar.gz
mv postfixadmin-2.93 /var/www/html/postfixadmin
chown -R apache:apache /var/www/html/postfixadmin
chmod -R 755 /var/www/html/postfixadmin
```
###配置postfixadmin
```
#先备份
cp /var/www/html/postfixadmin/config.inc.php /var/www/html/postfixadmin/config.inc.php.20180327
vim /var/www/html/postfixadmin/config.inc.php
#下列均需要修改
$CONF['configured'] = true;
$CONF['default_language'] = 'cn';
$CONF['database_type'] = 'mysql';
$CONF['database_host'] = 'localhost';
$CONF['database_user'] = 'postfix';
$CONF['database_password'] = 'postfix';
$CONF['database_name'] = 'postfix';
$CONF['encrypt'] = 'dovecot:CRAM-MD5';
$CONF['aliases'] = '1000';
$CONF['mailboxes'] = '1000';
$CONF['maxquota'] = '1000';
$CONF['fetchmail'] = 'NO';
$CONF['quota'] = 'YES';
$CONF['used_quotas'] = 'YES';
$CONF['new_quota_table'] = 'NO';
```
###启动postfixadmin
```
#启动apache
#如果已经启动，就restart
/etc/init.d/httpd restart
```
浏览器中打开
http://192.168.2.234/postfixadmin/setup.php

该页面是自检页面
![image.png](https://upload-images.jianshu.io/upload_images/3611412-884a53a991e016bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-047268be708ebd30.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-6d44ee7c2a63f33c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-d41119893383501a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
vim /var/www/html/postfixadmin/config.inc.php
# 修改为刚才画面上的值
$CONF['setup_password'] = 'changeme';
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-eae8ee4da15a75ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

回到web页面
![image.png](https://upload-images.jianshu.io/upload_images/3611412-bc262cb7df108c3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-d5baa705f3ca6191.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

登录： http://192.168.2.234/postfixadmin/login.php
![登录](https://upload-images.jianshu.io/upload_images/3611412-ab5bcf0fd36c0999.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-99447bc8fe56f2ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-b27597fb01831779.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-610fd78f8933c1ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-eee131f4c8d3bfdd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
新建两个用户，密码都是user2018
![image.png](https://upload-images.jianshu.io/upload_images/3611412-827509581772b24f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-9da3a5c9541a46ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-189cde28b0aa581e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以在此为该服务器设置一个还原点

# 四，邮件连接测试	
注意是 IMAP，虚拟用户只支持IMAP，即143端口
![image.png](https://upload-images.jianshu.io/upload_images/3611412-e41c39545954e75c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-67a10628513985a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

尝试建立一个不存在的账户的连接，说明服务器的验证生效了。
![image.png](https://upload-images.jianshu.io/upload_images/3611412-c4898eb8bad2b25c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/3611412-928d7e0682472319.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-7aaccb502d7815c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-2ab85feb76532992.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
使用user2 给 user1 回信
![image.png](https://upload-images.jianshu.io/upload_images/3611412-843a32e5d69b4871.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-241a6fd234cb9498.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以在此为该服务器设置一个还原点。

# 五，WEB端邮件系统	
## 1.下载roundcube webmail软件包
```
wget http://jaist.dl.sourceforge.net/project/roundcubemail/roundcubemail/1.1.4/roundcubemail-1.1.4-complete.tar.gz
#解压roundcube webmail软件包，并移动到apache根目录下，如下：
tar -xf roundcubemail-1.1.4-complete.tar.gz
mv roundcubemail-1.1.4 /var/www/html/webmail/
chown -R apache:apache /var/www/html/webmail/
chmod -R 755 /var/www/html/webmail/
ll /var/www/html/webmail/
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-b50a10bf3aa35111.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.配置webmail
```
#安装pear环境，为了下面更新PHP
wget http://pear.php.net/go-pear.phar
php go-pear.phar
```
![image.png](https://upload-images.jianshu.io/upload_images/3611412-2750bd97479e10b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图中间需要敲一个回车。

```
pear channel-update pear.php.net
pear install Auth_SASL Net_SMTP Net_IDNA2-0.1.1 Mail_Mime
```
修改配置：
```
vim /etc/php.ini +889
date.timezone = Asia/Chongqing
#以上修改完毕后，重启apache，使用如下命令：
/etc/init.d/httpd restart
```

建库，建用户
```
mysql -uroot -p 
#输入123456
mysql>create database webmail default character set utf8 collate utf8_bin;
mysql>grant all on webmail.* to 'webmail'@'%' identified by 'webmail';
mysql>flush privileges;
mysql>quit
```
测试新用户
```
mysql -uwebmail -pwebmail
```
## 3.安装roundcube webmail
http://192.168.2.234/webmail/installer/index.php
默认进行环境检验
![image.png](https://upload-images.jianshu.io/upload_images/3611412-c635eb4630461cd7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-4370515bb928f52b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
填写各种配置
![image.png](https://upload-images.jianshu.io/upload_images/3611412-cae5406ab20fd59d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-b185f132034ecd42.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-4eac9c7eaa7f99f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-1cdbc2469afd39f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-b36c2b064531d794.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-af29ba128d9f4aca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
点击【继续】
![image.png](https://upload-images.jianshu.io/upload_images/3611412-b9b16c06984d3ce3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
点击【初始化数据库】
![image.png](https://upload-images.jianshu.io/upload_images/3611412-51de1ad3c0be2792.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-1066335f7ac60b5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-60f110d01f12d145.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
WEB上收发邮件的测试
![image.png](https://upload-images.jianshu.io/upload_images/3611412-df9b2157d7626122.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-47bdaa7ab2b937b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-1e41891272c75ea6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
WEB上的登录测试
![image.png](https://upload-images.jianshu.io/upload_images/3611412-387bc314afba3381.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-296f2853cc70298e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4.测试WEB版
http://192.168.2.234/webmail/
输入之前建立的虚拟用户
![image.png](https://upload-images.jianshu.io/upload_images/3611412-64191dff6bd0f1a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3611412-334da28e3528a140.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

基于安全考虑，请删除 installer/index.php 这个文件。防止他人误操作。
可以在此为该服务器设置一个还原点。

**至此，整个服务器搭建完毕。**

# 六，维护类命令
```
#开启关闭 apache		
/etc/init.d/httpd start	

#开启关闭 mysql				
/etc/init.d/mysqld start
/etc/init.d/mysqld restart #重启  	
/etc/init.d/mysqld stop #停止  	
/etc/init.d/mysqld start #启动 	
		
#开启关闭防火墙		
chkconfig iptables off	
				
#开启关闭postfix		
/etc/init.d/postfix start

#查看发送邮件的Log	
tail -f /var/log/maillog

#列出POSTFIX非默认值的设置数据。
postconf -n

#开启关闭dovecot		
/etc/init.d/dovecot start	
chkconfig dovecot on	
cat /var/log/dovecot_info.log	
cat /var/log/dovecot.debug.log

#查看dovecot生效的非默认配置
doveconf -n
```
		
查看postfix对系统的影响		
	进入 /home/vmail/ 能看到 所有的虚拟域名	
	进入某一个虚拟域名，就能看到所有该域名下的用户

```		
#全局搜索文件		
find / -name  'xxx'	
#查看系统的整体安全LOG		
cat  /var/log/secure	
```

# 七，附件
## 1.附件1main.cf
```
# Global Postfix configuration file. This file lists only a subset
# of all parameters. For the syntax, and for a complete parameter
# list, see the postconf(5) manual page (command: "man 5 postconf").
#
# For common configuration examples, see BASIC_CONFIGURATION_README
# and STANDARD_CONFIGURATION_README. To find these documents, use
# the command "postconf html_directory readme_directory", or go to
# http://www.postfix.org/.
#
# For best results, change no more than 2-3 parameters at a time,
# and test if Postfix still works after every change.

# SOFT BOUNCE
#
# The soft_bounce parameter provides a limited safety net for
# testing.  When soft_bounce is enabled, mail will remain queued that
# would otherwise bounce. This parameter disables locally-generated
# bounces, and prevents the SMTP server from rejecting mail permanently
# (by changing 5xx replies into 4xx replies). However, soft_bounce
# is no cure for address rewriting mistakes or mail routing mistakes.
#
#soft_bounce = no

# LOCAL PATHNAME INFORMATION
#
# The queue_directory specifies the location of the Postfix queue.
# This is also the root directory of Postfix daemons that run chrooted.
# See the files in examples/chroot-setup for setting up Postfix chroot
# environments on different UNIX systems.
#
queue_directory = /var/spool/postfix

# The command_directory parameter specifies the location of all
# postXXX commands.
#
command_directory = /usr/sbin

# The daemon_directory parameter specifies the location of all Postfix
# daemon programs (i.e. programs listed in the master.cf file). This
# directory must be owned by root.
#
daemon_directory = /usr/libexec/postfix

# The data_directory parameter specifies the location of Postfix-writable
# data files (caches, random numbers). This directory must be owned
# by the mail_owner account (see below).
#
data_directory = /var/lib/postfix

# QUEUE AND PROCESS OWNERSHIP
#
# The mail_owner parameter specifies the owner of the Postfix queue
# and of most Postfix daemon processes.  Specify the name of a user
# account THAT DOES NOT SHARE ITS USER OR GROUP ID WITH OTHER ACCOUNTS
# AND THAT OWNS NO OTHER FILES OR PROCESSES ON THE SYSTEM.  In
# particular, don't specify nobody or daemon. PLEASE USE A DEDICATED
# USER.
#
mail_owner = postfix

# The default_privs parameter specifies the default rights used by
# the local delivery agent for delivery to external file or command.
# These rights are used in the absence of a recipient user context.
# DO NOT SPECIFY A PRIVILEGED USER OR THE POSTFIX OWNER.
#
#default_privs = nobody

# INTERNET HOST AND DOMAIN NAMES
# 
# The myhostname parameter specifies the internet hostname of this
# mail system. The default is to use the fully-qualified domain name
# from gethostname(). $myhostname is used as a default value for many
# other configuration parameters.
#
#myhostname = host.domain.tld
#myhostname = virtual.domain.tld
myhostname = mail.test.net

# The mydomain parameter specifies the local internet domain name.
# The default is to use $myhostname minus the first component.
# $mydomain is used as a default value for many other configuration
# parameters.
#
#mydomain = domain.tld
mydomain = test.net

# SENDING MAIL
# 
# The myorigin parameter specifies the domain that locally-posted
# mail appears to come from. The default is to append $myhostname,
# which is fine for small sites.  If you run a domain with multiple
# machines, you should (1) change this to $mydomain and (2) set up
# a domain-wide alias database that aliases each user to
# user@that.users.mailhost.
#
# For the sake of consistency between sender and recipient addresses,
# myorigin also specifies the default domain name that is appended
# to recipient addresses that have no @domain part.
#
#myorigin = $myhostname
myorigin = $mydomain

# RECEIVING MAIL

# The inet_interfaces parameter specifies the network interface
# addresses that this mail system receives mail on.  By default,
# the software claims all active interfaces on the machine. The
# parameter also controls delivery of mail to user@[ip.address].
#
# See also the proxy_interfaces parameter, for network addresses that
# are forwarded to us via a proxy or network address translator.
#
# Note: you need to stop/start Postfix when this parameter changes.
#
inet_interfaces = all
#inet_interfaces = $myhostname
#inet_interfaces = $myhostname, localhost
#inet_interfaces = localhost

# Enable IPv4, and IPv6 if supported
inet_protocols = ipv4

# The proxy_interfaces parameter specifies the network interface
# addresses that this mail system receives mail on by way of a
# proxy or network address translation unit. This setting extends
# the address list specified with the inet_interfaces parameter.
#
# You must specify your proxy/NAT addresses when your system is a
# backup MX host for other domains, otherwise mail delivery loops
# will happen when the primary MX host is down.
#
#proxy_interfaces =
#proxy_interfaces = 1.2.3.4

# The mydestination parameter specifies the list of domains that this
# machine considers itself the final destination for.
#
# These domains are routed to the delivery agent specified with the
# local_transport parameter setting. By default, that is the UNIX
# compatible delivery agent that lookups all recipients in /etc/passwd
# and /etc/aliases or their equivalent.
#
# The default is $myhostname + localhost.$mydomain.  On a mail domain
# gateway, you should also include $mydomain.
#
# Do not specify the names of virtual domains - those domains are
# specified elsewhere (see VIRTUAL_README).
#
# Do not specify the names of domains that this machine is backup MX
# host for. Specify those names via the relay_domains settings for
# the SMTP server, or use permit_mx_backup if you are lazy (see
# STANDARD_CONFIGURATION_README).
#
# The local machine is always the final destination for mail addressed
# to user@[the.net.work.address] of an interface that the mail system
# receives mail on (see the inet_interfaces parameter).
#
# Specify a list of host or domain names, /file/name or type:table
# patterns, separated by commas and/or whitespace. A /file/name
# pattern is replaced by its contents; a type:table is matched when
# a name matches a lookup key (the right-hand side is ignored).
# Continue long lines by starting the next line with whitespace.
#
# See also below, section "REJECTING MAIL FOR UNKNOWN LOCAL USERS".
#
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
#mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
#mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain,
#

# REJECTING MAIL FOR UNKNOWN LOCAL USERS
#
# The local_recipient_maps parameter specifies optional lookup tables
# with all names or addresses of users that are local with respect
# to $mydestination, $inet_interfaces or $proxy_interfaces.
#
# If this parameter is defined, then the SMTP server will reject
# mail for unknown local users. This parameter is defined by default.
#
# To turn off local recipient checking in the SMTP server, specify
# local_recipient_maps = (i.e. empty).
#
# The default setting assumes that you use the default Postfix local
# delivery agent for local delivery. You need to update the
# local_recipient_maps setting if:
#
# - You define $mydestination domain recipients in files other than
#   /etc/passwd, /etc/aliases, or the $virtual_alias_maps files.
#   For example, you define $mydestination domain recipients in    
#   the $virtual_mailbox_maps files.
#
# - You redefine the local delivery agent in master.cf.
#
# - You redefine the "local_transport" setting in main.cf.
#
# - You use the "luser_relay", "mailbox_transport", or "fallback_transport"
#   feature of the Postfix local delivery agent (see local(8)).
#
# Details are described in the LOCAL_RECIPIENT_README file.
#
# Beware: if the Postfix SMTP server runs chrooted, you probably have
# to access the passwd file via the proxymap service, in order to
# overcome chroot restrictions. The alternative, having a copy of
# the system passwd file in the chroot jail is just not practical.
#
# The right-hand side of the lookup tables is conveniently ignored.
# In the left-hand side, specify a bare username, an @domain.tld
# wild-card, or specify a user@domain.tld address.
# 
#local_recipient_maps = unix:passwd.byname $alias_maps
#local_recipient_maps = proxy:unix:passwd.byname $alias_maps
local_recipient_maps =

# The unknown_local_recipient_reject_code specifies the SMTP server
# response code when a recipient domain matches $mydestination or
# ${proxy,inet}_interfaces, while $local_recipient_maps is non-empty
# and the recipient address or address local-part is not found.
#
# The default setting is 550 (reject mail) but it is safer to start
# with 450 (try again later) until you are certain that your
# local_recipient_maps settings are OK.
#
unknown_local_recipient_reject_code = 550

# TRUST AND RELAY CONTROL

# The mynetworks parameter specifies the list of "trusted" SMTP
# clients that have more privileges than "strangers".
#
# In particular, "trusted" SMTP clients are allowed to relay mail
# through Postfix.  See the smtpd_recipient_restrictions parameter
# in postconf(5).
#
# You can specify the list of "trusted" network addresses by hand
# or you can let Postfix do it for you (which is the default).
#
# By default (mynetworks_style = subnet), Postfix "trusts" SMTP
# clients in the same IP subnetworks as the local machine.
# On Linux, this does works correctly only with interfaces specified
# with the "ifconfig" command.
# 
# Specify "mynetworks_style = class" when Postfix should "trust" SMTP
# clients in the same IP class A/B/C networks as the local machine.
# Don't do this with a dialup site - it would cause Postfix to "trust"
# your entire provider's network.  Instead, specify an explicit
# mynetworks list by hand, as described below.
#  
# Specify "mynetworks_style = host" when Postfix should "trust"
# only the local machine.
# 
#mynetworks_style = class
#mynetworks_style = subnet
#mynetworks_style = host

# Alternatively, you can specify the mynetworks list by hand, in
# which case Postfix ignores the mynetworks_style setting.
#
# Specify an explicit list of network/netmask patterns, where the
# mask specifies the number of bits in the network part of a host
# address.
#
# You can also specify the absolute pathname of a pattern file instead
# of listing the patterns here. Specify type:table for table-based lookups
# (the value on the table right-hand side is not used).
#
#mynetworks = 168.100.189.0/28, 127.0.0.0/8
#mynetworks = $config_directory/mynetworks
#mynetworks = hash:/etc/postfix/network_table

# The relay_domains parameter restricts what destinations this system will
# relay mail to.  See the smtpd_recipient_restrictions description in
# postconf(5) for detailed information.
#
# By default, Postfix relays mail
# - from "trusted" clients (IP address matches $mynetworks) to any destination,
# - from "untrusted" clients to destinations that match $relay_domains or
#   subdomains thereof, except addresses with sender-specified routing.
# The default relay_domains value is $mydestination.
# 
# In addition to the above, the Postfix SMTP server by default accepts mail
# that Postfix is final destination for:
# - destinations that match $inet_interfaces or $proxy_interfaces,
# - destinations that match $mydestination
# - destinations that match $virtual_alias_domains,
# - destinations that match $virtual_mailbox_domains.
# These destinations do not need to be listed in $relay_domains.
# 
# Specify a list of hosts or domains, /file/name patterns or type:name
# lookup tables, separated by commas and/or whitespace.  Continue
# long lines by starting the next line with whitespace. A file name
# is replaced by its contents; a type:name table is matched when a
# (parent) domain appears as lookup key.
#
# NOTE: Postfix will not automatically forward mail for domains that
# list this system as their primary or backup MX host. See the
# permit_mx_backup restriction description in postconf(5).
#
#relay_domains = $mydestination

# INTERNET OR INTRANET

# The relayhost parameter specifies the default host to send mail to
# when no entry is matched in the optional transport(5) table. When
# no relayhost is given, mail is routed directly to the destination.
#
# On an intranet, specify the organizational domain name. If your
# internal DNS uses no MX records, specify the name of the intranet
# gateway host instead.
#
# In the case of SMTP, specify a domain, host, host:port, [host]:port,
# [address] or [address]:port; the form [host] turns off MX lookups.
#
# If you're connected via UUCP, see also the default_transport parameter.
#
#relayhost = $mydomain
#relayhost = [gateway.my.domain]
#relayhost = [mailserver.isp.tld]
#relayhost = uucphost
#relayhost = [an.ip.add.ress]

# REJECTING UNKNOWN RELAY USERS
#
# The relay_recipient_maps parameter specifies optional lookup tables
# with all addresses in the domains that match $relay_domains.
#
# If this parameter is defined, then the SMTP server will reject
# mail for unknown relay users. This feature is off by default.
#
# The right-hand side of the lookup tables is conveniently ignored.
# In the left-hand side, specify an @domain.tld wild-card, or specify
# a user@domain.tld address.
# 
#relay_recipient_maps = hash:/etc/postfix/relay_recipients

# INPUT RATE CONTROL
#
# The in_flow_delay configuration parameter implements mail input
# flow control. This feature is turned on by default, although it
# still needs further development (it's disabled on SCO UNIX due
# to an SCO bug).
# 
# A Postfix process will pause for $in_flow_delay seconds before
# accepting a new message, when the message arrival rate exceeds the
# message delivery rate. With the default 100 SMTP server process
# limit, this limits the mail inflow to 100 messages a second more
# than the number of messages delivered per second.
# 
# Specify 0 to disable the feature. Valid delays are 0..10.
# 
#in_flow_delay = 1s

# ADDRESS REWRITING
#
# The ADDRESS_REWRITING_README document gives information about
# address masquerading or other forms of address rewriting including
# username->Firstname.Lastname mapping.

# ADDRESS REDIRECTION (VIRTUAL DOMAIN)
#
# The VIRTUAL_README document gives information about the many forms
# of domain hosting that Postfix supports.

# "USER HAS MOVED" BOUNCE MESSAGES
#
# See the discussion in the ADDRESS_REWRITING_README document.

# TRANSPORT MAP
#
# See the discussion in the ADDRESS_REWRITING_README document.

# ALIAS DATABASE
#
# The alias_maps parameter specifies the list of alias databases used
# by the local delivery agent. The default list is system dependent.
#
# On systems with NIS, the default is to search the local alias
# database, then the NIS alias database. See aliases(5) for syntax
# details.
# 
# If you change the alias database, run "postalias /etc/aliases" (or
# wherever your system stores the mail alias file), or simply run
# "newaliases" to build the necessary DBM or DB file.
#
# It will take a minute or so before changes become visible.  Use
# "postfix reload" to eliminate the delay.
#
#alias_maps = dbm:/etc/aliases
alias_maps = hash:/etc/aliases
#alias_maps = hash:/etc/aliases, nis:mail.aliases
#alias_maps = netinfo:/aliases

# The alias_database parameter specifies the alias database(s) that
# are built with "newaliases" or "sendmail -bi".  This is a separate
# configuration parameter, because alias_maps (see above) may specify
# tables that are not necessarily all under control by Postfix.
#
#alias_database = dbm:/etc/aliases
#alias_database = dbm:/etc/mail/aliases
alias_database = hash:/etc/aliases
#alias_database = hash:/etc/aliases, hash:/opt/majordomo/aliases

# ADDRESS EXTENSIONS (e.g., user+foo)
#
# The recipient_delimiter parameter specifies the separator between
# user names and address extensions (user+foo). See canonical(5),
# local(8), relocated(5) and virtual(5) for the effects this has on
# aliases, canonical, virtual, relocated and .forward file lookups.
# Basically, the software tries user+foo and .forward+foo before
# trying user and .forward.
#
#recipient_delimiter = +

# DELIVERY TO MAILBOX
#
# The home_mailbox parameter specifies the optional pathname of a
# mailbox file relative to a user's home directory. The default
# mailbox file is /var/spool/mail/user or /var/mail/user.  Specify
# "Maildir/" for qmail-style delivery (the / is required).
#
#home_mailbox = Mailbox
home_mailbox = Maildir/
 
# The mail_spool_directory parameter specifies the directory where
# UNIX-style mailboxes are kept. The default setting depends on the
# system type.
#
#mail_spool_directory = /var/mail
#mail_spool_directory = /var/spool/mail

# The mailbox_command parameter specifies the optional external
# command to use instead of mailbox delivery. The command is run as
# the recipient with proper HOME, SHELL and LOGNAME environment settings.
# Exception:  delivery for root is done as $default_user.
#
# Other environment variables of interest: USER (recipient username),
# EXTENSION (address extension), DOMAIN (domain part of address),
# and LOCAL (the address localpart).
#
# Unlike other Postfix configuration parameters, the mailbox_command
# parameter is not subjected to $parameter substitutions. This is to
# make it easier to specify shell syntax (see example below).
#
# Avoid shell meta characters because they will force Postfix to run
# an expensive shell process. Procmail alone is expensive enough.
#
# IF YOU USE THIS TO DELIVER MAIL SYSTEM-WIDE, YOU MUST SET UP AN
# ALIAS THAT FORWARDS MAIL FOR ROOT TO A REAL USER.
#
#mailbox_command = /some/where/procmail
#mailbox_command = /some/where/procmail -a "$EXTENSION"

# The mailbox_transport specifies the optional transport in master.cf
# to use after processing aliases and .forward files. This parameter
# has precedence over the mailbox_command, fallback_transport and
# luser_relay parameters.
#
# Specify a string of the form transport:nexthop, where transport is
# the name of a mail delivery transport defined in master.cf.  The
# :nexthop part is optional. For more details see the sample transport
# configuration file.
#
# NOTE: if you use this feature for accounts not in the UNIX password
# file, then you must update the "local_recipient_maps" setting in
# the main.cf file, otherwise the SMTP server will reject mail for    
# non-UNIX accounts with "User unknown in local recipient table".
#
#mailbox_transport = lmtp:unix:/var/lib/imap/socket/lmtp

# If using the cyrus-imapd IMAP server deliver local mail to the IMAP
# server using LMTP (Local Mail Transport Protocol), this is prefered
# over the older cyrus deliver program by setting the
# mailbox_transport as below:
#
# mailbox_transport = lmtp:unix:/var/lib/imap/socket/lmtp
#
# The efficiency of LMTP delivery for cyrus-imapd can be enhanced via
# these settings.
#
# local_destination_recipient_limit = 300
# local_destination_concurrency_limit = 5
#
# Of course you should adjust these settings as appropriate for the
# capacity of the hardware you are using. The recipient limit setting
# can be used to take advantage of the single instance message store
# capability of Cyrus. The concurrency limit can be used to control
# how many simultaneous LMTP sessions will be permitted to the Cyrus
# message store. 
#
# To use the old cyrus deliver program you have to set:
#mailbox_transport = cyrus

# The fallback_transport specifies the optional transport in master.cf
# to use for recipients that are not found in the UNIX passwd database.
# This parameter has precedence over the luser_relay parameter.
#
# Specify a string of the form transport:nexthop, where transport is
# the name of a mail delivery transport defined in master.cf.  The
# :nexthop part is optional. For more details see the sample transport
# configuration file.
#
# NOTE: if you use this feature for accounts not in the UNIX password
# file, then you must update the "local_recipient_maps" setting in
# the main.cf file, otherwise the SMTP server will reject mail for    
# non-UNIX accounts with "User unknown in local recipient table".
#
#fallback_transport = lmtp:unix:/var/lib/imap/socket/lmtp
#fallback_transport =

# The luser_relay parameter specifies an optional destination address
# for unknown recipients.  By default, mail for unknown@$mydestination,
# unknown@[$inet_interfaces] or unknown@[$proxy_interfaces] is returned
# as undeliverable.
#
# The following expansions are done on luser_relay: $user (recipient
# username), $shell (recipient shell), $home (recipient home directory),
# $recipient (full recipient address), $extension (recipient address
# extension), $domain (recipient domain), $local (entire recipient
# localpart), $recipient_delimiter. Specify ${name?value} or
# ${name:value} to expand value only when $name does (does not) exist.
#
# luser_relay works only for the default Postfix local delivery agent.
#
# NOTE: if you use this feature for accounts not in the UNIX password
# file, then you must specify "local_recipient_maps =" (i.e. empty) in
# the main.cf file, otherwise the SMTP server will reject mail for    
# non-UNIX accounts with "User unknown in local recipient table".
#
#luser_relay = $user@other.host
#luser_relay = $local@other.host
#luser_relay = admin+$local
  
# JUNK MAIL CONTROLS
# 
# The controls listed here are only a very small subset. The file
# SMTPD_ACCESS_README provides an overview.

# The header_checks parameter specifies an optional table with patterns
# that each logical message header is matched against, including
# headers that span multiple physical lines.
#
# By default, these patterns also apply to MIME headers and to the
# headers of attached messages. With older Postfix versions, MIME and
# attached message headers were treated as body text.
#
# For details, see "man header_checks".
#
#header_checks = regexp:/etc/postfix/header_checks

# FAST ETRN SERVICE
#
# Postfix maintains per-destination logfiles with information about
# deferred mail, so that mail can be flushed quickly with the SMTP
# "ETRN domain.tld" command, or by executing "sendmail -qRdomain.tld".
# See the ETRN_README document for a detailed description.
# 
# The fast_flush_domains parameter controls what destinations are
# eligible for this service. By default, they are all domains that
# this server is willing to relay mail to.
# 
#fast_flush_domains = $relay_domains

# SHOW SOFTWARE VERSION OR NOT
#
# The smtpd_banner parameter specifies the text that follows the 220
# code in the SMTP server's greeting banner. Some people like to see
# the mail version advertised. By default, Postfix shows no version.
#
# You MUST specify $myhostname at the start of the text. That is an
# RFC requirement. Postfix itself does not care.
#
#smtpd_banner = $myhostname ESMTP $mail_name
#smtpd_banner = $myhostname ESMTP $mail_name ($mail_version)

# PARALLEL DELIVERY TO THE SAME DESTINATION
#
# How many parallel deliveries to the same user or domain? With local
# delivery, it does not make sense to do massively parallel delivery
# to the same user, because mailbox updates must happen sequentially,
# and expensive pipelines in .forward files can cause disasters when
# too many are run at the same time. With SMTP deliveries, 10
# simultaneous connections to the same domain could be sufficient to
# raise eyebrows.
# 
# Each message delivery transport has its XXX_destination_concurrency_limit
# parameter.  The default is $default_destination_concurrency_limit for
# most delivery transports. For the local delivery agent the default is 2.

#local_destination_concurrency_limit = 2
#default_destination_concurrency_limit = 20

# DEBUGGING CONTROL
#
# The debug_peer_level parameter specifies the increment in verbose
# logging level when an SMTP client or server host name or address
# matches a pattern in the debug_peer_list parameter.
#
debug_peer_level = 2

# The debug_peer_list parameter specifies an optional list of domain
# or network patterns, /file/name patterns or type:name tables. When
# an SMTP client or server host name or address matches a pattern,
# increase the verbose logging level by the amount specified in the
# debug_peer_level parameter.
#
#debug_peer_list = 127.0.0.1
#debug_peer_list = some.domain

# The debugger_command specifies the external command that is executed
# when a Postfix daemon program is run with the -D option.
#
# Use "command .. & sleep 5" so that the debugger can attach before
# the process marches on. If you use an X-based debugger, be sure to
# set up your XAUTHORITY environment variable before starting Postfix.
#
debugger_command =



# If you can't use X, use this to capture the call stack when a
# daemon crashes. The result is in a file in the configuration
# directory, and is named after the process name and the process ID.
#
# debugger_command =
#
#
#
#
# Another possibility is to run gdb under a detached screen session.
# To attach to the screen sesssion, su root and run "screen -r
# <id_string>" where <id_string> uniquely matches one of the detached
# sessions (from "screen -list").
#
# debugger_command =
#
#
#

# INSTALL-TIME CONFIGURATION INFORMATION
#
# The following parameters are used when installing a new Postfix version.
# 
# sendmail_path: The full pathname of the Postfix sendmail command.
# This is the Sendmail-compatible mail posting interface.
# 
sendmail_path = /usr/sbin/sendmail.postfix

# newaliases_path: The full pathname of the Postfix newaliases command.
# This is the Sendmail-compatible command to build alias databases.
#
newaliases_path = /usr/bin/newaliases.postfix

# mailq_path: The full pathname of the Postfix mailq command.  This
# is the Sendmail-compatible mail queue listing command.
# 
mailq_path = /usr/bin/mailq.postfix

# setgid_group: The group for mail submission and queue management
# commands.  This must be a group name with a numerical group ID that
# is not shared with other accounts, not even with the Postfix account.
#
setgid_group = postdrop

# html_directory: The location of the Postfix HTML documentation.
#
html_directory = no

# manpage_directory: The location of the Postfix on-line manual pages.
#
manpage_directory = /usr/share/man

# sample_directory: The location of the Postfix sample configuration files.
# This parameter is obsolete as of Postfix 2.1.
#
sample_directory = /usr/share/doc/postfix-2.6.6/samples

# readme_directory: The location of the Postfix README files.
#
readme_directory = /usr/share/doc/postfix-2.6.6/README_FILES

#启用SMTP认证
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_application_name = smtpd
smtpd_sasl_auth_enable = yes
smtpd_sasl_local_domain = $myhostname
broken_sasl_auth_clients = yes
smtpd_recipient_restrictions = permit_mynetworks,permit_sasl_authenticated,reject_unauth_destination,reject_unknown_sender_domain
smtpd_sasl_security_restrictions = permit_mynetworks,permit_sasl_authenticated,reject_unauth_destination
smtpd_client_restrictions = permit_sasl_authenticated
smtpd_sasl_security_options = noanonymous
proxy_read_maps = $local_recipient_maps $mydestination $virtual_alias_maps $virtual_alias_domains $virtual_mailbox_maps $virtual_mailbox_domains $relay_recipient_maps $relay_domains $canonical_maps $sender_canonical_maps $recipient_canonical_maps $relocated_maps $transport_maps $mynetworks $virtual_mailbox_limit_maps
```
## 2.附件2main.cf
```
# Global Postfix configuration file. This file lists only a subset
# of all parameters. For the syntax, and for a complete parameter
# list, see the postconf(5) manual page (command: "man 5 postconf").
#
# For common configuration examples, see BASIC_CONFIGURATION_README
# and STANDARD_CONFIGURATION_README. To find these documents, use
# the command "postconf html_directory readme_directory", or go to
# http://www.postfix.org/.
#
# For best results, change no more than 2-3 parameters at a time,
# and test if Postfix still works after every change.

# SOFT BOUNCE
#
# The soft_bounce parameter provides a limited safety net for
# testing.  When soft_bounce is enabled, mail will remain queued that
# would otherwise bounce. This parameter disables locally-generated
# bounces, and prevents the SMTP server from rejecting mail permanently
# (by changing 5xx replies into 4xx replies). However, soft_bounce
# is no cure for address rewriting mistakes or mail routing mistakes.
#
#soft_bounce = no

# LOCAL PATHNAME INFORMATION
#
# The queue_directory specifies the location of the Postfix queue.
# This is also the root directory of Postfix daemons that run chrooted.
# See the files in examples/chroot-setup for setting up Postfix chroot
# environments on different UNIX systems.
#
queue_directory = /var/spool/postfix

# The command_directory parameter specifies the location of all
# postXXX commands.
#
command_directory = /usr/sbin

# The daemon_directory parameter specifies the location of all Postfix
# daemon programs (i.e. programs listed in the master.cf file). This
# directory must be owned by root.
#
daemon_directory = /usr/libexec/postfix

# The data_directory parameter specifies the location of Postfix-writable
# data files (caches, random numbers). This directory must be owned
# by the mail_owner account (see below).
#
data_directory = /var/lib/postfix

# QUEUE AND PROCESS OWNERSHIP
#
# The mail_owner parameter specifies the owner of the Postfix queue
# and of most Postfix daemon processes.  Specify the name of a user
# account THAT DOES NOT SHARE ITS USER OR GROUP ID WITH OTHER ACCOUNTS
# AND THAT OWNS NO OTHER FILES OR PROCESSES ON THE SYSTEM.  In
# particular, don't specify nobody or daemon. PLEASE USE A DEDICATED
# USER.
#
mail_owner = postfix

# The default_privs parameter specifies the default rights used by
# the local delivery agent for delivery to external file or command.
# These rights are used in the absence of a recipient user context.
# DO NOT SPECIFY A PRIVILEGED USER OR THE POSTFIX OWNER.
#
#default_privs = nobody

# INTERNET HOST AND DOMAIN NAMES
# 
# The myhostname parameter specifies the internet hostname of this
# mail system. The default is to use the fully-qualified domain name
# from gethostname(). $myhostname is used as a default value for many
# other configuration parameters.
#
#myhostname = host.domain.tld
#myhostname = virtual.domain.tld
myhostname = mail.test.net

# The mydomain parameter specifies the local internet domain name.
# The default is to use $myhostname minus the first component.
# $mydomain is used as a default value for many other configuration
# parameters.
#
#mydomain = domain.tld
mydomain = test.net

# SENDING MAIL
# 
# The myorigin parameter specifies the domain that locally-posted
# mail appears to come from. The default is to append $myhostname,
# which is fine for small sites.  If you run a domain with multiple
# machines, you should (1) change this to $mydomain and (2) set up
# a domain-wide alias database that aliases each user to
# user@that.users.mailhost.
#
# For the sake of consistency between sender and recipient addresses,
# myorigin also specifies the default domain name that is appended
# to recipient addresses that have no @domain part.
#
#myorigin = $myhostname
myorigin = $mydomain

# RECEIVING MAIL

# The inet_interfaces parameter specifies the network interface
# addresses that this mail system receives mail on.  By default,
# the software claims all active interfaces on the machine. The
# parameter also controls delivery of mail to user@[ip.address].
#
# See also the proxy_interfaces parameter, for network addresses that
# are forwarded to us via a proxy or network address translator.
#
# Note: you need to stop/start Postfix when this parameter changes.
#
inet_interfaces = all
#inet_interfaces = $myhostname
#inet_interfaces = $myhostname, localhost
#inet_interfaces = localhost

# Enable IPv4, and IPv6 if supported
inet_protocols = ipv4

# The proxy_interfaces parameter specifies the network interface
# addresses that this mail system receives mail on by way of a
# proxy or network address translation unit. This setting extends
# the address list specified with the inet_interfaces parameter.
#
# You must specify your proxy/NAT addresses when your system is a
# backup MX host for other domains, otherwise mail delivery loops
# will happen when the primary MX host is down.
#
#proxy_interfaces =
#proxy_interfaces = 1.2.3.4

# The mydestination parameter specifies the list of domains that this
# machine considers itself the final destination for.
#
# These domains are routed to the delivery agent specified with the
# local_transport parameter setting. By default, that is the UNIX
# compatible delivery agent that lookups all recipients in /etc/passwd
# and /etc/aliases or their equivalent.
#
# The default is $myhostname + localhost.$mydomain.  On a mail domain
# gateway, you should also include $mydomain.
#
# Do not specify the names of virtual domains - those domains are
# specified elsewhere (see VIRTUAL_README).
#
# Do not specify the names of domains that this machine is backup MX
# host for. Specify those names via the relay_domains settings for
# the SMTP server, or use permit_mx_backup if you are lazy (see
# STANDARD_CONFIGURATION_README).
#
# The local machine is always the final destination for mail addressed
# to user@[the.net.work.address] of an interface that the mail system
# receives mail on (see the inet_interfaces parameter).
#
# Specify a list of host or domain names, /file/name or type:table
# patterns, separated by commas and/or whitespace. A /file/name
# pattern is replaced by its contents; a type:table is matched when
# a name matches a lookup key (the right-hand side is ignored).
# Continue long lines by starting the next line with whitespace.
#
# See also below, section "REJECTING MAIL FOR UNKNOWN LOCAL USERS".
#
mydestination = 
#mydestination = mail.test.net
#mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
#mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain,
#

# REJECTING MAIL FOR UNKNOWN LOCAL USERS
#
# The local_recipient_maps parameter specifies optional lookup tables
# with all names or addresses of users that are local with respect
# to $mydestination, $inet_interfaces or $proxy_interfaces.
#
# If this parameter is defined, then the SMTP server will reject
# mail for unknown local users. This parameter is defined by default.
#
# To turn off local recipient checking in the SMTP server, specify
# local_recipient_maps = (i.e. empty).
#
# The default setting assumes that you use the default Postfix local
# delivery agent for local delivery. You need to update the
# local_recipient_maps setting if:
#
# - You define $mydestination domain recipients in files other than
#   /etc/passwd, /etc/aliases, or the $virtual_alias_maps files.
#   For example, you define $mydestination domain recipients in    
#   the $virtual_mailbox_maps files.
#
# - You redefine the local delivery agent in master.cf.
#
# - You redefine the "local_transport" setting in main.cf.
#
# - You use the "luser_relay", "mailbox_transport", or "fallback_transport"
#   feature of the Postfix local delivery agent (see local(8)).
#
# Details are described in the LOCAL_RECIPIENT_README file.
#
# Beware: if the Postfix SMTP server runs chrooted, you probably have
# to access the passwd file via the proxymap service, in order to
# overcome chroot restrictions. The alternative, having a copy of
# the system passwd file in the chroot jail is just not practical.
#
# The right-hand side of the lookup tables is conveniently ignored.
# In the left-hand side, specify a bare username, an @domain.tld
# wild-card, or specify a user@domain.tld address.
# 
#local_recipient_maps = unix:passwd.byname $alias_maps
#local_recipient_maps = proxy:unix:passwd.byname $alias_maps
local_recipient_maps =

# The unknown_local_recipient_reject_code specifies the SMTP server
# response code when a recipient domain matches $mydestination or
# ${proxy,inet}_interfaces, while $local_recipient_maps is non-empty
# and the recipient address or address local-part is not found.
#
# The default setting is 550 (reject mail) but it is safer to start
# with 450 (try again later) until you are certain that your
# local_recipient_maps settings are OK.
#
unknown_local_recipient_reject_code = 550

# TRUST AND RELAY CONTROL

# The mynetworks parameter specifies the list of "trusted" SMTP
# clients that have more privileges than "strangers".
#
# In particular, "trusted" SMTP clients are allowed to relay mail
# through Postfix.  See the smtpd_recipient_restrictions parameter
# in postconf(5).
#
# You can specify the list of "trusted" network addresses by hand
# or you can let Postfix do it for you (which is the default).
#
# By default (mynetworks_style = subnet), Postfix "trusts" SMTP
# clients in the same IP subnetworks as the local machine.
# On Linux, this does works correctly only with interfaces specified
# with the "ifconfig" command.
# 
# Specify "mynetworks_style = class" when Postfix should "trust" SMTP
# clients in the same IP class A/B/C networks as the local machine.
# Don't do this with a dialup site - it would cause Postfix to "trust"
# your entire provider's network.  Instead, specify an explicit
# mynetworks list by hand, as described below.
#  
# Specify "mynetworks_style = host" when Postfix should "trust"
# only the local machine.
# 
#mynetworks_style = class
#mynetworks_style = subnet
#mynetworks_style = host

# Alternatively, you can specify the mynetworks list by hand, in
# which case Postfix ignores the mynetworks_style setting.
#
# Specify an explicit list of network/netmask patterns, where the
# mask specifies the number of bits in the network part of a host
# address.
#
# You can also specify the absolute pathname of a pattern file instead
# of listing the patterns here. Specify type:table for table-based lookups
# (the value on the table right-hand side is not used).
#
mynetworks = 168.100.189.0/28, 127.0.0.0/8
#mynetworks = $config_directory/mynetworks
#mynetworks = hash:/etc/postfix/network_table

# The relay_domains parameter restricts what destinations this system will
# relay mail to.  See the smtpd_recipient_restrictions description in
# postconf(5) for detailed information.
#
# By default, Postfix relays mail
# - from "trusted" clients (IP address matches $mynetworks) to any destination,
# - from "untrusted" clients to destinations that match $relay_domains or
#   subdomains thereof, except addresses with sender-specified routing.
# The default relay_domains value is $mydestination.
# 
# In addition to the above, the Postfix SMTP server by default accepts mail
# that Postfix is final destination for:
# - destinations that match $inet_interfaces or $proxy_interfaces,
# - destinations that match $mydestination
# - destinations that match $virtual_alias_domains,
# - destinations that match $virtual_mailbox_domains.
# These destinations do not need to be listed in $relay_domains.
# 
# Specify a list of hosts or domains, /file/name patterns or type:name
# lookup tables, separated by commas and/or whitespace.  Continue
# long lines by starting the next line with whitespace. A file name
# is replaced by its contents; a type:name table is matched when a
# (parent) domain appears as lookup key.
#
# NOTE: Postfix will not automatically forward mail for domains that
# list this system as their primary or backup MX host. See the
# permit_mx_backup restriction description in postconf(5).
#
#relay_domains = $mydestination

# INTERNET OR INTRANET

# The relayhost parameter specifies the default host to send mail to
# when no entry is matched in the optional transport(5) table. When
# no relayhost is given, mail is routed directly to the destination.
#
# On an intranet, specify the organizational domain name. If your
# internal DNS uses no MX records, specify the name of the intranet
# gateway host instead.
#
# In the case of SMTP, specify a domain, host, host:port, [host]:port,
# [address] or [address]:port; the form [host] turns off MX lookups.
#
# If you're connected via UUCP, see also the default_transport parameter.
#
#relayhost = $mydomain
#relayhost = [gateway.my.domain]
#relayhost = [mailserver.isp.tld]
#relayhost = uucphost
#relayhost = [an.ip.add.ress]

# REJECTING UNKNOWN RELAY USERS
#
# The relay_recipient_maps parameter specifies optional lookup tables
# with all addresses in the domains that match $relay_domains.
#
# If this parameter is defined, then the SMTP server will reject
# mail for unknown relay users. This feature is off by default.
#
# The right-hand side of the lookup tables is conveniently ignored.
# In the left-hand side, specify an @domain.tld wild-card, or specify
# a user@domain.tld address.
# 
#relay_recipient_maps = hash:/etc/postfix/relay_recipients

# INPUT RATE CONTROL
#
# The in_flow_delay configuration parameter implements mail input
# flow control. This feature is turned on by default, although it
# still needs further development (it's disabled on SCO UNIX due
# to an SCO bug).
# 
# A Postfix process will pause for $in_flow_delay seconds before
# accepting a new message, when the message arrival rate exceeds the
# message delivery rate. With the default 100 SMTP server process
# limit, this limits the mail inflow to 100 messages a second more
# than the number of messages delivered per second.
# 
# Specify 0 to disable the feature. Valid delays are 0..10.
# 
#in_flow_delay = 1s

# ADDRESS REWRITING
#
# The ADDRESS_REWRITING_README document gives information about
# address masquerading or other forms of address rewriting including
# username->Firstname.Lastname mapping.

# ADDRESS REDIRECTION (VIRTUAL DOMAIN)
#
# The VIRTUAL_README document gives information about the many forms
# of domain hosting that Postfix supports.

# "USER HAS MOVED" BOUNCE MESSAGES
#
# See the discussion in the ADDRESS_REWRITING_README document.

# TRANSPORT MAP
#
# See the discussion in the ADDRESS_REWRITING_README document.

# ALIAS DATABASE
#
# The alias_maps parameter specifies the list of alias databases used
# by the local delivery agent. The default list is system dependent.
#
# On systems with NIS, the default is to search the local alias
# database, then the NIS alias database. See aliases(5) for syntax
# details.
# 
# If you change the alias database, run "postalias /etc/aliases" (or
# wherever your system stores the mail alias file), or simply run
# "newaliases" to build the necessary DBM or DB file.
#
# It will take a minute or so before changes become visible.  Use
# "postfix reload" to eliminate the delay.
#
#alias_maps = dbm:/etc/aliases
alias_maps = hash:/etc/aliases
#alias_maps = hash:/etc/aliases, nis:mail.aliases
#alias_maps = netinfo:/aliases

# The alias_database parameter specifies the alias database(s) that
# are built with "newaliases" or "sendmail -bi".  This is a separate
# configuration parameter, because alias_maps (see above) may specify
# tables that are not necessarily all under control by Postfix.
#
#alias_database = dbm:/etc/aliases
#alias_database = dbm:/etc/mail/aliases
alias_database = hash:/etc/aliases
#alias_database = hash:/etc/aliases, hash:/opt/majordomo/aliases

# ADDRESS EXTENSIONS (e.g., user+foo)
#
# The recipient_delimiter parameter specifies the separator between
# user names and address extensions (user+foo). See canonical(5),
# local(8), relocated(5) and virtual(5) for the effects this has on
# aliases, canonical, virtual, relocated and .forward file lookups.
# Basically, the software tries user+foo and .forward+foo before
# trying user and .forward.
#
#recipient_delimiter = +

# DELIVERY TO MAILBOX
#
# The home_mailbox parameter specifies the optional pathname of a
# mailbox file relative to a user's home directory. The default
# mailbox file is /var/spool/mail/user or /var/mail/user.  Specify
# "Maildir/" for qmail-style delivery (the / is required).
#
#home_mailbox = Mailbox
home_mailbox = Maildir/
 
# The mail_spool_directory parameter specifies the directory where
# UNIX-style mailboxes are kept. The default setting depends on the
# system type.
#
#mail_spool_directory = /var/mail
#mail_spool_directory = /var/spool/mail

# The mailbox_command parameter specifies the optional external
# command to use instead of mailbox delivery. The command is run as
# the recipient with proper HOME, SHELL and LOGNAME environment settings.
# Exception:  delivery for root is done as $default_user.
#
# Other environment variables of interest: USER (recipient username),
# EXTENSION (address extension), DOMAIN (domain part of address),
# and LOCAL (the address localpart).
#
# Unlike other Postfix configuration parameters, the mailbox_command
# parameter is not subjected to $parameter substitutions. This is to
# make it easier to specify shell syntax (see example below).
#
# Avoid shell meta characters because they will force Postfix to run
# an expensive shell process. Procmail alone is expensive enough.
#
# IF YOU USE THIS TO DELIVER MAIL SYSTEM-WIDE, YOU MUST SET UP AN
# ALIAS THAT FORWARDS MAIL FOR ROOT TO A REAL USER.
#
#mailbox_command = /some/where/procmail
#mailbox_command = /some/where/procmail -a "$EXTENSION"

# The mailbox_transport specifies the optional transport in master.cf
# to use after processing aliases and .forward files. This parameter
# has precedence over the mailbox_command, fallback_transport and
# luser_relay parameters.
#
# Specify a string of the form transport:nexthop, where transport is
# the name of a mail delivery transport defined in master.cf.  The
# :nexthop part is optional. For more details see the sample transport
# configuration file.
#
# NOTE: if you use this feature for accounts not in the UNIX password
# file, then you must update the "local_recipient_maps" setting in
# the main.cf file, otherwise the SMTP server will reject mail for    
# non-UNIX accounts with "User unknown in local recipient table".
#
#mailbox_transport = lmtp:unix:/var/lib/imap/socket/lmtp

# If using the cyrus-imapd IMAP server deliver local mail to the IMAP
# server using LMTP (Local Mail Transport Protocol), this is prefered
# over the older cyrus deliver program by setting the
# mailbox_transport as below:
#
# mailbox_transport = lmtp:unix:/var/lib/imap/socket/lmtp
#
# The efficiency of LMTP delivery for cyrus-imapd can be enhanced via
# these settings.
#
# local_destination_recipient_limit = 300
# local_destination_concurrency_limit = 5
#
# Of course you should adjust these settings as appropriate for the
# capacity of the hardware you are using. The recipient limit setting
# can be used to take advantage of the single instance message store
# capability of Cyrus. The concurrency limit can be used to control
# how many simultaneous LMTP sessions will be permitted to the Cyrus
# message store. 
#
# To use the old cyrus deliver program you have to set:
#mailbox_transport = cyrus

# The fallback_transport specifies the optional transport in master.cf
# to use for recipients that are not found in the UNIX passwd database.
# This parameter has precedence over the luser_relay parameter.
#
# Specify a string of the form transport:nexthop, where transport is
# the name of a mail delivery transport defined in master.cf.  The
# :nexthop part is optional. For more details see the sample transport
# configuration file.
#
# NOTE: if you use this feature for accounts not in the UNIX password
# file, then you must update the "local_recipient_maps" setting in
# the main.cf file, otherwise the SMTP server will reject mail for    
# non-UNIX accounts with "User unknown in local recipient table".
#
#fallback_transport = lmtp:unix:/var/lib/imap/socket/lmtp
#fallback_transport =

# The luser_relay parameter specifies an optional destination address
# for unknown recipients.  By default, mail for unknown@$mydestination,
# unknown@[$inet_interfaces] or unknown@[$proxy_interfaces] is returned
# as undeliverable.
#
# The following expansions are done on luser_relay: $user (recipient
# username), $shell (recipient shell), $home (recipient home directory),
# $recipient (full recipient address), $extension (recipient address
# extension), $domain (recipient domain), $local (entire recipient
# localpart), $recipient_delimiter. Specify ${name?value} or
# ${name:value} to expand value only when $name does (does not) exist.
#
# luser_relay works only for the default Postfix local delivery agent.
#
# NOTE: if you use this feature for accounts not in the UNIX password
# file, then you must specify "local_recipient_maps =" (i.e. empty) in
# the main.cf file, otherwise the SMTP server will reject mail for    
# non-UNIX accounts with "User unknown in local recipient table".
#
#luser_relay = $user@other.host
#luser_relay = $local@other.host
#luser_relay = admin+$local
  
# JUNK MAIL CONTROLS
# 
# The controls listed here are only a very small subset. The file
# SMTPD_ACCESS_README provides an overview.

# The header_checks parameter specifies an optional table with patterns
# that each logical message header is matched against, including
# headers that span multiple physical lines.
#
# By default, these patterns also apply to MIME headers and to the
# headers of attached messages. With older Postfix versions, MIME and
# attached message headers were treated as body text.
#
# For details, see "man header_checks".
#
#header_checks = regexp:/etc/postfix/header_checks

# FAST ETRN SERVICE
#
# Postfix maintains per-destination logfiles with information about
# deferred mail, so that mail can be flushed quickly with the SMTP
# "ETRN domain.tld" command, or by executing "sendmail -qRdomain.tld".
# See the ETRN_README document for a detailed description.
# 
# The fast_flush_domains parameter controls what destinations are
# eligible for this service. By default, they are all domains that
# this server is willing to relay mail to.
# 
#fast_flush_domains = $relay_domains

# SHOW SOFTWARE VERSION OR NOT
#
# The smtpd_banner parameter specifies the text that follows the 220
# code in the SMTP server's greeting banner. Some people like to see
# the mail version advertised. By default, Postfix shows no version.
#
# You MUST specify $myhostname at the start of the text. That is an
# RFC requirement. Postfix itself does not care.
#
#smtpd_banner = $myhostname ESMTP $mail_name
#smtpd_banner = $myhostname ESMTP $mail_name ($mail_version)

# PARALLEL DELIVERY TO THE SAME DESTINATION
#
# How many parallel deliveries to the same user or domain? With local
# delivery, it does not make sense to do massively parallel delivery
# to the same user, because mailbox updates must happen sequentially,
# and expensive pipelines in .forward files can cause disasters when
# too many are run at the same time. With SMTP deliveries, 10
# simultaneous connections to the same domain could be sufficient to
# raise eyebrows.
# 
# Each message delivery transport has its XXX_destination_concurrency_limit
# parameter.  The default is $default_destination_concurrency_limit for
# most delivery transports. For the local delivery agent the default is 2.

#local_destination_concurrency_limit = 2
#default_destination_concurrency_limit = 20

# DEBUGGING CONTROL
#
# The debug_peer_level parameter specifies the increment in verbose
# logging level when an SMTP client or server host name or address
# matches a pattern in the debug_peer_list parameter.
#
debug_peer_level = 2

# The debug_peer_list parameter specifies an optional list of domain
# or network patterns, /file/name patterns or type:name tables. When
# an SMTP client or server host name or address matches a pattern,
# increase the verbose logging level by the amount specified in the
# debug_peer_level parameter.
#
#debug_peer_list = 127.0.0.1
#debug_peer_list = some.domain

# The debugger_command specifies the external command that is executed
# when a Postfix daemon program is run with the -D option.
#
# Use "command .. & sleep 5" so that the debugger can attach before
# the process marches on. If you use an X-based debugger, be sure to
# set up your XAUTHORITY environment variable before starting Postfix.
#
debugger_command =



# If you can't use X, use this to capture the call stack when a
# daemon crashes. The result is in a file in the configuration
# directory, and is named after the process name and the process ID.
#
# debugger_command =
#
#
#
#
# Another possibility is to run gdb under a detached screen session.
# To attach to the screen sesssion, su root and run "screen -r
# <id_string>" where <id_string> uniquely matches one of the detached
# sessions (from "screen -list").
#
# debugger_command =
#
#
#

# INSTALL-TIME CONFIGURATION INFORMATION
#
# The following parameters are used when installing a new Postfix version.
# 
# sendmail_path: The full pathname of the Postfix sendmail command.
# This is the Sendmail-compatible mail posting interface.
# 
sendmail_path = /usr/sbin/sendmail.postfix

# newaliases_path: The full pathname of the Postfix newaliases command.
# This is the Sendmail-compatible command to build alias databases.
#
newaliases_path = /usr/bin/newaliases.postfix

# mailq_path: The full pathname of the Postfix mailq command.  This
# is the Sendmail-compatible mail queue listing command.
# 
mailq_path = /usr/bin/mailq.postfix

# setgid_group: The group for mail submission and queue management
# commands.  This must be a group name with a numerical group ID that
# is not shared with other accounts, not even with the Postfix account.
#
setgid_group = postdrop

# html_directory: The location of the Postfix HTML documentation.
#
html_directory = no

# manpage_directory: The location of the Postfix on-line manual pages.
#
manpage_directory = /usr/share/man

# sample_directory: The location of the Postfix sample configuration files.
# This parameter is obsolete as of Postfix 2.1.
#
sample_directory = /usr/share/doc/postfix-2.6.6/samples

# readme_directory: The location of the Postfix README files.
#
readme_directory = /usr/share/doc/postfix-2.6.6/README_FILES

#SMTP
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_application_name = smtpd
smtpd_sasl_auth_enable = yes
smtpd_sasl_local_domain = $myhostname
broken_sasl_auth_clients = yes
smtpd_recipient_restrictions = permit_mynetworks,permit_sasl_authenticated,reject_unauth_destination,reject_unknown_sender_domain
smtpd_sasl_security_restrictions = permit_mynetworks,permit_sasl_authenticated,reject_unauth_destination
smtpd_client_restrictions = permit_sasl_authenticated
smtpd_sasl_security_options = noanonymous
proxy_read_maps = $local_recipient_maps $mydestination $virtual_alias_maps $virtual_alias_domains $virtual_mailbox_maps $virtual_mailbox_domains $relay_recipient_maps $relay_domains $canonical_maps $sender_canonical_maps $recipient_canonical_maps $relocated_maps $transport_maps $mynetworks $virtual_mailbox_limit_maps

#SET virtual USER
virtual_mailbox_base = /home/vmail/
virtual_mailbox_domains = proxy:mysql:/etc/postfix/sql/mysql_virtual_domains_maps.cf
virtual_alias_maps =
  proxy:mysql:/etc/postfix/sql/mysql_virtual_alias_maps.cf,
  proxy:mysql:/etc/postfix/sql/mysql_virtual_alias_domain_maps.cf,
  proxy:mysql:/etc/postfix/sql/mysql_virtual_alias_domain_catchall_maps.cf
virtual_mailbox_maps =
  proxy:mysql:/etc/postfix/sql/mysql_virtual_mailbox_maps.cf,
  proxy:mysql:/etc/postfix/sql/mysql_virtual_alias_domain_mailbox_maps.cf
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000
virtual_transport = virtual
dovecot_destination_recipient_limit = 1
```
