# Linux 挖矿木马清理方案（2023-10-01）

从你提供的脚本**完整特征**可以判定：这是**典型的Linux挖矿木马（kinsing/kdevtmpfsi家族）** 恶意脚本，具备**进程查杀自我保护、权限篡改、文件驻留、网络下载恶意程序、定时任务持久化** 等完整攻击能力。

我给你整理**分优先级、可直接复制执行**的完善处理方案，**严格按顺序操作**（先止损，再清除，最后加固）：

# 一、【紧急止损：立即执行！10分钟内完成】

⚠️ 核心目标：杀死恶意进程、阻断网络、撤销恶意权限，防止木马自我修复

## 1. 强制终止所有恶意进程（全覆盖脚本+木马进程）

```Bash

# 杀死脚本中指定的所有恶意进程 + 伪装进程
kill -9 $(pgrep -f 'kdevtmpfsi|pppsssdm|cf94c8fd2|kerneldx|pg_mem|postgres-system|postgres_dm|rumpostgreswk|kthreaddk|memory|kinsing|postgres-kernel|perfctl|httpd\ \(malicious\)') 2>/dev/null

# 杀死 /proc/exe 标记为 (deleted) 的僵尸恶意进程（木马核心隐藏进程）
for pid in /proc/[0-9]*; do
  pid=${pid##*/}
  result=$(ls -l /proc/$pid/exe 2>/dev/null)
  if echo "$result" | grep -q -F '(deleted)'; then
    [ $$ -ne $pid ] && kill -9 $pid 2>/dev/null
  fi
done

# 杀死父进程，防止进程复活
pkill -9 -f 'sh|bash' -v | grep -E 'promocioni|httpd|perfctl' | awk '{print $1}' | xargs kill -9 2>/dev/null
```

## 2. 阻断恶意网络通信

```Bash

# 封禁木马下载服务器IP（永久拉黑）
iptables -A INPUT -s 200.4.115.1 -j DROP
iptables -A OUTPUT -d 200.4.115.1 -j DROP
# 关闭木马通信端口
iptables -A INPUT -p tcp --dport 44870 -j DROP
iptables -A INPUT -p tcp --dport 63582 -j DROP
# 保存防火墙规则
iptables-save > /etc/iptables.rules
```

## 3. 撤销木马篡改的 /tmp 执行权限（木马核心运行目录）

脚本主动执行了 `mount -o remount,exec /tmp` 开启执行权限，**立即关闭**：

```Bash

mount -o remount,noexec,nosuid /tmp
mount -o remount,noexec,nosuid /var/tmp
mount -o remount,noexec,nosuid /dev/shm
```

## 4. 锁定恶意文件，防止木马再生

```Bash

# 锁定木马目录，禁止写入/删除
chattr +i /tmp/.perf.c /tmp/.xdiag 2>/dev/null
```

---

# 二、【彻底清除：删除所有恶意残留】

⚠️ 核心目标：删除木马文件、定时任务、系统劫持，根除驻留

## 1. 删除所有恶意文件/目录（脚本明确指定）

```Bash

# 强制删除木马核心文件
rm -rf /tmp/.perf.c /tmp/.xdiag /tmp/httpd /tmp/.install.pid* /tmp/init /tmp/mysql
# 删除内存盘木马文件（重启消失，主动清理）
rm -rf /dev/shm/[a-zA-Z]* 2>/dev/null
# 删除木马临时文件
rm -rf /tmp/vei* /tmp/ver 2>/dev/null
```

## 2. 清除定时任务（挖矿木马**最核心持久化方式**）

脚本篡改了 `crontab` 权限，必须彻底清理：

```Bash

# 清空所有用户定时任务
crontab -r 2>/dev/null
for user in $(cut -f1 -d: /etc/passwd); do crontab -r -u $user 2>/dev/null; done

# 删除系统级定时任务
rm -rf /var/spool/cron/* /etc/cron.d/* /etc/cron.hourly/* /etc/cron.daily/* /etc/cron.weekly/* /etc/cron.monthly/*
# 修复定时任务权限
chmod 600 /etc/crontab
chown root:root /etc/crontab
chattr +i /etc/crontab
```

## 3. 修复动态库劫持（高危！脚本操作了 `/etc/ld.so.preload`）

```Bash

# 清空恶意预加载文件（木马常用劫持方式）
> /etc/ld.so.preload
chattr +i /etc/ld.so.preload
```

## 4. 清除系统恶意环境变量

```Bash

unset AAZHDE VEI
echo > /etc/profile
echo > /root/.bashrc
```

---

# 三、【深度查杀：根除Rootkit/挖矿木马】

## 1. 安装专业查杀工具

```Bash

# CentOS/RHEL
yum install -y chkrootkit rkhunter

# Ubuntu/Debian
apt install -y chkrootkit rkhunter
```

## 2. 全盘扫描

```Bash

# 扫描rootkit（系统级后门）
chkrootkit
rkhunter --check

# 全盘搜索木马特征文件（MD5/大小匹配）
find / -type f -size 10999887c -exec md5sum {} \; | grep "06f5325811050e0e76ca40be71d045ca"
```

---

# 四、【永久加固：防止二次入侵】

## 1. 永久禁用高危目录执行权限

```Bash

# 写入fstab，重启后依然生效
echo 'tmpfs /tmp tmpfs defaults,noexec,nosuid 0 0' >> /etc/fstab
echo 'tmpfs /var/tmp tmpfs defaults,noexec,nosuid 0 0' >> /etc/fstab
echo 'tmpfs /dev/shm tmpfs defaults,noexec,nosuid 0 0' >> /etc/fstab
```

## 2. 系统权限加固

```Bash

# 锁定关键系统文件
chattr +i /etc/passwd /etc/shadow /etc/group /etc/gshadow
# 修复/tmp权限
chmod 1777 /tmp
```

## 3. 账号/SSH安全加固

```Bash

# 重置root密码（强制复杂密码）
passwd root

# 删除陌生SSH公钥（防止免密登录）
> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys

# 禁止root远程SSH登录（可选，推荐）
sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
systemctl restart sshd
```

## 4. 关闭不必要服务

kinsing 木马常利用 **Redis未授权、Docker未授权、弱密码** 入侵：

```Bash

# 如未使用Redis/Docker，直接关闭
systemctl stop redis docker
systemctl disable redis docker
```

---

# 五、【溯源：找到入侵入口】

## 1. 查看恶意进程日志

```Bash

# 查看当前网络连接（确认无恶意外联）
netstat -antp | grep -E '44870|63582|200.4.115.1'

# 查看系统登录日志
last
lastb

# 查看系统日志
grep -i 'attack\|malware\|ssh' /var/log/secure
```

## 2. 常见入侵入口

1. Redis未授权访问（最常见）

2. Docker API未授权访问

3. SSH弱密码爆破

4. Web应用漏洞（上传漏洞、命令执行）

---

# 六、【最终验证：确认清理完成】

执行以下命令，**无任何输出** 即为清理干净：

```Bash

# 检查恶意进程
pgrep -f 'kdevtmpfsi|kinsing|perfctl|httpd\(malicious\)'
# 检查恶意文件
ls /tmp/.perf.c /tmp/.xdiag /tmp/httpd
# 检查网络连接
netstat -antp | grep 200.4.115.1
# 检查定时任务
crontab -l
```

---

# 重要警告

1. **不要直接重启服务器**！重启前必须完成以上所有清理，否则木马会自启；

2. 若服务器有业务运行，**先备份数据**，再执行清理；

3. 清理完成后，**24小时内持续监控** 进程/CPU/网络，防止木马残留。
> （注：文档部分内容可能由 AI 生成）