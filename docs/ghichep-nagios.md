## Ghi chép cài đặt NAGIOS

### Lịch sử tài liệu
- 03.04.2017: Tạo tài liệu

### 1. Chuẩn bị môi trường

- Chuẩn bị các máy chủ
  - Nagios Server: Ubuntu Server 14.04 64 bit
  - Nagios Client: Ubuntu Server 14.04 64 bit, CentOS 7.x 64 bit
  
 - Phiên bản Nagios: Nagios core 4.3.1  
 
### 2. Cài đặt trên Server

- Chuyển qua quyền `root` để thực hiện các bước cài đặt
  ```sh
  root
  ```

- Cài đặt apache
  ```sh
  sudo apt-get install wget build-essential apache2 php5 php5-gd libgd-dev unzip apache2-utils
  ```

- Tao user `nagios`, groups `nagcmd` và phân quyền 
  ```sh
  sudo useradd nagios
  sudo groupadd nagcmd
  sudo usermod -a -G nagcmd nagios
  sudo usermod -a -G nagcmd www-data
  ```

####  2.1 Tải các gói cài đặt của `Nagios Core`
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
- Now copy event handlers scripts under libexec directory. These binaries provides multiple events triggers for your Nagios web interface
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
  services nagios start
  ```

- Thực hiện lệnh dưới để khởi động nagios khi reboot lại máy chủ.
  ```sh
  sudo update-rc.d nagios defaults
  ```
  
#### 2.5 Truy cập vào web của nagios
 
- Địa chỉ truy cập
  ```sh
  http://IP_cua_may/nagios
  ```
 
- Nhập tài khoản và mật khẩu đã tạo ở trên vào.
 
 
