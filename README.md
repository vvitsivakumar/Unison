# Unison
Unison is a bi-directional file synchronization tool that works across multiple operating systems, including Windows, macOS, and Unix-like systems such as Linux. Its primary purpose is to keep two replicas of files and directories synchronized across different hosts or storage devices, ensuring that both locations remain identical.

# 🖥️ High Availability File Sync with Unison

## Unison File Sync using `www-data` User (Two-Way Sync)

This guide sets up Unison for syncing `/var/www/html` between two Linux servers using the `www-data` user to ensure proper file ownership.

# 🚨 RUN ALL AS ROOT
---

## ✅ Requirements

* Two Linux servers (e.g., `192.168.1.10` and `192.168.1.20`)
* Root access on both servers
* SSH access between them

---

## 🔧 Step 1: Install Unison and SSH

Run on **both servers**:

```
apt update
apt install unison openssh-server -y
```

---

## 📁 Step 2: Create and Set Up Target Directory

Run on **both servers**:

```
mkdir -p /var/www/html
chown -R www-data:www-data /var/www/html
chmod -R 775 /var/www/html
```

---

## 🔐 Step 3: Set Password and SSH Access for `www-data`

### Create SSH directory on **both servers**:

```
sudo systemctl stop apache2  # or nginx
sudo usermod -d /home/www-data www-data
sudo chsh -s /bin/bash www-data
sudo mkdir -p /home/www-data
sudo chown -R www-data:www-data /home/www-data
sudo systemctl start apache2  # or nginx
```

If the output like 
```
usermod: user www-data is currently used by process 1573
```

```
kill -9 1573
```
Again Create SSH directory on **both servers** 

### Set password on **both servers**:

```
chsh -s /bin/bash www-data
passwd www-data
```

Verify user shell:

```
getent passwd www-data
```

Expected output:

```
www-data:x:33:33:www-data:/home/www-data:/bin/bash
```

### Generate SSH key on the Primary Server:

```
sudo -u www-data mkdir -p /home/www-data/.ssh
sudo -u www-data chmod 700 /home/www-data/.ssh
sudo -u www-data ssh-keygen -t rsa -b 2048
```

Just press `Enter` to accept the default file location.

Copy key to the secondary server:

```
sudo -u www-data ssh-copy-id www-data@192.168.1.20
```

Test SSH (no password prompt):

```
sudo -u www-data ssh www-data@192.168.1.20
```

### Generate SSH key on the Secondary Server:

```
sudo -u www-data mkdir -p /home/www-data/.ssh
sudo -u www-data chmod 700 /home/www-data/.ssh
sudo -u www-data ssh-keygen -t rsa -b 2048
```

Just press `Enter` to accept the default file location.

Copy key to the primary server:

```
sudo -u www-data ssh-copy-id www-data@192.168.1.10
```

Test SSH:

```
sudo -u www-data ssh www-data@192.168.1.10
```

## 🔁 Step 4: Create Unison Profile

### On Primary Server (192.168.1.10):

```
mkdir -p /home/www-data/.unison
touch /home/www-data/.unison/faveo-sync.prf
chown -R www-data:www-data /home/www-data/.unison
chown www-data:www-data /home/www-data/.unison/faveo-sync.prf
chmod 700 /home/www-data/.unison
chmod 600 /home/www-data/.unison/faveo-sync.prf
nano /home/www-data/.unison/faveo-sync.prf
```

Paste the following content:

```ini
root = /var/www/html
root = ssh://www-data@192.168.1.20//var/www/html
auto = true
batch = true
prefer = newer
confirmbigdel = false
log = true
```
Replace 192.168.1.20 to Secondary Sever IP

Verify ownership:

```
ls -l /home/www-data/.unison/faveo-sync.prf
```

Expected output:

```
-rw------- 1 www-data www-data 147 Aug  3 08:05 /home/www-data/.unison/faveo-sync.prf
```

---

### On Secondary Server (192.168.1.20):

```
mkdir -p /home/www-data/.unison
touch /home/www-data/.unison/faveo-sync.prf
chown -R www-data:www-data /home/www-data/.unison
chown www-data:www-data /home/www-data/.unison/faveo-sync.prf
chmod 700 /home/www-data/.unison
chmod 600 /home/www-data/.unison/faveo-sync.prf
nano /home/www-data/.unison/faveo-sync.prf
```

Paste the following content:

```ini
root = /var/www/html
root = ssh://www-data@192.168.1.10//var/www/html
auto = true
batch = true
prefer = newer
confirmbigdel = false
log = true
```

Replace 192.168.1.10 to Primary Sever IP

Verify ownership:

```
ls -l /home/www-data/.unison/faveo-sync.prf
```

Expected output:

```
-rw------- 1 www-data www-data 147 Aug  3 08:05 /home/www-data/.unison/faveo-sync.prf
```

---

## ⏱️ Step 5: Automate Sync with Cron

On **both servers**, edit crontab for `www-data`:

```
touch /var/log/unison.log
chown www-data:www-data /var/log/unison.log
chmod 664 /var/log/unison.log
crontab -u www-data -e
```

### Add this line to run sync every minute:

```
* * * * * /usr/bin/unison faveo-sync >> /var/log/unison.log 2>&1
```

### Or with timestamped log entries:

```
* * * * * echo "$(date) - Running Unison Sync" >> /var/log/unison.log && /usr/bin/unison faveo-sync >> /var/log/unison.log 2>&1
```

---

## 🔎 Verify Cron

List current cron jobs for `www-data`:

```
crontab -u www-data -l
```

---

## 🧹 Optional: Monthly Log Cleanup

Edit root's crontab:

```
crontab -e
```

Add this to clean the log on the 1st of each month:

```
0 0 1 * * > /var/log/unison.log
```

Or with permission reset:

```
0 0 1 * * echo -n "" > /var/log/unison.log && chown www-data:www-data /var/log/unison.log && chmod 664 /var/log/unison.log
```

---

## 🛠️ Final Checklist and Troubleshooting Commands

Check if Unison Cron Job is Running

```
grep CRON /var/log/syslog | grep www-data
```
or
```
grep --text CRON /var/log/syslog | grep www-data
```

Monitor the Unison Log Live
```
tail -f /var/log/unison.log
```

Test Unison Sync Manually

```
sudo -u www-data unison faveo-sync
```

See If Cron Daemon is Active
```
systemctl status cron
```

Restart Cron if Needed
```
sudo systemctl restart cron
```

Confirm Cron Entries for www-data
```
crontab -u www-data -l
```

## ✅ Summary

* Automatic file sync every minute
* Files retain `www-data:www-data` ownership
* No manual `chown` needed
* Ideal for web servers like Apache/Nginx

---
