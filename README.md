# ProjectMen


ANGGOTA KELOMPOK:

MUHAMMAD NUR HAQIQI - 22.83.0886 TK-02(Ketua) - System Security Administrator (Fenrir717)

Georel Jeferson Fransiskus Bonai - 22.83.0833(Anggota 1) - Mail Server Administrator (Quetzalcoatlos23)

Muhammad Akbar Firmansyah - 22.83.0870(Anggota 2) -  CMS Wordpress Configurator

Muhammad Fadhil Mundzir Sakaria - 22.83.0829(Anggota 3) - Database Administrator (A4VA4TA4R)

Willy Agustianus - 22.83.0882(Anggota 4) - Roundcube Webmail Configurator

Janssensius Rifaldo Lhezu  -  22.83.0861(Anggota 5) - Web Server Administrator


**TOPOLOGI:**
![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/15940bb6-a453-4a1a-bd59-ae71f2bf8628)


**SCENARIO:**
Proyek ini bertujuan untuk mengimplementasikan langkah-langkah keamanan pada tiga server virtual (VM), masing-masing bernama VM1, VM2, dan VM3. Berikut adalah konfigurasi keamanan yang akan diterapkan:

**1. VM1 (Honeypot):**
- IP: 192.168.20.1
- Berfungsi sebagai honeypot untuk mendeteksi potensi ancaman.
- Terinstal Honeypot dengan aturan iptables yang mengalihkan paket yang masuk ke VM2 (IP 192.168.20.2) dengan tujuan port 22, akan dialihkan ke VM1. Penyerang yang mencoba mengakses SSH di VM2 akan terjebak ke Honeypot di VM1.

**2. VM2 (Server Utama):**
- IP: 192.168.20.2
- Terinstal Web Server (CMS WordPress) dan Mail Server (Roundcube).
- Layanan dilindungi oleh UFW dan WAF (ModSecurity2) untuk meningkatkan keamanan.
- Akses SSH ditutup dan hanya dapat diakses melalui Port Knocking (Knockd) yang dikombinasikan dengan VPN Server (OpenVPN).
- SSH hanya dapat diakses melalui VPN dengan Subnet VPN 20.10.20.0/24 pada interface tap0.
- Interface tambahan untuk area lokal dengan IP 10.10.10.1, digunakan untuk komunikasi dengan VM3.
- Akan ditambahkan Monitoring Log Server dengan Promtail & LOKI dan Rsyslog yang akan divisualisasikan dengan Grafana.
- Penggunaan SSL Certificate pada layanan HTTP/Apache2 untuk mengamankan setiap halaman web. Web server menggunakan port 443, mail server (SMTPS 465 dan IMAPS 993), serta Grafana menggunakan HTTPS pada port 443.

**3. VM3 (Server Backup):**
- IP: 10.10.10.2
- Berfungsi sebagai server backup.
- Hanya dapat berkomunikasi dengan VM2 di jaringan lokal (10.10.10.1).
- Tidak terhubung langsung ke internet untuk meningkatkan keamanan.
- Menerima backup konfigurasi secara rutin dari VM2 dengan penjadwalan(Crontab).

Proyek ini bertujuan untuk menciptakan lingkungan server yang aman dengan mengimplementasikan praktik keamanan yang canggih seperti honeypot, port knocking, VPN, dan konfigurasi otomatis backup. Seluruh konfigurasi akan didokumentasikan dengan baik di dalam repositori ini untuk memudahkan pengelolaan dan pemeliharaan sistem keamanan.

**Konfigurasi pada Setiap VM**

 [1.VM1](#1-Konfigurasi-Honeypot-pada-VM1)
 
 [2.VM2](#2-Konfigurasi-Server-pada-VM2)
 
 [3.VM3](#3-Konfigurasi-Server-pada-VM3)

## 1. Konfigurasi Honeypot pada VM1

### 1.1 Konfigurasi Adapter Network di VM1

**Langkah 1: Buka direktori utama Konfigurasi Adapter**
```
nano /etc/network/interfaces
```
**Langkah 2: Edit File Konfigurasi seperti ini**
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface

allow-hotplug enp0s3
auto enp0s3
iface enp0s3 inet dhcp

auto enp0s8
iface enp0s8 inet static
 address 192.168.20.1
 netmask 255.255.255.0

```
**Langkah 3: Restart Layanan Networking**
```
systemctl restart networking
```

### 1.2 Konfigurasi Honeypot

**Langkah 1: Instalasi paket dan Depedensi yang dibutuhkan**
```
apt-get install postgresql
apt-get install python3-psycopg2
apt-get install libpq-dev
apt install python3-pip -y
apt install python3.11-venv
```
**Langkah 2: Instalasi Honeypots di virtual Environment Python3**
```
mkdir Honeypots
python3 -m venv Honeypots
cd Honeypots/
source bin/activate
pip3 install honeypots
```
**Langkah 3: Konfigurasi dan Running**
```
(Honeypots) root@VM1:~/Honeypots# honeypots -h
Qeeqbox/honeypots customizable honeypots for monitoring network traffic, bots activities, and username\password
credentials

Arguments:
  --setup               target honeypot E.g. ssh or you can have multiple E.g ssh,http,https
  --list                list all available honeypots
  --kill                kill all honeypots
  --verbose             Print error msgs

Honeypots options:
  --ip                  Override the IP
  --port                Override the Port (Do not use on multiple!)
  --username            Override the username
  --password            Override the password
  --config              Use a config file for honeypots settings
  --options             Extra options

General options:
  --termination-strategy {input,signal}
                        Determines the strategy to terminate by
  --test                Test a honeypot
  --auto                Setup the honeypot with random port

Chameleon:
  --chameleon           reserved for chameleon project
  --sniffer             sniffer - reserved for chameleon project
  --iptables            iptables - reserved for chameleon project
```
Kita akan Running Honeypots sesuai sekenario saja yaitu SSH
```
python3 -m honeypots --setup ssh --username root --password root --port 22 --options interactive
```

### Konfigurasi Rules IPTABLES

**Langkah 1: Instalasi paket iptables**
```
apt-get install iptables
apt-get install iptables-persistent
```
**Langkah 2: Rule Iptables untuk menghubungkan VM2 ke internet**
```
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```
**Langkah 3: Rule iptables untuk mengalihkan/meredirect paket yang masuk ke VM2 (IP 192.168.20.2) dengan tujuan port 22**
```
iptables -t nat -A PREROUTING -p tcp --dport 22 -j DNAT --to 192.168.20.1
```
**Langkah 4: Save Konfigurais iptables secara permanen dengan persistent**
```
iptables-save > /etc/iptables/rules.v4
```
or
```
dpkg-reconfigure iptables-persistent
```

## 2. Konfigurasi Server pada VM2

### Konfigurasi Adapter Network VM2
**Langkah 1: Buka Konfigurasi utama Networking**
```
nano /etc/network/interfaces
```
**Langkah 2: Edit konfigurasi interfaces**
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network iterface
#auto enp0s3
#allow-hotplug enp0s3
#iface enp0s3 inet dhcp

auto enp0s8
iface enp0s8 inet static
 address 192.168.20.2
 netmask 255.255.255.0
post-up ip route add default via 192.168.20.1 dev enp0s8

auto enp0s9
iface enp0s9 inet static
 address 10.10.10.1
 netmask 255.255.255.0
```
**Langkah 3: Restart**
```
systemctl restart networking
```

### Apa saja yang diKonfigurasi? ###
**1. Web Server**

**2. Mail Server**

**3. Database Server**

**4. VPN Server**

**5. DNS Server**

**6. Port Knocking**

**7. Monitoring Server**

### 1. Web Server

### A. Instalasi dan Konfigurasi Apache2

### B. Konfigurasi CMS Wordpress pada Apache2

### C. Konfigurasi WAF(Web Application Firewall)

**Langkah 1: Instalasi Paket Modsecurity2**
```
apt install libapache2-mod-security2
```
**Langkah 2: Enable modul modsecurity2**
```
a2enmod security2
```
**Langkah 3: Restart Apache2**
```
systemctl restart apache2
```
**Langkah 4: Konfigurasi Modsecurity**
```
nano /etc/apache2/mods-enabled/security2.conf

<IfModule security2_module>
        # Default Debian dir for modsecurity's persistent data
        SecDataDir /var/cache/modsecurity

        # Include all the *.conf files in /etc/modsecurity.
        # Keeping your local configuration in that directory
        # will allow for an easy upgrade of THIS file and
        # make your life easier
        IncludeOptional /etc/modsecurity/*.conf

        # Include OWASP ModSecurity CRS rules if installed
        IncludeOptional /usr/share/modsecurity-crs/*.load
</IfModule>
```
disini terlihat bahwa konfigurasinya terdapat kata "*.conf" yang berarti bahwa apache akan meninclude semua file tersebut yang ada di direktori /etc/modsecurity/
maka dari itu kita harus rename file nya

**Langkah 5: Rename File Konfigurasi utama**
```
mv /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
```
**Langkah 6: Masuk ke File Konfigurasi utama**
```
nano /etc/modsecurity/modsecurity.conf
```
**Langkah 7: Edit Konfigurasi ini**
```
#SecRuleEngine DetectionOnly(ganti ke on)

SecRuleEngine On

#SecAuditLogParts ABDEFHIJZ(ubah ke:)
SecAuditLogParts ABCFHZ
```
agar tampilan log nya lebih simple dan mudah dibaca

**Langkah 8: Restart Apache2**
```
systemctl restart apache2
```
**Langkah 9: Instalasi OWASP CoreRuleSet(CRS)**
```
wget https://github.com/coreruleset/coreruleset/archive/refs/tags/v3.3.5.tar.gz
tar xvf v3.3.5.tar.gz
```
**Langkah 10: Membuat direktori untuk menyimpan CRS**
```
mkdir /etc/apache2/modsecurity-crs/
```
**Langkah 11: Pindahkan File CRS ke Direktori yang dibuat**
```
mv coreruleset-3.3.5/ /etc/apache2/modsecurity-crs/
cd /etc/apache2/modsecurity-crs/coreruleset-3.3.5/
```
**Langkah 12: Ubah nama Example file Konfigurasinya**
```
mv crs-setup.conf.example crs-setup.conf
```
**Langkah 13: ke direktori security2.conf**
```
nano /etc/apache2/mods-enabled/security2.conf
```
**Langkah 14: Ganti CRS rules nya ke CRS yang baru tadi**
```
#IncludeOptional /usr/share/modsecurity-crs/*.load(Ubahlah line ini ke Konfigurasi dibawah ini)
IncludeOptional /etc/apache2/modsecurity-crs/coreruleset-3.3.5/crs-setup.conf
IncludeOptional /etc/apache2/modsecurity-crs/coreruleset-3.3.5/rules/*.conf
```
**Langkah 15: Test Konfigurasi Apache**
```
apache2ctl -t
Syntax OK
```
jika Syntax OK maka lanjut dengan restart layanan
**Langkah 16: Restart Layanan Apache2**
```
systemctl restart apache2
```


### D. Mengamankan Halaman-Halaman Utama Wordpress dengan IP Filtering

**Langkah 1: ke direktori a2site pada Apache2**
```
nano /etc/apache2/sites-available/000-default.conf
```
**Langkah 2: Edit Konfigurasi**
```
<VirtualHost *:80>
    ServerAdmin fenrir@mail.projectman.my.id
    DocumentRoot /var/www/html

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    <Directory "/var/www/html">
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    <Files "wp-config.php">
        <RequireAny>
            Require ip 20.10.20.0/24
            Require host projectman.my.id
        </RequireAny>
    </Files>


    <IfModule mod_headers.c>
        Header set X-XSS-Protection "1; mode=block"
    </IfModule>
</VirtualHost>
```
konfigurasi ini akan memblokir akses ke dirktori yang paling penting yaitu "wp-config.php" netwrok VPN saja.

### E. Instalasi SSL Certificate pada Web server(443 HTTPS)

SSL Certificate nya akan Menggunakan self-Singed SSL dari Openssl

**Langkah 1: insall paket Openssl**
```
apt-get install openssl
```
**Langkah 2: Buat File .Key untuk certificate SSL nya**
```
openssl genpkey -algorithm RSA -out projectman.my.id.key -aes256

buat juga untuk subdomain mail
openssl genpkey -algorithm RSA -out mail.projectman.my.id.key -aes256
```
silahkan ikuti proses ini
```
.....+..+.......+...+...+.....+....+..+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*......+......+....+..+......+.........+....+..+............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*........+.+.....+......+...+......+..........+.....+....+..+......+...+.+...+...+............+.....+............+.........+....+............+.....+.+..+.+...........+.+..+...+.........+...+...+..........+.....+...+................+..............+....+...+...+...........+.........+.........+.+.........+...+..+..........+..+.+...............+..+.+..+.......+......+..+...+.+.........+..+....+...+.....+.+............+.....+............+...+...+............+...+....+......+.........+......+.....+....+...+..+............+................+..+.......+...........+...+.+...........+...+.........+.......+...............+...+......+.....................+.........+..+...+.+...+.....................+.....+....+......+..+............+................+......+...............+.....+......+....+...+...+..+...+....+......+..................+..+.........+.....................+...+......+.+........+.......+...+..+....+.....+......+.............+.........+.....+.+..+.......+...............+...........+.......+..............+...+.+......+...+........+....+.....+.............+..+.+...+......+.....+....+..+....+......+..+.......+...+......+...........+...+.+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
........+...+......+......+.+..................+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*........+.....+.+..+...+......+....+..............+.+..+.........+.........+....+..+.+.....+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+..+....+...+..+....+.....+..........+.....................+.....+...............+...+..............................+......+.+.....+.........+.+......+............+...+..+...+.........+.+.....+...+......+.+..+.+.........+.....+.....................+.+.....+...............+..........+...+...........+....+...+........+....+........+....+...+.....+.+......+......+..+..........+............+....................+...+......+.+...+......+..+............+.+..............+.+..+...+.+...+......+..+....+........+...............+...+...+.+...........+...+....+...+.....+...+......+.+...+....................+....+...+...............+..+......+.......+.....+.......+.....+.....................+......+.+...+.........+............+...+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
root@VM2:~/ssl# ls
projectman.my.id.key
```
**Langkah 3: Membuat File Certificate(.crt) dari key tersebut**
```
openssl req -new -x509 -key projectman.my.id.co.id.key -out projectman.my.crt -days 365 -sha256

buat juga untuk subdomain mail
openssl req -new -x509 -key mail.projectman.my.id.key -out mail.projectman.my.crt -days 365 -sha256
```
ikuti langkah-langkah ini
```
.....+..+.......+...+...+.....+....+..+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*......+......+....+..+......+.........+....+..+............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*........+.+.....+......+...+......+..........+.....+....+..+......+...+.+...+...+............+.....+............+.........+....+............+.....+.+..+.+...........+.+..+...+.........+...+...+..........+.....+...+................+..............+....+...+...+...........+.........+.........+.+.........+...+..+..........+..+.+...............+..+.+..+.......+......+..+...+.+.........+..+....+...+.....+.+............+.....+............+...+...+............+...+....+......+.........+......+.....+....+...+..+............+................+..+.......+...........+...+.+...........+...+.........+.......+...............+...+......+.....................+.........+..+...+.+...+.....................+.....+....+......+..+............+................+......+...............+.....+......+....+...+...+..+...+....+......+..................+..+.........+.....................+...+......+.+........+.......+...+..+....+.....+......+.............+.........+.....+.+..+.......+...............+...........+.......+..............+...+.+......+...+........+....+.....+.............+..+.+...+......+.....+....+..+....+......+..+.......+...+......+...........+...+.+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
........+...+......+......+.+..................+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*........+.....+.+..+...+......+....+..............+.+..+.........+.........+....+..+.+.....+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+..+....+...+..+....+.....+..........+.....................+.....+...............+...+..............................+......+.+.....+.........+.+......+............+...+..+...+.........+.+.....+...+......+.+..+.+.........+.....+.....................+.+.....+...............+..........+...+...........+....+...+........+....+........+....+...+.....+.+......+......+..+..........+............+....................+...+......+.+...+......+..+............+.+..............+.+..+...+.+...+......+..+....+........+...............+...+...+.+...........+...+....+...+.....+...+......+.+...+....................+....+...+...............+..+......+.......+.....+.......+.....+.....................+......+.+...+.........+............+...+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
root@VM2:~/ssl# ls
projectman.my.id.key
root@VM2:~/ssl# openssl req -new -x509 -key projectman.my.id.key -out projectman.my.id.crt -days 365 -sha256
Enter pass phrase for projectman.my.id.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:IN
State or Province Name (full name) [Some-State]:Yogyakarta
Locality Name (eg, city) []:Sleman
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Projectman.my.id
Organizational Unit Name (eg, section) []:IT(Cyber Security)
Common Name (e.g. server FQDN or YOUR name) []:projectman.my.id
Email Address []:fenrir@mail.projectman.my.id
```
**Langkah 4: Gabungkan menjadi 1 file(optional**
```
cat projectman.my.id.key projectman.my.id.crt > projectman.my.id.pem
```
**Langkah 5: copy file ke Direktori yang sesuai**
```
cp projectman.my.id.crt /etc/ssl/certs/
cp projectman.my.id.key /etc/ssl/private/
cp mail.projectman.my.id.key /etc/ssl/private/
cp mail.projectman.my.crt /etc/ssl/certs/
```
**Langkah 6: Konfigurasi ke file .htaccess di Apache2
```
nano /etc/apache2/sites-available/000-default.conf
```
**Langkah 7: ubah isi filenya**
```
<VirtualHost *:80>
    ServerName projectman.my.id

    <If "%{REMOTE_ADDR} !~ m#^20\.10\.20\.#">
        Redirect permanent / https://projectman.my.id/
    </If>
</VirtualHost>

<VirtualHost *:443>
    ServerAdmin fenrir@mail.projectman.my.id
    DocumentRoot /var/www/html

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    <Directory "/var/www/html">
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    <Files "wp-config.php">
        <RequireAny>
            Require ip 20.10.20.0/24
            Require host projectman.my.id
        </RequireAny>
    </Files>

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/projectman.my.id.crt
    SSLCertificateKeyFile /etc/ssl/private/projectman.my.id.key

    <IfModule mod_headers.c>
        Header set X-XSS-Protection "1; mode=block"
    </IfModule>
</VirtualHost>
```
**Langkah 8: Ubah juga .htaccess pada roundcube.conf**
```
nano /etc/apache2/sites-available/roundcube.conf
```
**Langkah 9: Ubahlah isinya**
```
<VirtualHost *:80>
    ServerName mail.projectman.my.id

    <If "%{REMOTE_ADDR} !~ m#^20\.10\.20\.#">
        Redirect permanent / https://mail.projectman.my.id/
    </If>
</VirtualHost>

<VirtualHost *:443>
    ServerAdmin fenrir@projectman.my.id
    DocumentRoot /var/www/roundcube/public_html
    ServerName mail.projectman.my.id

    <Directory "/var/www/roundcube/public_html">
        Options FollowSymLinks
        AllowOverride All
        Require all granted

        RewriteEngine On
        RewriteRule ^roundcube(.*)$ /var/www/roundcube/public_html/roundcube$1 [L]

        <FilesMatch "^/(config|temp|logs)">
            Deny from all
        </FilesMatch>
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/roundcube_error.log
    CustomLog ${APACHE_LOG_DIR}/roundcube_access.log combined

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/mail.projectman.my.crt
    SSLCertificateKeyFile /etc/ssl/private/mail.projectman.my.id.key

    <IfModule mod_headers.c>
        Header set X-XSS-Protection "1; mode=block"
    </IfModule>
</VirtualHost>
```
**Langkah 10: Aktifkan Modul ssl**
```
a2enmod ssl
a2enmod rewrite
```
**Langkah 11: Restart Layanan Apache2**
```
systemctl restart apache2
```
Jika ada notifikasi seperti ini berarti masukkkan key ssl anda
```
root@VM2:~# systemctl restart apache2
🔐 Enter passphrase for SSL/TLS keys for mail.projectman.my.id:443 (RSA): ****
```

### 2. Mail Server

### A. Instalasi dan Konfigurasi Postfix
**Langkah 1: Install Postfix**
```
apt -y install postfix sasl12-bin
```
```
# Pada contoh disini, pilih [No Configuration]
# Dikarenakan kita akan mengonfigurasinya secara Manual

+------+ Postfix Configuration +-------+
| General type of mail configuration:  |
|                                      |
|       No configuration               |
|       Internet Site                  |
|       Internet with smarthost        |
|       Satellite system               |
|       Local only                     |
|                                      |
|                                      |
|       <Ok>           <Cancel>        |
|                                      |
+--------------------------------------+
```
**Langkah 2: Backup file main.cf**
*Dikarenakan kita akan mengubah file main.cf, jadi ada baiknya dibackup dulu supaya jika terjadi kesalahan kita bisa membalikannya*
```
cp /usr/share/postfix/main.cf.dist /etc/postfix/main.cf
```
**Langkah 3: Konfigurasi Main.cf**
```
nano /etc/postfix/main.cf
```
**Ubahlah beberapa baris kode seperti dibawah ini:**
```
# Penjelasan: "Uncomment" artinya menghilangkan tanda pagar, sedangkan "Comment out" adalah sebaliknya
# Baris 82 : uncomment
mail_owner = postfix

# Baris 98 : uncomment dan berikan hostname
myhostname = mail.projectman.my.id

# Baris 106 : uncomment dan berikan domainname
mydomain = projectman.my.id

# Baris 127 : uncomment
myorigin = $mydomain

# Baris 141 : uncomment
inet_interfaces = all

# Baris 189 : uncomment
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

# Baris 232 : uncomment
local_recipient_maps = unix:passwd.byname $alias_maps

# Baris 277 : uncomment
mynetworks_style = subnet

# Baris 294 : Tambahkan Jaringan Lokalmu
mynetworks = 127.0.0.0/8, 192.168.20.0/24

# Baris 416 : uncomment
alias_maps = hash:/etc/aliases

# Baris 427 : uncomment
alias_database = hash:/etc/aliases

# Baris 449 : uncomment
home_mailbox = Maildir/

# Baris 585: comment out dan tambahkan baris kode dibawah
#smtpd_banner = $myhostname ESMTP $mail_name (Debian/GNU)
smtpd_banner = $myhostname ESMTP

# Baris 659 : Tambahkan
sendmail_path = /usr/sbin/postfix

# Baris 664 : Tambahkan
newaliases_path = /usr/bin/newaliases

# Baris 669 : Tambahkan
mailq_path = /usr/bin/mailq

# Baris 675 : Tambahkan
setgid_group = postdrop

# Baris 679 : comment out
#html_directory =

# Baris 683 : comment out
#manpage_directory =

# Baris 688 : comment out
#sample_directory =

# Baris 692 : comment out
#readme_directory =

# Baris 692 : Jika IPv6 dipakai, ganti ke [all], tapi pada kasus ini saya tidak menggunakan IPv6
inet_protocols = ipv4

# Tambahkan kode dibawah ini di baris paling akhir
# disable SMTP VRFY command
disable_vrfy_command = yes

# require HELO command to sender hosts
smtpd_helo_required = yes

# Batasi ukuran Email
# Contoh dibawah adalah batas 10M bytes
message_size_limit = 10240000

# SMTP-Auth settings
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname
smtpd_recipient_restrictions = permit_mynetworks, permit_auth_destination, permit_sasl_authenticated, reject
```
**Langkah 4: Restart Layanan Postfix**
```
newaliases
```
```
systemctl restart postfix
```
#### B. Instalasi dan Konfigurasi Dovecot
**Langkah 1: Install Dovecot**
```
apt -y install-core dovecot-pop3d dovecot-imapd
```

**Langkah 2: Konfigurasi file dovecot.conf**
```
nano /etc/dovecot/dovecot.conf
```
**Ubahlah beberapa baris kode seperti dibawah ini:**
```
# Baris 30 : uncomment
listen = *, ::
```

**Langkah 3: Konfigurasi file 10-auth.conf**
```
nano /etc/dovecot/conf.d/10-auth.conf
```
**Ubahlah beberapa baris kode seperti dibawah ini:**
```
# Baris 10 : uncomment dan ubah (allow plain text auth)
disable_plaintext_auth = no

# Baris 100 : Tambahkan
auth_mechanisms = plain login
```

**Langkah 4: Konfigurasi file 10-mail.conf**
```
nano /etc/dovecot/conf.d/10-mail.conf
```
**Ubahlah beberapa baris kode seperti dibawah ini:**
```
# Baris 30 : Ubah ke Maildir
mail_location = maildir:~/Maildir
```

**Langkah 5: Konfigurasi file 10-master.conf**
```
nano /etc/dovecot/conf.d/10-master.conf
```
**Ubahlah beberapa baris kode seperti dibawah ini:**
```
# Baris 107-109 : uncomment dan tambahkan
  # Postfix smtp-auth
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
```

**Langkah 6: Restart layanan Dovecot**
```
systemctl restart dovecot
```

### C. Konfigurasi Webmail Roundcube

### D. Mengamankan Roundcube secara Umum

Sebelum Mengamankan Roundcube lebih jauh seperti Memasang WAF,kita Terlebih dahulu perlu mengamankan Direktor-Direktori yang berpotensi menjadi sasaran para Penyerang
yang direkomendasikan dari pihak roundcube

**Langkah 1: Hapus Installer Roundcube**
```
rm -rf /var/www/roundcube/installer
```
**Langkah 2: Membuat .htmaccess dan Melakukan a2site hanya untuk direktori /public_html saja pada roundcube**
```
nano /etc/apache2/sites-available/roundcube.conf
```
**Langkah 3: Isi Konfigurasi seperti dibawah ini**
```
<VirtualHost *:80>
    ServerAdmin fenrir@projectman.my.id
    DocumentRoot /var/www/roundcube/public_html
    ServerName mail.projectman.my.id

    <Directory "/var/www/roundcube/public_html">
        Options FollowSymLinks
        AllowOverride All
        Require all granted

        RewriteEngine On
        RewriteRule ^roundcube(.*)$ /var/www/roundcube/public_html/roundcube$1 [L]

        <FilesMatch "^/(config|temp|logs)">
            Deny from all
        </FilesMatch>
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/roundcube_error.log
    CustomLog ${APACHE_LOG_DIR}/roundcube_access.log combined
</VirtualHost>
```
Konfigurasi ini akan melakukan a2site ke halaman /roundcube/public_html saja dan memblokir akses ke direktori
config/temp/logs sesuai rekomendasi roundcube

### E. Instalasi SSL Certificate pada Protocol Mail Server (SMTPS dan IMAPS)
**Langkah 1: Membuat Certificate TLS1.1 untuk SMTP 465**
```
openssl req -new -newkey rsa:2048 -nodes -keyout mail.projectman.my.id.key -out mail.projectman.my.id.csr
openssl x509 -req -days 365 -in  mail.projectman.my.id.csr -signkey mail.projectman.my.id.key -out  mail.projectman.my.id.crt
```
**Langkah 2: Pastikan versinya**
```
root@VM2:/home/ssl# openssl x509 -in mail.projectman.my.id.crt -text -noout
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            3c:d9:6a:a5:1e:92:5c:d6:ac:8c:36:6e:c0:9b:18:48:4a:0d:bf:d2
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = IN, ST = Yogyakarta, L = Sleman, O = mail.projactman.my.id, OU = IT(Cyber Security), CN = mail.projectman.my.id, emailAddress = fenrir@mail.projectman.my.id
        Validity
            Not Before: Nov 12 18:19:15 2023 GMT
            Not After : Nov 11 18:19:15 2024 GMT
        Subject: C = IN, ST = Yogyakarta, L = Sleman, O = mail.projactman.my.id, OU = IT(Cyber Security), CN = mail.projectman.my.id, emailAddress = fenrir@mail.projectman.my.id
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:bb:85:d6:1d:0f:15:e7:5b:ac:55:85:bc:5e:70:
                    b6:43:04:d3:3b:bc:57:e5:ad:91:ee:ba:60:30:e7:
                    40:e7:ff:31:e8:fb:e1:31:f4:bb:db:41:48:c1:32:
                    e6:f0:58:be:9b:90:07:6d:5f:34:93:3b:ea:8d:6d:
                    9f:73:6a:80:35:38:1a:e1:3f:94:bf:63:81:3f:b3:
                    a6:90:94:91:96:50:4b:54:e9:bb:ab:ec:f1:4f:35:
                    4c:c2:78:e5:c2:3a:78:ee:19:92:0a:d0:4c:83:a6:
                    c8:cd:25:b1:0b:d2:1d:1f:97:e0:91:9f:5d:0d:da:
                    43:18:22:8d:6e:d3:4a:5d:fe:dd:a8:0f:25:a4:2e:
                    ff:16:ca:0e:fe:9a:fa:0e:6a:da:43:00:dc:04:40:
                    2e:2f:86:0f:f2:15:eb:35:a1:60:29:d5:97:04:79:
                    c6:d7:1a:63:8d:e2:7e:98:d0:70:46:6e:c8:f9:ed:
                    42:da:28:c6:a4:82:cd:65:ea:8f:73:39:3f:26:a9:
                    79:e4:43:13:3f:5b:af:57:0b:fa:6d:f0:76:03:8d:
                    67:bd:ce:c7:26:77:b4:bf:8f:5c:d5:e0:2a:30:39:
                    2d:c7:89:ac:60:53:c0:15:6c:b7:9a:4d:16:55:37:
                    b5:18:f5:65:84:00:52:07:38:f7:0d:80:46:4b:ec:
                    17:27
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        10:8a:c6:83:29:fa:2c:83:52:4b:4c:69:00:e3:01:f5:c8:db:
        37:7c:15:c4:f8:58:ce:45:b8:f6:f5:f3:c4:78:7e:04:67:da:
        6b:03:b1:3f:a2:5f:75:87:22:31:fa:74:0c:27:58:e9:6b:f7:
        a7:0e:30:fd:78:36:9f:75:98:80:fc:87:24:b4:f4:52:37:7d:
        dd:31:7c:e3:00:c3:71:f9:fd:dd:f2:3c:86:8d:d7:17:d9:d9:
        6d:03:8c:4c:a7:9a:4f:70:e0:2a:ac:b8:1f:36:ef:8a:1a:c9:
        cc:56:f1:07:79:54:01:0b:b2:da:3f:46:86:49:4d:b5:d7:41:
        c4:d9:28:e2:ca:27:99:a8:d6:90:be:03:23:9f:0c:9d:46:fc:
        74:8d:3c:31:7d:cf:3f:0e:f4:b8:a9:8e:1f:dc:f9:4d:82:6b:
        6a:6e:f0:40:3a:4a:d8:54:39:7d:c4:43:fc:4e:e5:a4:82:4a:
        8f:f6:fd:f7:54:05:a3:10:c1:85:06:ba:0b:24:6a:e5:3e:82:
        b2:1c:c0:f4:9d:2b:c5:d2:3e:2d:76:c5:52:c6:bc:20:85:14:
        a7:b5:27:0e:89:8a:1d:cc:30:17:ee:bd:03:0d:00:f6:34:60:
        95:fb:f6:7f:37:33:98:f4:93:09:02:1d:ce:c6:c0:18:ca:f8:
        61:83:4f:be
```
**Langkah 3: Buka Direktori main.cf**
```
nano /etc/postfix/main.cf
```
**Langkah 4: Tambahkan di baris akhir**
```
inet_protocols = ipv4
disable_vrfy_command = yes
smtpd_helo_required = yes
message_size_limit = 10240000

smtpd_use_tls = yes
smtpd_tls_cert_file = /home/ssl/mail.projectman.my.id.crt
smtpd_tls_key_file = /home/ssl/mail.projectman.my.id.key
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtpd_tls_security_level=may
```
**Langkah 5: Buka File master.cf**
```
nano /etc/postfix/master.cf
```
**Langkah 6: Edit Konfigurasinya**
```
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/submissions
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
```
**Langkah 7: Konfigurasi Dovecot**
```
nano /etc/dovecot/conf.d/10-ssl.conf
```
**Langkah 8: Edit Konfigurasi**
```
# line 6 : ganti
ssl = yes

# line 12,13 : ganti ke direktori ssl certificate
ssl_cert = </etc/ssl/certs/mail.projectman.my.crt
ssl_key = </etc/ssl/private/mail.projectman.my.id.key
```
ini masih menggunakan Certificate SSL yang versi sebelumnya yaitu 1.3(yang digunakan untuk HTTPS)
jadi yang menggunakan TLS 1.1 hanya SMTP saja
**Langkah 9: Buka file Konfigurasi utama Webmailroundcube**
```
nano /var/www/roundcube/config/config.inc.php
```
**Langkah 10: Sesuaikan Konfigurasinya**
```
// IMAP host chosen to perform the log-in.
// See defaults.inc.php for the option description.
$config['imap_host'] = 'ssl://mail.projectman.my.id:993';

// SMTP server host (for sending mails).
// See defaults.inc.php for the option description.
$config['smtp_host'] = 'ssl://mail.projectman.my.id:465';
$config['smtp_auth_type'] = 'PLAIN';
// SMTP username (if required) if you use %u as the username Roundcube
// will use the current username for login
$config['smtp_user'] = '%u';

// SMTP password (if required) if you use %p as the password Roundcube
// will use the current user's password for login
$config['smtp_pass'] = '%p';

// provide an URL where a user can get support for this Roundcube installation
// PLEASE DO NOT LINK TO THE ROUNDCUBE.NET WEBSITE HERE!
$config['support_url'] = '';

$config['imap_conn_options'] = array(
    'ssl' => array(
        'verify_peer' => false,
        'allow_self_signed' => true,
    ),
);
$config['smtp_conn_options'] = array(
    'ssl' => array(
        'verify_peer' => false,
        'allow_self_signed' => true,
    ),
);
```
sesuaikan seperti ini

**Langkah 11: Cek apakah SMTP bisa berjalan dengan SSL certificate TLS 1.1**
```
openssl s_client -connect mail.projectman.my.id:465
```
Hasilnya
```
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1587 bytes and written 407 bytes
Verification error: self-signed certificate
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 18 (self-signed certificate)
---
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: A41B409BE7975113C514A26E55886073951B14801B5B5235351B9F8537074F0F
    Session-ID-ctx:
    Resumption PSK: 63AD4A6554D8E34D3E7867763A5422CF98236F77D8C2B0F138931EED56347517A8D4AB84A08ABA840322B05D8FBFA455
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 7200 (seconds)
    TLS session ticket:
    0000 - c1 f1 aa 61 4c 07 bc bd-db fd e2 67 fb c3 2c d7   ...aL......g..,.
    0010 - f9 0d d3 53 7c 5f c1 b3-5f f9 03 4b 4d dd 55 d5   ...S|_.._..KM.U.
    0020 - ab 92 42 5e 96 52 c0 51-96 19 43 63 57 7d c0 61   ..B^.R.Q..CcW}.a
    0030 - 22 e1 45 1c 6f 5e c8 8b-d7 e1 eb 5b 0d 08 6d 73   ".E.o^.....[..ms
    0040 - 28 5a bd e4 58 c9 b1 d2-2c fc fd 5f 8c 2c 16 73   (Z..X...,.._.,.s
    0050 - 0a b6 b8 da d0 df 05 a5-8b 03 1c bd 06 fa 90 8d   ................
    0060 - 4d ff 70 17 aa 59 c0 44-7e 39 67 93 48 ba 96 53   M.p..Y.D~9g.H..S
    0070 - 0b d4 ec 1d 64 75 7d e0-da 5f d6 cb 59 66 bf ed   ....du}.._..Yf..
    0080 - f6 40 08 00 52 ee 01 12-a2 06 8e b5 1e c8 fe a7   .@..R...........
    0090 - f2 63 6f 25 8a 88 ed 10-ef 91 e3 fb de 71 a6 1a   .co%.........q..
    00a0 - e5 5e 6d e6 9f 64 7c 6f-df 93 4e 66 4d 70 fe 81   .^m..d|o..NfMp..
    00b0 - b0 d6 64 bd 38 8f a4 60-9f 41 01 47 0f 7c 33 49   ..d.8..`.A.G.|3I
    00c0 - 76 65 aa 71 e6 6c 93 fa-f9 8c ee 87 53 c6 c8 22   ve.q.l......S.."

    Start Time: 1699814108
    Timeout   : 7200 (sec)
    Verify return code: 18 (self-signed certificate)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
220 mail.projectman.my.id ESMTP
```

### 3. Database Server
### A. Instalasi dan konfigurasi Mariadb dan Phpmyadmin
## Instalasi MariaDB
**Langkah 1: Instalasi Paket MariaDB**
```
sudo apt-get update
sudo apt-get install mariadb-server
```
## Konfigurasi MariaDB
**Langkah 1: Jalankan Perintah ini**
```
mysql_secure_installation
```
**Langkah 2: Ikuti Konfigurasi dibawah ini**

Anda Bisa Mengikuti Langkah-Langkah ini:
```
NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user. If you've just installed MariaDB, and
haven't set the root password yet, you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password or using the unix_socket ensures that nobody
can log into the MariaDB root user without the proper authorisation.

You already have your root account protected, so you can safely answer 'n'.

Switch to unix_socket authentication [Y/n] n
 ... skipping.

You already have your root account protected, so you can safely answer 'n'.

Change the root password? [Y/n] n
 ... skipping.

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n
 ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```
Bagian "Disallow root login remotely" bisa diatur sesuai keperluan,karena pada bagian perlu itu login dengan root account.

## Instalasi dan Konfigurasi Phpmyadmin
**Langkah 1: Lakukan instalasi paket**
```
apt-get install phpmyadmin
```
**Langkah 2: Konfigurasi Phpmyadmin**
Bisa langsung Konfigurasi otomatis agar bisa ditampilkan ke Web Server anda
![01](https://github.com/Xzhacts-Crew/ProjectMen/assets/148263594/b47f0218-35c4-4ec5-9f40-318bc205eb0c)

atau bisa Menggunakan Command Line
```
ln -s /etc/phpmyadmin/apache.conf /etc/apache2/conf-available/phpmyadmin.conf
a2enconf phpmyadmin
systemctl reload apache2
```
## Menguji Konfigurasi
berhasil dibuka melalui Webserver Apache2

![02](https://github.com/Xzhacts-Crew/ProjectMen/assets/148263594/b605cab2-ea0a-47aa-87f4-db1d6b3f9ffc)

### B. Mengamankan MariaDB dan phpmyadmin dengan UFW dan IP FIlTERING

**Langkah 1: Membuka direktori utama Mariadb**
```
nano /etc/mysql/mariadb.cnf
```
**Langkah 2: Mengedit Konfigurasi**
```
#
# If the same option is defined multiple times, the last one will apply.
#
# One can use all long options that the program supports.
# Run program with --help to get a list of available options and with
# --print-defaults to see which it would actually understand and use.
#
# If you are new to MariaDB, check out https://mariadb.com/kb/en/basic-mariadb-articles/

#
# This group is read both by the client and the server
# use it for options that affect everything
#
[client-server]
# Port or socket location where to connect
#port = 3306
socket = /run/mysqld/mysqld.sock

[mysqld]
bind-address = 127.0.0.1
port = 3306
log-bin
# Import all .cnf files from configuration directory
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mariadb.conf.d/
```
ini akan membuat database server hanya bisa diakses dari localhost

**Langkah 3: Restart**
```
systemctl restart mysqld mariadb.service
```

**Langkah 4: Buka Konfigurasi .htmaccess phpmyadmin**
```
nano /etc/apache2/conf-available/phpmyadmin.conf
```
**Langkah 5: Edit Konfigurasi ini**
```
# phpMyAdmin default Apache configuration

Alias /phpmyadmin /usr/share/phpmyadmin

<Directory /usr/share/phpmyadmin>
    Options SymLinksIfOwnerMatch
    DirectoryIndex index.php

    # limit libapache2-mod-php to files and directories necessary by pma
    <IfModule mod_php7.c>
        php_admin_value upload_tmp_dir /var/lib/phpmyadmin/tmp
        php_admin_value open_basedir /usr/share/phpmyadmin/:/usr/share/doc/phpmyadmin/:/etc/phpmyadmin/:/var/lib/phpmyadmin/:/usr/share/php/:/usr/share/javascript/
    </IfModule>

    # PHP 8+
    <IfModule mod_php.c>
        php_admin_value upload_tmp_dir /var/lib/phpmyadmin/tmp
        php_admin_value open_basedir /usr/share/phpmyadmin/:/usr/share/doc/phpmyadmin/:/etc/phpmyadmin/:/var/lib/phpmyadmin/:/usr/share/php/:/usr/share/javascript/
    </IfModule>

        Require ip 20.10.20.0/24
</Directory>

# Disallow web access to directories that don't need it
<Directory /usr/share/phpmyadmin/templates>
    Require all denied
</Directory>
<Directory /usr/share/phpmyadmin/libraries>
    Require all denied
</Directory>
```
**Langkah 6: Reload**
```
systemctl reload apache2
```
**Langkah 7: Block akses ke port 3306 dari luar**
```
ufw deny 3306
```

### 4. Instalasi dan Konfigurasi OPENVPN

### A. Instalasi OpenVPN dan Konfigurasi
**Langkah 1: Instalasi paket Openvpn & iptables**
```
apt-get install -y openvpn easy-rsa iptables
```
**Langkah 2: Buka direktori easyrsa**
```
cd /usr/share/easy-rsa
```
**Langkah 3: Membuat file PKI**
```
./easyrsa init-pki
```
**Langkah 4: Membuat CA**
```
./easyrsa build-ca

* Notice:
Using Easy-RSA configuration from: /usr/share/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)


Enter New CA Key Passphrase:
Re-Enter New CA Key Passphrase:
Using configuration from /usr/share/easy-rsa/pki/93c96b9d/temp.9d426c95
.+.........+.........+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.....+...................+..+.+..................+.........+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...+.....+.......+...+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
...+..+.+........+.......+...+.........+............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+...+...............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+...+............+......+.......+..+...+...............+..........+.....+..........+..+..........+..+....+......+...+......+.....+......+.+...+......+..............+......+.........+.+.....................+.....+....+...+.....+.+.....+.......+..+......................+.....+...............+.+........+....+......+.....+......+......+....+.........+..............+...+....+........+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:projectman.my.id

* Notice:

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/usr/share/easy-rsa/pki/ca.crt
```
**Langkah 5: Membuat CA certificate untuk Server**
```
./easyrsa build-server-full server1 nopass

* Notice:
Using Easy-RSA configuration from: /usr/share/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)

.....+....+...+.....+...+................+.....+.+.....+...+....+.....+......+......+.+...+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.....+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*......+.....+....+.....................+...+...+............+..+...+.+.........+........+....+........+......+....+..+....+.....+.......+..+......+.......+.........+...+.......................+....+.....+...+.+............+..+......+.......+...+.....+......+.+...+..+.........+.........+....+...+...+..+.........+....+......+...........+...+.+......+..+..........+...+...+.....................+......+..+..........+...+.....+...+.........+.+..+....+...+...........+...............+.+...+..+.+..............+....+......+.....+.+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
.+.+...+...........+.+......+...+.....+......+....+............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+...+...........+....+...+..+..........+...........+.+......+..+.+.....+....+.....+.......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...+...........+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----
* Notice:

Keypair and certificate request completed. Your files are:
req: /usr/share/easy-rsa/pki/reqs/server1.req
key: /usr/share/easy-rsa/pki/private/server1.key


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 825 days:

subject=
    commonName                = server1


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes

Using configuration from /usr/share/easy-rsa/pki/957ade41/temp.1fbdc4d7
Enter pass phrase for /usr/share/easy-rsa/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'server1'
Certificate is to be certified until Feb 14 05:10:06 2026 GMT (825 days)

Write out database with 1 new entries
Database updated

* Notice:
Certificate created at: /usr/share/easy-rsa/pki/issued/server1.crt

```
**Langkah 6: Membuat Client Certificate**
```
./easyrsa build-client-full client1 nopass
*** Notice:
Using Easy-RSA configuration from: /usr/share/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)

..+....+..............+............+...+.+...+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...+..+.+......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.................+......+.........+...+..+......+...+............+.......+..+................+.....+....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
...+......+...+...........+.+........................+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+............+..+...............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*....+............+.+..............+.+..+.............+.....+......+.........+.........+.+......+...+........+.........+......+.......+...+..+..........+............+......+..+.+.....+.........+......+.+......+...+......+...+...+..+...+....+...+...+........+......+...+.+...........+.........+............+....+........+............+.+..+...+...+....+..+...+...+.......+..+....+.................+.......+..................+..+............+.+...+.....+..........+...........+...+......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----
* Notice:

Keypair and certificate request completed. Your files are:
req: /usr/share/easy-rsa/pki/reqs/client1.req
key: /usr/share/easy-rsa/pki/private/client1.key


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a client certificate for 825 days:

subject=
    commonName                = client1


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes

Using configuration from /usr/share/easy-rsa/pki/993657cf/temp.3261277e
Enter pass phrase for /usr/share/easy-rsa/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'client1'
Certificate is to be certified until Feb 14 05:12:07 2026 GMT (825 days)

Write out database with 1 new entries
Database updated

* Notice:
Certificate created at: /usr/share/easy-rsa/pki/issued/client1.crt
**
```
**Langkah 7: Generate DH paramenter**
```
./easyrsa gen-dh

* Notice:
Using Easy-RSA configuration from: /usr/share/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)

Generating DH parameters, 2048 bit long safe prime
..........++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*

* Notice:

DH parameters of size 2048 created at /usr/share/easy-rsa/pki/dh.pem
```
**Langkah 8: Membuat TLS Auth Key**
```
openvpn --genkey secret ./pki/ta.key
```
**Langkah 9: Copy sample Config Openvpn**
```
cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server/
```
**Langkah 10: Copy Certificate yang digenerate ke Direktori utama Openvpn**
```
cp -pR /usr/share/easy-rsa/pki/{issued,private,ca.crt,dh.pem,ta.key} /etc/openvpn/server/
```
**Langkah 11: Buka Direktori utama Openvpn**
```
nano /etc/openvpn/server/server.conf
```
**Langkah 12: Ubahlah isinya seperti**
```
# line 32 :  Menggunakan Port default 1194
port 1194
# line 35 : menggunakan Protocol UDP
;proto tcp
proto udp
# line 53 : ubah ke ke Tap 
dev tap
;dev tun

# line 78 : Arahkan ke direktori spesifik dari Certificate tadi
ca ca.crt
cert issued/server1.crt
key private/server1.key

# line 85 : Aarahkan ke lokasi file dh.pem
dh dh.pem

# line 101 : Subnet yang digunakan VPN saya akan menyesuaikan topologi
server 20.10.20.0 255.255.255.0

# line 142 : Change Routing table ke network local(yang menuju VM3)
push "route 10.10.10.0 255.255.255.0"

# line 231 : Settingan keepalive 
keepalive 10 120

# line 244 : File Auth Key
tls-auth ta.key 0

# line 281 : aktifkan persistent
persist-key
persist-tun

# line 306 : Log level ke 3 (0 - 9,artinya debug level)
verb 3
```
**Langkah 13: Membuat script bridge.sh**
```
nano /etc/openvpn/server/add-bridge.sh
```
isi seperti ini
```
#!/bin/bash

# network interface dari VPN
IF=enp0s8
# interface yang digunakan VPN tunnel
VPNIF=tap0

echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -A FORWARD -i ${VPNIF} -j ACCEPT
iptables -t nat -A POSTROUTING -o ${IF} -j MASQUERADE
```
**Langkah 14: Membuat script remove bridge.sh**
```
nano /etc/openvpn/server/remove-bridge.sh

#!/bin/bash
# sama seperti sebelumnya
IF=enp0s8
# interface VPN
VPNIF=tap0

echo 0 > /proc/sys/net/ipv4/ip_forward
iptables -D FORWARD -i  ${VPNIF} -j ACCEPT
iptables -t nat -D POSTROUTING -o ${IF} -j MASQUERADE
```
**Langkah 15: Ubah hak akses file dengan chmod**
```
chmod 700 /etc/openvpn/server/{add-bridge.sh,remove-bridge.sh}
```
**Langkah 16: Tambahkan Script Tersebut ke Service Openvpn**
```
nano /lib/systemd/system/openvpn-server@.service

# tambahkan ke section [service]
[Service]
.....
.....
ExecStartPost=/etc/openvpn/server/add-bridge.sh
ExecStopPost=/etc/openvpn/server/remove-bridge.sh
```
**Langkah 17: Restart Layanan**
```
systemctl daemon-reload
systemctl enable --now openvpn-server@server
```

**Langkah 18: Jika terjadi error seperti salah penempatan folder bisa gunakan command ini**
```
cp -r /etc/openvpn/server/!(add-bridge.sh|remove-bridge.sh) /etc/openvpn/
```

### 5. DNS Server
### A. Instalasi dan Konfigurasi BIND (Berkeley Internet Name Domain)
**Langkah 1: Instalasi BIND**
```
apt -y install bind9 bind9utils
```

**Langkah 2: Copy file untuk Konfigurasi "Forward" dan "Reverse"**
```
cd /etc/bind
root@VM2:/etc/bind# ls

bind.keys  db.127  db.empty  named.conf                named.conf.local    rndc.key
db.0       db.255  db.local  named.conf.default-zones  named.conf.options  zones.rfc1918
cp db.local db.forward
cp db.127 db.reverse
```
**Langkah 3: Konfigurasi file db.forward**
```
nano db.forward
```
**Ubahlah Konfigurasi seperti dibawah ini:**
```
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     projectman.my.id. root.projectman.my.id. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      projectman.my.id.
@       IN      A       192.168.20.2
ns      IN      A       192.168.20.2
www     IN      A       192.168.20.2
mail    IN      A       192.168.20.2
@       IN      MX      10 mail.projectman.my.id.
```

**Langkah 4: Konfigurasi file db.reverse**
```
nano db.reverse
```
**Ubahlah Konfigurasi seperti dibawah ini:**
```
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     projectman.my.id. root.projectman.my.id. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      projectman.my.id.
10      IN      PTR     projectman.my.id.
```

**Langkah 5: Buka Konfigurasi named.conf.local untuk konfigurasi DNS Zones**
```
nano named.conf.local
```
**Ubahlah Konfigurasi seperti dibawah ini:**
```
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "projectman.my.id" {
        type master;
        file "/etc/bind/db.forward";
};
zone "20.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/db.reverse";
};
```

**Langkah 6: Konfigurasi Forwarders**
```
nano named.conf.options
```
**Tambahkan DNS Forwarders**
```
options {
        directory "/var/cache/bind";

        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

        // forwarders {
        //      192.168.20.2;
        //      8.8.8.8;
        // };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation auto;

        listen-on-v6 { any; };
};
```

**Langkah 7: Konfigurasi DNS diperangkat Server**
```
nano /etc/resolv.conf
```
**Ubahlah Konfigurasi seperti dibawah ini:**
```
domain projectman.my.id
search projectman.my.id
nameserver 192.168.20.2
nameserver 8.8.8.8
```
**Langkah 8: Restart Layanan BIND9**
```
systemctl restart bind9
```

#### B. Pengetesan Konfigurasi DNS
**Langkah 1: Instalasi DNS Resolver**
```
apt-get install dnsutils
```

**Langkah 2: Test DNS**
```
root@VM2:/etc/bind# nslookup projectman.my.id
Server:         192.168.20.2
Address:        192.168.20.2#53

Name:   projectman.my.id
Address: 192.168.20.2

root@VM2:/etc/bind# nslookup 192.168.20.2
2.20.168.192.in-addr.arpa     name = projectman.my.id.
```

### 6. Instalasi dan Konfigurasi Port Knocking(Knockd)


### A. Instalasi Kncokd

**Langkah 1: Instalasi knockd**
```
apt-get install knockd
```
**Langkah 2: Buka File Konfigurasi utama Knockd**
```
nano /etc/knockd.conf
```
**Langkah 3:Edit Sequence Number seperti sekanrio**
```
[options]
        logfile = /var/log/knockd.log

[openSSH]
        sequence    = 3200,7600,9900
        seq_timeout = 5
        command     = ufw allow from %IP% to any port 9029 comment 'Port Knocking Access'
        tcpflags    = syn

[closeSSH]
        sequence    = 9900,7600,3200
        seq_timeout = 5
        command     = ufw delete allow from %IP% to any port 9029 comment 'Port Knocking Access'
        tcpflags    = syn
```
**Langkah 4: Buka Konfigurasi untuk listen Interface nya**
```
nano /etc/default/knockd
```
**Langkah 5: Edit Konfigurasi listen interface knockd**
```
# control if we start knockd at init or not
# 1 = start
# anything else = don't start
# PLEASE EDIT /etc/knockd.conf BEFORE ENABLING
START_KNOCKD=1

# command line options
KNOCKD_OPTS="-i tap0"
```
**Langkah 6: Restart Layanan Knockd**
```
systemctl restart knockd
systemctl enable knockd
```


### 7. Instalasi dan Konfigurasi Monitoring Log Server(Loki,Promtail,Rsyslog)

### A. Instalasi Loki& Promtail

**Langkah 1: Instalasi Paket-Paket Yang diperlukan**
```
wget -q -O /usr/share/keyrings/grafana.key https://packages.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://packages.grafana.com/oss/deb stable main" | tee -a /etc/apt/sources.list.d/grafana.list
apt-get update
apt-get install loki
apt-get install promtail
apt-get install rsyslog
apt-get install grafana
apt-get install acl
```
**Langkah 2: berikan akses ke direktori /var/log/*" kepada user Promtail**
```
setfacl -R -m u:promtail:rX /var/log
```
**Langkah 3: Buka Konfigurasi Utama Promtail**
```
nano /etc/promtail/config.yml
```
**Langkah 4: Tambahkan Job Monitoring**
```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      __path__: /var/log/*log
```
Pada bagian job_name bisa anda tambahkan ke direktori yang anda ingin kan, misal direktori /var/www/html/error.log. itu bisa ditambahkan sebagai target log yang akan dicapture dan di export oleh Promtail

**Langkah 5: Ke Konfigurasi utama Loki**
```
nano /etc/loki/config.yml
```
**Langkah 6: Ubah Sesuai Kebutuhan**
```
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

# By default, Loki will send anonymous, but uniquely-identifiable usage and configuration
# analytics to Grafana Labs. These statistics are sent to https://stats.grafana.org/
#
# Statistics help us better understand how Loki is used, and they show us performance
# levels for most users. This helps us prioritize features and documentation.
# For more information on what's sent, look at
# https://github.com/grafana/loki/blob/main/pkg/usagestats/stats.go
# Refer to the buildReport method to see what goes into a report.
#
# If you would like to disable reporting, uncomment the following lines:
#analytics:
#  reporting_enabled: false
```
anda bisa mengganti port atau path penyimpanan log dari Loki

**Langkah 7: Buka halaman http://Domain atau IP anda:3100/metrics**
![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/ab856e8f-4235-443b-be0b-c6f3a32014a5)

### B. Visualisasi Log Ke Grafana

**Langkah 1: Enable Grafana**
```
systemctl enable grafana-server
systemctl start grafana-server
```
**Langkah 2: Buka Konfigurasi Utama Grafana**
```
nano /etc/grafana/grafana.ini
```
**Langkah 3: Edit Konfigurasi**
```
#Hilangkan tanda ";"
[server]
# Protocol (http, https, h2, socket)
protocol = http

# This is the minimum TLS version allowed. By default, this value is empty. Accepted values are: TLS1.2, TLS1.3. If nothing is set TLS1.2 would be taken
;min_tls_version = ""

# The ip address to bind to, empty will bind to all interfaces
;http_addr =

# The http port  to use
http_port = 3000

# The public facing domain name used to access grafana from a browser
domain = localhost
```
**Langkah 4: Restart Grafana**
```
systemctl restart grafana-server
```
**Langkah 5: Login ke Grafana**

![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/1ae7d71c-17b4-4e5e-ae23-6070ee4485d3)
Login = "admin password="admin"

**Langkah 6: Add Data Source**

![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/92aff166-f66d-443d-b958-cf36bda9e2ac)
'Home > Connection > Data Source

**Langkah 7: Search Loki**
![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/4202a014-48da-4504-b090-a50f4532f102)

**Langkah 8: Tambahkan URL dimana Loki Berjalan**
![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/acf063e9-30f6-4fe9-8454-5e808825e564)


**Langkah 8: Membuat Dashboard baru**
![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/fcacae1f-41cd-4adc-90cd-8cf824ea285c)


**Langkah 9: Pilih Data Source Loki**
![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/715b9a54-4290-4e9b-b3d2-4ed121646d25)


Kemudian pada bagian Label Filters : Pilih "job/filename" kemudian pilih "varlogs"

**Langkah 10: Klik Visualisation dan pilih Logs**
![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/26726a9e-0801-4499-aecf-2c9ec9a4a266)
kemudian klik "run Query"

**Langkah 11:Klik Apply dan save untuk Membuat Dashboard**
![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/1d2db53e-7ab4-4fe5-a67f-f09f8e4516e2)


 Sekarang Muncul sebuah Tampilan Log,dimana anda bisa explore dan membuat dashboard Log anda sendiri

**Langkah 12: ini Dashoard Dari Monitoring Logs nya**
![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/64aa9451-7c2c-4eda-a60e-fa76658f4901)

### C. Mengamankan Grafana dengan UFW dan SSL Cerfificate

**Langkah 1: Membuat Self Singed Certificate**
```
openssl genpkey -algorithm RSA -out server.key
```
**Langkah 2: Membuat CSR**
```
openssl req -new -key server.key -out server.csr
```
**Langkah 3: Membuat CA Certificate**
```
openssl x509 -req -in server.csr -signkey server.key -out server.crt -days 365
```
**Langkah 4: Ubah Pemilik dan Hak Akses Certificate nya**
```
chown grafana:grafana server.crt
chown grafana:grafana server.key
chmod 776 server.key server.crt
```
**Langkah 5: Pindahkan file nya**
```
mv server.key server.crt /etc/grafana/
```
**Langkah 7: Buka Direktori Konfigurasi Grafana**
```
nano /etc/grafana/grafana.ini
```
**Langkah 8: Edit Konfigurasinya**
```
[server]
# Protocol (http, https, h2, socket)
protocol = https

# This is the minimum TLS version allowed. By default, this value is empty. Accepted values are: TLS1.2, TLS1.3. If nothing is set TLS1.2 would be taken
;min_tls_version = ""

# The ip address to bind to, empty will bind to all interfaces
http_addr = 20.10.20.1

# The http port  to use
http_port = 3000

# The public facing domain name used to access grafana from a browser
domain = localhost

# Redirect to correct domain if host header does not match domain
# Prevents DNS rebinding attacks
;enforce_domain = false

# The full public facing url you use in browser, used for redirects and emails
# If you use reverse proxy and sub path specify full url (with sub path)
;root_url = %(protocol)s://%(domain)s:%(http_port)s/

# Serve Grafana from subpath specified in `root_url` setting. By default it is set to `false` for compatibility reasons.
;serve_from_sub_path = false

# Log web requests
;router_logging = false

# the path relative working path
;static_root_path = public

# enable gzip
;enable_gzip = false

# https certs & key file
cert_file = /etc/grafana/server.crt
cert_key = /etc/grafana/server.key
```
**Langkah 8: (Optional)Cek  Versi TLS/TLS**
```
openssl rsa -in server.key -text -noout
openssl rsa -in crt.key -text -noout

Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            30:9b:3f:96:b3:7d:91:ed:7f:75:ba:92:65:b5:5a:80:81:ea:31:2a
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = IN, ST = yogyakarta, L = Sleman, O = Projectman, OU = IT(Cyber Security), CN = 20.10.20.1, emailAddress = fenrir@mail.projectman.my.id
        Validity
            Not Before: Nov 13 09:42:14 2023 GMT
            Not After : Nov 12 09:42:14 2024 GMT
        Subject: C = IN, ST = yogyakarta, L = Sleman, O = Projectman, OU = IT(Cyber Security), CN = 20.10.20.1, emailAddress = fenrir@mail.projectman.my.id
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:cb:dd:c4:62:c9:70:5f:b6:80:75:ba:87:b3:73:
                    44:b1:f4:71:c4:23:89:ce:34:51:e5:06:b1:ac:ec:
                    48:1b:ff:19:3f:47:0a:a1:90:8f:1b:e4:c7:c5:32:
                    02:80:05:7a:c0:e6:2d:e0:d7:39:1c:6d:db:23:c6:
                    53:22:9d:b1:04:0d:a1:fd:da:25:d8:8f:19:86:9e:
                    d5:93:9d:97:77:20:f6:f6:98:42:bd:52:fb:1d:68:
                    e0:c0:ae:d3:54:a9:a7:a3:9b:47:4e:8f:38:2f:d2:
                    63:dc:ad:80:ed:d1:22:32:83:c5:df:c6:01:13:88:
                    18:d9:ee:5c:25:fd:ed:45:fc:9d:24:34:ac:06:5c:
                    66:cf:3f:b6:60:e7:7d:95:d9:1e:33:4d:9d:36:9e:
                    49:4f:bc:41:aa:ef:08:67:b8:a0:4a:29:bb:95:51:
                    86:c0:eb:04:5e:62:a4:74:f1:53:ba:17:6c:03:2f:
                    f0:91:32:ed:4c:e3:a7:da:55:ae:9f:03:f0:f8:04:
                    00:04:b7:ee:93:02:44:1e:3a:53:e0:36:d8:64:6a:
                    da:0f:76:b0:07:d1:5e:c1:f9:31:a1:bb:15:9a:9b:
                    0d:39:e0:13:f0:41:a9:57:3a:72:87:6c:0d:52:e1:
                    8d:ea:c5:87:81:3a:8b:72:1e:75:58:e2:19:ce:e7:
                    ea:35
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha256WithRSAEncryption
```
**Langkah 9: Cek halaman Homepage Grafana**
![image](https://github.com/Xzhacts-Crew/ProjectMen/assets/147627144/9ee0c03b-708f-489f-b33a-514433d994b9)


**Langkah 10: Mengamankan Grafana dan Metrics dari Promtail dengan ufw**
```
ufw allow from 20.10.20.0/24 to any port 3000
ufw allow from 20.10.20.0/24 to any port 3100
ufw reload
```
jadi Hanya Subnet dari VPN saja yang bisa Mengakses Monitoring Log Server nya


## 3. Konfigurasi Server Backup VM3

### A. Konfigurasi Adapter Network VM3

**Langkah 1: Buka Direktori utama interfaces**
```
nano /etc/network/interfaces
```
**Langkah 2: Edit Konfigurasi nya**
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network iterface
#auto enp0s3
#allow-hotplug enp0s3
#iface enp0s3 inet dhcp

auto enp0s3
iface enp0s3 inet static
 address 10.10.10.2
 netmask 255.255.255.0
```
**Langkah 3: Restart Networking**
```
systemctl restart networking
```


### B. Instalasi Rsync

**Langkah 1: Instalasi paket Rsync dan Mariadb-Client& cron di server utama(10.10.10.1)**
```
apt install rsync mariadb-client cron
```
**Langkah 2: instalasi Rsync di Server backup(10.10.10.2)**
```
apt install rsync
```
**Langkah 3: Membuat Script Backup Sqldump di Server utama(10.10.10.1)**
```
mkdir /home/mysql_backup
cd /home/mysql_backup
touch backup.sh
nano backup.sh
```
**Langkah 4: isi Script backup(10.10.10.1)**
```
#!/bin/bash

# Nama File Backup
BACKUP_FILE="mysql_backup_$(date +%Y%m%d%H%M%S).sql"

# Eksekusi mysqldump
mysqldump -u root -h localhost -p1 --all-databases > /tmp/$BACKUP_FILE

# Compression (opsional)
gzip /tmp/$BACKUP_FILE

# Konfigurasi rsync pada server utama
SOURCE_DIR="/tmp"
DESTINATION_DIR="10.10.10.2::backup_module"

# Eksekusi rsync tanpa opsi --delete
rsync -avz $SOURCE_DIR/$BACKUP_FILE.gz $DESTINATION_DIR

# Hapus file dump sementara
rm /tmp/$BACKUP_FILE.gz
```

**Langkah 5: Membuat Script untuk sinkornisasi Direktori Konfigurasi di Server Utama dan Server Backup**
```
touch sync.sh
nano sync.sh
```
**Langkah 6: Isi Script Sync(10.10.10.1)**
```
#!/bin/bash

# Fungsi untuk menyinkronkan modul rsync
sync_module() {
    SOURCE_DIR=$1
    MODULE_NAME=$2

    # Backup direktori ke file tar di /tmp dengan nama yang mencakup nama modul
    BACKUP_FILE="/tmp/backup_$(date +%Y%m%d%H%M%S).tar.gz"
    tar -czf $BACKUP_FILE $SOURCE_DIR

    # Synchronize backup file ke modul rsync
    rsync -avz $BACKUP_FILE $MODULE_NAME


    rm $BACKUP_FILE
}


sync_module "/var/www/" "10.10.10.2::module_sync_var_www"


sync_module "/etc/postfix/" "10.10.10.2::module_sync_etc_postfix"


sync_module "/etc/apache2/" "10.10.10.2::module_sync_etc_apache2"


sync_module "/etc/openvpn/" "10.10.10.2::module_sync_etc_openvpn"


sync_module "/etc/grafana/" "10.10.10.2::module_sync_etc_grafana"


sync_module "/etc/loki/" "10.10.10.2::module_sync_etc_loki"


sync_module "/etc/promtail/" "10.10.10.2::module_sync_etc_promtail"


sync_module "/etc/dovecot/" "10.10.10.2::module_sync_etc_dovecot"


```

**Langkah 7: beri akses untuk eksekusi**
```
chmod +x backup.sh
chmod +x sync.sh
```
**Langkah 8: membuat File Module Backup untuk Menerima dan Mengarahkan File bakcup ke direktori yang sesuai di Server Backup(10.10.10.2)**
```
mkdir /home/mysql_backup
mkdir /home/backup
#ini isi subfolder dari /home/backup/
mkdir www_backup
mkdir postfix_backup
mkdir apache2_backup
mkdir openvpn_backup
mkdir grafana_backup
mkdir loki_backup
mkdir promtail_backup
mkdir dovecot_backup
```
**Langkah 9: Beri Hak akses ke Direktori yang akan menyimpan file Backup di Server Backup(10.10.10.2)**
```
chmod 1777 /home/mysql_backup
chmod -R 1777 /home/backup/
```
**Langkah 10: Edit Konfigurasi Utama Rsync(10.10.10.2)**
```
uid = nobody
gid = nogroup
use chroot = yes
max connections = 4
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock
log file = /var/log/rsyncd.log

[backup_module]
    path = /home/mysql_backup/
    comment = Rsync Backup
    read only = no
    list = yes
    hosts allow = 10.10.10.1

[module_sync_var_www]
    path = /home/backup/www_backup/
    comment = Rsync Sync /var/www
    read only = no
    list = yes
    hosts allow = 10.10.10.1

[module_sync_etc_postfix]
    path = /home/backup/postfix_backup/
    comment = Rsync Sync /etc/postfix
    read only = no
    list = yes
    hosts allow = 10.10.10.1

[module_sync_etc_apache2]
    path = /home/backup/apache2_backup/
    comment = Rsync Sync /etc/apache2
    read only = no
    list = yes
    hosts allow = 10.10.10.1

[module_sync_etc_openvpn]
    path = /home/backup/openvpn_backup/
    comment = Rsync Sync /etc/openvpn
    read only = no
    list = yes
    hosts allow = 10.10.10.1

[module_sync_etc_grafana]
    path = /home/backup/grafana_backup/
    comment = Rsync Sync /etc/grafana
    read only = no
    list = yes
    hosts allow = 10.10.10.1

[module_sync_etc_loki]
    path = /home/backup/loki_backup/
    comment = Rsync Sync /etc/loki
    read only = no
    list = yes
    hosts allow = 10.10.10.1


[module_sync_etc_promtail]
    path = /home/backup/promtail_backup/
    comment = Rsync Sync /etc/promtail
    read only = no
    list = yes
    hosts allow = 10.10.10.1

[module_sync_etc_dovecot]
    path = /home/backup/dovecot_backup/
    comment = Rsync Sync /etc/dovecot
    read only = no
    list = yes
    hosts allow = 10.10.10.1

```
**Langkah 11: Restart**
```
systemctl enable rsync
systemctl restart rsync
```

**Langkah 12: Script untuk file/log rotate(10.10.10.2)**
```
nano backup_rotate.sh
```
jadi saya akan membuat script yang bisa menghapus file backup yang sudah berumur lebih dari 7 hari(7+)

**Langkah 13: isi script**
```
#!/bin/bash

BACKUP_DIRS=(
    "/home/mysql_backup/"
    "/home/backup/www_backup/"
    "/home/backup/postfix_backup/"
    "/home/backup/apache2_backup/"
    "/home/backup/openvpn_backup/"
    "/home/backup/grafana_backup/"
    "/home/backup/loki_backup/"
    "/home/backup/promtail_backup/"
)

for dir in "${BACKUP_DIRS[@]}"; do
    find $dir -type f -mtime +7 -exec rm {} \;
done

systemctl restart rsync
```

### 4.2 Menjadwalkan Script Backup secara Rutin(Crontab)

**Langkah 1: Buka Direktori Utama Crontab(10.10.10.1)**
```
crontab -e
```
**Langkah 2: Jadwalkan Sesuai keinginan(10.10.10.1)**

Format penjadwalan cron adalah sebagai berikut:

Menit (0 - 59)

Jam (0 - 23)

Hari dalam bulan (1 - 31)

Bulan (1 - 12)

Hari dalam minggu (0 - 6, 0 adalah Minggu)


(disini jadwal nya 3 hari sekali setiap jam 2 pagi)
```
0 2 */3 * * /home/backup/backup.sh
0 2 */3 * * /home/backup/sync.sh
```

**Langkah 3: Verifikasi Job cron(10.10.10.1)**
```
crontab -l
```

**Langkah 4: direktori crontab di VM3(10.10.10.2)**
```
crobtab -e
```
**Langkah 5: edit direktori crontab(10.10.10.2)**
```
0 2 * * 0 /home/backup/backup_rotate.sh
```
jadi script akan terjadwalkan setiap seminggu sekali yang,akan menghapus file backup yang sudah berumur lebih dari 7+ hari

**Langkah 6: Apply script(10.10.10.2)**
```
crontab -l
```









