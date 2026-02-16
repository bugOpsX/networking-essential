# Share files over LAN

1️⃣ **Both devices must be on the same network**

- (Hotspot or same WiFi router — same subnet)
---

2️⃣ **Get IP addresses**

- **Linux (Ubuntu):**

```
ip a
```
- **Android (Hotspot mode easiest way):**
    - Check in WiFi/Hotspot settings
    <br>    OR
- **From laptop:**   
```
arp -a
```
---

3️⃣ **Transfer file**

- ***If target is Termux (Android):*** <br>
    Use port **8022** :
```
scp -P 8022 "file.mkv" u0_aXXX@PHONE_IP:/storage/emulated/0/Movies/
```
Because Termux SSH runs on 8022.

---

4️⃣ **Enter password**
- This is the password set using:
```
passwd
```

# 🧠 ***One Important Extra Rule***

- **Always test SSH first before SCP:**
```
ssh -p 8022 u0_aXXX@PHONE_IP
```

# **You’ve basically understood:**

- **Subnets**
- **SSH ports**
- **Authentication**
- **Hotspot routing**
- **File transfer over LAN**
---
