# Segmentacija mreže: praktični vodič za mrežne inženjere

> **Repo putanja (lokalno):** `C:\Users\Max\repos\linux-security-hardening\docs\diagrams`  
> **Namjena:** README.md dokument za objašnjenje koncepta segmentacije mreže, dizajna zona, politika i implementacije (SMB/Enterprise/Cloud/OT), uz šablone za provjeru i “traffic matrix”.

---

## Sadržaj (TOC)

- [1. Uvod](#1-uvod)
- [2. Šta je segmentacija mreže](#2-šta-je-segmentacija-mreže)
- [3. Zašto je segmentacija mreže važna](#3-zašto-je-segmentacija-mreže-vaŽna)
- [4. Kako segmentacija radi: tri glavne svrhe](#4-kako-segmentacija-radi-tri-glavne-svrhe)
- [5. Kada raditi segmentaciju](#5-kada-raditi-segmentaciju)
- [6. Design principles](#6-design-principles)
- [7. Politike segmentacije mreže](#7-politike-segmentacije-mreže)
- [8. Tipovi i metode segmentacije](#8-tipovi-i-metode-segmentacije)
- [9. Implementation playbook](#9-implementation-playbook)
- [10. Primjeri segmentacije](#10-primjeri-segmentacije)
- [11. Kako se testira segmentacija](#11-kako-se-testira-segmentacija)
- [12. Šta treba da pokaže dijagram mrežnog segmenta](#12-šta-treba-da-pokaže-dijagram-mrežnog-segmenta)
- [13. DLP + PCI: segmentacija u usklađenosti](#13-dlp--pci-segmentacija-u-usklađenosti)
- [14. Segmentacija vs ZTNA (nulta pouzdanost)](#14-segmentacija-vs-ztna-nulta-pouzdanost)
- [15. Segmentacija vs mikrosegmentacija](#15-segmentacija-vs-mikrosegmentacija)
- [16. Traffic matrix (šablon)](#16-traffic-matrix-šablon)
- [17. Enterprise / SMB / Cloud / OT smjernice](#17-enterprise--smb--cloud--ot-smjernice)
  - [17.1 Enterprise](#171-enterprise)
  - [17.2 SMB](#172-smb)
  - [17.3 Cloud](#173-cloud)
  - [17.4 OT/ICS](#174-otics)
- [18. Najčešće greške i anti-patterni](#18-najčešće-greške-i-anti-patterni)
- [19. Brza kontrolna lista](#19-brza-kontrolna-lista)

---

## 1. Uvod

Segmentacija mreže je jedna od najefikasnijih arhitektonskih mjera za smanjenje rizika, stabilniji rad servisa i lakše upravljanje. Dobra segmentacija nije “više VLAN-ova”, nego jasno definisane zone povjerenja, kontrolisani tokovi saobraćaja između zona i provjerljive politike koje se mogu održavati kroz vrijeme.

---

## 2. Šta je segmentacija mreže

**Segmentacija mreže** je namjerno razdvajanje mreže na logičke ili fizičke segmente (zone) tako da se saobraćaj između segmenata **kontroliše** definisanim pravilima (vatrozid, ACL, SDN politike), a ne da se “slobodno preliva” kroz cijelu infrastrukturu.

U praksi, segment je skup uređaja/sistema koji dijele isti nivo povjerenja i slične sigurnosne potrebe (npr. korisnički računari, serverski segment, upravljački segment, gostujuća mreža, IoT).

---

## 3. Zašto je segmentacija mreže važna

Segmentacija:

- smanjuje površinu napada i ograničava bočno kretanje (lateral movement)
- ograničava širenje incidenata (malver/ransomware)
- omogućava princip najmanjih privilegija na mrežnom nivou (ko smije pričati s kim)
- poboljšava stabilnost i performanse (manje broadcast/multicast “buke”, jasnije rute)
- olakšava usklađenost (npr. PCI DSS) kroz izolaciju CDE i sužavanje obuhvata kontrola

---

## 4. Kako segmentacija radi: tri glavne svrhe

1. **Izolacija i ograničavanje štete**  
   Incident u jednom segmentu ne smije automatski postati incident cijele mreže.

2. **Kontrola tokova saobraćaja (policy enforcement)**  
   “East-west” i “north-south” saobraćaj treba prolaziti kroz kontrolne tačke gdje se primjenjuju pravila, bilježi i nadzire.

3. **Operativna jasnoća (upravljanje i promjene)**  
   Jasne granice olakšavaju dijagnostiku, planiranje adresiranja, QoS i upravljanje promjenama.

---

## 5. Kada raditi segmentaciju

Segmentaciju treba raditi gotovo uvijek, a posebno kada imate:

- različite klase uređaja: korisnici, serveri, administracija, IoT, video-nadzor, printeri
- kritične sisteme: AD/DNS, ERP, finansije, backup, virtualizacija
- goste i BYOD: gostujući Wi-Fi mora biti odvojen od interne mreže
- treće strane: vendor VPN, eksterni servis, partneri
- OT/ICS: industrijske mreže (PLC/SCADA) zahtijevaju strogu izolaciju i kontrolu protokola
- regulatorne zahtjeve: izolacija CDE i sužavanje PCI scope-a

---

## 6. Design principles

Ovo su praktični principi koji čine segmentaciju održivom:

- **Zone po nivou povjerenja, ne po “org chart-u”**  
  Segmenti moraju odražavati rizik i potrebe, ne samo organizacijsku strukturu.

- **Deny-by-default između zona**  
  Dozvoli samo ono što je nužno i dokumentuj “zašto”.

- **Minimalni tokovi (least privilege na mreži)**  
  Ne dozvoljavaj “any-any”. Svaki tok ima svrhu, vlasnika i rok revizije.

- **Kontrolne tačke na pravim mjestima**  
  Inter-VLAN rutiranje bez kontrole nije segmentacija. Kontrola mora biti provjerljiva (vatrozid/ACL/SDN).

- **Management plane izolacija**  
  Upravljanje mrežom (SSH/HTTPS/SNMP/NETCONF) mora biti odvojeno od korisničkog segmenta.

- **Dokazivost (auditability)**  
  Moraš moći dokazati da su kontrole efektivne: logovi, testovi, revizije.

- **Automatizacija gdje god je moguće**  
  U dinamičnim okruženjima (cloud, VM, kontejnere) ručna pravila brzo postanu tehnički dug.

---

## 7. Politike segmentacije mreže

Minimalni set politika:

1. **Model zona i povjerenja**  
   Definiši zone (Users, Servers, Mgmt, DMZ, Guest, IoT, Backup, CDE…).

2. **Matrica dozvoljenog saobraćaja**  
   Izvor → odredište → protokol/port → svrha → vlasnik servisa → evidencija promjene.

3. **Upravljanje izuzecima**  
   Svaki izuzetak ima vlasnika, rizik, kompenzacione mjere i rok.

4. **Standardi imenovanja i dokumentacije**  
   VLAN/VRF nazivi, IP plan, opis pravila, ticket/CR referenca.

5. **Revizija i lifecycle pravila**  
   Periodični pregled pravila i uklanjanje neiskorištenih/privremenih pravila.

---

## 8. Tipovi i metode segmentacije

### 8.1 Tipovi segmentacije (vrste)

- **Fizička segmentacija:** odvojena infrastruktura (air-gap, odvojeni preklopnici/linkovi)
- **Logička segmentacija (L2/L3):** VLAN, podmreže, rutiranje, VRF
- **Sigurnosne zone:** zone na vatrozidu, interni vatrozid (east-west)
- **Aplikacijska segmentacija:** 3-tier (web/app/db), servisne zone
- **Mikrosegmentacija:** kontrola workload-to-workload (VM/kontejner)
- **Cloud segmentacija:** VPC/VNet, security groups, NACL, odvojeni računi/projekti

### 8.2 Metode segmentacije (tehnike)

- VLAN + inter-VLAN kontrola (ACL/vatrozid)
- Podmreže i rutiranje uz filtriranje
- VRF (virtuelno rutiranje i prosljeđivanje) za snažnu izolaciju
- Zone na vatrozidu (perimetar i interno)
- ACL na L3 preklopnicima/usmjerivačima
- NAC / 802.1X (segmentacija po identitetu uređaja/korisnika)
- SDN/overlay (VXLAN/EVPN) i centralne politike
- Host-based vatrozid/politike (workload segmentacija)
- Kontejnerske politike (npr. Kubernetes NetworkPolicy)

---

## 9. Implementation playbook

### 9.1 Priprema (ne preskakati)

- inventar uređaja i servisa (makar realan spisak)
- klasifikacija podataka (javno/interno/povjerljivo/PCI)
- identifikacija tokova (ko mora s kim komunicirati)

**Ishod:** početna “traffic matrix” (šablon u sekciji 16).

### 9.2 Dizajn

- definisanje zona + IP plan (VLAN ID, podmreže, gateway, DHCP/DNS)
- izbor kontrolnih tačaka (vatrozid/ACL/SDN)
- definisanje matrice tokova i “deny-by-default” politike

### 9.3 Implementacija kontrola

- kreiranje VLAN-ova/podmreža/VRF-ova
- rutiranje (statičko/OSPF/BGP) + filtriranje ruta gdje treba
- pravila na vatrozidu/ACL-u: minimalno, eksplicitno, uz logovanje gdje je opravdano
- izolacija upravljačke ravni (Mgmt segment)

### 9.4 Postepeno uvođenje

- “pilot” segment (Guest ili IoT), zatim serverski, pa korisnički
- praćenje logova blokiranog saobraćaja i korekcija matrice tokova

### 9.5 Dokumentacija i održavanje

- dijagrami, matrica tokova, procedura promjene, plan testiranja
- periodične revizije pravila i uklanjanje zastarjelih izuzetaka

---

## 10. Primjeri segmentacije

### 10.1 SMB primjer (najčešći)

| Zona | VLAN | IP opseg | Primjeri uređaja | Tipična pravila |
|---|---:|---|---|---|
| Users | 10 | 10.10.10.0/24 | klijenti | prema serverima samo potrebni portovi |
| Servers | 20 | 10.10.20.0/24 | AD/DNS, file, app | admin samo iz Mgmt; ograničiti east-west |
| Mgmt | 30 | 10.10.30.0/24 | admin stanice, NMS | SSH/HTTPS/SNMP samo iz Mgmt |
| Guest | 40 | 10.10.40.0/24 | gosti | samo internet |
| IoT/Printers | 50 | 10.10.50.0/24 | printeri/kamere | Users → printer portovi; blok ostalo |
| DMZ | 60 | 10.10.60.0/24 | reverse proxy/web | DMZ → interno strogo ograničeno |
| Backup | 70 | 10.10.70.0/24 | backup repo | dozvoli samo potrebne backup portove |

### 10.2 Data centar (3-tier)

- **DMZ/Web** → **App** → **DB**
- Web ne smije direktno do baze; App je jedini koji priča s DB.

### 10.3 OT/ICS koncept

- IT zona, OT zona, Jump zona, Monitoring zona  
- dozvoliti samo nužne protokole i samo prema određenim odredištima

---

## 11. Kako se testira segmentacija

### 11.1 Funkcionalno testiranje

- provjera DNS/DHCP po segmentima
- test portova (npr. `nc`, `curl`, `Test-NetConnection`)
- provjera ruta i gateway-a

### 11.2 Sigurnosno testiranje (da zabranjeno ne prolazi)

- skeniranje iz “niže” zone prema “višoj” i očekivani **deny**
- tcpdump/Wireshark na kontrolnim tačkama
- pregled logova vatrozida (drop/deny događaji)
- simulacija bočnog kretanja (lateral movement) i dokaz blokade

### 11.3 Kontinuirano testiranje i revizija

- nakon svake promjene pravila
- periodično (npr. kvartalno) za uklanjanje tehničkog duga

---

## 12. Šta treba da pokaže dijagram mrežnog segmenta

Dobar dijagram segmentacije treba prikazati:

- zone/segmente i njihov nivo povjerenja
- VLAN ID / podmreže / VRF (ako se koristi)
- kontrolne tačke: vatrozid, L3 gateway, NAC, proxy, DLP
- smjer i tip tokova (npr. Users → App: TCP 443)
- kritične servise i vlasništvo (tim/odgovorna osoba)
- vanjske veze: internet, VPN, partner, oblak

**ASCII koncept (minimalno):**
```text
[Internet]
    |
 [Perimetarski vatrozid]
    |-------------------|
   DMZ               VPN/Remote
    |                   |
[Reverse Proxy]     [ZTNA/VPN GW]
    |
 [Interni vatrozid / zone]
  |     |       |        |
Users  Servers  Mgmt     IoT
```
---

## 13. DLP + PCI: segmentacija u usklađenosti

### 13.1 DLP (Data Loss Prevention)

DLP je skup procesa i alata koji detektuju i/ili sprječavaju iznošenje osjetljivih podataka (e-mail, web upload, cloud sync, USB, štampa). Segmentacija pomaže DLP-u jer:

- sužava gdje se osjetljivi podaci smiju nalaziti (npr. samo u CDE ili "sensitive" zoni)
- uvodi "chokepoints" (proxy/egress vatrozid) gdje je realno nadzirati izlazni saobraćaj
- olakšava pravila (npr. CDE nema direktan izlaz na internet osim eksplicitno dozvoljenog)

### 13.2 PCI: kompatibilnost, CDE i šta se kvalifikuje kao PCI

U kontekstu PCI DSS, segmentacija se često koristi za izolaciju **CDE (Cardholder Data Environment)** i sužavanje obuhvata ("scope-a"). U praksi:

- CDE uključuje sisteme koji **pohranjuju, obrađuju ili prenose** podatke kartica
- sve što ima **neograničenu povezanost** prema CDE može ući u obuhvat kontrole
- "PCI podatak" tipično uključuje PAN (broj kartice) i druge elemente *cardholder data*, dok je *sensitive authentication data* (npr. CVV/PIN) posebno strogo tretiran

> **Napomena:** Ako segmentaciju koristiš da smanjiš PCI obuhvat, moraš imati dokaz da su segmentacione kontrole efektivne (testiranje + dokumentacija).

---

## 14. Segmentacija vs ZTNA (nulta pouzdanost)

**Segmentacija** kontroliše tokove između zona (granice i politike na mrežnom nivou).  
**ZTNA** fokusira se na pristup aplikacijama prema identitetu i kontekstu (ko si, stanje uređaja, politika po sesiji), bez implicitnog povjerenja u "internu mrežu".

Najbolji rezultat u praksi je **kombinacija**: ZTNA za "ko/koliko/odakle", segmentacija za "kuda smije teći saobraćaj" i ograničavanje štete.

### Brzo poređenje

| Aspekt | Segmentacija mreže | ZTNA |
|---|---|---|
| Osnova kontrole | zone, IP, portovi, protokoli | identitet, kontekst, stanje uređaja |
| Tipična primjena | interna izolacija i kontrola tokova | pristup aplikacijama (posebno remote/BYOD) |
| Snaga | smanjuje "blast radius" | precizniji pristup po sesiji |
| Rizik | izuzeci "any-any" potkopaju sigurnost | zavisi od identiteta/integracija |

---

## 15. Segmentacija vs mikrosegmentacija

- **Segmentacija (klasična):** grublje zone (Users/Servers/DMZ), kontrola na vatrozidu/L3 granici.
- **Mikrosegmentacija:** fine-grained kontrola između workload-ova (VM/kontejner), često kroz SDN ili host-based politike.

Mikrosegmentacija ima smisla kad je okruženje dinamično (mnogo servisa, česte promjene) i kad je rizik "east-west" saobraćaja visok.

### Kada mikrosegmentacija posebno pomaže

- veliki data centri ili cloud okruženja sa mnogo servisa
- potreba za jakom kontrolom komunikacije "service-to-service"
- automatizacija politika (IaC/SDN) i dobra vidljivost tokova

---

## 16. Traffic matrix (šablon)

Ovu tabelu možeš direktno kopirati i popunjavati. Preporuka: svaka linija treba imati poslovni razlog, vlasnika i referencu na promjenu (ticket/CR).

| ID | Izvor zona/segment | Odredište zona/segment | Izvor (IP/CIDR/objekt) | Odredište (IP/CIDR/objekt) | Protokol | Port(ovi) | Smjer | Svrha / aplikacija | AuthN/AuthZ (ako postoji) | Kontrolna tačka (FW/ACL/SG) | Log (Y/N) | Kritičnost (L/M/H) | Vlasnik | Ticket/CR | Status (Plan/Test/Prod) | Napomena |
|---:|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Users | Servers | 10.10.10.0/24 | 10.10.20.10 | TCP | 443 | -> | Web pristup aplikaciji | SSO/LDAP | Internal FW | Y | M | App Team | CHG- | Plan |  |
| 2 | Mgmt | Network Devices | 10.10.30.0/24 | mgmt-objects | TCP | 22,443 | -> | Upravljanje uređajima | Admin RBAC | Mgmt FW/ACL | Y | H | NetOps | CHG- | Plan |  |
| 3 | Servers | Backup | 10.10.20.0/24 | 10.10.70.10 | TCP | (popuni) | -> | Backup replikacija | mTLS/Keys | Internal FW | Y | H | Infra | CHG- | Plan |  |

---

## 17. Enterprise / SMB / Cloud / OT smjernice

### 17.1 Enterprise

**Ciljevi:** skalabilnost, standardizacija, više domena, kompleksni tokovi, audit.

Preporuke:
- vatrozidne zone + VRF za jaku izolaciju (npr. prod/dev, tenant, CDE)
- centralizovano upravljanje politikama (*policy-as-code* gdje je moguće)
- NAC/802.1X (dinamičke politike, posture check)
- logging + SIEM integracija, segmentacione kontrole kao "guardrails"
- redovno testiranje i revizija (kvartalno ili nakon promjena)

### 17.2 SMB

**Ciljevi:** jednostavnost, mali tim, brzo održavanje.

Preporuke:
- 5–8 osnovnih zona (Users/Servers/Mgmt/Guest/IoT/DMZ/Backup)
- inter-VLAN kontrola (preferirano preko vatrozida)
- jasna "deny-by-default" pravila između Guest/IoT i interne mreže
- dokumentuj sve izuzetke i redovno uklanjaj privremena pravila

### 17.3 Cloud

**Ciljevi:** segmentacija po projektu/računu, identitet, "ephemeral" workload.

Preporuke:
- odvojeni računi/projekti za prod vs dev (gdje je moguće)
- VPC/VNet segmenti + routing kontrola
- Security Groups / NSG kao primarna kontrola "east-west"
- NACL kao dodatni sloj (stateless) gdje ima smisla
- privatne veze (PrivateLink/peering) uz minimalne rute
- automatizacija: IaC (Terraform), policy guardrails, kontinuirani compliance

### 17.4 OT/ICS

**Ciljevi:** sigurnost + dostupnost, kontrolisani protokoli, minimalne promjene.

Preporuke:
- stroga separacija IT/OT + jump host zona
- whitelist protokola i odredišta (najmanji set)
- pasivni monitoring gdje je moguće, promjene kroz kontrolisani proces
- backup/restore plan i incident plan specifičan za OT
- fizička segmentacija ili snažna logička izolacija gdje rizik opravdava

---

## 18. Najčešće greške i anti-patterni

- "Imamo VLAN-ove" bez inter-VLAN kontrola (u praksi: nema segmentacije)
- "any-any" izuzeci koji postanu trajni
- Mgmt nije izolovan (upravljanje iz Users segmenta)
- pravila bez vlasnika i bez svrhe
- nema periodične revizije (pravila se gomilaju)
- segmentacija "na papiru" bez testova i dokaza

---

## 19. Brza kontrolna lista

- [ ] definisane zone i nivo povjerenja
- [ ] "deny-by-default" između zona
- [ ] matrica tokova (traffic matrix) popunjena i odobrena
- [ ] Mgmt segment izolovan i strogo kontrolisan
- [ ] kontrolne tačke loguju relevantne allow/deny događaje
- [ ] dijagram segmentacije + IP plan su ažurni
- [ ] testiranje: funkcionalno + sigurnosno + periodično
- [ ] izuzeci imaju vlasnika, rok i reviziju
