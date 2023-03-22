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
## BIND9
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
## Apache
```
# Update Host and install apache2
sudo apt update -y
sudo apt install apache2 -y

# Go to the default directory
cd /etc/apache2/

# Creat new site configuration
sudo vim sites-available/www.smartlearn.dmz.conf
<VirtualHost *:80>
    ServerName          smartlearn.dmz
    ServerAlias         www.smartlearn.dmz
    ServerAdmin         root@root.com
    DocumentRoot        /var/www/smartlearn.dmz/

    ErrorLog            ${APACHE_LOG_DIR}/error.log
    CustomLog           ${APACHE_LOG_DIR}/access.log combined

    <Directory /var/www/smartlearn.dmz/>
        Require all granted
    </Directory>
</VirtualHost>

# Create website-file directory and change the owner/group to www-data
sudo mkdir /var/www/smartlearn.dmz
sudo chown www-data:www-data /var/www/smartlearn.dmz

# Create index.html file
sudo vim /var/www/smartlearn.dmz/index.html
<html>
    <head>
        <title>Welcome to smartlearn.dmz!</title>
    </head>
    <body>
        <h1>Success!  The smartlearn.dmz virtual host is working!</h1>
    </body>
</html>

# Change the owner of the index.html 
sudo chown www-data:www-data /var/www/smartlearn.dmz/index.html

# Enable virtualhostconfig and disable default virtualhostconfig 
sudo a2ensite www.smartlearn.dmz.conf 
sudo a2dissite www.smartlearn.dmz.conf 

# Reload sites and configs 
sudo systemctl reload apache2

# Add new dns record
sudo vim /etc/bind/db.smartlearn.dmz
...
www IN A 192.168.220.13

sudo vim /etc/bind/db.192.168.220
...
13 IN CNAME www.smartlearn.dmz

# Restart the dns to load the new configuration
sudo systemctl restart bind9

# Testing 
curl -v www.smartlearn.dmz
*   Trying 192.168.220.13:80...
* Connected to 192.168.220.13 (192.168.220.13) port 80 (#0)
> GET / HTTP/1.1
> Host: 192.168.220.13
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Wed, 22 Mar 2023 10:25:15 GMT
< Server: Apache/2.4.52 (Ubuntu)
< Last-Modified: Wed, 15 Mar 2023 10:08:10 GMT
< ETag: "b6-5f6ed8575632f"
< Accept-Ranges: bytes
< Content-Length: 182
< Vary: Accept-Encoding
< Content-Type: text/html
< 
<html>
    <head>
        <title>Welcome to smartlearn.dmz!</title>
    </head>
    <body>
        <h1>Success!  The smartlearn.dmz virtual host is working!</h1>
    </body>
</html>
* Connection #0 to host 192.168.220.13 left intact
```