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

**Gunakan Policy Kit (Polkit) agar User dapat Menjalankan Perintah untuk Mengatur Server tanpa Autentikasi**
```console
# Buat Polkit Rule
sudo nano /etc/polkit-1/rules.d/50-allow-zomboid.rules

# Tambahkan rule untuk service Zomboid
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units" &&
        action.lookup("unit") == "zomboid.service" &&
        subject.user == "pzuser") {
        return polkit.Result.YES;
    }
});

# Restart Polkit
sudo systemctl restart polkit
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

