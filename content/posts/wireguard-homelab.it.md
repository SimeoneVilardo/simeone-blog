---
author: ["Simeone Vilardo"]
title: "Advanced WireGuard on Docker: Split Tunneling, No-NAT e Hardening su Linux Moderno"
date: "2025-01-20"
description: "Configurare Wireguard sicuro per un homelab"
summary: "Configurare WireGuard tramite Docker (utilizzando immagini popolari come `wg-easy`) è spesso venduta come un'operazione da 5 minuti. E lo è, se il tuo unico obiettivo è avere un tunnel funzionante senza preoccuparti troppo di cosa accade sotto il cofano. Tuttavia, quando si hanno requisiti ingegneristici specifici — come l'integrazione con un DNS locale (es. Pi-hole), la necessità di log reali (No-NAT) e una sicurezza granulare (Firewalling) — la configurazione out-of-the-box mostra tutti i suoi limiti, specialmente su distribuzioni Linux moderne (Debian 12/13, Ubuntu 22.04+) che utilizzano `nftables` e un sottosistema Docker complesso."
tags: ["networking", "homelab"]
series: ["Network", "Homelab"]
---

Configurare WireGuard tramite Docker (utilizzando immagini popolari come `wg-easy`) è spesso venduta come un'operazione da 5 minuti. E lo è, se il tuo unico obiettivo è avere un tunnel funzionante senza preoccuparti troppo di cosa accade "sotto il cofano".

Tuttavia, quando si hanno requisiti ingegneristici specifici — come l'integrazione con un DNS locale (es. Pi-hole), la necessità di log reali (No-NAT) e una sicurezza granulare (Firewalling) — la configurazione "out-of-the-box" mostra tutti i suoi limiti, specialmente su distribuzioni Linux moderne (Debian 12/13, Ubuntu 22.04+) che utilizzano `nftables` e un sottosistema Docker complesso.

In questo articolo analizzeremo come costruire un'architettura **VPN Split Tunnel** robusta, risolvendo i conflitti tra `iptables-legacy` e `nftables` e gestendo correttamente il flusso dei pacchetti attraverso le catene di Docker.

## 1. L'Obiettivo Architetturale

Vogliamo ottenere un ambiente dove:

1. **Split Tunnel Reale:** I client inviano nel tunnel **solo** il traffico destinato ai servizi interni e al DNS. Tutto il resto (Internet) viaggia sulla connessione del client (4G/Wi-Fi).
2. **Osservabilità (No-NAT):** Il server DNS (Pi-hole) deve vedere l'IP reale del client VPN (es. `10.13.13.2`), non l'IP del gateway Docker.
3. **Sicurezza "Least Privilege":** Anche se un client manipola la sua configurazione per inviare tutto il traffico, il server deve scartare (DROP) tutto ciò che non è esplicitamente autorizzato.

## 2. Configurazione Client-Side: `WG_ALLOWED_IPS`

Il primo livello di gestione del traffico avviene sul **Client**. WireGuard non ha un meccanismo di "push" delle rotte dinamico come OpenVPN; le rotte vengono definite staticamente nella configurazione del peer.

Nel `docker-compose.yml` di `wg-easy`, usiamo la variabile `WG_ALLOWED_IPS` per generare le config dei client:

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy
    network_mode: host
    environment:
      # Sottorete interna della VPN
      - WG_DEFAULT_ADDRESS=10.13.13.x
      # IP del DNS (Pi-hole)
      - WG_DEFAULT_DNS=192.168.0.2
      
      # FONDAMENTALE: Split Tunneling
      # Istruisce il client a creare rotte nel tunnel SOLO per questi IP/Range.
      # Se qui mettessimo 0.0.0.0/0, il client perderebbe la connettività internet
      # (perché il nostro firewall lato server bloccherà l'uscita verso WAN).
      - WG_ALLOWED_IPS=192.168.0.2/32,192.168.0.4/32,10.13.13.0/24

```

**Nota Tecnica:** Questa è una configurazione che risiede sul dispositivo dell'utente. Un utente malizioso (o curioso) può modificarla manualmente sul proprio smartphone impostando `AllowedIPs = 0.0.0.0/0`. Per questo motivo, non possiamo fidarci solo di questa configurazione: serve un firewall severo lato server (vedi Sezione 5).

## 3. Visibilità e Routing: Perché uccidere il Masquerading

La configurazione di default di `wg-easy` abilita il **Masquerading (NAT)**. Questo significa che il server VPN sostituisce l'IP sorgente del pacchetto (`10.13.13.2`) con il proprio IP LAN (`192.168.0.4`) prima di inoltrarlo.

* **Pro:** Non serve configurare nulla sulla rete locale.
* **Contro:** I log del Pi-hole diventano inutili (tutto sembra arrivare dal server VPN).

**La Soluzione No-NAT:**
Disabilitiamo il masquerading (vedremo come nella Sezione 4). Tuttavia, senza NAT, quando il pacchetto arriva al Pi-hole con mittente `10.13.13.2`, il Pi-hole cercherà di rispondere inviando il pacchetto al suo Gateway predefinito (il Router Internet), che scarterà il pacchetto perché non conosce la rete `10.13.13.0/24`.

Dobbiamo aggiungere una **Rotta Statica** sul Gateway della LAN (il router di casa) o direttamente sugli host target (Pi-hole):

> *Destination Network:* `10.13.13.0/24` -> *Next Hop (Gateway):* `192.168.0.4` (IP del Server VPN).

Inoltre, su Pi-hole (Settings -> DNS), bisogna abilitare **"Permit all origins"**, altrimenti le query provenienti da una subnet diversa (`10.x.x.x`) verranno ignorate per policy di sicurezza predefinita.

## 4. Il Paradosso di Docker e il Firewall "Split Brain"

Qui arriviamo al cuore del problema tecnico su sistemi Linux moderni.

### Il Problema: Legacy vs NFT

L'immagine `wg-easy` tenta, all'avvio, di iniettare regole permissive usando i binari `iptables` all'interno del container. Spesso questi sono `iptables-legacy`. L'host (Debian 13) usa `iptables-nft` (wrapper per nftables). Questo crea due set di regole disgiunti che possono andare in conflitto o essere ignorati.

### Il Problema: Le regole di Default

Di default, `wg-easy` esegue comandi simili a questi:

```bash
iptables -A FORWARD -i wg0 -j ACCEPT
iptables -A FORWARD -o wg0 -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.13.13.0/24 -j MASQUERADE

```

Se noi vogliamo gestire la sicurezza e rimuovere il NAT, dobbiamo inibire questo comportamento. Lo facciamo "hackerando" la variabile `WG_POST_UP` nel compose:

```yaml
    environment:
      # Un comando dummy che impedisce l'esecuzione degli script default
      - WG_POST_UP=echo "Skipping default rules"
      - WG_POST_DOWN=echo "Nothing to do"

```

### Il Blocco: Perché la connettività muore?

Appena rimuoviamo le regole automatiche sopra citate (specialmente le prime due di `ACCEPT`), la VPN smette di funzionare verso i container Docker. Perché?

Docker, per isolare i container, imposta la **Policy della catena FORWARD su DROP**.

Quando un pacchetto arriva da `wg0` (rete non gestita da Docker) diretto a un container:

1. **Senza regole esplicite:** Docker non riconosce l'interfaccia, scorre le sue catene, non trova match, e applica il `DROP` finale.
2. **Con Masquerading (se fosse attivo):** Il pacchetto sembrerebbe provenire dall'IP dell'host o del gateway. Docker tende a fidarsi del traffico "locale", quindi spesso lascia passare i pacchetti nattati. Questo è il motivo per cui il NAT spesso "risolve" magicamente i problemi: nasconde la provenienza esterna del traffico.

Poiché noi **non vogliamo usare il NAT**, il pacchetto si presenta con l'IP alieno `10.13.13.x` e viene brutalmente scartato dalla policy di Docker.

### La Soluzione: `DOCKER-USER` Chain

Docker fornisce una catena specifica, chiamata `DOCKER-USER`, che viene valutata **prima** delle regole di blocco interne. Dobbiamo inserire qui le nostre eccezioni.

È fondamentale usare `-I` (Insert) e non `-A` (Append) per assicurarsi che la regola sia in cima alla lista.

```bash
# Permetti ai pacchetti della VPN di entrare nel routing di Docker
sudo iptables -I DOCKER-USER -i wg0 -j ACCEPT

# Permetti alle risposte di tornare indietro
sudo iptables -I DOCKER-USER -o wg0 -j ACCEPT

```

*Nota: Questi comandi vanno resi persistenti (es. via systemd o rc.local) poiché al riavvio Docker resetta le catene.*

## 5. NFTables: La Whitelist Finale (Server-Side)

Ora che abbiamo aperto le porte di Docker tramite `DOCKER-USER`, il traffico scorre. Ma scorre *troppo*. Senza ulteriori filtri, un client VPN potrebbe provare a raggiungere qualsiasi cosa.

Dobbiamo configurare `nftables` sull'host per applicare una logica di **Whitelist rigorosa**.

File `/etc/nftables.conf`:

```nft
#!/usr/sbin/nft -f

flush ruleset

table inet my_shield {
    # Catena INPUT: Protegge l'Host stesso
    chain input {
        type filter hook input priority 0; policy accept;
        
        # Accetta traffico VPN solo verso l'IP del server (es. 192.168.0.4)
        iifname "wg0" ip daddr 192.168.0.4 accept
        
        # Blocca tentativi di accesso ad altre porte/servizi dell'host
        iifname "wg0" drop
    }

    # Catena FORWARD: Gestisce il traffico passante (verso Docker, LAN, Internet)
    chain forward {
        type filter hook forward priority 0; policy accept;

        # Accetta connessioni stabilite (necessario per il ritorno dei pacchetti)
        ct state established,related accept

        # --- REGOLA CRUCIALE PER DOCKER ---
        # Quando accedi a un container (es. porta 8080), Docker fa DNAT verso un IP interno (172.x...).
        # Questa regola accetta esplicitamente il traffico "nattato" (destinato ai container).
        iifname "wg0" ct status dnat accept

        # Accesso al Pi-hole (IP fisico)
        iifname "wg0" ip daddr 192.168.0.2 accept
        
        # Accesso diretto al server (Ridondanza di sicurezza)
        iifname "wg0" ip daddr 192.168.0.4 accept

        # --- DROP FINALE ---
        # Se un client prova a uscire su Internet o accedere ad altri IP LAN, viene bloccato qui.
        iifname "wg0" drop
    }
}

```

## Conclusione

Abbiamo sostituito la "magia" delle configurazioni automatiche con una comprensione profonda del packet flow.
Il risultato è un sistema dove:

1. **Docker** accetta i pacchetti VPN grezzi (grazie a `DOCKER-USER`).
2. **Il Routing** gestisce il ritorno dei pacchetti (grazie alle rotte statiche, senza NAT).
3. **NFTables** agisce come guardiano finale, bloccando ogni connessione non strettamente necessaria.

Questo setup non è solo "funzionante", è **osservabile** e **sicuro by-design**.
