## Ghi chép cài đặt NAGIOS

### Lịch sử tài liệu
- 03.04.2017: Tạo tài liệu

### 1. Chuẩn bị môi trường

- Chuẩn bị các máy chủ
  - Nagios Server: Ubuntu Server 14.04 64 bit. IP: `172.16.69.211`
  - Nagios Client1 : Ubuntu Server 14.04 64 bit. IP: IP: `172.16.69.209`
  
 - Phiên bản Nagios: Nagios core 4.3.1
 - Thực hiện cài đặt nagios với quyền `root` 
 
### 2. Cài đặt trên Server

- Chuyển qua quyền `root` để thực hiện các bước cài đặt
  ```sh
  su - 
  ```

- Cài đặt apache
  ```sh
  sudo apt-get -y install wget build-essential apache2 php5 php5-gd libgd-dev unzip apache2-utils libgd2-xpm-dev openssl libssl-dev xinetd
  ```

- Tao user `nagios`, groups `nagcmd` và phân quyền 
  ```sh
  sudo useradd nagios
  sudo groupadd nagcmd
  sudo usermod -a -G nagcmd nagios
  sudo usermod -a -G nagcmd www-data
  ```

####  2.1 Tải các gói cài đặt của `Nagios Core`
- Lưu ý: Bước này thực hiện trên máy chủ `Nagios Server`
- Tải `nagios core 4.3.1` và thực hiện biên dịch.
  ```sh
  cd /opt/
  wget https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.3.1.tar.gz
  tar xzf nagios-4.3.1.tar.gz
  cd nagios-4.3.1
  sudo ./configure --with-command-group=nagcmd
  sudo make all
  sudo make install
  sudo make install-init
  sudo make install-config
  sudo make install-commandmode
  ```
- Copy các script để mở rộng 
  ```sh
  cp -R contrib/eventhandlers/ /usr/local/nagios/libexec/
  chown -R nagios:nagios /usr/local/nagios/libexec/eventhandlers
  ```

- Dùng lệnh `vi` để tạo fie `/etc/apache2/conf-available/nagios.conf`
  ```sh
  sudo vi /etc/apache2/conf-available/nagios.conf
  ```

với nội dung dưới

  ```sh
  ScriptAlias /nagios/cgi-bin "/usr/local/nagios/sbin"

  <Directory "/usr/local/nagios/sbin">
     Options ExecCGI
     AllowOverride None
     Order allow,deny
     Allow from all
     AuthName "Restricted Area"
     AuthType Basic
     AuthUserFile /usr/local/nagios/etc/htpasswd.users
     Require valid-user
  </Directory>

  Alias /nagios "/usr/local/nagios/share"

  <Directory "/usr/local/nagios/share">
     Options None
     AllowOverride None
     Order allow,deny
     Allow from all
     AuthName "Restricted Area"
     AuthType Basic
     AuthUserFile /usr/local/nagios/etc/htpasswd.users
     Require valid-user
  </Directory>
  ```

#### 2.2 Cấu hình xác thực cho apache khi đăng nhập vào nagios

- Thực hiện lệnh dưới và nhập mật khẩu cho tài khoản là `nagiosadmin`, nhớ mật khẩu này để truy cập vào web khi cài đặt xong nagios 
  ```sh
  htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
  ```
  
  ```sh
  sudo a2enconf nagios
  sudo a2enmod cgi rewrite
  sudo service apache2 restart
  ```

#### 2.3 Cài đặt `Nagios Plugin`

- Tải `Nagios Plugin 2.1.4`
  ```sh
  cd /opt
  wget http://www.nagios-plugins.org/download/nagios-plugins-2.1.4.tar.gz
  tar xzf nagios-plugins-2.1.4.tar.gz
  cd nagios-plugins-2.1.4
  ```

- Biên dịch nagios plugin
  ```sh
  sudo ./configure --with-nagios-user=nagios --with-nagios-group=nagios
  sudo make
  sudo make install
  ```
  
#### 2.4 Kiểm chứng lại xem nagios
 
- Sử dụng lệnh dưới để xem nagios đã cài đúng hay chưa
  ```sh
  /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
  ```
   
- Khởi động nagios
  ```sh
  service nagios start
  ```

- Thực hiện lệnh dưới để khởi động nagios khi reboot lại máy chủ.
  ```sh
  sudo update-rc.d nagios defaults
  ```

#### 2.5 Cài đặt NRPE phía server 
- Phía server nagios sẽ có `check NRPE` để thực hiện gửi các câu lệnh tới `NRPE service` ở phía client để yêu cầu thực thi các script check các dịch vụ cần giám sát. Do vậy cần cài đặt gói NRPE trên server.
- Thực hiện tải gói NRPE, giải nén và cài đặt trên `Nagios Server`

  ```sh
  cd /opt/
  curl -L -O http://downloads.sourceforge.net/project/nagios/nrpe-2.x/nrpe-2.15/nrpe-2.15.tar.gz
  tar xvf nrpe-*.tar.gz
  cd nrpe-*
  ```

- Thực hiện lệnh để cấu hình NRPE
  ```sh
  ./configure --enable-command-args --with-nagios-user=nagios --with-nagios-group=nagios --with-ssl=/usr/bin/openssl --with-ssl-lib=/usr/lib/x86_64-linux-gnu
  ```

- Biên dịch NRPE 
  ```sh
  make all
  sudo make install
  sudo make install-xinetd
  sudo make install-daemon-config
  ```
  
- Sửa file cấu hình của NRPE, dùng vi để mở file `/etc/xinetd.d/nrpe`
  ```sh
  vi /etc/xinetd.d/nrpe
  ```

- Sửa dòng `only_from` thành dòng chứa địa chỉ IP của máy Nagios Server, trong ví dụ này là `172.16.69.211`
  ```sh
  only_from       = 127.0.0.1,172.16.69.211
  ```

- Khởi động lại xinetd
  ```sh
  service xinetd restart
  ```

- Lưu ý: cần cài đặt NRPE trên Nagios Server để sau bước cài nrpe trên client xong thì sẽ dùng `check_nrpe` để kiểm tra  nrpe trên client xem đã hoạt động hay chưa.
  
#### 2.6 Truy cập vào web của nagios
 
- Địa chỉ truy cập
  ```sh
  http://IP_cua_may/nagios
  ```
 
- Nhập tài khoản và mật khẩu đã tạo ở trên vào.
 

### 3. Cài đặt giám sát các host (cài đặt phía client).
- Nagios sử dụng cơ chế NRPE (Nagios Remote Plugin Executor) để giám sát các host ở xa. Ngoài NRPE còn nhiều cơ chế khác.
- NRPE là một tiện ích bổ sung thêm cùng với bộ nagios để thực thi các plugin trên client (plugin là các script thực thi phía clinet). Mô hình của NRPE như sau: http://prntscr.com/es90vs
- Thực hiện lệnh dưới để cài các gói bổ trợ và NRPE phía các host client
```sh
apt-get -y install nagios-nrpe-server nagios-plugins
```

- Mở file cấu `/etc/nagios/nrpe.cfg` bằng lệnh `vi`
  ```sh
  vim /etc/nagios/nrpe.cfg
  ```

- Sửa dòng `allowed_hosts` để khai báo IP của máy chủ Nagios Server: `172.16.69.211`
  ```sh
  allowed_hosts=127.0.0.1, 172.16.69.211
  ```
  
- Khởi động lại nrpe service trên Client 
  ```sh
  sudo /etc/init.d/nagios-nrpe-server restart
  ```
  
- Sau đó, đứng trên máy chủ `Nagios Server` và thực hiện lệnh dưới để kiểm tra xem đã kết nối từ Nagios Server tới Client thông qua NRPE được hay chưa, kết quả của lệnh sẽ là phiên bản của NRPE được cài đặt phía client.
  ```sh
  root@u14-lab:/opt# /usr/local/nagios/libexec/check_nrpe -H 172.16.69.209
  NRPE v2.15
  ```

- Nếu kết quả như trên thì việc cài đặt và chuẩn bị đã hoàn tất, chúng ta sẽ chuyển sang các bước tiếp theo.
  
#### Bước khai báo các host để nagios giám sát 

## Ghi chú:

### Check các service ngay và luôn
- Khi cài đặt và khai báo host để nagios giám sát xong, cần phải chờ xxx phút thì các service mới được giám sát (link ảnh), trạng thái lúc này là pending. Do vậy để Nagios Server check các service này ngay thì thực hiện như sau:
- Bước 1: click vào service đang ở trạng thái pending: 
- Bước 2: Click vào dòng `Re-schedule the next check of this service`. Tham khảo: http://prntscr.com/es7ar0
- Bước 3: Click vào nút commit. Tham khảo: http://prntscr.com/es7b2u
- Bước 4: Quay lại tab `Host` để xem trạng thái của service được update, lúc này service sẽ được update ngay mà không phải chờ sau thời gian mặc định nữa.
 
## Tham khảo
- https://tecadmin.net/install-nagios-monitoring-server-on-ubuntu/
 
