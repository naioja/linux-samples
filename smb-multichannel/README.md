Samba server side configuration for multichannel in `smb.conf`:

```
[global]
interfaces = eth1, eth2
server multi channel support = yes
aio read size = 1
aio write size = 1
```

Client side `mount` options, where N is the total number of channels the client will try to open:

```
vers=3.11,multichannel,max_channels=N
```
