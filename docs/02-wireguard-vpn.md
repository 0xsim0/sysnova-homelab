# Projekt 2: Sicherer Fernzugriff mit WireGuard

## Ziel

Ich wollte meinen Homelab-Server auch außerhalb meines Heimnetzes
erreichen können. Dafür habe ich WireGuard auf meinem Ubuntu-Server
eingerichtet und den Zugang mit einem Linux-Laptop und einem
Windows-PC getestet.

Die Installation und Grundkonfiguration des Servers habe ich
bereits in Projekt 1 dokumentiert.

## Aufbau

| Gerät | Aufgabe | VPN-Adresse |
|---|---|---|
| Ubuntu-Server | VPN-Endpunkt und SSH-Server | <SERVER_VPN_IP> |
| Linux-Laptop | Erster VPN-Client | <LINUX_VPN_IP> |
| Windows-PC | Zweiter VPN-Client | <WINDOWS_VPN_IP> |
| Router | Portweiterleitung und Dynamic DNS | Nicht veröffentlicht |

Alle Adressen und Schlüssel in dieser Dokumentation sind Platzhalter.
Die Beispiele sind nicht direkt ausführbar.

## So funktioniert die Verbindung

1. Der Client löst den Dynamic-DNS-Namen meines Heimanschlusses auf.
2. Er sendet verschlüsselte WireGuard-Pakete an den Router.
3. Der Router leitet UDP-Port 51820 an den Ubuntu-Server weiter.
4. WireGuard prüft den Schlüssel des Clients.
5. Über den VPN-Tunnel verbinde ich mich per SSH mit dem Server.

SSH benötigt zusätzlich seinen eigenen SSH-Schlüssel.

## Installation und Schlüssel

Auf Ubuntu habe ich WireGuard über die Paketverwaltung installiert:

```bash
sudo apt update
sudo apt install wireguard
wg --version
```

Auf Windows verwende ich die offizielle WireGuard-App.

Jedes Gerät hat ein eigenes WireGuard-Schlüsselpaar:

- Der private Schlüssel bleibt auf dem jeweiligen Gerät.
- Der öffentliche Schlüssel wird beim Verbindungspartner eingetragen.
- Jeder Client erhält eine eigene VPN-Adresse.

Die Konfigurationsdateien auf Ubuntu liegen unter `/etc/wireguard`.
Der Ordner ist mit den Rechten 700 geschützt, die Dateien mit 600.
Eigentümer ist root.

## Beispielkonfiguration des Servers

Die echten Schlüssel und Adressen sind nicht enthalten.

```ini
[Interface]
Address = <SERVER_VPN_IP>/24
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>

[Peer]
PublicKey = <LINUX_PUBLIC_KEY>
AllowedIPs = <LINUX_VPN_IP>/32

[Peer]
PublicKey = <WINDOWS_PUBLIC_KEY>
AllowedIPs = <WINDOWS_VPN_IP>/32
```

`AllowedIPs` ordnet jedem Peer seine VPN-Adresse zu.
`/32` bezeichnet genau eine IPv4-Adresse.

## Beispielkonfiguration eines Clients

```ini
[Interface]
Address = <CLIENT_VPN_IP>/32
PrivateKey = <CLIENT_PRIVATE_KEY>

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <DDNS_HOSTNAME>:51820
AllowedIPs = <SERVER_VPN_IP>/32
```

Nur der Verkehr zur VPN-Adresse des Servers wird durch den Tunnel
geschickt. Normales Surfen läuft weiterhin über den Internetzugang
des Clients.

Ein Zugriff auf das gesamte Heimnetz oder eine Weiterleitung des
gesamten Internetverkehrs ist in diesem Projekt nicht eingerichtet.

## Firewall und Router

Die Ubuntu-Firewall UFW ist aktiv.

Die eingerichteten Freigaben erlauben:

- SSH aus dem Heimnetz.
- WireGuard über UDP-Port 51820 am LAN-Adapter des Servers.
- SSH über den VPN-Adapter von den beiden festgelegten Client-Adressen
  zur VPN-Adresse des Servers.

Im Router wird ausschließlich UDP-Port 51820 für dieses Projekt
an den Server weitergeleitet. Für SSH-Port 22 wurde keine
Portweiterleitung eingerichtet.

Eine anfangs nur für das Heimnetz angelegte WireGuard-Regel wurde
durch eine allgemeinere Regel ergänzt. Die engere Regel ist dadurch
überflüssig und kann entfernt werden.

## Dynamic DNS

Da sich die öffentliche IP-Adresse des Heimanschlusses ändern kann,
habe ich einen Hostnamen bei No-IP eingerichtet.

Der Router aktualisiert den DNS-Eintrag mit eigenen DDNS-Zugangsdaten.
In der WireGuard-Konfiguration steht der Hostname als Endpoint.

Beim kostenlosen No-IP-Tarif muss der Hostname alle 30 Tage
bestätigt werden.

Wenn sich die öffentliche IP ändert, muss ein bereits laufender
Client den Namen gegebenenfalls erneut auflösen. Auf Linux kann
dafür der Tunnel neu gestartet werden:

```bash
sudo systemctl restart wg-quick@wg0
```

Dieser Befehl wird auf dem Client ausgeführt und unterbricht
dessen VPN-Verbindung kurz.

## Automatischer Start auf dem Server

```bash
sudo systemctl enable wg-quick@wg0
```

Damit startet der VPN-Endpunkt nach einem Serverneustart automatisch.

## Durchgeführte Tests

| Test | Ergebnis |
|---|---|
| Linux-Laptop verbindet sich im Heimnetz per VPN | Erfolgreich |
| Linux-Laptop verbindet sich über Handy-Hotspot und mobile Daten | Erfolgreich |
| Verbindung mit Dynamic-DNS-Namen als Endpoint | Erfolgreich |
| Windows-PC verbindet sich über das Mobilfunknetz | Erfolgreich |
| SSH über die VPN-Adresse des Servers | Erfolgreich |
| Serverneustart und erneute VPN-Anmeldung von Windows | Erfolgreich |
| Windows-SSH-Anmeldung ausschließlich mit Public-Key-Verfahren | Erfolgreich |

Beim Außentest waren die Clients vom Heimnetz getrennt.
Dadurch konnte ich prüfen, ob die Verbindung tatsächlich über
das Internet und die Portweiterleitung funktioniert.

## Befehle zur Kontrolle

Auf dem Server:

```bash
systemctl is-enabled wg-quick@wg0
systemctl is-active wg-quick@wg0
sudo wg show
sudo ufw status numbered
```

Bei `wg show` habe ich geprüft:

- Ist der erwartete Peer vorhanden?
- Gibt es nach einem Verbindungsversuch einen aktuellen Handshake?
- Steigen die Zähler für empfangene und gesendete Daten?

Ein aktivierter Tunnel allein beweist noch keine erfolgreiche Verbindung.
Deshalb habe ich zusätzlich eine SSH-Anmeldung getestet.

## Probleme und Lösungen

### Portweiterleitung im Router

Der Router akzeptierte denselben Wert als Start- und Endport nicht.
Für den einzelnen UDP-Port habe ich das öffentliche Endport-Feld
leer gelassen. Danach ließ sich die Regel speichern.

### Verbindung zum Handy-Hotspot

Der Linux-Laptop konnte sich zunächst nicht mit dem Hotspot verbinden.
Ich habe zuerst die WLAN- und Internetverbindung geprüft.
Nachdem die Hotspot-Verbindung funktionierte, war der VPN-Test möglich.

### Falscher SSH-Schlüsselpfad unter Windows

Ein SSH-Befehl verwies auf eine nicht vorhandene Schlüsseldatei.
Die Verbindung funktionierte trotzdem über eine andere verfügbare
Identität.

Ich habe die vorhandenen Dateinamen geprüft. Auf dem Windows-PC
liegt der Standardschlüssel `id_ed25519`.

Ein weiterer Test mit ausschließlich erlaubter Public-Key-Anmeldung
war erfolgreich.

## Was ich gelernt habe

- Öffentliche IP, Heimnetz-IP und VPN-IP haben unterschiedliche Aufgaben.
- Eine Portweiterleitung und eine Server-Firewall sind getrennte Einstellungen.
- WireGuard-Schlüssel und SSH-Schlüssel werden unabhängig voneinander verwendet.
- Jeder VPN-Client kann einen eigenen Schlüssel und eine eigene Adresse erhalten.
- Dynamic DNS hält einen festen Namen mit der aktuellen öffentlichen IP verbunden.
- Ein Test im Heimnetz ersetzt keinen Test von außerhalb.
- Einstellungen sollten auch nach einem Neustart geprüft werden.

## Datenschutz in diesem Repository

Nicht veröffentlicht werden:

- Private Schlüssel und Passwörter
- DDNS-Key-Zugangsdaten
- Echte öffentliche IP-Adressen und der echte DDNS-Hostname
- Persönliche Netzwerkadressen und Gerätekennungen
- Unbearbeitete VPN-Konfigurationen
- Screenshots oder Protokolle mit vertraulichen Angaben

Die Konfigurationsbeispiele enthalten ausschließlich Platzhalter.

## Ergebnis

Ich kann meinen Ubuntu-Server von Linux und Windows über einen
WireGuard-Tunnel per SSH verwalten. Der Zugang funktioniert auch
außerhalb meines Heimnetzes und nach einem Serverneustart.

Als nächste Schritte möchte ich weitere Dienste gezielt bereitstellen
und eine Datensicherung mit Wiederherstellungstest einrichten.
