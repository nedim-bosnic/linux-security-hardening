# SSH hardening (BOS)

> Cilj: smanjiti površinu napada na SSH, ograničiti pristup, povećati vidljivost (logging/audit) i uvesti zaštite od brute-force pokušaja.

## Brzi checklist (sažeto)
- [ ] Ažuriraj OpenSSH i OS pakete
- [ ] Onemogući password login, koristi samo ključeve
- [ ] Zabraniti root login preko SSH
- [ ] Ograniči korisnike/grupe (AllowUsers/AllowGroups)
- [ ] Uključi moderne kriptografske postavke (KEX/Ciphers/MACs)
- [ ] Rate limit / zaštita (fail2ban, firewall, sshguard)
- [ ] Logging i audit (journald/syslog, auditd)
- [ ] Test i rollback plan prije restartovanja servisa

---

## 1) Preduslovi i priprema

### 1.1 Provjeri verzije
```bash
ssh -V
sshd -T | head
```

### 1.2 Backup konfiguracije
```bash
sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F)
sudo cp -a /etc/ssh/ssh_config /etc/ssh/ssh_config.bak.$(date +%F) 2>/dev/null || true
```

### 1.3 Obezbijedi drugi terminal
Prije izmjena, ostavi jednu aktivnu SSH sesiju otvorenu (da ne izgubiš pristup).

---

## 2) Ključevi umjesto lozinki

### 2.1 Kreiranje ključa na klijentu
```bash
ssh-keygen -t ed25519 -a 64 -C "nedim@client"
```

### 2.2 Kopiranje ključa na server
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
```

Ako nema `ssh-copy-id`:
```bash
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

---

## 3) Hardenovanje servera (`/etc/ssh/sshd_config`)

> Napomena: neke distribucije koriste include fajlove (`/etc/ssh/sshd_config.d/*.conf`). Provjeri šta je aktivno.

### 3.1 Osnovna pravila (preporučeno)
Uredi:
```bash
sudo nano /etc/ssh/sshd_config
```

Preporučene stavke (prilagodi korisniku/okruženju):

```conf
# Ne dozvoli root login
PermitRootLogin no

# Samo ključevi (bez lozinki)
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no

# Ako koristiš PAM za druge stvari, ostavi yes; ako ne treba, može no
UsePAM yes

# Ograniči koje korisnike puštaš
AllowUsers user1 user2
# ili:
# AllowGroups sshusers

# Ne koristi X11 forwarding (ako ne treba)
X11Forwarding no

# Ugasiti TCP forwarding ako ne treba (tuneli/proxy)
AllowTcpForwarding no
PermitTunnel no

# Smanji napadnu površinu
PermitEmptyPasswords no
LoginGraceTime 30
MaxAuthTries 3
MaxSessions 2

# Log detaljnije (ili INFO ako ti je previše)
LogLevel VERBOSE
```

### 3.2 Moderni algoritmi (primjer)
Ovo je primjer “modernog” seta; može varirati po verziji OpenSSH. Prije primjene provjeri podršku preko `sshd -T`.

```conf
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

### 3.3 Port 22 i “security by obscurity”
Promjena porta može smanjiti šum skenera, ali nije zamjena za hardening.
Ako mijenjaš port, otvori firewall i dokumentuj.

```conf
#Port 22
#Port 2222
```

### 3.4 Match blokovi (primjer)
Razdvoji politike po korisniku/grupi:

```conf
Match User deploy
    AllowTcpForwarding yes
    X11Forwarding no
    ForceCommand /usr/local/bin/deploy-shell
```

---

## 4) Validacija konfiguracije (obavezno prije restarta)

### 4.1 Test sintakse
```bash
sudo sshd -t
```

### 4.2 Pregled efektivne konfiguracije
```bash
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|allowusers|allowgroups|loglevel|maxauthtries|allowtcpforwarding|x11forwarding'
```

### 4.3 Restart / reload servisa
Debian/Ubuntu:
```bash
sudo systemctl reload ssh
# ili:
sudo systemctl restart ssh
```

RHEL/Fedora:
```bash
sudo systemctl reload sshd
# ili:
sudo systemctl restart sshd
```

---

## 5) Firewall i ograničavanje pristupa

### 5.1 Samo iz određenih IP adresa (primjer sa nftables)
```bash
sudo nft add rule inet filter input ip saddr { 203.0.113.10, 198.51.100.20 } tcp dport 22 accept
```

### 5.2 UFW (Ubuntu)
```bash
sudo ufw allow from 203.0.113.10 to any port 22 proto tcp
sudo ufw enable
sudo ufw status verbose
```

> Ako koristiš port 2222, zamijeni 22 → 2222.

---

## 6) Zaštita od brute-force (fail2ban)

### 6.1 Instalacija
Debian/Ubuntu:
```bash
sudo apt update
sudo apt install -y fail2ban
```

### 6.2 Minimalni jail (sshd)
Kreiraj:
```bash
sudo nano /etc/fail2ban/jail.d/sshd.local
```

Primjer:
```conf
[sshd]
enabled = true
maxretry = 4
findtime = 10m
bantime = 1h
```

Restart:
```bash
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
```

Provjera:
```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---

## 7) Logging i audit

### 7.1 Journald / syslog
Provjeri logove:
```bash
sudo journalctl -u ssh -n 200 --no-pager 2>/dev/null || true
sudo journalctl -u sshd -n 200 --no-pager 2>/dev/null || true
```

### 7.2 Audit (auditd) – primjer pravila
Instalacija (Debian/Ubuntu):
```bash
sudo apt install -y auditd
```

Dodaj pravilo za praćenje `sshd_config`:
```bash
echo "-w /etc/ssh/sshd_config -p wa -k sshd_config" | sudo tee /etc/audit/rules.d/sshd.rules
sudo augenrules --load
```

Pretraga događaja:
```bash
sudo ausearch -k sshd_config
```

---

## 8) “Emergency” rollback plan

Ako izgubiš pristup nakon izmjena:
1) Prijavi se preko konzole/ILO/VM console
2) Vrati backup:
```bash
sudo cp -a /etc/ssh/sshd_config.bak.* /etc/ssh/sshd_config
sudo sshd -t
sudo systemctl restart sshd 2>/dev/null || sudo systemctl restart ssh
```

---

## 9) Test komande (klijent)

Testaj sa verbose:
```bash
ssh -vvv user@server
```

Ako koristiš drugi port:
```bash
ssh -p 2222 -vvv user@server
```

Provjeri koji ključ koristi:
```bash
ssh -v -i ~/.ssh/id_ed25519 user@server
```

---

## 10) Preporuke (kratko)
- Preferiraj `ed25519` ključeve.
- Onemogući lozinke na serveru i koristi ključeve + (po potrebi) MFA preko PAM rješenja.
- Ograniči pristup po IP-u gdje je realno moguće.
- Drži logove, prati pokušaje, i koristi fail2ban/sshguard.
- Svaku izmjenu testiraj sa `sshd -t` prije restartovanja servisa.
