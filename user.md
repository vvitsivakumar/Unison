# Unison
Unison is a bi-directional file synchronization tool that works across multiple operating systems, including Windows, macOS, and Unix-like systems such as Linux. Its primary purpose is to keep two replicas of files and directories synchronized across different hosts or storage devices, ensuring that both locations remain identical.

# 🖥️ High Availability File Sync with Unison

This document outlines how to set up **file synchronization** between two web servers (`worker1` and `worker2`) using **Unison** for a high-availability web hosting setup.

---

## 🌐 Server Details

| Role       | IP Address       | Directory to Sync     |
|------------|------------------|------------------------|
| worker1    | 192.168.1.10    | /var/www/faveo         |
| worker2    | 192.168.1.20     | /var/www/faveo         |

---

# 🚨  RUN ALL AS ROOT 

## ✅ Prerequisites (on both servers)

```bash
sudo apt update
sudo apt install unison openssh-server -y
```

Ensure the directory exists on both servers:

```bash
sudo mkdir -p /var/www/faveo
sudo chown -R www-data:www-data /var/www/faveo
sudo chmod -R 775 /var/www/faveo
```

Ensure same user/group (`www-data`) exists with same UID/GID.

---

## 🔐 Passwordless SSH Setup

Run this **on worker1** to allow SSH into worker2 without password:

```bash
ssh-keygen     # Press Enter for all prompts
ssh-copy-id worker2@192.168.1.20
```

Run this **on worker2** to allow SSH into worker1 without password:

```bash
ssh-keygen     # Press Enter for all prompts
ssh-copy-id worker1@192.168.1.10
```

Test from both directions:

```bash
ssh worker2@192.168.1.20
ssh worker1@192.168.1.10
```

---

## 📄 Create Unison Profile

On **worker1**, create profile:

```bash
mkdir -p ~/.unison
nano ~/.unison/faveo-sync.prf
```

Paste:

```
root = /var/www/faveo
root = ssh://worker2@192.168.1.20//var/www/faveo
auto = true
batch = true
prefer = newer
confirmbigdel = false
log = true
```

Do the same on **worker2**, change the second root to:

```
root = ssh://worker1@192.168.1.10//var/www/faveo
```

---

## ⏲️ Automate with Cron

Run `crontab -e` on **both servers** and add:

```
* * * * * /usr/bin/unison faveo-sync && chown -R www-data:www-data /var/www/html >> /var/log/unison.log 2>&1
```

This runs sync every minutes.

---

## 🔄 How This Works

- Syncs changes in `/var/www/faveo` between both nodes
- If `worker1` is down, `worker2` continues working and vice versa
- Sync resumes automatically when the downed node is back

---

## ⚠️ Notes

- Avoid simultaneous writes to the same file on both nodes (to prevent conflicts)
- Use load balancer with **sticky sessions** or direct writes to one node

---

## ✅ Optional: Dry Run

To test without making changes:

```bash
unison faveo-sync -dryrun
```

---

## 🔚 Summary

With Unison + Cron:
- You get **active-active sync** between two nodes
- Each node can recover if the other is offline
- Lightweight, CLI-friendly HA setup
