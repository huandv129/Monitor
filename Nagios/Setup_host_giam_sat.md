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





















