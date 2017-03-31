# Các ghi chép về OMD Check_MK

### Tham khảo

### Nagios, OMD và check MK
- https://cunninghamshane.com/check_mk-agent-setup-and-configuration/
- https://www.rosehosting.com/blog/how-to-install-nagios3-and-check_mk-on-an-ubuntu-12-04-lts-vps/
- http://blog.unicsolution.com/2013/11/best-monitoring-solution-omd-nagios.html
- https://github.com/congto/omd
- https://www.digitalocean.com/community/tutorials/how-to-use-open-monitoring-distribution-with-check_mk-on-ubuntu-14-04

## Mô hình

- Gồm 2 máy: 
  - MK_Server: Ubuntu server 14.04 64 bit - có NICs kết nối internet.
  - MK_Client: Ubuntu server 14.04 64 bit hoặc CentOS 7.x 64 bit. Có kết nói internet và cùng dải với máy server

## Cài đặt OMD trên Server
### Cài đặt repos cho OMD

- Khai báo GPG KEY 
```sh
wget -q "https://labs.consol.de/repo/stable/RPM-GPG-KEY" -O - | sudo apt-key add -
```

- Khai báo repos cho OMD
```sh
echo "deb http://labs.consol.de/repo/stable/ubuntu $(lsb_release -cs) main" > /etc/apt/sources.list.d/labs-consol-stable.list
apt-get update
```

- Kiểm tra phiên bản của OMD có thể được cài đặt
```sh
apt-cache search omd
```

- Kết quả như ảnh sau: http://prntscr.com/eldnfd

- Lựa chọn phiên bản cài của OMD, trong hướng dẫn này sử dụng phiên bản OMD native bản trong hướng dẫn này là `omd-1.30` (Có 02 phiên bản của OMD là: OMD native và OMD-LAB)

- Thực hiện lệnh để cài đặt OMD
```sh
apt-get -y install omd 
```

- Chú ý: 
  - Nếu cài đặt OMD-LAB thì lựa sử dụng lệnh `apt-get -y install omd-labs-edition`
  - Trong quá trình cài đặt cần nhập mật khẩu cho MySQL
  - Đợi khoảng 15p việc cài đặt sẽ hoàn tất (thường lúc cài sẽ không có lỗi, vì OMD sinh ra để giúp cài đặt Nagios core và các plugin nhanh chóng)

- Tiếp tục dùng các lệnh mà OMD cung cấp để tại site monitor, ví dụ này sẽ lấy tên là "monitoring" (Có thể tạo nhiều site với các tên khác nhau nhé)

```sh
omd create monitoring
```

- Kết quả của lệnh trên, chú ý sẽ có thông báo tài khoản `omdadmin` và mật khẩu `omd`

```sh
Adding /omd/sites/monitoring/tmp to /etc/fstab.
Creating temporary filesystem /omd/sites/monitoring/tmp...OK
Restarting Apache...^[[AAH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1. Set the 'ServerName' directive globally to suppress this message
Created new site monitoring with version 1.30.

  The site can be started with omd start monitoring.
  The default web UI is available at http://hostname_may_server/monitoring/
  The admin user for the web applications is omdadmin with password omd.
  Please do a su - monitoring for administration of this site.
```

- Khở động site `monitoring` vừa tạo.

```sh
omd start monitoring
```

- Quan sát output trên terminal để truy cập vào link qua web
- Khi mở web ra thì chọn giao diện check_MK như trong ảnh http://prntscr.com/epyz9c 
- Khi truy cập web, cần nhập tài khoản là `omdadmin` và mật khẩu là `omd` (có thể đổi mật khẩu theo link dưới)

- Thực hiện đổi mật khẩu mặc định cho OMD (tham khảo trong https://www.digitalocean.com/community/tutorials/how-to-use-open-monitoring-distribution-with-check_mk-on-ubuntu-14-04#changing-administrative-password)


## Cài đặt check_MK trên Client
- Bước này sẽ cài đặt check_MK lên các client (host được OMD giám sát).

### Cài đặt check_MK trên Ubuntu Server 14.04 64bit

- Khai báo GPG KEY 
```sh
wget -q "https://labs.consol.de/repo/stable/RPM-GPG-KEY" -O - | sudo apt-key add -
```

- Khai báo repos cho OMD
```sh
echo "deb http://labs.consol.de/repo/stable/ubuntu $(lsb_release -cs) main" > /etc/apt/sources.list.d/labs-consol-stable.list
apt-get update
```
- Kiểm tra gói check_mk
```sh
root@u14-ctl:~# apt-cache search check_mk
```

- Kết quả lệnh trên

```sh
check-mk-agent - general purpose nagios-plugin for retrieving data
check-mk-agent-logwatch - general purpose nagios-plugin for retrieving data
check-mk-config-icinga - general purpose nagios-plugin for retrieving data
check-mk-config-nagios3 - general purpose nagios-plugin for retrieving data
check-mk-doc - general purpose nagios-plugin for retrieving data (documentation)
check-mk-livestatus - general purpose nagios-plugin for retrieving data
check-mk-multisite - general purpose nagios-plugin for retrieving data
check-mk-server - general purpose nagios-plugin for retrieving data
libmonitoring-livestatus-perl - Perl API for check_mk livestatus to access runtime
nagstamon - Nagios status monitor which takes place in systray or on desktop
shinken-module-commandfile - Command file module for Arbiter, Receiver or Poller
check-mk-raw-1.2.6p16 - Check_MK is a full featured system monitoring
```

- Tải check_MK cho máy client, chọn gói `check-mk-raw-1.2.6p16`
```sh
apt-get install check-mk-agent
```

- Nếu báo lỗi thì cài thêm
```sh
apt-get -f install
```

- Chỉnh sửa file check mk phía client để cho phép nó được hoạt động, mở file bằng lệnh `vi /etc/xinetd.d/check_mk` và sửa dòng dưới thành


```sh
disable        = no
```

- Khởi động lại xinetd

```sh
service xinetd restart
```

- Port của check mk hoạt động là 6556, kiểm tra lại port 6556 đã hoạt động hay chưa

```sh
netstat -atunp | grep 6556
```

- Sau đó học cách add node trên GUI Check mk ở đây
https://www.digitalocean.com/community/tutorials/how-to-use-open-monitoring-distribution-with-check_mk-on-ubuntu-14-04#monitoring-the-first-host




