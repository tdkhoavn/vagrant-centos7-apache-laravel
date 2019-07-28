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

```bash
vagrant init centos/7
truncate Vagrantfile -s 0
sudo vi Vagrantfile
```

Modify the new `Vagrantfile` with below content. Remember change the `config.vm.hostname` and `config.vm.network` with your own setting.

```Vagrantfile
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

![vagrant startup](img/Screenshot&#32;from&#32;2019-07-18&#32;22-28-17.png)

Vagrant usage command

```bash
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

# remove box image
vagrant box remove <box_name>
```

How to ssh to vagrant box using `root` account

```bash
vagrant ssh
sudo vi /etc/ssh/sshd_config
```

> Find the line `PasswordAuthentication no` and changed to `PasswordAuthentication yes`.

```bash
sudo systemctl restart sshd
exit
ssh root@192.168.0.100 # password: vagrant
```

## 2. Installing LAMP stack

### 2.1. Prerequisites

Enable both repositories on your system using the following commands on your CentOS 7 system.

```bash
sudo rpm -Uvh http://dl.fedoraproject.org/pub/epel/7/x86_64/Packages/e/epel-release-7-11.noarch.rpm
sudo rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
```

### 2.2. Apache 2.4

#### Installing Apache 2.4

```bash
sudo yum --enablerepo=epel,remi install httpd
systemctl enable httpd
systemctl start httpd
vi /etc/httpd/conf/httpd.conf
servername localhost # Add line
systemctl restart httpd
systemctl status httpd
```

![apache status](/img/Screenshot&#32;from&#32;2019-07-08&#32;22-48-38.png)

#### Setting up a Firewall

```bash
sudo systemctl enable firewalld
sudo systemctl start firewalld
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --add-service=http --permanent
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent # or
sudo firewall-cmd --zone=public --add-service=https --permanent
sudo firewall-cmd --zone=public --add-port=443/tcp --permanent # or
sudo firewall-cmd --reload
```

#### Setting up Virtual Hosts

> Change `vagrant101.dev` with your own folder name

```bash
sudo mkdir -p /var/www/vagrant101.dev/public
sudo mkdir -p /var/www/vagrant101.dev/log
sudo chown -R $USER:$USER /var/www/vagrant101.dev/public
sudo chmod -R 755 /var/www/
cd /var/www/vagrant101.dev/public
sudo vi index.html
```

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Page Title</title>
</head>
<body>

<h1>This is a Heading</h1>
<p>This is a paragraph.</p>

</body>
</html>
```

```bash
sudo mkdir -p /etc/httpd/sites-available /etc/httpd/sites-enabled
sudo vi /etc/httpd/conf/httpd.conf
```

```conf
IncludeOptional sites-enabled/*.conf # Add this line
```

```bash
sudo vi /etc/httpd/sites-available/vagrant101.dev.conf
```

```conf
<VirtualHost *:80>
    ServerName www.vagrant101.dev
    ServerAlias vagrant101.dev
    DocumentRoot /var/www/vagrant101.dev/public
    ErrorLog /var/www/vagrant101.dev/log/error.log
    CustomLog /var/www/vagrant101.dev/log/requests.log combined
</VirtualHost>
```

```bash
sudo ln -s /etc/httpd/sites-available/vagrant101.dev.conf /etc/httpd/sites-enabled/vagrant101.dev.conf
```

#### Adjusting SELinux Permissions

Adjusting Apache Policies Universally (not recommended)

```bash
sudo setsebool -P httpd_unified 1
```

Adjusting Apache Policies on a Directory

```bash
sudo yum provides /usr/sbin/semanage
sudo yum whatprovides /usr/sbin/semanage # or
sudo yum install policycoreutils-python
sudo yum install setroubleshoot
sudo ls -dZ /var/www/vagrant101.dev/log
sudo semanage fcontext -a -t httpd_log_t "/var/www/vagrant101.dev/log(/.*)?"
sudo restorecon -R -v /var/www/vagrant101.dev/log
sudo ls -dZ /var/www/vagrant101.dev/log/
sudo systemctl restart httpd
```

### 2.3. Installing MySQL 8.0

```bash
sudo rpm -Uvh https://repo.mysql.com/mysql80-community-release-el7-1.noarch.rpm
sudo yum install mysql-server
sudo systemctl enable mysqld
sudo systemctl start mysqld
sudo grep "A temporary password" /var/log/mysqld.log | tail -n1 # password string locate after root@localhost:
mysql_secure_installation # input your new password
Change the password for root?               - y
Remove anonymous users?                     - y
Disallow root login remotely?               - y
Remove test database and access to it?      - y
Reload privilege tables now?                - y
sudo systemctl restart mysqld
sudo mysql -u root -p # input your new password
```

### 2.4. Installing PHP 7.3

```bash
sudo yum --enablerepo=epel,remi-php73 install php php-cli php-fpm php-mysqlnd php-zip php-devel php-gd php-mcrypt php-mbstring php-curl php-xml php-pear php-bcmath php-json
sudo systemctl restart httpd
```

### 2.5. Check installed version

```bash
httpd -v

Server version: Apache/2.4.6 (CentOS)
Server built:   Apr 24 2019 13:45:48
```

```bash
mysql -V

mysql  Ver 8.0.16 for Linux on x86_64 (MySQL Community Server - GPL)
```

```bash
php -v

Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.3.7, Copyright (c) 1998-2018 Zend Technologies
```

## 3. Installing Composer

```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"

php -r "if (hash_file('sha384', 'composer-setup.php') === '48e3236262b34d30969dca3c37281b3b4bbe3221bda826ac6a9a62d6444cdb0dcd0615698a5cbe587c3f0fe57a54d8f5') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"

sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer

php -r "unlink('composer-setup.php');"

composer
```

## 4. Installing NodeJS v10

```bash
sudo yum install -y gcc-c++ make
curl -sL https://rpm.nodesource.com/setup_10.x | sudo -E bash -
sudo yum install nodejs
```

Check NodeJS and NPM version

```bash
node -v # v10.16.0
npm -v #6.9.0
```

## 5. Installing Git 2.22.0 + Ungit 0.10.3

```bash
sudo yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel
sudo yum install gcc perl-ExtUtils-MakeMaker
sudo yum install wget
cd /usr/src
sudo wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.22.0.tar.gz
sudo tar xzf git-2.22.0.tar.gz
sudo rm -rf git-2.22.0.tar.gz
sudo make prefix=/usr/local/git all
sudo make prefix=/usr/local/git install
sudo vi /etc/bashrc
export PATH=/usr/local/git/bin:$PATH # Add this line
source /etc/bashrc
git --version # git version 2.22.0
```

Installing Ungit

```bash
sudo npm install -g ungit@0.10.3
sudo firewall-cmd --zone=public --add-port=8448/tcp --permanent
sudo firewall-cmd --reload
```

## 6. Rainloop + Dovecot + Postfix

Edit hosts file

```bash
sudo vi /etc/hosts
127.0.0.1   localhost #change to this
::1         localhost #change to this
```

Add catchall user

```bash
sudo adduser catchall
sudo passwd catchall # password: catchall
sudo firewall-cmd --permanent --add-service=smtp --permanent
sudo firewall-cmd --permanent --add-service=imaps --permanent
sudo firewall-cmd --reload
```

PCRE regrex

```bash
sudo vi /etc/aliases.regexp
/(?!^root$|^catchall$)^.*$/ catchall # add this line
```

Dovecot

```bash
sudo yum install dovecot
sudo vi /etc/dovecot/conf.d/10-mail.conf
mail_location = maildir:~/Maildir # Add this line
```

Postfix

```bash
sudo cp /etc/postfix/main.cf /etc/postfix/main.cf.bak
sudo cp /etc/postfix/master.cf /etc/postfix/master.cf.bak
sudo vi /etc/postfix/main.cf
```

```cf
home_mailbox = Maildir/ # uncomment
alias_maps = hash:/etc/aliases, pcre:/etc/aliases.regexp # change this line
```

```bash
sed -i "s/postmaster:\troot/postmaster:\tcatchall/" /etc/aliases # find postmaster:\troot replace by postmaster:\tcatchall
```

```bash
sudo vi /etc/postfix/transport_maps
```

```file
/^.*@.*$/ local # add this
```

```bash
sudo postalias /etc/aliases
sudo postmap /etc/postfix/transport
sudo systemctl restart postfix
sudo systemctl restart dovecot
sudo systemctl enable dovecot
sudo setsebool -P httpd_can_network_connect 1
```

Install Rainloop

```bash
sudo yum install unzip
sudo mkdir /var/www/rainloop
sudo wget -qO- https://www.rainloop.net/repository/webmail/rainloop-latest.zip
sudo unzip rainloop-latest.zip -d /var/www/rainloop/
sudo find /var/www/rainloop -type d -exec chmod 755 {} \;
sudo find /var/www/rainloop -type f -exec chmod 644 {} \;
sudo chown -R apache:apache /var/www/rainloop/
sudo chcon -R -t httpd_sys_rw_content_t /var/www/rainloop/data
sudo semanage fcontext -a -t httpd_sys_rw_content_t /var/www/rainloop/data
sudo restorecon -R -v /var/www/rainloop/data
sudo systemctl restart httpd
```

Access domain `your_vagrant_ipaddress/rainloop/?admin` and authenticate with info `admin|12345`

![rainloop](/img/Screenshot&#32;from&#32;2019-07-25&#32;23-55-25.png)

Access domain `your_vagrant_ipaddress/rainloop/?admin` and authenticate with info `catchall@localhost|12345`

Test send mail

```bash
sudo vi ~/mail.txt
```

```txt
To: ms_receiver@email.com
Subject: From Mr.Sender with love
From: mr_sender@myserver.com
And here goes the e-mail body, bla bla bla...
```

```bash
sendmail user@example.com  < ~/email.txt
```

![rainloop test email](/img/Screenshot&#32;from&#32;2019-07-26&#32;00-01-59.png)

## 7. Samba 4 for File Sharing

```bash
sudo yum install samba samba-client samba-common
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```
