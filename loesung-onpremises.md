# Lösung 1: On-Premises-Webserver (Nginx auf Linux-VM)

> **Status:** Empfohlene Lösung für schnelle Umsetzung mit minimalem Betriebsaufwand

---

## Zusammenfassung

Diese Lösung ersetzt den bestehenden FTP-Server durch einen modernen Linux-Webserver
(Nginx), der die Daten des AIUB unter **<https://download.aiub.unibe.ch/>** öffentlich
zugänglich macht. Die Synchronisation erfolgt täglich automatisch vom HPC-Cluster via `rsync`.
Die Lösung ist einfach zu betreiben und verursacht keine laufenden Cloud-Kosten, falls die
Universität eine virtuelle Maschine bereitstellt.

---

## Architekturübersicht

```
┌─────────────────────────────────────────────────────────┐
│  HPC-Cluster (gesichertes Netz)                         │
│  /data/gnss/  ──── rsync über SSH (täglich, cronjob) ──▶│──┐
└─────────────────────────────────────────────────────────┘  │
                                                              ▼
                                              ┌──────────────────────────┐
                                              │  Linux-VM (Ubuntu 24.04) │
                                              │  /srv/data/              │
                                              │                          │
                                              │  Nginx (autoindex on)    │
                                              │  Let's Encrypt (HTTPS)   │
                                              └──────────┬───────────────┘
                                                         │ HTTPS (Port 443)
                                                         │ HTTP  (Port 80, redirect)
                                              ┌──────────▼───────────────┐
                                              │  Öffentliche Nutzer      │
                                              │  https://download.aiub.  │
                                              │         unibe.ch/        │
                                              └──────────────────────────┘
```

---

## Technische Komponenten

| Komponente | Produkt / Technologie | Zweck |
|---|---|---|
| Webserver | Nginx ≥ 1.24 | Dateien bereitstellen, Verzeichnislisten |
| Betriebssystem | Ubuntu 24.04 LTS | Stabile, langfristig unterstützte Basis |
| TLS-Zertifikat | Let's Encrypt + certbot | Kostenloses HTTPS, automatische Erneuerung |
| Dateitransfer | rsync über SSH | Tägliche Synchronisation vom HPC |
| Überwachung | UptimeRobot (kostenlos) | Verfügbarkeitsmonitoring per E-Mail |

---

## Verzeichnisauflistung

Nginx unterstützt mit der Direktive `autoindex on` eine **automatische Verzeichnisliste**, die
identisch mit der aktuellen HTTP-Ansicht des FTP-Servers aussieht. Nutzer können die
Verzeichnisstruktur im Browser navigieren und einzelne Dateien herunterladen – ganz ohne
zusätzliche Software oder Skripte.

**Nginx-Konfiguration (Auszug):**

```nginx
server {
    listen 443 ssl;
    server_name download.aiub.unibe.ch;

    root /srv/data;
    charset utf-8;

    location / {
        autoindex on;
        autoindex_exact_size off;   # Dateigrößen in KB/MB statt Bytes
        autoindex_localtime on;     # Zeitangaben in Lokalzeit
    }
}
```

---

## Synchronisation vom HPC-Cluster

Die Synchronisation wird als täglicher Cronjob auf dem HPC-Cluster eingerichtet:

```bash
# /etc/cron.d/aiub-sync (auf dem HPC-Cluster)
# Täglich um 02:00 Uhr
0 2 * * * hpcuser rsync -avz --delete --progress \
    /data/gnss/ \
    deploy@download.aiub.unibe.ch:/srv/data/ \
    >> /var/log/aiub-sync.log 2>&1
```

**Erklärung der Optionen:**

- `-a` — Archivmodus (Berechtigungen, Zeitstempel, Symlinks erhalten)
- `-v` — Ausführliche Ausgabe ins Logfile
- `-z` — Komprimierung während der Übertragung
- `--delete` — Auf dem Ziel gelöschte Dateien entfernen

**SSH-Authentifizierung:**

```bash
# SSH-Schlüsselpaar auf dem HPC generieren
ssh-keygen -t ed25519 -f ~/.ssh/aiub_sync -C "aiub-hpc-sync"

# Öffentlichen Schlüssel auf der VM autorisieren
ssh-copy-id -i ~/.ssh/aiub_sync.pub deploy@download.aiub.unibe.ch
```

---

## HTTPS und Zertifikatsverwaltung

Let's Encrypt stellt kostenlose, automatisch erneuerte TLS-Zertifikate bereit.

```bash
# certbot installieren (Ubuntu 24.04)
sudo snap install --classic certbot

# Zertifikat ausstellen und Nginx automatisch konfigurieren
sudo certbot --nginx -d download.aiub.unibe.ch

# Automatische Erneuerung testen
sudo certbot renew --dry-run
```

Die automatische Erneuerung alle 90 Tage erfolgt über einen systemd-Timer, der mit certbot
bereits eingerichtet wird – kein manueller Eingriff erforderlich.

---

## Erforderliche Kompetenzen und Governance

> **Wichtig:** Der Betrieb einer Linux-VM auf der Universitätsinfrastruktur unterliegt den
> IT-Governance-Richtlinien und dem Härtungsleitfaden der Universität Bern. Die verantwortliche
> Person (Systemengineer) muss über die nachfolgend beschriebenen Kenntnisse verfügen oder
> sich diese aneignen.

### Fachliche Anforderungen an den Systembetreiber

| Kompetenzbereich | Beschreibung |
|---|---|
| **Linux-Systemadministration** | Installation, Konfiguration und Wartung von Ubuntu/Debian-Servern; Paketverwaltung (`apt`); Dienstverwaltung (`systemd`); Benutzerverwaltung |
| **Server-Härtung (Hardening)** | Umsetzung des universitären Härtungsleitfadens: minimale Dienste, Firewall-Konfiguration (`ufw`/`nftables`), SSH-Absicherung (Key-only, kein Root-Login), Deaktivierung unnötiger Netzwerkports |
| **Nginx-Konfiguration** | Virtual Hosts, TLS-Einstellungen, Sicherheitsheader, Log-Verwaltung |
| **Netzwerk-Grundlagen** | DNS, Firewallregeln, HTTPS/TLS, SSH-Tunneling |
| **Monitoring und Compliance** | Installation und Betrieb der vorgeschriebenen Agenten (siehe unten) |

### Vorgeschriebene Software-Agenten (Uni Bern)

Gemäss den IT-Governance-Vorgaben der Universität Bern müssen auf jeder VM, die am
Universitätsnetz betrieben wird, folgende Agenten installiert und konfiguriert werden:

| Agent | Zweck | Aufwand |
|---|---|---|
| **Rapid7 InsightVM** | Schwachstellen-Scanning und Compliance-Reporting. Erkennt fehlende Patches, unsichere Konfigurationen und bekannte Sicherheitslücken. | Einmalig: Installation und Registrierung (~30 min). Laufend: Schwachstellenberichte prüfen und Massnahmen umsetzen. |
| **Snow Agent** | Software-Asset-Management. Inventarisiert installierte Software für die Lizenzverwaltung der Universität. | Einmalig: Installation (~15 min). Laufend: keine manuelle Aktion. |

### Server-Härtung: Mindestmassnahmen

Die folgenden Massnahmen sind gemäss gängiger Praxis und universitärem Härtungsleitfaden
umzusetzen:

```bash
# SSH absichern (/etc/ssh/sshd_config)
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers deploy

# Firewall konfigurieren
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP (Redirect auf HTTPS)
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable

# Automatische Sicherheitsupdates
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

> **Hinweis für den Forscher:** Falls diese Kompetenzen nicht vorhanden sind und die IT-Abteilung
> den Aufbau und Betrieb nicht übernehmen kann, ist eine **Cloud-Lösung (Lösung 2 oder 3)**
> möglicherweise besser geeignet, da dort keine Server-Härtung, kein Patching und keine
> Agenten-Installation erforderlich sind.

---

## Betrieb und Wartung

| Aufgabe | Häufigkeit | Aufwand |
|---|---|---|
| Systemupdates (unattended-upgrades) | Automatisch täglich | Keine manuelle Aktion |
| Schwachstellenberichte (Rapid7) prüfen | Monatlich | 15–30 min |
| Kritische Patches manuell einspielen | Bei Bedarf (Rapid7-Alert) | 30–60 min |
| Nginx-Logs prüfen (bei Bedarf) | Bei Fehlerfall | 5–10 min |
| Speicherplatz prüfen | Monatlich | 2 min |
| Datensicherung (VM-Snapshot) | Wöchentlich | Automatisch (Hypervisor) |
| certbot-Erneuerung | Alle 90 Tage | Automatisch |
| Snow/Rapid7-Agent-Updates | Halbjährlich | 15 min |

---

## Kostenschätzung

### Option A: VM auf Uni-Infrastruktur (Empfohlen)

| Posten | Monatliche Kosten |
|---|---|
| VM (durch Uni bereitgestellt) | CHF 0.– |
| Speicher (durch Uni bereitgestellt) | CHF 0.– |
| TLS-Zertifikat (Let's Encrypt) | CHF 0.– |
| **Total** | **CHF 0.–/Monat** |

*Voraussetzung: Die Universität Bern stellt eine VM mit öffentlicher IP-Adresse und
mindestens 600 GB Speicher bereit.*

### Option B: Externer VPS-Anbieter (falls keine Uni-Infrastruktur verfügbar)

| Posten | Produkt | Monatliche Kosten |
|---|---|---|
| Server | Hetzner CPX11 (2 vCPU, 2 GB RAM) | ~CHF 5.– |
| Speicher | Hetzner Volume 600 GB | ~CHF 24.– |
| Traffic | In Hetzner-Tarif inbegriffen (20 TB) | CHF 0.– |
| TLS-Zertifikat | Let's Encrypt | CHF 0.– |
| **Total** | | **~CHF 29.–/Monat** |

> **Hinweis:** Hetzner ist ein in Deutschland (EU) ansässiger Anbieter mit Rechenzentren in
> Nürnberg, Falkenstein und Helsinki. Alternative Anbieter sind OVHcloud (FR) oder Infomaniak (CH).

---

## Voraussetzungen

- [ ] Linux-VM (Ubuntu 24.04 LTS) mit öffentlicher IP-Adresse und ≥ 600 GB Speicher
- [ ] DNS-Eintrag: `download.aiub.unibe.ch` → IP-Adresse der VM
- [ ] SSH-Zugang vom HPC-Cluster zur VM (Port 22)
- [ ] Port 80 und 443 in der Firewall der VM geöffnet

---

## Implementierungsschritte

### Schritt 1 — VM bereitstellen

```bash
# Ubuntu 24.04 LTS installieren (via Cloud-Init oder manuell)
# Systemaktualisierung
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx certbot python3-certbot-nginx rsync
```

### Schritt 2 — Nginx konfigurieren

```bash
# Konfigurationsdatei erstellen
sudo nano /etc/nginx/sites-available/download.aiub.unibe.ch
```

```nginx
server {
    listen 80;
    server_name download.aiub.unibe.ch;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name download.aiub.unibe.ch;

    root /srv/data;
    charset utf-8;

    # Sicherheitsheader
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;

    location / {
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
    }

    # Zugriffslogs
    access_log /var/log/nginx/download.aiub.unibe.ch.access.log;
    error_log  /var/log/nginx/download.aiub.unibe.ch.error.log;
}
```

```bash
sudo ln -s /etc/nginx/sites-available/download.aiub.unibe.ch \
           /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### Schritt 3 — HTTPS aktivieren

```bash
sudo certbot --nginx -d download.aiub.unibe.ch
```

### Schritt 4 — Datenspeicher vorbereiten

```bash
sudo mkdir -p /srv/data
sudo chown deploy:www-data /srv/data
sudo chmod 755 /srv/data
```

### Schritt 5 — SSH-Schlüssel und ersten Datentransfer einrichten

```bash
# Auf dem HPC-Cluster:
ssh-keygen -t ed25519 -f ~/.ssh/aiub_sync -C "aiub-hpc-sync"

# Öffentlichen Schlüssel auf die VM kopieren:
ssh-copy-id -i ~/.ssh/aiub_sync.pub deploy@download.aiub.unibe.ch

# Erster vollständiger Sync-Test (Trockenübung):
rsync -avz --dry-run /data/gnss/ deploy@download.aiub.unibe.ch:/srv/data/
```

### Schritt 6 — Automatischen Cronjob einrichten

```bash
# Auf dem HPC-Cluster:
crontab -e
# Eintrag hinzufügen:
0 2 * * * rsync -avz --delete /data/gnss/ deploy@download.aiub.unibe.ch:/srv/data/ >> /home/hpcuser/sync.log 2>&1
```

### Schritt 7 — Überwachung einrichten

- UptimeRobot-Konto unter <https://uptimerobot.com> erstellen (kostenlos)
- HTTP(S)-Monitor für `https://download.aiub.unibe.ch/` einrichten
- E-Mail-Benachrichtigung bei Ausfall konfigurieren

---

## Vorteile und Nachteile

| ✅ Vorteile | ❌ Nachteile |
|---|---|
| Niedrigste Betriebskomplexität nach Einrichtung | VM braucht gelegentliche OS-Updates |
| Native Verzeichnislisten (autoindex) | Einzelner Server (Single Point of Failure) |
| rsync ist einfach, zuverlässig und inkrementell | 600 GB Speicher auf VPS ~CHF 29/Monat |
| Let's Encrypt: keine Zertifikatskosten | Uni-Governance: Härtung, Rapid7, Snow Agent erforderlich |
| Keine Abhängigkeit von Cloud-Anbietern | Erfordert Linux-Sysadmin-Kompetenzen |
| Daten bleiben in der Schweiz / EU | Schwachstellen-Scanning muss regelmässig geprüft werden |

---

*Dieses Dokument beschreibt Lösung 1 von 4. Für einen vollständigen Vergleich aller Lösungen
siehe **README.md**.*
