### Hướng dẫn cài đặt Nagios Core 4.3.1 trên CentOS 7

* Cài đặt các gói thư viện

`yum install gcc glibc glibc-common gd gd-devel make net-snmp openssl-devel xinetd unzip httpd php php-fpm curl wget -y`

* Mở port 80 cho HTTPD trên Firewalld

```
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --reload
```

Tắt SELinux:

```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

* Tạo user cho Nagios

Tạo user nagios trên máy chủ cài đặt Nagios Server

`useradd -m -s /bin/bash nagios`

-m: Tạo thư mục home cho user

-s: User sử dụng Bash Shell mặc định

Tạo group nagcmd cho phép sử dụng thư mục `Web UI`, thêm nagios và apache:

```
groupadd nagcmd
usermod -a -G nagcmd nagios
usermod -a -G nagcmd apache
```

* Cài đặt Nagios Core và Plugin

```
cd /opt
wget http://prdownloads.sourceforge.net/sourceforge/nagios/nagios-4.3.1.tar.gz
```

Sau khi tải xong, chúng ta cùng giải nén và bắt đầu phần biên dịch Nagios Core và Plugin trên máy chủ mà ta muốn cài Nagios.

Bước 1: Giải nén source Nagios


`tar xf nagios-4.3.1.tar.gz`

Bước 2: Biên dịch Nagios

`cd nagios-4.3.1`

```
./configure --with-command-group=nagcmd 
make all
make install
make install-commandmode
make install-init
make install-config
make install-webconf
```
```
 General Options:
 -------------------------
        Nagios executable:  nagios
        Nagios user/group:  nagios,nagios
       Command user/group:  nagios,nagcmd
             Event Broker:  yes
        Install ${prefix}:  /usr/local/nagios
    Install ${includedir}:  /usr/local/nagios/include/nagios
                Lock file:  ${prefix}/var/nagios.lock
   Check result directory:  ${prefix}/var/spool/checkresults
           Init directory:  /etc/rc.d/init.d
  Apache conf.d directory:  /etc/httpd/conf.d
             Mail program:  /bin/mail
                  Host OS:  linux-gnu
          IOBroker Method:  epoll

 Web Interface Options:
 ------------------------
                 HTML URL:  http://localhost/nagios/
                  CGI URL:  http://localhost/nagios/cgi-bin/
 Traceroute (used by WAP):


Review the options above for accuracy.  If they look okay,
type 'make all' to compile the main program and CGIs.

[root@nagios nagios-4.3.1]#
```

Bước 4: Cài đặt password cho nagiosadmin, khi đăng nhập Web:

`htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin`

Bước 5: Tải gói Plugin và giải nén

```
cd /opt
wget https://nagios-plugins.org/download/nagios-plugins-2.2.0.tar.gz
tar xzf nagios-plugins-2.2.0.tar.gz
```

Bước 6: Biên dịch các Plugin từ source code

```
cd nagios-plugins-2.2.0
./configure --with-nagios-user=nagios --with-nagios-group=nagios --with-openssl
make
make install
```

### 2. Khởi động Nagios Server

Khởi động lại Apache và chạy nagios, cho phép khởi động cùng hệ thống:

```
systemctl enable httpd
systemctl restart httpd
service nagios restart
```


Để kiểm tra, hãy truy cập vào giao diện Web và đăng nhập bằng nagiosadmin và Password vừa tạo ở địa chỉ:

http://192.168.100.201/nagios

<img src="/img/1.jpg">


### 3. Giám sát host/dịch vụ trên Linux


* 3.1 Giám sát thông qua NRPE

* 3.1.1 Cài đặt NRPE trên Nagios Server

NRPE - (Nagios Remote Plugin Executor) là một công cụ đi kèm để theo dõi tài nguyên hệ thống, nó còn được biết như một Agent để theo dõi các host từ xa (Remote hosts).

Mục đích của việc cài đặt này là để biên dịch ra plugin check_nrpe.

Bước 1: Tải và Giải nén source gói NRPE

```
cd /opt
curl -L -O http://downloads.sourceforge.net/project/nagios/nrpe-2.x/nrpe-2.15/nrpe-2.15.tar.gz

tar xf nrpe-*.tar.gz
```

Bước 2: Biên dịch NRPE từ source

```
cd nrpe-*

./configure --enable-command-args --with-nagios-user=nagios \
--with-nagios-group=nagios --with-ssl=/usr/bin/openssl \
--with-ssl-lib=/usr/lib/x86_64-linux-gnu

make all
make install
```

Bước 3: Thêm câu lệnh check_nrpe

Mở file /usr/local/nagios/etc/objects/commands.cfg:

`vi /usr/local/nagios/etc/objects/commands.cfg`

Thêm câu lệnh sau:

```
...
define command{
    command_name check_nrpe
    command_line $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
}
```

Thoát và lưu lại file.

* 3.1.2 Cài đặt NRPE trên host cần giám sát

Trên host Linux cần giám sát, chúng ta cần thực hiện các bước sau:

Bước 1: Cập nhật repo cho host

Đối với client sử dụng CentOS, cài đặt repo epel-release:

`yum install epel-release`

Đối với client sử dụng Ubuntu:

`apt-get update`


Bước 2: Cài đặt NRPE và các Plugin trên host cần giám sát

Đối với client sử dụng CentOS, cài đặt repo `epel-release`:

`yum install nrpe nagios-plugins-all`

Đối với client sử dụng Ubuntu:

`apt-get install nagios-plugins nagios-nrpe-server`

Bước 3: Cấu hình NRPE trên host cần giám sát

Bước này có thể làm trên cả 2 distro CentOS và Ubuntu.

Sửa file cấu hình NRPE

`vi /etc/nagios/nrpe.cfg`

Ở bài này, host web01 sử dụng OS là CentOS 7, vì vậy hãy làm theo những hướng dẫn cấu hình trên CentOS.

Cho phép Server Nagios có quyền truy cập và sử dụng NRPE

Tìm trường allowed_hosts và thêm địa chỉ IP Nagios server của bạn vào. Mỗi IP cách nhau bởi dấu phẩy (,):

`allowed_hosts=127.0.0.1, 192.168.100.201`

Đừng thoát khỏi file, làm tiếp theo bước dưới để khai báo câu lệnh cho NRPE.

Thêm câu lệnh để check các dịch vụ

Trên 2 distro CentOS và Ubuntu, thư mục chứa các plugin trên client là khác nhau:

Với CentOS, thư mục chứa Plugin ở `/usr/lib64/nagios/plugins/`





