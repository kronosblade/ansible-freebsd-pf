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

### GeoBlocking Dinamico
Integrazione con [ipdbtools](https://it-notes.dragas.net/2024/06/16/freebsd-blocking-country-access) per il filtraggio geografico per paese (ISO 3166-1 alpha-2).

- Sostituzione atomica della tabella PF (`pfctl -T replace -f`), nessuna finestra senza blocco.
- Cron job per aggiornamento quotidiano e al riavvio.
- Paesi configurabili tramite `geoblock_countries` (es. `"CN:RU:KP:IR"`).

### Unbound DNS Resolver
Resolver locale con **DNS-over-TLS** verso Quad9 (`9.9.9.9`) e Cloudflare (`1.1.1.1`) sulla porta 853.

- Cache locale con TTL configurabile.
- Protezione DNS rebinding per indirizzi privati.
- Servito solo sulla LAN interna (`10.0.0.0/24`).

### ISC DHCP Server
Assegna IP ai client della LAN interna con gateway e DNS del router come default.

### Suricata IDS *(detection only)*
Intrusion Detection System in modalità passiva (pcap) sull'interfaccia WAN, analizza il traffico senza interromperlo, nessuna modalità inline/IPS. Ruleset Emerging Threats Open con aggiornamento automatico notturno via cron. Invocabile separatamente con `--tags suricata`.

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

# Tuning kernel (sysctl NAT/forwarding)
ansible-playbook -i hosts.ini site.yml -K --tags tuning

# Aggiornamenti di sicurezza
ansible-playbook -i hosts.ini site.yml -K --tags upgrade
```

## Comandi di Monitoraggio

```sh
# IP bannati per brute force
doas pfctl -t bruteforce -T show

# IP bloccati per nazione
doas pfctl -t blocked_countries -T show

# Log firewall in tempo reale
doas tcpdump -n -e -ttt -i pflog0

# Stato connessioni attive
doas pfctl -s states
```

## Licenza

Progetto rilasciato sotto licenza MIT.
