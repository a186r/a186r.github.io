## 在ubuntu上安装nextcloud

主要记录在ubuntu上安装nextcloud的具体步骤

#### 在ubuntu上安装nextcloud的条件

- PHP
- Apache/Nginx Web服务器
- MySQL/MariaDB数据库服务器

#### 一、安装PHP

```shell
sudo apt install -y php-cli php-fpm php-json php-intl php-imagick php-pdo php-mysql php-zip php-gd php-mbstring php-curl php-xml php-pear php-bcmath
```

#### 二、安装MySQL/MariaDB数据库服务器

NextCloud可以使用MySQL、MariaDB、PostgreSQL或SQLite数据库来存储其数据。本文将使用MySQL数据库服务器。

1. 执行命令 `sudo apt-get install mysql-server`
2. 执行 `sudo apt-get install mysql-client`
3. 执行 `sudo apt-get install libmysqlclient-dev`
4. 检查是否安装成功 `sudo netstat -tap | grep mysql` 如果显示LISTEN，表示安装成功
5. 查看工作状态 `systemctl status mysql.service` 如果没有在运行，则输入命令 `sudo systemctl start mysql` 启动mysql

mysql安装完成了，接下里是一些配置：

在ubuntu中安装mysql不会弹出设置账号密码的框，会自己默认设置好账号和密码，配置文件在 `/etc/mysql/debian.cnf` 中。因此我们需要先使用配置文件中的账号密码登陆mysql，然后再建立root账户。

1. 登陆mysql `mysql -u debian-sys-maint -p` 输入配置文件中对应的账户密码
2. `mysql> update mysql.user set authentication_string=password("root") where user="root";` 

注意：第2步中的`password("root")` 将 `root`改为自己的密码即可。执行第2步会产生警告

3. 使用下面的命令消除警告

   `mysql>update mysql.user set plugin="mysql_native_password";`

4. 更新权限

   `mysql>flush privileges;`

5. 重启mysql即可

   `sudo service mysql restart`

至此，mysql安装就完成了，可以使用 `mysql -u root -p` 登入到mysql中。

mysql安装完成之后，就需要为nextcloud创建数据库和用户：

1. 执行下面的命令，创建nextcloud数据库和用户

   `mysql -u root -p`先 登入mysql

   然后执行下面的命令创建数据库和用户

   ```mysql
   CREATE USER 'nextcloud'@'localhost' identified by 'StrongPassword';(此处StrongPassword应该更换为自己的密码)
   
   CREATE DATABASE nextcloud;
   
   GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';
   
   FLUSH PRIVILEGES;
   ```

2. 确认用户可以使用提供的密码连接到数据库

   `mysql -u nextcloud -p`

   ```mysql
   Enter password: <ENTER PASSWORD>
   
   Welcome to the MariaDB monitor.  Commands end with ; or \g.
   
   Your MariaDB connection id is 34
   
   MariaDB [(none)]> SHOW DATABASES;
   
   |　Database　|
   
   |　information_schema　|
   
   |　nextcloud　|
   
   2 rows in set (0.00 sec)
   
   MariaDB [(none)]> QUIT
   
   Bye
   ```

   

#### 三、下载并安装nextcloud

前往[nextcloud]("https://nextcloud.com/")官网下载最新版本的nextcloud压缩包，下载文件后将其解压，并且移动到/srv

`sudo mv nextcloud /srv`

将目录权限更改为`www-datauser`:

`sudo chown -R www-data:www-data /srv/nextcloud/`

#### 四、安装和配置Apache Web服务器

1. 安装Apache HTTP Server

   `sudo apt install -y apache2  libapache2-mod-php`

2. 为Nextcloud创建VirtualHost文件

   `sudo vim /etc/apache2/conf-enabled/nextcloud.conf`

3. 设置VirtualHost配置文件

   ```shell
   <VirtualHost *:80>
   
   ServerAdmin admin@example.com
   
   DocumentRoot /srv/nextcloud/
   
   ServerName example.com
   
   ServerAlias www.example.com
   
   ErrorLog /var/log/apache2/nextcloud-error.log
   
   CustomLog /var/log/apache2/nextcloud-access.log combined
   
   <Directory /srv/nextcloud/>
   
   Options +FollowSymlinks
   
   AllowOverride All
   
   Require all granted
   
   SetEnv HOME /srv/nextcloud
   
   SetEnv HTTP_HOME /srv/nextcloud
   
   <IfModule mod_dav.c>
   
   Dav off
   
   </IfModule>
   
   </Directory>
   
   </VirtualHost>
   ```

4. 启动所需的Apache模块并重新启动服务

   ```shell
   sudo a2enmod rewrite dir mime env headers
   
   sudo systemctl restart apache2
   ```



此时nextcloud已经安装完成，访问域名:端口即可进入设置

首先创建管理员账户

然后设置存储路径，存储路径需要将目录权限更改为`www-datauser`

最后设置数据库连接设置（都是nextcloud，密码是前文提到的StrongPassword）
