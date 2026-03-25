# Linux 服务器恶意脚本攻击全面应对教程

# Complete Tutorial: Immediate & Thorough Response to Linux Server Malicious Script Attacks

This script is a **typical Linux mining malware (kinsing/kdevtmpfsi family)** with full malicious capabilities: self-protection via process killing, permission tampering, file persistence, malicious program downloading, and cron job persistence.

Below is a **priority-ordered, copy-pasteable English tutorial** for full remediation (stop threats first → eradicate malware → harden the system).

---

## 1. Emergency Containment (Execute Immediately! Finish in 10 Minutes)

**Goal**: Kill malicious processes, block network connections, revoke malicious permissions, and prevent malware self-repair.

### 1.1 Force Kill All Malicious Processes

```Bash

# Kill all malicious processes listed in the script + disguised processes
kill -9 $(pgrep -f 'kdevtmpfsi|pppsssdm|cf94c8fd2|kerneldx|pg_mem|postgres-system|postgres_dm|rumpostgreswk|kthreaddk|memory|kinsing|postgres-kernel|perfctl|httpd\ \(malicious\)') 2>/dev/null

# Kill zombie malicious processes marked (deleted) in /proc/exe (core hidden malware processes)
for pid in /proc/[0-9]*; do
  pid=${pid##*/}
  result=$(ls -l /proc/$pid/exe 2>/dev/null)
  if echo "$result" | grep -q -F '(deleted)'; then
    [ $$ -ne $pid ] && kill -9 $pid 2>/dev/null
  fi
done

# Kill parent processes to prevent malware resurrection
pkill -9 -f 'sh|bash' -v | grep -E 'promocioni|httpd|perfctl' | awk '{print $1}' | xargs kill -9 2>/dev/null
```

### 1.2 Block Malicious Network Traffic

```Bash

# Permanently blacklist the malware download server IP
iptables -A INPUT -s 200.4.115.1 -j DROP
iptables -A OUTPUT -d 200.4.115.1 -j DROP
# Block malware communication ports
iptables -A INPUT -p tcp --dport 44870 -j DROP
iptables -A INPUT -p tcp --dport 63582 -j DROP
# Save firewall rules
iptables-save > /etc/iptables.rules
```

### 1.3 Revoke Execute Permissions on /tmp (Malware Core Runtime Directory)

The script runs `mount -o remount,exec /tmp` to enable execution permissions—**disable them immediately**:

```Bash

mount -o remount,noexec,nosuid /tmp
mount -o remount,noexec,nosuid /var/tmp
mount -o remount,noexec,nosuid /dev/shm
```

### 1.4 Lock Malicious Directories to Prevent Regeneration

```Bash

# Lock malware directories to block writing/deletion
chattr +i /tmp/.perf.c /tmp/.xdiag 2>/dev/null
```

---

## 2. Complete Malware Removal (Eradicate All Residues)

**Goal**: Delete malware files, cron jobs, system hijacks, and eliminate persistence.

### 2.1 Delete All Malicious Files/Directories

```Bash

# Force delete core malware files
rm -rf /tmp/.perf.c /tmp/.xdiag /tmp/httpd /tmp/.install.pid* /tmp/init /tmp/mysql
# Delete malware files in RAM disk (temporary storage, clean proactively)
rm -rf /dev/shm/[a-zA-Z]* 2>/dev/null
# Delete malware temporary files
rm -rf /tmp/vei* /tmp/ver 2>/dev/null
```

### 2.2 Clean Up Malicious Cron Jobs (Critical Persistence Mechanism)

The script tampers with cron permissions—full cleanup required:

```Bash

# Clear all user cron jobs
crontab -r 2>/dev/null
for user in $(cut -f1 -d: /etc/passwd); do crontab -r -u $user 2>/dev/null; done

# Delete system-wide cron jobs
rm -rf /var/spool/cron/* /etc/cron.d/* /etc/cron.hourly/* /etc/cron.daily/* /etc/cron.weekly/* /etc/cron.monthly/*
# Restore cron file permissions
chmod 600 /etc/crontab
chown root:root /etc/crontab
chattr +i /etc/crontab
```

### 2.3 Fix Dynamic Library Hijacking (High Risk! The script targets /etc/[ld.so](ld.so).preload)

```Bash

# Empty malicious preload file (common malware hijacking method)
> /etc/ld.so.preload
chattr +i /etc/ld.so.preload
```

### 2.4 Clear Malicious System Environment Variables

```Bash

unset AAZHDE VEI
echo > /etc/profile
echo > /root/.bashrc
```

---

## 3. Deep Scan & Rootkit Eradication

**Goal**: Detect and remove system-level rootkits/backdoors.

### 3.1 Install Professional Scanning Tools

```Bash

# CentOS/RHEL
yum install -y chkrootkit rkhunter

# Ubuntu/Debian
apt install -y chkrootkit rkhunter
```

### 3.2 Full System Scan

```Bash

# Scan for rootkits (system-level backdoors)
chkrootkit
rkhunter --check

# Full-system search for malware files (matching MD5/size)
find / -type f -size 10999887c -exec md5sum {} \; | grep "06f5325811050e0e76ca40be71d045ca"
```

---

## 4. Permanent Hardening (Prevent Re-infection)

**Goal**: Block malware execution vectors and secure the system long-term.

### 4.1 Permanently Disable Execution Permissions on High-Risk Directories

```Bash

# Add to fstab to retain settings after reboot
echo 'tmpfs /tmp tmpfs defaults,noexec,nosuid 0 0' >> /etc/fstab
echo 'tmpfs /var/tmp tmpfs defaults,noexec,nosuid 0 0' >> /etc/fstab
echo 'tmpfs /dev/shm tmpfs defaults,noexec,nosuid 0 0' >> /etc/fstab
```

### 4.2 System File Permission Hardening

```Bash

# Lock critical system files
chattr +i /etc/passwd /etc/shadow /etc/group /etc/gshadow
# Restore /tmp permissions
chmod 1777 /tmp
```

### 4.3 Account & SSH Security Hardening

```Bash

# Reset root password (enforce strong password)
passwd root

# Delete unknown SSH public keys (block passwordless login)
> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys

# Disable remote root SSH login (recommended)
sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
systemctl restart sshd
```

### 4.4 Stop Unnecessary Services

Kinsing malware often invades via **unauthorized Redis/Docker access or weak passwords**:

```Bash

# Stop Redis/Docker if not in use
systemctl stop redis docker
systemctl disable redis docker
```

---

## 5. Threat Tracing (Find the Intrusion Vector)

### 5.1 Check Malicious Process & Network Logs

```Bash

# Verify no malicious outbound connections
netstat -antp | grep -E '44870|63582|200.4.115.1'

# Check system login logs
last
lastb

# Check system security logs
grep -i 'attack\|malware\|ssh' /var/log/secure
```

### 5.2 Common Intrusion Vectors

1. Unauthorized Redis access (most frequent)

2. Unauthorized Docker API access

3. SSH brute-force attacks with weak passwords

4. Web application vulnerabilities (file upload, remote code execution)

---

## 6. Final Verification (Confirm Full Cleanup)

**No output = successful cleanup** when running these commands:

```Bash

# Check for malicious processes
pgrep -f 'kdevtmpfsi|kinsing|perfctl|httpd\(malicious\)'
# Check for malicious files
ls /tmp/.perf.c /tmp/.xdiag /tmp/httpd
# Check for malicious network connections
netstat -antp | grep 200.4.115.1
# Check for malicious cron jobs
crontab -l
```

---

## Critical Warnings

1. **Do NOT reboot the server directly**! Complete all cleanup steps first, or the malware will restart automatically.

2. **Back up data first** if the server runs production services before executing removal commands.

3. **Monitor the server 24/7** after cleanup (processes, CPU, network) to catch any remaining malware residues.
> （注：文档部分内容可能由 AI 生成）