# ansible-freebsd-pf
![CI Status](https://github.com/kronosblade/ansible-freebsd-pf/actions/workflows/linting.yml/badge.svg)

# FreeBSD Ansible Router & Firewall
Questo repository contiene un'automazione Ansible per trasformare un'istanza FreeBSD in un router di rete professionale con firewalling avanzato e protezione dinamica. Nel caso specifico ho utilizzato Proxmox VE per creare due interfacce virtuali da "attaccare" alla macchina virtuale.
Questa config può essere applicata tranquillamente in ambiente reale.

La prima, la più interna, rappresenta la rete che ospita un servizio d'esempio di chat vocale, un'istanza di Mumble configurata su un container LXC. Il protocollo UDP richiede stabilità ed efficienza, FreeBSD con PF come Firewall si sposa bene con questi requisiti dato il network stack maturo e ottimizzato. 

# Architettura Logica
Il sistema è progettato per gestire il traffico in entrata e in uscita filtrando le minacce prima che raggiungano i servizi interni (come Mumble o SSH).
# Componenti Chiave
1. PF Firewall (Packet Filter) è un filtro di pacchetti stateful con licenza BSD, un componente fondamentale per la gestione dei firewall. È paragonabile a netfilter (iptables), ipfw e ipfilter.
PF è stato sviluppato per OpenBSD, ma è stato portato su molti altri sistemi operativi. 
    - La configurazione è suddivisa in:
        a. Macro e Tabelle: Gestione centralizzata di interfacce e gruppi di IP (Trusted LAN, Bruteforce, GeoBlock).
        b. Polizza Default: "Block all" in ingresso per massimizzare la sicurezza (Whitelisting).
        c. Rate Limiting: Protezione contro attacchi Brute Force su SSH e Mumble. Se un IP supera le soglie di connessione, viene inserito automaticamente in una tabella di blocco a livello kernel.

2. GeoBlocking Dinamico. Per questa sezione ringrazio il lavoro del Prof. [Stefano Marinelli](https://it-notes.dragas.net/2024/06/16/freebsd-blocking-country-access):
    - Integrazione con ipdbtools per il filtraggio geografico.
    - Automazione: Uno script (update_blocked_countries.sh) scarica i database IP aggiornati.
    - Efficienza: Migliaia di range IP vengono caricati in tabelle PF ottimizzate per non impattare sulle prestazioni della CPU.
    - Cron Job: Aggiornamento automatico quotidiano e al riavvio del sistema.

3. Gestione IaC (Infrastructure as Code)
    Separazione Dati/Logica: Variabili sensibili (IP, porte) segregate in group_vars.
    Idempotenza: Ansible assicura che il router sia sempre nello stato desiderato senza riconfigurazioni manuali.
    Sicurezza: Accesso SSH limitato a chiavi pubbliche e privilegi gestiti tramite doas.

# Guida Rapida
    1. Configurazione: Copia hosts.ini.example e group_vars/bsd_router.yml.example rimuovendo l'estensione *.example*.
    2. Personalizzazione: Inserisci il tuo admin_ip e le interfacce di rete (es. vtnet0).

## Deployment:
    Bash
    ansible-playbook -i hosts.ini site.yml

# Comandi di Monitoraggio
    1. Vedere ban attivi: 
        `doas pfctl -t bruteforce -T show`
    2. Vedere IP bloccati per nazione: 
        `doas pfctl -t blocked_countries -T show`
    3. Log in tempo reale: 
        `doas tcpdump -n -e -ttt -i pflog0`

# Licenza
Progetto rilasciato sotto licenza MIT.
