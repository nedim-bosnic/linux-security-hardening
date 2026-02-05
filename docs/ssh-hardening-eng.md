# SSH Hardening (EN)

> Goal: reduce SSH attack surface, restrict access, improve visibility (logging/audit), and add brute-force protections.

## Quick checklist
- [ ] Update OpenSSH and OS packages
- [ ] Disable password auth; use keys only
- [ ] Disable SSH root login
- [ ] Restrict users/groups (AllowUsers/AllowGroups)
- [ ] Use modern crypto settings (KEX/Ciphers/MACs)
- [ ] Rate-limit / protection (fail2ban, firewall, sshguard)
- [ ] Logging and auditing (journald/syslog, auditd)
- [ ] Test and keep a rollback plan before restarting

---

## 1) Prep

### 1.1 Check versions
```bash
ssh -V
sshd -T | head
```

### 1.2 Back up configs
```bash
sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F)
sudo cp -a /etc/ssh/ssh_config /etc/ssh/ssh_config.bak.$(date +%F) 2>/dev/null || true
```

### 1.3 Keep a second session
Keep one SSH session open while applying changes.

---

## 2) Keys instead of passwords

### 2.1 Generate a key on the client
```bash
ssh-keygen -t ed25519 -a 64 -C "nedim@client"
```

### 2.2 Copy key to the server
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
```

If `ssh-copy-id` is unavailable:
```bash
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

---

## 3) Server hardening (`/etc/ssh/sshd_config`)

> Note: some distros use include snippets (`/etc/ssh/sshd_config.d/*.conf`). Verify what is active.

### 3.1 Recommended baseline
Edit:
```bash
sudo nano /etc/ssh/sshd_config
```

```conf
PermitRootLogin no

PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no

UsePAM yes

AllowUsers user1 user2
# or:
# AllowGroups sshusers

X11Forwarding no

AllowTcpForwarding no
PermitTunnel no

PermitEmptyPasswords no
LoginGraceTime 30
MaxAuthTries 3
MaxSessions 2

LogLevel VERBOSE
```

### 3.2 Modern algorithms (example)
Validate availability with `sshd -T`:

```conf
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

### 3.3 Port changes
```conf
#Port 22
#Port 2222
```

### 3.4 Match blocks (example)
```conf
Match User deploy
    AllowTcpForwarding yes
    X11Forwarding no
    ForceCommand /usr/local/bin/deploy-shell
```

---

## 4) Validate before reload/restart

### 4.1 Syntax check
```bash
sudo sshd -t
```

### 4.2 Inspect effective config
```bash
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|allowusers|allowgroups|loglevel|maxauthtries|allowtcpforwarding|x11forwarding'
```

### 4.3 Reload / restart
Debian/Ubuntu:
```bash
sudo systemctl reload ssh
# or:
sudo systemctl restart ssh
```

RHEL/Fedora:
```bash
sudo systemctl reload sshd
# or:
sudo systemctl restart sshd
```

---

## 5) Firewall-based access control

### 5.1 Allow only trusted IPs (nftables example)
```bash
sudo nft add rule inet filter input ip saddr { 203.0.113.10, 198.51.100.20 } tcp dport 22 accept
```

### 5.2 UFW (Ubuntu)
```bash
sudo ufw allow from 203.0.113.10 to any port 22 proto tcp
sudo ufw enable
sudo ufw status verbose
```

---

## 6) Brute-force protection (fail2ban)

### 6.1 Install
```bash
sudo apt update
sudo apt install -y fail2ban
```

### 6.2 Minimal jail for sshd
```bash
sudo nano /etc/fail2ban/jail.d/sshd.local
```

```conf
[sshd]
enabled = true
maxretry = 4
findtime = 10m
bantime = 1h
```

```bash
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---

## 7) Logging and auditing

### 7.1 Journald / syslog
```bash
sudo journalctl -u ssh -n 200 --no-pager 2>/dev/null || true
sudo journalctl -u sshd -n 200 --no-pager 2>/dev/null || true
```

### 7.2 auditd (example)
```bash
sudo apt install -y auditd
echo "-w /etc/ssh/sshd_config -p wa -k sshd_config" | sudo tee /etc/audit/rules.d/sshd.rules
sudo augenrules --load
sudo ausearch -k sshd_config
```

---

## 8) Emergency rollback
```bash
sudo cp -a /etc/ssh/sshd_config.bak.* /etc/ssh/sshd_config
sudo sshd -t
sudo systemctl restart sshd 2>/dev/null || sudo systemctl restart ssh
```

---

## 9) Client-side testing
```bash
ssh -vvv user@server
ssh -p 2222 -vvv user@server
ssh -v -i ~/.ssh/id_ed25519 user@server
```

---

## 10) Notes
- Prefer `ed25519` keys.
- Disable passwords and use keys + (optionally) PAM-based MFA.
- Restrict by IP where practical.
- Monitor logs; use fail2ban/sshguard.
- Always run `sshd -t` before restarting SSH.
