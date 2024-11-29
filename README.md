ServerMama

Tugas OS Server Teknik Komputer

**Server Services Overview**
Web Server (WordPress)
SQL Server (MySQL)
DNS Server (BIND9)
Mail Server (Postfix + Dovecot)
Monitoring Server (Prometheus + Grafana)
Load Balancer (HAProxy)

**Operating System**
Ubuntu 24.04.1 LTS

**Install LAMP Stack**
```console
# Install Apache, MySQL, PHP
sudo apt install apache2 mysql-server php libapache2-mod-php php-mysql -y

sudo systemctl enable apache2

sudo systemctl start apache2

sudo mysql_secure_installation

sudo apt install php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc -y
```

**Konfigurasi MySQL Database untuk Wordpress**
```console
# Login MySQL
sudo mysql

# Buat database and user untuk WordPress
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'admin'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON wordpress.* TO 'admin'@'localhost';
FLUSH PRIVILEGES;
EXIT;

```



**Download dan Konfigurasi Wordpress**
```console
# Download Wordpress
curl -O https://wordpress.org/latest.tar.gz

# Extract Wordpress
tar -xzvf latest.tar.gz

# Pindahkan Wordpress ke web root
sudo mv wordpress /var/www/html/

# Atur permissions pada direktori wordpress tersebut
sudo chown -R www-data:www-data /var/www/html/wordpress
sudo chmod -R 755 /var/www/html/wordpress
```

**Konfigurasi Apache untuk WordPress**
```console
# Edit konfigurasi
sudo nano /etc/apache2/sites-enabled/wordpress.conf

<VirtualHost *:80>
    DocumentRoot /var/www/html/wordpress
    <Directory /var/www/html/wordpress>
        Options FollowSymLinks
        AllowOverride Limit Options FileInfo
        DirectoryIndex index.php
        Require all granted
    </Directory>
    <Directory /var/www/html/wordpress/wp-content>
        Options FollowSymLinks
        Require all granted
    </Directory>
</VirtualHost>

# Enable site dan Apache mod_rewrite
sudo a2ensite wordpress
sudo a2enmod rewrite
sudo systemctl reload apache2
```

**Finalize WordPress Setup**
```console
# Buka server IP address di browser
http://192.168.18.203

# Ikuti langkah-langkah instalasi:
1. Pilih bahasa
2. Masukkan detail database :
  - Database Name : wordpress
  - Username : admin
  - Password : password
  - Table Prefix : wp_
3. Jalankan instalasi
4. Site Title : Server Mama
5. Username : admin
6. Password : @!admin227887
7. Email : admin@mama.com
7. Install WordPress
```

**DNS Server (BIND9)**
```console
# Install BIND9
sudo apt install bind9 bind9-utils -y

# Konfigurasi DNS Server
sudo cp /etc/bind/db.127 /etc/bind/db.ip
sudo cp /etc/bind/db.local /etc/bind/db.domain

# Edit file db.domain
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     mama.com. root.mama.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      mama.com.
@       IN      A       192.168.18.203
www     IN      A       192.168.18.203

# Edit file db.ip
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     mama.com root.mama.com. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      mama.com
203     IN      PTR     mama.com

# Edit named.conf.local
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "mama.com"{
        type master;
        file "/etc/bind/db.domain";
};

zone "18.168.192.in-addr.arpa"{
        type master;
        file "/etc/bind/db.ip";
};

# Selanjutnya, konfigurasi named.conf.option
sudo nano /etc/bind/named.conf.option

# Selanjutnya restart bind9
sudo systemctl restart bind9.service
```

