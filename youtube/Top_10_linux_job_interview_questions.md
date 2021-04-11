## [Top 10 linux interview questions](https://www.youtube.com/watch?v=l0QGLMwR-lY)

### How to check kernel version of current system

uname -a (or -v or -r)

### How to check current IP

#### Old way

ifconfig

- eth0 is ethernet
- inet is ipv4
- inet 6 is ipv6

#### New way

ip addr show eth0

### How to check for free disk space 
**Disk free**

df -ah 

sda1 is root partition. 

See Avail and Use%

### How to check if a service is running

#### Old OS
service [service name] status 

service [service name] start to start 

#### New OS 
systemctl status [service name] 

### How to check total size of folders' contents
**Disk use**

du -sh [dir path]

### How to check for open ports on machine 
sudo netstat -tulpn 

sudo is necessary to see PID/Program name column

### Check CPU usage of a process 

ps aux | grep [process name]

or 

top or htop (not installed by default)

### How to mount a new volume 

To for instance mount the dev/sda2 file system to the /mnt directory:

Go to /mnt dir in root 

mount /dev/sda2 /mnt 

mount command with no args shows existing mounts. 

### How to look up something you don't know

man [command]

info 

### How to find answers on things you can't find in man pages
serverfault

stackoverflow
