# Linux Command Line Quick Reference Guide

## System Info
```bash
uname -a
hostname
uptime
whoami
```

## File & Directory Management
```bash
ls
ls -al
pwd
cd <dir>
mkdir <dir>
rm file
rm -rf <dir>
cp file1 file2
mv file1 file2
```

## File Viewing
```bash
cat file
tac file
less file
more file
head file
tail file
tail -f file
```

## Search
```bash
grep "text" file
grep -r "text" .
find . -name "*.log"
```

## Disk & Memory
```bash
df -h
du -sh *
free -h
top
htop
```

## Networking
```bash
ifconfig
ip addr show
ping google.com
curl https://example.com
wget https://example.com
netstat -tulpn
ss -tulpn
```

## Permissions
```bash
chmod 755 file
chmod +x script.sh
chown user:user file
```

## Processes
```bash
ps aux
ps aux | grep <name>
kill <pid>
kill -9 <pid>
```

## Packages
### Debian/Ubuntu
```bash
sudo apt update
sudo apt install <pkg>
sudo apt remove <pkg>
```

### CentOS/RHEL
```bash
sudo yum install <pkg>
sudo yum remove <pkg>
```

## Archives
```bash
tar -cvf archive.tar folder/
tar -xvf archive.tar
zip -r file.zip folder/
unzip file.zip
```

## SSH & File Transfer
```bash
ssh user@host
scp file user@host:/path
rsync -avh source/ dest/
```

## Systemd Services
```bash
sudo systemctl start service
sudo systemctl stop service
sudo systemctl restart service
sudo systemctl status service
```

## Docker (bonus)
```bash
docker ps
docker logs <container>
docker exec -it <container> bash
```

## Useful Shortcuts
```bash
Ctrl + C   # stop process
Ctrl + Z   # background
Ctrl + D   # logout/EOF
Ctrl + R   # search history
```
