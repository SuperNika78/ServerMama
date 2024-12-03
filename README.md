Tugas OS Server Teknik Komputer

- William Hanjaya(23.83.0959)

**Project Zomboid Dedicated Server with Access from a Website**

[![Watch the video](https://shared.cloudflare.steamstatic.com/store_item_assets/steam/apps/108600/ss_eca8be032b3f5508bf5bea74cfbc823a4df047ce.600x338.jpg?t=1691508011)](https://video.cloudflare.steamstatic.com/store_trailers/256865885/movie480_vp9.webm?t=1639992657)



**Server Services Overview**
Game Server (Project Zomboid)
Web Server (NGINX)
DNS Server (BIND9)
Mail Server (Postfix + Dovecot)
Monitoring Server (Prometheus + Grafana)



**Operating System**
Ubuntu 24.04.1 LTS

**Install Game Server (Project Zomboid)**
```console
# Install SteamCMD
sudo add-apt-repository multiverse; sudo dpkg --add-architecture i386; sudo apt update
sudo apt install steamcmd

# Jangan jalankan server pada user root. Buat user baru
sudo adduser pzuser

# Install Zomboid server pada /opt/pzserver:
sudo mkdir /opt/pzserver
sudo chown pzuser:pzuser /opt/pzserver

# Log in as pzuser
sudo -u pzuser -i

# Buat file konfigurasi /home/pzuser/update_zomboid.txt yang akan mengelola steamcmd
cat >$HOME/update_zomboid.txt <<'EOL'
// update_zomboid.txt
//
@ShutdownOnFailedCommand 1 //set to 0 if updating multiple servers at once
@NoPromptForPassword 1
force_install_dir /opt/pzserver/
//for servers which don't need a login
login anonymous
app_update 380870 validate
quit
EOL

# Sekarang instal Project Zomboid Server.
export PATH=$PATH:/usr/games
steamcmd +runscript $HOME/update_zomboid.txt

# Firewall yang umum digunakan untuk membuka port di Linux adalah UFW (Uncomplication Firewall). Port yang dibuka untuk server ini adalah 16261/udp, 16262/udp
sudo ufw allow 16261/udp
sudo ufw allow 16262/udp
sudo ufw reload

# Selanjutnya saya akan menjalankan server melalui systemd. Sebelumnya jalankan terlebih dahulu servernya untuk membuat password admin
cd /opt/pzserver/
bash start-server.sh

# Lalu, pastikan Anda tidak lagi login sebagai pzuser. Perintah system.d harus dijalankan sebagai root. Jika sudah login sebagai **root** jalankan perintah berikut
cat >/etc/systemd/system/zomboid.service <<'EOL'
[Unit]
Description=Project Zomboid Server
After=network.target

[Service]
PrivateTmp=true
Type=simple
User=pzuser
WorkingDirectory=/opt/pzserver/
ExecStart=/bin/sh -c "exec /opt/pzserver/start-server.sh </opt/pzserver/zomboid.control"
ExecStop=/bin/sh -c "echo save > /opt/pzserver/zomboid.control; sleep 15; echo quit > /opt/pzserver/zomboid.control"
Sockets=zomboid.socket
KillSignal=SIGCONT

[Install]
WantedBy=multi-user.target
EOL

# Kedua, tambahkan "zomboid.socket FIFO service", ini akan memungkinkan kita mengeluarkan perintah ke server, dan melakukan shutdown dengan aman
cat >/etc/systemd/system/zomboid.socket <<'EOL'
[Unit]
BindsTo=zomboid.service

[Socket]
ListenFIFO=/opt/pzserver/zomboid.control
FileDescriptorName=control
RemoveOnStop=true
SocketMode=0660
SocketUser=pzuser

EOL

# Setelah Anda selesai melakukannya, Anda memiliki perintah berikut yang tersedia
# start a server
systemctl start zomboid.socket

# stop a server
systemctl stop zomboid

# restart a server
systemctl restart zomboid

# check server status (ctrl-c to exit)
systemctl status zomboid
```

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

