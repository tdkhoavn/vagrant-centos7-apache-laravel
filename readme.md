# Setting up a Laravel Development

- [Setting up a Laravel Development](#setting-up-a-laravel-development)
  - [1. CentOS 7 Vagrant box](#1-centos-7-vagrant-box)
  - [2. Installing LAMP stack](#2-installing-lamp-stack)
    - [2.1. Prerequisites](#21-prerequisites)
    - [2.2. Apache 2.4](#22-apache-24)
      - [Installing Apache 2.4](#installing-apache-24)
      - [Setting up a Firewall](#setting-up-a-firewall)
      - [Setting up Virtual Hosts](#setting-up-virtual-hosts)
      - [Adjusting SELinux Permissions](#adjusting-selinux-permissions)
    - [2.3. Installing MySQL 8.0](#23-installing-mysql-80)
    - [2.4. Installing PHP 7.3](#24-installing-php-73)
    - [2.5. Check installed version](#25-check-installed-version)
  - [3. Installing Composer](#3-installing-composer)
  - [4. Installing NodeJS v10](#4-installing-nodejs-v10)
  - [5. Installing Git 2.22.0 + Ungit 0.10.3](#5-installing-git-2220--ungit-0103)
  - [6. Rainloop + Dovecot + Postfix](#6-rainloop--dovecot--postfix)
  - [7. Samba 4 for File Sharing](#7-samba-4-for-file-sharing)

## 1. CentOS 7 Vagrant box

```cmd
vagrant init centos/7
truncate Vagrantfile -s 0
vi Vagrantfile
```

Modify the new `Vagrantfile` with below content. Remember change the `config.vm.hostname` and `config.vm.network` with your own setting.

```Ruby
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.hostname = "centos7_192.168.0.100"
  config.vm.network "public_network", ip: "192.168.0.100", netmask: "255.255.255.0", gateway: "192.168.0.1"
  config.ssh.insert_key = false
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--memory", "512", "--cpus", "1" ,"--ioapic", "on"]
  end
end
```

Startup vagrant box by command `vagrant up --provision`. Your result should be look like this.

Some Vagrant usage command

```cmd
# Connection to box via SSH
vagrant ssh

# Reload box
vagrant reload --provision

# Check status current box
vagrant status

# Check status all box
vagrant global-status

# Shutdown box
vagrant halt

# Destroy box
vagrant destroy --force

# Remove box image
vagrant box remove <box_name>
```

How to ssh to vagrant box using `root` account

```cmd
vagrant ssh
vi /etc/ssh/sshd_config
```

> Find the line `PasswordAuthentication no` and changed to `PasswordAuthentication yes`.

```cmd
systemctl restart sshd
exit
ssh root@192.168.0.100 # password: vagrant
```

## 2. Installing LAMP stack

### 2.1. Prerequisites

Enable both repositories on your system using the following commands on your CentOS 7 system.

```cmd
rpm -Uvh http://dl.fedoraproject.org/pub/epel/7/x86_64/Packages/e/epel-release-7-11.noarch.rpm
rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
```

### 2.2. Apache 2.4

#### Installing Apache 2.4

```cmd
yum --enablerepo=epel,remi install httpd
systemctl enable httpd
systemctl start httpd
vi /etc/httpd/conf/httpd.conf
servername localhost # Add line
systemctl restart httpd
systemctl status httpd
```

#### Setting up a Firewall

```cmd
systemctl enable firewalld
systemctl start firewalld
firewall-cmd --get-active-zones
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-port=80/tcp --permanent # or
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --zone=public --add-port=443/tcp --permanent # or
firewall-cmd --reload
```

#### Setting up Virtual Hosts

> Change `example.vagrant.test` with your own folder name

```cmd
mkdir -p /var/www/vhosts/example.vagrant.test/public
mkdir -p /var/www/vhosts/example.vagrant.test/log
chown -R $USER:$USER /var/www/vhosts/example.vagrant.test/public
chmod -R 755 /var/www/vhosts
cd /var/www/vhosts/example.vagrant.test/public
vi index.php
```

```php
<?php

echo phpinfo();
```

```cmd
mkdir -p /etc/httpd/conf.d/vhosts
vi /etc/httpd/conf/httpd.conf
```

```apache
IncludeOptional conf.d/vhosts/*.conf # Add this line
DirectoryIndex index.php index.html
AddType application/x-httpd-php .php
```

```cmd
vi /etc/httpd/conf.d/vhosts/example.vagrant.test.conf
```

```apache
<VirtualHost *:80>
    ServerName www.example.vagrant.test
    ServerAlias example.vagrant.test
    DocumentRoot /var/www/vhosts/example.vagrant.test/public
    ErrorLog /var/www/vhosts/example.vagrant.test/log/error.log
    CustomLog /var/www/vhosts/example.vagrant.test/log/requests.log combined
</VirtualHost>
```

#### Adjusting SELinux Permissions

Adjusting Apache Policies Universally (not recommended)

```cmd
setsebool -P httpd_unified 1
```

Adjusting Apache Policies on a Directory

```cmd
yum provides /usr/sbin/semanage
yum whatprovides /usr/sbin/semanage # or
yum install policycoreutils-python setroubleshoot
ls -dZ /var/www/vhosts/example.vagrant.test/log
semanage fcontext -a -t httpd_log_t "/var/www/vhosts/example.vagrant.test/log(/.*)?"
restorecon -R -v /var/www/vhosts/example.vagrant.test/log
systemctl restart httpd
```

### 2.3. Installing MySQL 8.0

```cmd
rpm -Uvh https://repo.mysql.com/mysql80-community-release-el7-1.noarch.rpm
yum install mysql-server
systemctl enable mysqld
systemctl start mysqld
grep "A temporary password" /var/log/mysqld.log | tail -n1 # password string locate after root@localhost:
mysql_secure_installation # input your new password
Change the password for root?               - y
Remove anonymous users?                     - y
Disallow root login remotely?               - y
Remove test database and access to it?      - y
Reload privilege tables now?                - y
systemctl restart mysqld
mysql -u root -p # input your new password
```

### 2.4. Installing PHP 7.3

```cmd
yum --enablerepo=epel,remi-php73 install php php-cli php-fpm php-mysqlnd php-zip php-devel php-gd php-mcrypt php-mbstring php-curl php-xml php-pear php-bcmath php-json
systemctl restart httpd
```

### 2.5. Check installed version

```cmd
httpd -v

Server version: Apache/2.4.6 (CentOS)
Server built:   Apr 24 2019 13:45:48
```

```cmd
mysql -V

mysql  Ver 8.0.16 for Linux on x86_64 (MySQL Community Server - GPL)
```

```cmd
php -v

Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.3.7, Copyright (c) 1998-2018 Zend Technologies
```

## 3. Installing Composer

```cmd
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"

php -r "if (hash_file('sha384', 'composer-setup.php') === 'a5c698ffe4b8e849a443b120cd5ba38043260d5c4023dbf93e1558871f1f07f58274fc6f4c93bcfd858c6bd0775cd8d1') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"

php composer-setup.php --install-dir=/usr/local/bin --filename=composer

php -r "unlink('composer-setup.php');"

composer
```

## 4. Installing NodeJS v10

```cmd
yum install -y gcc-c++ make
curl -sL https://rpm.nodesource.com/setup_10.x | -E bash -
yum install nodejs
```

Check NodeJS and NPM version

```cmd
node -v # v10.16.0
npm -v #6.9.0
```

## 5. Installing Git 2.22.0 + Ungit 0.10.3

```cmd
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel
yum install gcc perl-ExtUtils-MakeMaker
yum install wget
cd /usr/src
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.22.0.tar.gz
tar xzf git-2.22.0.tar.gz
rm -rf git-2.22.0.tar.gz
cd git-2.22.0/
make prefix=/usr/local/git all
make prefix=/usr/local/git install
vi /etc/bashrc
export PATH=/usr/local/git/bin:$PATH # Add this line
source /etc/bashrc
git --version # git version 2.22.0
```

Installing Ungit

```cmd
npm install -g ungit@0.10.3
firewall-cmd --zone=public --add-port=8448/tcp --permanent
firewall-cmd --reload
```

Make Ungit autostart

```cmd
vim /etc/init.d/ungit
```

```ungit
#!/bin/sh
#
# unigit        This shell script takes care of starting and stopping
#               the Ungit.(git web client)
#
# chkconfig: - 64 36
# description:  Ungit server.
# processname: ungit

PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin
NAME=ungit
   :
case "$1" in
  start)
        echo -n "Starting $NAME: "
        ungit > /dev/null &
        echo "OK"
        ;;
  stop)
        echo -n "Stopping $NAME: "
        killall node
        echo "OK"
        ;;
esac

exit 0
```

```cmd
chkconfig ungit on
chmod 777 /etc/init.d/ungit
/etc/init.d/ungit start
echo "<?php" > /root/autostart.php
echo "\$buf = shell_exec('ps aux | grep ungit');" >> /root/autostart.php
echo 'if(!preg_match("/server\.js/",$buf)){' >> /root/autostart.php
echo "system('ungit > /dev/null &');" >> /root/autostart.php
echo "}" >> /root/autostart.php
```

## 6. Rainloop + Dovecot + Postfix

Edit hosts file

```cmd
vi /etc/hosts
127.0.0.1   localhost #change to this
::1         localhost #change to this
```

Add catchall user

```cmd
adduser catchall
passwd catchall # password: catchall
firewall-cmd --zone=public --add-service=smtp --permanent
firewall-cmd --zone=public --add-service=imaps --permanent
firewall-cmd --reload
```

PCRE regrex

```cmd
vi /etc/aliases.regexp
/(?!^root$|^catchall$)^.*$/ catchall # add this line
```

Dovecot

```cmd
yum install dovecot
vi /etc/dovecot/conf.d/10-mail.conf
mail_location = maildir:~/Maildir # Add this line
```

Postfix

```cmd
cp /etc/postfix/main.cf /etc/postfix/main.cf.bak
cp /etc/postfix/master.cf /etc/postfix/master.cf.bak
vi /etc/postfix/main.cf
```

```cf
home_mailbox = Maildir/ # uncomment
alias_maps = hash:/etc/aliases, pcre:/etc/aliases.regexp # change this line
transport_maps = pcre:/etc/postfix/transport_maps
```

```cmd
sed -i "s/postmaster:\troot/postmaster:\tcatchall/" /etc/aliases # find postmaster:\troot replace by postmaster:\tcatchall
```

```cmd
vi /etc/postfix/transport_maps
```

```file
/^.*@.*$/ local # add this
```

```cmd
postalias /etc/aliases
postmap /etc/postfix/transport
systemctl restart postfix
systemctl restart dovecot
systemctl enable dovecot
setsebool -P httpd_can_network_connect 1
```

Install Rainloop

```cmd
yum install unzip
mkdir /var/www/vhosts/rainloop
wget https://www.rainloop.net/repository/webmail/rainloop-latest.zip
unzip rainloop-latest.zip -d /var/www/vhosts/rainloop/
rm -rf rainloop-latest.zip
find /var/www/vhosts/rainloop -type d -exec chmod 755 {} \;
find /var/www/vhosts/rainloop -type f -exec chmod 644 {} \;
chown -R apache:apache /var/www/vhosts/rainloop/
chcon -R -t httpd_sys_rw_content_t /var/www/vhosts/rainloop/data
semanage fcontext -a -t httpd_sys_rw_content_t /var/www/vhosts/rainloop/data
restorecon -R -v /var/www/vhosts/rainloop/data
systemctl restart httpd
```

Create a rainloop virtual hosts

```cmd
vim /etc/httpd/conf.d/vhosts/rainloop.conf
```

```apache
Alias /rainloop /var/www/vhosts/rainloop

<Directory /var/www/vhosts/rainloop>
    AddDefaultCharset UTF-8

   <IfModule mod_authz_core.c>
     <RequireAny>
       Require all granted
     </RequireAny>
   </IfModule>
    Options SymLinksIfOwnerMatch
    DirectoryIndex index.php
</Directory>
```

```cmd
systemctl restart httpd
systemctl restart dovecot
```

Access domain `your_vagrant_ipaddress/rainloop/?admin` and login with info `admin|12345`

Config ```Domains`` sections

![rainloop](/img/Screenshot&#32;from&#32;2019-08-19&#32;22-21-33.png)

Config ```Login``` section

![rainloop](/img/Screenshot&#32;from&#32;2019-08-19&#32;22-14-00.png)

Access domain `your_vagrant_ipaddress/rainloop` and login with info `catchall|12345`

Test send mail. Press ```Ctr + D``` to send mail.

```cmd
yum install mailx
echo "This is message body" | mail -s "This is Subject" -r "sender<sender@mail.com>" someone@example.com
```

## 7. Samba 4 for File Sharing

```cmd
yum install samba samba-client samba-common
firewall-cmd --permanent --add-service=samba
firewall-cmd --reload
cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```
