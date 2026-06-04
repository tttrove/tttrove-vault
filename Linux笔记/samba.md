apt update && apt install samba


```conf
[dev]
   comment = Development shared folder
   path = /root
   browseable = yes
   writable = yes
   read only = no
   force user = root
   force group = root
   create mask = 0644
   directory mask = 0755
   guest ok = no
   
```