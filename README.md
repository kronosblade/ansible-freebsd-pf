![CI Status](https://github.com/kronosblade/ansible-freebsd-pf/actions/workflows/linting.yml/badge.svg)

# FreeBSD Ansible Router & Firewall

Automazione Ansible per trasformare un'istanza FreeBSD in un router di rete con firewalling avanzato, DNS cifrato, VPN e protezione dinamica. La configurazione è stata sviluppata su Proxmox VE con due interfacce virtuali, ma è applicabile in qualsiasi ambiente.

## Topologia

```
Internet
    │
[vtnet0] ext_if — 192.168.0.30/24
    │
 FreeBSD Router
    │
[vtnet1] int_if — 10.0.0.1/24
    │
 LAN interna (DHCP 10.0.0.100–200)
    └── Mumble (LXC container)
```

## Componenti

### PF Firewall
Filtro di pacchetti stateful, default-deny in ingresso (whitelist). La configurazione è suddivisa in macro/tabelle, NAT, redirect e regole di filtraggio.

- **Rate limiting SSH**: `max-src-conn-rate` + `overload <bruteforce> flush global` — gli IP che superano la soglia vengono bannati automaticamente a livello kernel.
- **Rate limiting Mumble**: limiti permissivi con `source-track rule` per non penalizzare client legittimi dietro NAT.
- **Anti-spoofing**: `antispoof` su entrambe le interfacce.

### Filtraggio Geografico Dinamico
Integrazione con [ipdbtools](https://it-notes.dragas.net/2024/06/16/freebsd-blocking-country-access) per il filtraggio per paese (ISO 3166-1 alpha-2). La modalità è selezionabile tramite la variabile `geoblock_mode`:

- **`whitelist`** (default del progetto): ammette solo i paesi in `geoblock_countries`, blocca tutto il resto del traffico WAN. Lista predefinita: UE + EEA + Regno Unito + Svizzera (32 codici, GB esplicito).
- **`blacklist`**: comportamento legacy -> blocca solo i paesi elencati.

Il passaggio fra le due modalità avviene cambiando una sola variabile e rilanciando il playbook. La tabella PF `<geo_countries>` viene aggiornata atomicamente (`pfctl -T replace -f`); cron job giornaliero e al boot. In modalità whitelist viene aggiunto un `pass <trusted_lan>` prima del blocco per evitare lockout dell'amministratore.

### Unbound DNS Resolver
Resolver locale con **DNS-over-TLS** verso Quad9 (`9.9.9.9`) e Cloudflare (`1.1.1.1`) sulla porta 853.

- Cache locale con TTL configurabile.
- Protezione DNS rebinding per indirizzi privati.
- Servito solo sulla LAN interna (`10.0.0.0/24`).

### ISC DHCP Server
Assegna IP ai client della LAN interna con gateway e DNS del router come default.

### Suricata IDS *(detection only)*
Intrusion Detection System in modalità passiva (pcap) sull'interfaccia WAN, analizza il traffico senza interromperlo, nessuna modalità inline/IPS. Ruleset Emerging Threats Open con aggiornamento automatico notturno via cron. Invocabile separatamente con `--tags suricata`.

### Endpoint dedicato *(opt-in)*
Role separata `gaming` per esporre un host LAN su porte specifiche con redirect e regole di filtraggio dedicate. Le regole sono distribuite in due anchor PF distinti (`gaming.rdr` e `gaming.filter`) per rispettare l'ordine di valutazione richiesto dal kernel. Attivabile solo con `--tags gaming`, richiede `gaming_host`, `gaming_lan_port`, `gaming_wan_port` configurati.

### VPN WireGuard *(opt-in)*
Role separata `wireguard` che configura un server WireGuard sul kernel host (modulo `if_wg`, nessuna jail) per dare ai client remoti accesso alla rete upstream e — con la configurazione consigliata — a tutto Internet via il router. Le regole PF sono splittate in tre anchor distinti: `wg.nat` (sezione translation), `wg.allow` (pass del tunnel UDP, incluso PRIMA del geoblock per restare raggiungibile anche da paesi non in whitelist) e `wg.filter` (pass del traffico decapsulato).

Caratteristiche:

- Porta UDP non standard di default (`51871`).
- Chiave privata server generata sul router in modo idempotente (mai salvata nel repo).
- Peer dichiarati in `group_vars/bsd_router.yml` con `public_key` + `allowed_ips` + `preshared_key` opzionale.
- Unbound estende automaticamente il bind sull'IP del tunnel quando la role è attiva (con `ip-transparent`), così i client possono usare il resolver interno come DNS.

Attivabile solo con `--tags wireguard`, richiede `wg_peers` non vuoto in `group_vars`.

#### Aggiungere un client (smartphone Android come esempio)

Prerequisiti:

- Record DNS pubblico che punta al router. Se è dietro Cloudflare il record deve essere **DNS only** (nuvola grigia): il proxy non gestisce UDP.
- Port forward UDP sul gateway upstream verso l'IP WAN del router FreeBSD, sulla porta `wg_port`.

Procedura:

**1. Sul router**, genera coppia chiavi del client e PresharedKey:
```bash
doas mkdir -p /usr/local/etc/wireguard/clients
cd /usr/local/etc/wireguard/clients
doas sh -c 'umask 077; wg genkey | tee android01.key | wg pubkey > android01.pub'
doas sh -c 'umask 077; wg genpsk > android01.psk'
```

**2. Componi la config che il client importerà** (sostituisci `tunnel.example.org` con il tuo dominio):
```bash
doas sh -c 'cat > android01.conf <<EOF
[Interface]
PrivateKey = $(cat android01.key)
Address = 10.10.0.2/24
DNS = 10.10.0.1

[Peer]
PublicKey = $(cat /usr/local/etc/wireguard/server.pub)
PresharedKey = $(cat android01.psk)
Endpoint = tunnel.example.org:51871
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
EOF'
```

> **Importante:** `AllowedIPs = 0.0.0.0/0, ::/0` è la configurazione consigliata (full tunnel). Senza `0.0.0.0/0` il client manda nel tunnel solo il traffico verso le subnet elencate, lasciando passare il resto via rete locale del telefono — il browser non userà la VPN. Inoltre, nell'app WireGuard Android, **assicurarsi che l'opzione "Exclude private IPs" sia disattivata**: quando attiva, sostituisce silenziosamente `0.0.0.0/0` con uno schema che esclude le RFC1918, neutralizzando l'accesso sia a Internet sia alle reti private interne.

**3. Genera il QR code e scansionalo dall'app:**
```bash
doas pkg install -y libqrencode
doas qrencode -t ansiutf8 < android01.conf
```

App WireGuard Android → `+` → "Scansione da codice QR" → punta al terminale.

**4. Aggiorna `group_vars/bsd_router.yml`** con la pubblica e la PSK del peer (il file è gitignored):
```bash
echo "public_key:    $(doas cat /usr/local/etc/wireguard/clients/android01.pub)"
echo "preshared_key: $(doas cat /usr/local/etc/wireguard/clients/android01.psk)"
```
```yaml
wg_peers:
  - name: "android01"
    public_key: "<output_pub>"
    allowed_ips: "10.10.0.2/32"
    preshared_key: "<output_psk>"
```

**5. Applica.** Al primo deploy della role serve un full run (per rigenerare `pf.conf` e `unbound.conf` con gli include della role) seguito dal run mirato:
```bash
ansible-playbook -i hosts.ini site.yml
ansible-playbook -i hosts.ini site.yml --tags wireguard
```
Per i client successivi (peer aggiunto a `wg_peers` su un router già configurato) basta:
```bash
ansible-playbook -i hosts.ini site.yml --tags wireguard
```

**6. Cleanup material sensibile sul router** dopo aver scansionato il QR — la chiave privata del client e la PSK in chiaro non hanno motivo di restare lato server:
```bash
doas rm /usr/local/etc/wireguard/clients/android01.{key,conf,psk}
```

#### Verifica della connessione
```bash
doas wg show wg0                # handshake recente, byte rx/tx > 0
doas pfctl -ss | grep 10.10     # stati attivi del peer
doas pfctl -sn | grep 10.10     # regola NAT del tunnel caricata
```

## Guida Rapida

**1. Prerequisiti**
```bash
pip install ansible ansible-lint
ansible-galaxy collection install -r requirements.yml
```

**2. Configurazione**
```bash
cp hosts.ini.example hosts.ini
cp group_vars/bsd_router.yml.example group_vars/bsd_router.yml
# Modifica i due file con le tue variabili (IP, interfacce, porte, chiave SSH)
```

**3. Deploy**
```bash
ansible-playbook -i hosts.ini site.yml
```

**4. Task opzionali** (richiedono `--tags` esplicito)
```bash
# Suricata IDS
ansible-playbook -i hosts.ini site.yml --tags suricata

# Endpoint dedicato (gaming/servizio su host LAN)
ansible-playbook -i hosts.ini site.yml --tags gaming

# Server VPN WireGuard (vedi sezione "Aggiungere un client" qui sopra)
ansible-playbook -i hosts.ini site.yml --tags wireguard

# Tuning kernel (sysctl NAT/forwarding)
ansible-playbook -i hosts.ini site.yml -K --tags tuning

# Aggiornamenti di sicurezza
ansible-playbook -i hosts.ini site.yml -K --tags upgrade
```

## Comandi di Monitoraggio

```sh
# IP bannati per brute force
doas pfctl -t bruteforce -T show

# Tabella geo (whitelist o blacklist a seconda di geoblock_mode)
doas pfctl -t geo_countries -T show

# Log firewall in tempo reale
doas tcpdump -n -e -ttt -i pflog0

# Stato connessioni attive
doas pfctl -s states
```

## Licenza

Progetto rilasciato sotto licenza MIT.
