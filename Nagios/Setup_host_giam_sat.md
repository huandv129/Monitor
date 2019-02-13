### 1 Cài đặt NRPE trên host cần giám sát

Trên host Linux cần giám sát, chúng ta cần thực hiện các bước sau:

Bước 1: Cập nhật repo cho host

Đối với client sử dụng CentOS, cài đặt repo epel-release:

`yum install epel-release -y`

Đối với client sử dụng Ubuntu:

`apt-get update -y`


Bước 2: Cài đặt NRPE và các Plugin trên host cần giám sát

Đối với client sử dụng CentOS, cài đặt repo `epel-release`:

`yum install nrpe nagios-plugins-all -y`

Đối với client sử dụng Ubuntu:

`apt-get install nagios-plugins nagios-nrpe-server -y`

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

<img src="/img/5.jpg">

Với Ubuntu, thư mục chứa Plugin ở /usr/lib/nagios/plugins/

Vì vậy, để thêm các lệnh check dịch vụ ở 2 distro cũng phải khai báo đúng đường dẫn trỏ tới thư mục chứa các plugin. Ở ví dụ này, tôi sẽ check 2 dịch vụ SSH và HTTP qua NRPE:

Với CentOS:

<img src="/img/6.png">

Với Ubuntu:

<img src="/img/7.png">

```
cat /etc/xinetd.d/nrpe
service nrpe
{
    flags           = REUSE
    socket_type     = stream
    port            = 5666
    wait            = no
    user            = nagios
    group           = nagios
    server          = /usr/local/nagios/bin/nrpe
    server_args     = -c /usr/local/nagios/etc/nrpe.cfg --inetd
    log_on_failure  += USERID
    disable         = no
    only_from       = 192.168.30.80
}
```

Để check các dịch vụ khác, chúng ta thêm câu lệnh tương tự với hướng dẫn bên trên. Lưu ý, cần chạy thử plugin trước để có hướng dẫn sử dụng.

Bước 4: Khởi động lại dịch vụ

Với host sử dụng CentOS

```
systemctl restart nrpe.service
systemctl enable nrpe.service
```

Với host sử dụng Ubuntu

`service nagios-nrpe-server restart`

Sau khi cài đặt và cấu hình NRPE trên host mà chúng ta muốn giám sát, chúng ta cần phải thêm host đó vào cấu hình Nagios Server trước khi bắt đầu giám sát nó.

Thêm thông tin host trên Nagios Server

Bước 1: Cấu hình Nagios Server

Chúng ta đặt tất cả các file cấu hình host giám sát vào một thư mục, sửa file cấu hình chính của nagios:

`vi /usr/local/nagios/etc/nagios.cfg`

Tìm và bỏ "#" ở dòng:

```
...
cfg_dir=/usr/local/nagios/etc/servers
...
```

Tạo thư mục để lưu trữ file cấu hình các host cần giám sát:

`mkdir /usr/local/nagios/etc/servers`

Bước 2: Tạo file cấu hình cho host giám sát trên Nagios Server

Trên Nagios Server, tạo file cấu hình cho mỗi host mà bạn muốn giám sát chúng ở folder /usr/local/nagios/etc/servers/. Trong trường hợp của tôi, tôi sẽ đặt tên cho nó là web01.cfg

```
vi /usr/local/nagios/etc/servers/web01.cfg

define host {
        use                             linux-server
        host_name                       web01
        alias                           My Apache server
        address                         192.168.100.201
        max_check_attempts              5
        check_period                    24x7
        notification_interval           30
        notification_period             24x7
}
define service {
        use                             generic-service
        host_name                       web01
        service_description             SSHMonitor
        check_command                   check_nrpe!check_ssh
}
define service {
        use                             generic-service
        host_name                       web01
        service_description             HTTPMonitor
        check_command                   check_nrpe!check_http
        notifications_enabled           1		
}
```



Sau khi sửa xong, chúng ta lưu lại file và khởi động lại nagios server.

`service nagios restart`

Kiểm tra trên Web UI của Nagios Server

Vào giao diện Web để kiểm tra lại:

`http://192.168.100.201/nagios`

Bước 1: Bấm vào Service

Chúng ta sẽ thấy một host mới có tên web01 được thêm mới và các dịch vụ đang ở trạng thái PENDING.

<img src="/img/6.jpg">

Mặc định khi mới thêm host, Nagios Server chưa check các dịch vụ. Để check xem dịch vụ trên host có chạy hay không? Chúng ta bấm vào dịch vụ cần check, ở đây tôi sẽ chọn dịch vụ HTTP.

<img src="/img/7.jpg">

Bước 2: Bấm vào Re-schedule the next check of this service

Khi click vào, chúng ta thấy một thông báo "This service has not yet been checked, so status information is not avaliable.", Đừng lo lắng, hãy click vào phần tô đỏ Re-schedule the next check of this service để force lượt check tiếp theo.

<img src="/img/8.jpg">

Bước 3: Click tiếp vào Commit.

<img src="/img/9.jpg">

Bước 4: Click tiếp vào Done.

<img src="/img/10.jpg">

Tương tự, nếu bạn muốn check dịch vụ khác. Với ví dụ, này là SSH, các bạn thao tác lại như với dịch vụ HTTP mà tôi vừa hướng dẫn bên trên.

Sau khi Nagios Server check xong các dịch vụ, Dashboard sẽ hiển thị như sau:

<img src="/img/11.jpg">


Giám sát dịch vụ MySQL

https://github.com/hoangdh/meditech-ghichep-nagios/blob/master/docs/thuchanh-nagios/6.Monitor-MySQL.md

Giám sát dịch vụ RabbitMQ

https://github.com/hoangdh/meditech-ghichep-nagios/blob/master/docs/thuchanh-nagios/5.Monitor-RabbitMQ.md



