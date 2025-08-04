# Unison

Unison is a bi-directional file synchronization tool that works across multiple operating systems, including Windows, macOS, and Unix-like systems such as Linux. Its primary purpose is to keep two replicas of files and directories synchronized across different hosts or storage devices, ensuring that both locations remain identical.

# 🖥️ High Availability File Sync with Unison on Red Hat / RHEL-Based Systems

## Unison File Sync using `apache` User (Two-Way Sync)

This guide sets up Unison for syncing `/var/www/html` between two RHEL-based Linux servers (like CentOS, AlmaLinux, Rocky Linux) using the `apache` user to ensure proper file ownership.

# ⚠️ RUN ALL AS ROOT

---

## ✅ Requirements

* Two RHEL-based servers (e.g., `192.168.1.10` and `192.168.1.20`)
* Root access on both servers
* SSH access between them

---

## 🔧 Step 1: Install Unison and SSH

Run on **both servers**:

```
yum install epel-release -y
yum install unison openssh-server -y
```

Enable and start SSH:

```
systemctl enable sshd
systemctl start sshd
```

---

## 📁 Step 2: Create and Set Up Target Directory

Run on **both servers**:

```
mkdir -p /var/www/html
chown -R apache:apache /var/www/html
chmod -R 775 /var/www/html
```

If `apache` does not exist, create it:

```
groupadd apache
useradd -g apache -s /bin/bash -m apache
```

---

## 🔐 Step 3: Set Password and SSH Access for `apache`

### Configure Home Directory:

```
usermod -d /home/apache apache
mkdir -p /home/apache
chown -R apache:apache /home/apache
```

### Set password:

```
passwd apache
```

### Generate SSH key on the Primary Server:

```
sudo -u apache mkdir -p /home/apache/.ssh
sudo -u apache chmod 700 /home/apache/.ssh
sudo -u apache ssh-keygen -t rsa -b 2048
```

Copy key to the secondary server:

```
sudo -u apache ssh-copy-id apache@192.168.1.20
```

Test SSH:

```
sudo -u apache ssh apache@192.168.1.20
```

### Repeat on the Secondary Server:

Same steps as above, replacing IP with `192.168.1.10`

---

## 🔁 Step 4: Create Unison Profile

### On Primary Server (192.168.1.10):

```
mkdir -p /home/apache/.unison
touch /home/apache/.unison/faveo-sync.prf
chown -R apache:apache /home/apache/.unison
chmod 700 /home/apache/.unison
chmod 600 /home/apache/.unison/faveo-sync.prf
nano /home/apache/.unison/faveo-sync.prf
```

Content:

```ini
root = /var/www/html
root = ssh://apache@192.168.1.20//var/www/html
auto = true
batch = true
prefer = newer
confirmbigdel = false
log = true
```

### On Secondary Server (192.168.1.20):

Update profile with:

```ini
root = /var/www/html
root = ssh://apache@192.168.1.10//var/www/html
auto = true
batch = true
prefer = newer
confirmbigdel = false
log = true
```

---

## ⏱️ Step 5: Automate Sync with Cron

Enable cron service:

```
systemctl enable crond
systemctl start crond
```

Prepare log file:

```
touch /var/log/unison.log
chown apache:apache /var/log/unison.log
chmod 664 /var/log/unison.log
```

Edit cron for apache:

```
crontab -u apache -e
```

Add:

```
* * * * * /usr/bin/unison faveo-sync >> /var/log/unison.log 2>&1
```

Or with timestamp:

```
* * * * * echo "$(date) - Running Unison Sync" >> /var/log/unison.log && /usr/bin/unison faveo-sync >> /var/log/unison.log 2>&1
```

---

## 🔎 Verify Cron

```
crontab -u apache -l
grep CRON /var/log/cron | grep apache
tail -f /var/log/unison.log
```

---

## 🪑 Optional: Monthly Log Cleanup

Edit root crontab:

```
crontab -e
```

Add:

```
0 0 1 * * echo -n "" > /var/log/unison.log && chown apache:apache /var/log/unison.log && chmod 664 /var/log/unison.log
```

---

## ✅ Summary

* Apache/Nginx friendly sync
* Files stay owned by `apache`
* Fully automated every minute
* Works securely over SSH
* Ideal for Red Hat / RHEL-like systems
