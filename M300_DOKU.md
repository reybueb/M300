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
; Zone fuer smartlearn.lan
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
; Reverse Zone fuer smartlearn.dmz
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
; Zone fuer smartlearn.dmz
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
; Reverse Zone fuer smartlearn.dmz
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
# Auf Client & Server fuer search-domain eintragen
sudo resolvectl dns eth0 192.168.220.13
sudo resolvectl domain eth0 smartlearn.dmz smartlearn.lan
```
```
# Testen
journalctl -f -u named
named-checkzone <zone> <zonefile>
nslookup <fqdn/ip> <dnsserverip>
```