# M300 Documentation
## SSH
### Files
| Filename | Use |
| -------- | --- |
| id_rsa | Private Key | 
| id_rsa.pub | Public Key wich we will copy on the remote host |
| kown_hosts | Hosts wich You where connected to |
| authorized_keys | Public Keys from user which have been connecting to this host |
### Use
```
# Private and Public Key generation
ssh-keygen

# Copy the Public Key on the hosts
ssh-copy-id <user@host>

# Connect to the host
ssh <user@host>
```
## vmls3: BIND9
```
sudo apt install bind9 -y
```
```
# named.conf
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";

# named.conf.options
acl smartlearn-networks {
        192.168.210.0/24;
        192.168.220.0/24;
};

options {
        directory "/var/cache/bind";
        
        forwarders {
          8.8.8.8;
        };

        allow-recursion { smartlearn-networks; };
        allow-query { smartlearn-networks; };
        allow-query-cache { smartlearn-networks; };
        
        dnssec-validation auto;
        
        listen-on-v6 { any; };
};

# named.conf.local
zone "smartlearn.lan" {
    type master;
    file "/etc/bind/db.smartlearn.lan";
    allow-update {
        none;
    };
};

zone "smartlearn.dmz" {
    type master;
    file "/etc/bind/db.smartlearn.dmz";
    allow-update {
        none;
    };
};

zone "210.168.192.in-addr.arpa" IN {
    type master;
    file "/etc/bind/db.192.168.210";
    allow-update {
        none;
    };
};

zone "220.168.192.in-addr.arpa" IN {
    type master;
    file "/etc/bind/db.192.168.220";
    allow-update {
        none;
    };
};

# db.smartlearn.lan
;
; Zone smartlearn.lan
;
$TTL    600
@       IN       SOA     vmls3.smartlearn.dmz. root.smartlearn.ch. (
        2023030101       ;
        1H               ; 
        2H               ;
        1D               ;
        1H)              ;

@       IN       NS      vmls3.smartlearn.dmz.
vmlf1   IN       A       192.168.210.1
vmls4   IN       A       192.168.210.64
vmls5   IN       A       192.168.210.65
vmlp1   IN       A       192.168.210.31

# db.192.168.210
;
; Reverse Zone smartlearn.lan
;
$TTL    600
@       IN       SOA     vmls3.smartlearn.dmz. root.smartlearn.ch. (
        2023030101       ;
        1H               ; 
        2H               ;
        1D               ;
        1H)              ;

@       IN     NS    vmls3.smartlearn.lan.
1       IN     PTR   vmlf1.smartlearn.lan.
64      IN     PTR   vmls4.smartlearn.lan.
65      IN     PTR   vmls5.smartlearn.lan.
31      IN     PTR   vmlp1.smartlearn.lan.

# db.smartlearn.dmz
;
; Zone smartlearn.dmz
;
$TTL    600
@       IN       SOA     vmls3.smartlearn.dmz.   root.smartlearn.ch. (
        202303010        ;
        1H               ; 
        2H               ;
        1D               ;
        1H)              ;

@       IN       NS      vmls3.smartlearn.dmz.
vmlf1   IN       A       192.168.220.1
vmls3   IN       A       192.168.220.13

# db.192.168.220
;
; Reverse Zone smartlearn.dmz
;
$TTL    600
@       IN       SOA     vmls3.smartlearn.dmz. root.smartlearn.ch. (
        2023030101       ;
        1H               ; 
        2H               ;
        1D               ;
        1H)              ;

@       IN     NS    vmls3.smartlearn.dmz.
1       IN     PTR   vmlf1.smartlearn.dmz.
13      IN     PTR   vmls3.smartlearn.dmz.
```
```
# On Client & DNS-Server for search-domain
sudo resolvectl dns eth0 192.168.220.13
sudo resolvectl domain eth0 smartlearn.dmz smartlearn.lan
```
```
# Testing
journalctl -f -u named
named-checkconf <conffile>
named-checkzone <zone> <zonefile>
nslookup <fqdn/ip> <dnsserverip>
```
## vmls3 & vmls5: Disk partitionate & format & mount 
```
sudo apt update
sudo apt install lshw -y

# List all Disks and partitions
lsblk

# partitionate
sudo fdisk /dev/sdb

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): 

Using default response p.
Partition number (1-4, default 1): 
First sector (2048-2097151, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-2097151, default 2097151): 

Created a new partition 1 of type 'Linux' and of size 1023 MiB.

Command (m for help): w

# formate
sudo mkfs.ext4 /dev/sdb1

# Creat direcotry
sudo mkdir /www

# Edit file for auto-mount by restart
sudo vim /etc/fstab
/dev/sdb1 /www ext4 defaults 1 2 

# mount
sudo mount -a 
sudo mount
...
/dev/sdb1 on /www type ext3 (rw)
```
## vmls3 & vmls5: Apache
```
# Update Host and install apache2
sudo apt update -y
sudo apt install apache2 -y
sudo chown -R www-data:www-data /www
```
### vmls3: For www.smartlearn.dmz
```
# Creat new site configuration
sudo vim /etc/apache2/sites-available/www.smartlearn.dmz.conf
<VirtualHost *:80>
    ServerName          smartlearn.dmz
    ServerAlias         www.smartlearn.dmz
    ServerAdmin         webmaster@smartlearn.dmz
    DocumentRoot        /www/www.smartlearn.dmz
    <Directory /www/www.smartlearn.dmz>
        Require all granted
    </Directory>
    ErrorLog            ${APACHE_LOG_DIR}/error.log
    CustomLog           ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

# Enable virtualhostconfig and disable default virtualhostconfig 
sudo a2ensite www.smartlearn.dmz.conf 
sudo a2dissite 000-default.conf 

# Add sites-files to /www/
ll /www/
...
drwxrwxr-x  4 www-data www-data  4096 Mar 29 10:39 www.smartlearn.dmz/

# Reload sites and configs 
sudo systemctl reload apache2
```
### vmls5: For www.smartlearn.lan & ku1.smartlearn.lan & ku2.smartlearn.lan 
```
# Creat new site configuration
sudo vim /etc/apache2/sites-available/www.smartlearn.lan.conf
<VirtualHost *:80>
    ServerName          smartlearn.lan
    ServerAlias         www.smartlearn.lan
    ServerAdmin         webmaster@smartlearn.lan
    DocumentRoot        /www/smartlearn.lan
    <Directory /www/ku1.smartlearn.lan>
        Require all granted
    </Directory>
    ErrorLog            ${APACHE_LOG_DIR}/error.log
    CustomLog           ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

sudo vim /etc/apache2/sites-available/ku1.smartlearn.lan.conf
<VirtualHost *:80>
    ServerAdmin         webmaster@smartlearn.lan
    ServerName          ku1.smartlearn.lan
    ServerAlias         www.ku1.smartlearn.lan
    DocumentRoot        /www/ku1.smartlearn.lan
    <Directory /www/ku1.smartlearn.lan>
        Require all granted
    </Directory>
    ErrorLog            ${APACHE_LOG_DIR}/error.log
    CustomLog           ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

sudo vim /etc/apache2/sites-available/ku2.smartlearn.lan.conf
<VirtualHost *:80>
    ServerAdmin         webmaster@smartlearn.lan
    ServerName          ku2.smartlearn.lan
    ServerAlias         www.ku2.smartlearn.lan
    DocumentRoot        /www/ku2.smartlearn.lan
    <Directory /www/ku2.smartlearn.lan>
        Require all granted
    </Directory>
    ErrorLog            ${APACHE_LOG_DIR}/error.log
    CustomLog           ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

# Enable virtualhostconfig and disable default virtualhostconfig 
sudo a2ensite www.smartlearn.lan.conf 
sudo a2ensite ku1.smartlearn.lan.conf
sudo a2ensite ku2.smartlearn.lan.conf
sudo a2dissite 000-default.conf 

# Add sites-files to /www/
ll /www/
...
drwxrwxr-x  4 www-data www-data  4096 Mar 29 11:13 ku1.smartlearn.lan/
drwxrwxr-x  4 www-data www-data  4096 Mar 29 11:13 ku2.smartlearn.lan/
...
drwxrwxr-x  4 www-data www-data  4096 Mar 29 11:13 www.smartlearn.lan/


# Reload sites and configs 
sudo systemctl reload apache2
```
### vmls3: Add DNS Records
```
sudo vim /etc/bind/db.smartlearn.lan
...
www     IN       A       192.168.210.65
@       IN       A       192.168.210.65
ku1     IN       A       192.168.210.65
ku2     IN       A       192.168.210.65

sudo vim /etc/bind/db.smartlearn.dmz
...
www     IN       A       192.168.220.13
@       IN       A       192.168.220.13

# Restart the dns to load the new configuration
sudo systemctl restart bind9
```

### Testing 
curl -sI www.smartlearn.lan | head -1
HTTP/1.1 200 OK

curl -sI smartlearn.lan | head -1
HTTP/1.1 200 OK

curl -sI www.ku1.smartlearn.lan | head -1
HTTP/1.1 200 OK

curl -sI www.ku2.smartlearn.lan | head -1
HTTP/1.1 200 OK

curl -sI ku1.smartlearn.lan | head -1
HTTP/1.1 200 OK

curl -sI ku2.smartlearn.lan | head -1
HTTP/1.1 200 OK

curl -sI www.smartlearn.dmz | head -1
HTTP/1.1 200 OK

curl -sI smartlearn.dmz | head -1
HTTP/1.1 200 OK
```