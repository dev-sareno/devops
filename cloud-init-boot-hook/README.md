# Cloud-init
Cloud-init only runs script the first time the server started. We can change it to always run everytime the server rebooted using [cloud-boothook](https://cloudinit.readthedocs.io/en/latest/explanation/format.html)

## Linux/Ubuntu
```shell
#cloud-boothook
#!/bin/bash
DEBIAN_FRONTEND=noninteractive apt update -y
DEBIAN_FRONTEND=noninteractive apt install curl -y

echo "Hey! this runs every time the server rebooted"
```
