---
title: "Hvordan få fjerntilgang til Homeassistant med duckdns og swag."
date: 2022-10-10
layout: post
---

For å få tilgang til HomeAssistant utenfor ditt eget lokalnett, så har man to alternativer:

1.  Homeassistant cloud(Nabu casa)
Nabu casa opptrer som en proxy til din lokale HomeAssistant-installasjon.
Skybasert, og koster USD 7,50 per måned.  Svært enkel å sette opp, man trenger stort sett bare å klikke på "Enable" i GUI-et til HomeAssistant og legge inn kredittkort så er man i gang.

2.  Ordne selv.
For å kunne nå din HomeAssistant-installasjon fra internett, så er det en del betingelser som må være på plass
    - Statisk IP-adresse eller dynamisk DNS-oppføring.
    - Brannmuren i hjemmenettet ditt må åpnes opp i relevante porter.
    - Forbindelsen bør sikres med et TLS-sertifikat, slik at tredjepart ikke kan avlytte trafikken.

Det høres komplisert ut, men egentlig så er det ikke uoverkommelig å gjennomføre.

Du må ha følgende på plass:
- Brukernavn og passord til ruteren din, og ruteren må ha mulighet til å sette åpne opp brannmur, samt sette opp videresending av åpne porter(port forwarding)
- En datamaskin som kan kjøre docker-containere, kan gjerne være samme maskin som kjører HomeAssistant.

# Dynamisk DNS-oppføring

Det finnes flere leverandører av dynamisk DNS-oppføringer, blant annet dyndns, no-ip, duckdns med flere.
HomeAssistant har innebygget støtte for duckdns, så da er det naturlig å velge denne.
Duckdns gir deg inntil fem gratis domeneoppføringer, så den er veldig gunstig å bruke.  All bruk av dynamisk dns krever en klient som er installert lokalt, som med jevne mellomrom gir leverandør beskjed om hvilken ekstern ip-adresse ruteren har, slik at leverandøren av DNS-oppslaget kan holde seg oppdatert med hvilken ip-addresse DNS-oppføringen skal peke på.

Som tidligere nevnt så støtter HomeAssistant duckdns, og sørger automatisk for å holde DNS-oppføring i synk med ekstern ip-adresse.

Fremgangsmåte:

1.  Gå til https://duckdns.org i en nettleser, logg inn med Github,Google,Twitter eller Reddit-bruker.  Eller lag din egen bruker.
2.  Finn deg ett unikt subdomene-navn, og klikk på "add domain"
3.  Noter ned subdomene-navn og access-token, du vil trenge dem litt senere.
![Duckdns illustrasjon](/img/duckdns.png)

4.  Åpne opp `configuration.yaml` i din HomeAssistant-installasjon, og legg til følgende:

```yaml

http:
  ip_ban_enabled: true
  login_attempts_threshold: 3
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.1.0/24  # Local Lan, endre denne dersom din ruter ikke har 192.168.1.0/24 som nett.
    - 172.10.0.0/24  # Docker network
duckdns:
  domain: "DITT-DOMENE"
  access_token: "DITT_DUCKDNS_ACCESS_TOKEN_HER"

```
**NB** domain skal kun inneholde det du valgte i subdomain i pkt 2, ikke hele domenet.

5.  Gjør en restart av HomeAssistant.

Vent noen minutter, og gjør følgende fra et kommandolinjevindu:

```sh
ping DITT-DOMENE.duckdns.org

```

Får du svar tilbake, så har du nå opprettet en permanent DNS-oppføring som peker på hjemmenettet ditt.


## Port-forwarding

Før vi kan begynne å bruke HomeAssistant fra utsiden av lokalnettet, så må det åpnes opp for det i brannmuren.
Siden vi skal kjøre over TLS/SSL, så må portene 80 og 443 åpnes opp i brannmuren for TCP.
I tillegg så må de to portene videresendes(port-forwardes) til en maskin i ditt lokalnett som kan kjøre docker.
Det kan gjerne være samme maskin som kjører HomeAssistant, den dockercontaineren vi skal sette opp er veldig lettvekts.

Hvordan man åpner opp brannmur og setter opp videresending av porter varierer fra ruter til ruter, konsulter manualen til ruteren din dersom du får problemer her.

Lag en fil i en mappe, kall denne for `docker-compose.yml` , og gi den følgende innhold:

```yaml
version: '3'
services:
  swag:
    image: ghcr.io/linuxserver/swag
    container_name: swag
    cap_add:
      - NET_ADMIN
    environment:
      - PUID=1001
      - PGID=1001
      - TZ=Europe/Berlin
      - URL=DITT-DOMENE.duckdns.org
      - VALIDATION=duckdns
      - DUCKDNSTOKEN=DITT-DUCKDNS-TOKEN
      - SUBDOMAINS=wildcard
    volumes:
      - ./homeassistant.subdomain.conf:/config/nginx/proxy-confs/homeassistant.subdomain.conf
    ports:
      - 443:443
      - 80:80
    restart: unless-stopped

```
Erstatt `DITT-DOMENE` og `DITT-DUCKDNS-TOKEN` med dine verdier.
I tillegg så oppretter du en fil til, kall denne for `homeassistant.subdomain.conf`, og gi den følgende innhold:

```ini
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name homeassistant.*;

    include /config/nginx/ssl.conf;

    client_max_body_size 0;

    location / {

        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        set $upstream_app IP_TIL_DIN_HA_INSTALLASJON;
        set $upstream_port 8123;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;

    }

    location ~ ^/(api|local|media)/ {
        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        set $upstream_app IP_TIL_DIN_HA_INSTALLASJON;
        set $upstream_port 8123;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;
    }
}
```
Erstatt `IP_TIL_DIN_HA_INSTALLASJON` med ip-adresse til maskinen som kjører HomeAssistant.
Lagre filen, og kjør `docker-compose up -d`
Første gang dette gjøres, så lastes containeren ned, containeren vil kontakte letsencrypt for å få et SSL-sertifikat, og sette opp dette.  Dette kan ta noen minutter.  Du kan følge med på progresjonen med å skrive `docker-compose logs swag -f`

Når loggfilen til slutt skriver ut noe slikt, så er den klar:

```
swag             | [ls.io-init] done.
swag             | s6-rc: info: service 99-ci-service-check successfully started
swag             | Server ready
```

swag fungerer i dette tilfellet som api-gateway og ssl-terminator.  Det vil si at alle kall som kommer inn til den som starter med https://homeassistant.[ditt_domene].duckdns.org vil bli videresendt til http://ip-til-homeassistant-server:8123


Hvis alle stegene ovenfor er gjort riktig, så skal du nå kunne gå i en nettleser til
https://homeassistant.ditt-domene.duckdns.org
(Erstatt ditt-domene med domenet du registrerte hos duckdns)

Gratulerer, du har nå en HomeAssistant-installasjon som er tilgjengelig fra det store internett!
