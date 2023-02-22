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

```