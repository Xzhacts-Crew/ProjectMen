# ProjectMen


ANGGOTA KELOMPOK:

MUHAMMAD NUR HAQIQI - 22.83.0886 TK-02(Ketua) - System security Administrator(Fenrir717)

Georel Jeferson Fransiskus Bonai - 22.83.0833(Anggota 1) - Mail server administrator(Quetzalcoatlos23)

Muhammad Akbar Firmansyah - 22.83.0870(Anggota 2) -  CMS wordpress configurator

Muhammad Fadhil Mundzir Sakaria - 22.83.0829(Anggota 3) - Database administrator(A4VA4TA4R)

Willy Agustianus - 22.83.0882(Anggota 4) - Roundcube webmail configurator

Janssensius Rifaldo Lhezu  -  22.83.0861(Angoota 5) - web server administrator


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
 [VM1](#1-Konfigurasi-Honeypot-pada-VM1)
 [VM2](#2-Konfigurasi-Server-pada-VM2)
 [VM3](#VM3)

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

### Apa saja yang diKonfiguras? ###
**1. Web Server**

**2. Mail Server**

**3. Database Server**

**4. VPN Server**

**5. DNS Server**

**6. Port Knocking**

**7. Monitoring Server**

### A. Web Server

### B. Instalasi dan Konfigurasi Apache2

### C. Konfigurasi CMS Wordpress pada Apache2

### D. Konfigurasi WAF(Web Application Firewall)

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


