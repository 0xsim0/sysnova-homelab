# sysNova Homelab

Ich baue mit meinem alten PC ein Homelab auf, um Linux,
Netzwerke und später Cloud- und DevOps-Technologien praktisch zu lernen.

## Hardware

- AMD Ryzen 5 2600
- 32 GB RAM
- Kingston SSD mit 240 GB für Ubuntu
- WD HDD mit 1 TB für Daten
- Verbindung über LAN und einen Switch zum Speedport Smart 4

## Grundinstallation

- Ubuntu Server installiert
- System auf der SSD, Datenplatte unter /srv eingebunden
- SSH-Zugriff mit Schlüsseln von Windows und Linux getestet
- UFW-Firewall aktiviert: SSH nur aus dem Heimnetz erlaubt
- Gleichbleibende IPv4-Adresse im Router reserviert
- Persönlichen Datenordner unter /srv/data/simo angelegt

## Durchgeführte Tests

- Nach einem Neustart ist SSH erreichbar.
- Die Datenplatte ist weiterhin unter /srv eingebunden.
- Die Testdatei ist vorhanden und lesbar.
- Die Firewall bleibt aktiv.
- systemctl --failed zeigt keine fehlgeschlagenen Dienste.

## Übung: Dateirechte

Mit chmod 000 habe ich der Testdatei alle Zugriffsrechte entzogen.
Beim Lesen mit cat erschien "Permission denied".

Mit chmod 600 habe ich dem Eigentümer Lesen und Schreiben erlaubt.
Danach konnte ich die Datei wieder lesen.

## Nächste Schritte

- Netzwerk und VPN einrichten
- Weitere Dienste auf dem Homelab betreiben
- Datensicherung einrichten und Wiederherstellung testen
