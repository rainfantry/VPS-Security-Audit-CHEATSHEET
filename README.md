# VPS-Security-Audit-CHEATSHEET
Linux VPS/Terminal commands to enumerate system security 

# VPS Security Audit Cheatsheet

One-liner commands for quick security assessment. Copy-paste friendly.

---

## 🔍 SYSTEM INFO

```bash
# OS and kernel
uname -a && cat /etc/os-release | head -5

# Uptime and load
uptime

# Users with login shells
grep -E '/bin/(ba)?sh' /etc/passwd

# Who's logged in now
w

# Last 10 logins
last -10

# Failed login attempts
grep -i "failed\|invalid" /var/log/auth.log | tail -30
```

---

## 🔥 FIREWALL

```bash
# UFW status
ufw status verbose

# IPtables rules
iptables -L -n --line-numbers

# UFW numbered (for deletion)
ufw status numbered
```

---

## 🌐 NETWORK / OPEN PORTS

```bash
# Listening ports with process names
ss -tlnp

# Listening + established
ss -tunap

# What's bound to 0.0.0.0 (internet exposed)
ss -tlnp | grep "0.0.0.0"

# Active outbound connections
ss -tnp | grep ESTAB

# Check specific port
lsof -i :22
```

---

## 🔐 SSH SECURITY

```bash
# SSH config highlights
grep -E '^(Port|PermitRootLogin|PasswordAuthentication|PubkeyAuthentication|PermitEmptyPasswords|AllowUsers|AllowGroups)' /etc/ssh/sshd_config

# SSH authorized keys (who can login)
cat ~/.ssh/authorized_keys

# All users' authorized keys
find /home -name authorized_keys -exec echo "=== {} ===" \; -exec cat {} \; 2>/dev/null
```

---

## 🛡️ FAIL2BAN

```bash
# Status
systemctl status fail2ban

# All jails
fail2ban-client status

# Specific jail (sshd)
fail2ban-client status sshd

# Banned IPs across all jails
fail2ban-client banned

# Unban an IP
fail2ban-client set sshd unbanip 1.2.3.4
```

---

## 👤 USERS & PRIVILEGES

```bash
# Users with UID 0 (root equivalent)
awk -F: '($3 == 0) {print}' /etc/passwd

# Users in sudo group
getent group sudo

# Sudoers file
cat /etc/sudoers | grep -v "^#" | grep -v "^$"

# Recently modified users
ls -lt /home

# Password status for all users
cat /etc/shadow | cut -d: -f1,2 | grep -v ":\*\|:!"
```

---

## 📁 SENSITIVE FILES

```bash
# Check permissions on critical files
ls -la /etc/shadow /etc/passwd /etc/sudoers

# World-writable files in /etc
find /etc -type f -perm -002 2>/dev/null

# World-writable directories
find / -type d -perm -002 2>/dev/null 2>&1 | grep -v proc

# SUID binaries (potential privesc)
find / -perm -4000 2>/dev/null

# SGID binaries
find / -perm -2000 2>/dev/null

# Files modified in last 24h
find /etc -mtime -1 2>/dev/null

# Large files in /tmp (suspicious)
find /tmp -size +10M 2>/dev/null
```

---

## 🐳 DOCKER

```bash
# Running containers
docker ps

# All containers
docker ps -a

# Docker images
docker images

# Exposed ports from containers
docker ps --format "{{.Names}}: {{.Ports}}"
```

---

## ⚙️ SERVICES & PROCESSES

```bash
# Running services
systemctl list-units --type=service --state=running

# Enabled services (start on boot)
systemctl list-unit-files --type=service --state=enabled

# All processes (tree view)
ps auxf

# Processes by memory
ps aux --sort=-%mem | head -15

# Processes by CPU
ps aux --sort=-%cpu | head -15

# Listening services by user
ss -tlnp | awk '{print $7}' | sort -u
```

---

## 📧 MAIL / SMTP

```bash
# Postfix relay settings
grep -E '^(mynetworks|relay_domains|smtpd_recipient_restrictions)' /etc/postfix/main.cf

# Mail queue
mailq

# Is port 25 exposed?
ss -tlnp | grep ":25"
```

---

## 🗄️ DATABASE

```bash
# PostgreSQL auth config
cat /etc/postgresql/*/main/pg_hba.conf | grep -v "^#" | grep -v "^$"

# MySQL users (if accessible)
mysql -e "SELECT User,Host FROM mysql.user;"

# Is DB exposed externally?
ss -tlnp | grep -E ":(5432|3306|27017)"
```

---

## 📜 LOGS

```bash
# Auth log (logins, sudo, su)
tail -100 /var/log/auth.log

# Failed logins
grep "Failed" /var/log/auth.log | tail -30

# Successful logins
grep "Accepted" /var/log/auth.log | tail -30

# Syslog
tail -100 /var/log/syslog

# Kernel messages
dmesg | tail -50

# Nginx access (last 20)
tail -20 /var/log/nginx/access.log

# Nginx errors
tail -20 /var/log/nginx/error.log
```

---

## ⏰ CRON JOBS

```bash
# Root crontab
crontab -l

# System cron
cat /etc/crontab

# Cron directories
ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/

# All user crontabs
for user in $(cut -f1 -d: /etc/passwd); do echo "=== $user ===" && crontab -u $user -l 2>/dev/null; done
```

---

## 🔒 KERNEL SECURITY

```bash
# ASLR (should be 2)
sysctl kernel.randomize_va_space

# SYN cookies (should be 1)
sysctl net.ipv4.tcp_syncookies

# IP forwarding (0 unless router)
sysctl net.ipv4.ip_forward

# All security-related sysctls
sysctl -a 2>/dev/null | grep -E "(syn|forward|accept|icmp|martian)"
```

---

## 📦 PACKAGES & UPDATES

```bash
# Last apt update
stat /var/cache/apt/pkgcache.bin | grep Modify

# Unattended upgrades status
systemctl status unattended-upgrades

# Pending updates
apt list --upgradable 2>/dev/null

# Installed packages count
dpkg -l | wc -l
```

---

## 🚨 QUICK RED FLAGS

```bash
# Suspicious processes (miners, shells)
ps aux | grep -iE "(kworker|xmrig|minerd|kdevtmpfsi|nc -|/bin/sh -i|bash -i)"

# Weird outbound connections
ss -tnp | grep -vE "(443|80|22|53):" | grep ESTAB

# Recently modified binaries
find /usr/bin /usr/sbin -mtime -7 2>/dev/null

# Hidden files in /tmp
ls -la /tmp/.*  2>/dev/null

# Check for rootkits (if rkhunter installed)
rkhunter --check --skip-keypress

# Unusual listening ports
ss -tlnp | grep -vE ":(22|80|443|25|53|5432|3306)"
```

---

## 🔑 SSH KEYS & SECRETS

```bash
# Private keys (should not exist on server)
find / -name "id_rsa" -o -name "*.pem" 2>/dev/null

# AWS credentials
cat ~/.aws/credentials 2>/dev/null

# Environment variables with secrets
env | grep -iE "(key|secret|token|pass|api)"

# .env files
find /var/www /opt /home -name ".env" 2>/dev/null -exec cat {} \;

# Git credentials
find / -name ".git-credentials" 2>/dev/null

# Bash history (secrets typed)
cat ~/.bash_history | grep -iE "(pass|key|token|secret|curl.*auth)"
```

---

## 📊 FULL AUDIT ONE-LINER

```bash
# Run everything at once (copy entire block)
echo "=== SYSTEM ===" && uname -a && echo -e "\n=== USERS ===" && grep -E '/bin/(ba)?sh' /etc/passwd && echo -e "\n=== LOGGED IN ===" && w && echo -e "\n=== UFW ===" && ufw status verbose && echo -e "\n=== LISTENING ===" && ss -tlnp && echo -e "\n=== SSH CONFIG ===" && grep -E '^(Port|PermitRootLogin|PasswordAuthentication)' /etc/ssh/sshd_config && echo -e "\n=== FAIL2BAN ===" && fail2ban-client status 2>/dev/null && echo -e "\n=== WORLD-WRITABLE ===" && find /etc -type f -perm -002 2>/dev/null && echo -e "\n=== SUID ===" && find /usr -perm -4000 2>/dev/null && echo -e "\n=== FAILED LOGINS ===" && grep -i "failed" /var/log/auth.log 2>/dev/null | tail -10 && echo -e "\n=== OUTBOUND ===" && ss -tnp | grep ESTAB | head -10
```

---