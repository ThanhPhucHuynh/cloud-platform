# Prepare - update os

master - master.tpk8s.com - 4r/2core \
node   - node01.tpk8s.com - 4r/2core

|Node   |Domain             |                 |
|-------|-------------------|-----------------|
| master|master.tpk8s.com   | ram 4 - 2 core  |
| node  | node01.tpk8s.com  | ram 2 - 2 core  |

```bash
sudo yum -y update && sudo systemctl reboots
```
