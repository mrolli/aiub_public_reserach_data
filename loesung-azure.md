# Lösung 2: Microsoft Azure (Blob Storage + Static Website + Front Door)

> **Status:** Vollständig verwaltete Cloud-Lösung ohne eigene Server

---

## Zusammenfassung

Diese Lösung nutzt **Azure Blob Storage** mit aktiviertem Static-Website-Hosting als zentralen
Datenspeicher. Die Dateien werden täglich per `azcopy` vom HPC-Cluster in die Azure Cloud
übertragen. Statische `index.html`-Dateien werden während der Synchronisation automatisch
erzeugt und ermöglichen die Verzeichnisnavigation im Browser. **Azure Front Door Standard**
stellt den benutzerdefinierten Domain-Namen `download.aiub.unibe.ch` mit HTTPS bereit.

> **Hinweis:** Der frühere Dienst *Azure CDN Standard from Microsoft (classic)* wird am
> 30. September 2027 eingestellt und nimmt seit August 2025 keine neuen Konfigurationen mehr
> an. Der Nachfolger ist **Azure Front Door Standard**, der in dieser Lösung verwendet wird.

Es werden **keine virtuellen Maschinen** betrieben – der gesamte Stack ist vollständig
von Microsoft verwaltet.

---

## Architekturübersicht

```text
┌─────────────────────────────────────────────────────────┐
│  HPC-Cluster (gesichertes Netz)                         │
│                                                         │
│  Schritt 1: azcopy sync /data/gnss/ → Azure Blob        │
│  Schritt 2: Python-Skript generiert index.html-Dateien  │
│  Schritt 3: azcopy kopiert index.html-Dateien hoch      │
└─────────────────────────────────────────────────────────┘
                          │
                          │  HTTPS (azcopy / Azure SDK)
                          ▼
         ┌────────────────────────────────────┐
         │  Azure Blob Storage                │
         │  Region: Switzerland North (Zürich)│
         │  Container: $web (Static Website)  │
         │  Tier: Hot                         │
         └──────────────┬─────────────────────┘
                        │ Origin
                        ▼
         ┌────────────────────────────────────┐
         │  Azure Front Door (Standard)       │
         │  Endpoint: download.aiub.unibe.ch  │
         │  HTTPS: Managed Certificate        │
         └──────────────┬─────────────────────┘
                        │
                        ▼
         ┌────────────────────────────────────┐
         │  Öffentliche Nutzer                │
         │  https://download.aiub.unibe.ch/   │
         └────────────────────────────────────┘
```

---

## Azure-Dienste im Überblick

| Azure-Dienst | Zweck | Tarif |
| --- | --- | --- |
| Azure Blob Storage | Datenspeicher, Static Website Hosting | Hot Tier |
| Azure Front Door | Benutzerdefinierte Domain, HTTPS, Caching, globale Verteilung | Standard |
| Azure DNS (optional) | DNS-Verwaltung in Azure | Standard |

---

## Verzeichnisauflistung: Lösung des zentralen Problems

Azure Blob Storage bietet **keine automatische Verzeichnisnavigation** wie ein klassischer
Webserver. Das Problem wird durch die Generierung von **statischen `index.html`-Dateien**
gelöst, die bei jeder Synchronisation neu erzeugt werden.

### Ablauf

```text
1. azcopy sync        → Datendateien hochladen
2. Python-Skript      → index.html pro Verzeichnis generieren
3. azcopy copy        → index.html-Dateien hochladen
```

### Python-Skript zur Index-Generierung

Das folgende Skript läuft auf dem HPC-Cluster nach dem Daten-Sync und erzeugt für jedes
Verzeichnis eine `index.html`, die das Aussehen einer klassischen Apache-Verzeichnisliste
nachahmt:

```python
#!/usr/bin/env python3
"""
generate_index.py — Erzeugt index.html-Dateien für Azure Blob Static Website
Aufruf: python3 generate_index.py /data/gnss /tmp/index_output
"""
import os
import sys
from pathlib import Path
from datetime import datetime

TEMPLATE_TOP = """<!DOCTYPE html>
<html><head>
  <meta charset="utf-8">
  <title>Index of {path}</title>
  <style>
    body {{ font-family: monospace; margin: 2em; }}
    table {{ border-collapse: collapse; width: 100%; }}
    th, td {{ text-align: left; padding: 4px 16px 4px 0; }}
    hr {{ border: 1px solid #aaa; }}
    a {{ text-decoration: none; color: #0066cc; }}
    a:hover {{ text-decoration: underline; }}
  </style>
</head><body>
<h1>Index of {path}</h1><hr>
<table>
<tr><th>Name</th><th>Last modified</th><th>Size</th></tr>
<tr><td><a href="../">../</a></td><td></td><td>-</td></tr>
"""

TEMPLATE_DIR  = '<tr><td><a href="{name}/">{name}/</a></td><td>{mtime}</td><td>-</td></tr>\n'
TEMPLATE_FILE = '<tr><td><a href="{name}">{name}</a></td><td>{mtime}</td><td>{size}</td></tr>\n'
TEMPLATE_BOT  = "</table><hr></body></html>\n"

def human_size(n):
    for unit in ("B", "KB", "MB", "GB"):
        if n < 1024:
            return f"{n:.0f} {unit}"
        n /= 1024
    return f"{n:.1f} TB"

def generate(src_root: Path, out_root: Path):
    for dirpath, dirnames, filenames in os.walk(src_root):
        dirnames.sort()
        filenames.sort()
        rel = Path(dirpath).relative_to(src_root)
        out_dir = out_root / rel
        out_dir.mkdir(parents=True, exist_ok=True)

        web_path = "/" + str(rel) + ("/" if str(rel) != "." else "")
        lines = [TEMPLATE_TOP.format(path=web_path)]

        for d in dirnames:
            mtime = datetime.fromtimestamp(
                (Path(dirpath) / d).stat().st_mtime
            ).strftime("%Y-%m-%d %H:%M")
            lines.append(TEMPLATE_DIR.format(name=d, mtime=mtime))

        for f in filenames:
            fpath = Path(dirpath) / f
            mtime = datetime.fromtimestamp(fpath.stat().st_mtime).strftime("%Y-%m-%d %H:%M")
            size  = human_size(fpath.stat().st_size)
            lines.append(TEMPLATE_FILE.format(name=f, mtime=mtime, size=size))

        lines.append(TEMPLATE_BOT)
        (out_dir / "index.html").write_text("".join(lines), encoding="utf-8")

if __name__ == "__main__":
    generate(Path(sys.argv[1]), Path(sys.argv[2]))
    print("Index-Generierung abgeschlossen.")
```

---

## Synchronisation vom HPC-Cluster

### Installation von azcopy auf dem HPC

```bash
# azcopy herunterladen (Linux, amd64)
curl -L https://aka.ms/downloadazcopy-v10-linux -o azcopy.tar.gz
tar -xf azcopy.tar.gz
sudo mv azcopy_*/azcopy /usr/local/bin/
azcopy --version
```

### Authentifizierung

Für die automatisierte Synchronisation empfiehlt sich ein **Azure Service Principal** (technischer
Benutzer) mit eingeschränkten Berechtigungen auf den Storage-Container:

```bash
# Einmalig: Login mit Service Principal (Credentials in .env-Datei)
export AZCOPY_SPA_APPLICATION_ID="<app-id>"
export AZCOPY_SPA_CLIENT_SECRET="<secret>"
export AZCOPY_TENANT_ID="<tenant-id>"
azcopy login --service-principal --application-id $AZCOPY_SPA_APPLICATION_ID \
             --tenant-id $AZCOPY_TENANT_ID
```

### Sync-Skript (täglich via Cronjob)

```bash
#!/bin/bash
# /home/hpcuser/aiub_azure_sync.sh

set -euo pipefail

DATA_DIR="/data/gnss"
INDEX_TMP="/tmp/aiub_index"
STORAGE_URL="https://<storageaccount>.blob.core.windows.net/\$web"
LOG="/home/hpcuser/azure_sync.log"

echo "$(date): Sync gestartet" >> "$LOG"

# Schritt 1: Datendateien synchronisieren
azcopy sync "$DATA_DIR" "$STORAGE_URL" \
    --recursive=true \
    --delete-destination=true \
    --log-level=INFO >> "$LOG" 2>&1

# Schritt 2: index.html-Dateien generieren
rm -rf "$INDEX_TMP"
python3 /home/hpcuser/generate_index.py "$DATA_DIR" "$INDEX_TMP"

# Schritt 3: index.html-Dateien hochladen
azcopy copy "$INDEX_TMP/*" "$STORAGE_URL" \
    --recursive=true \
    --content-type="text/html" >> "$LOG" 2>&1

echo "$(date): Sync abgeschlossen" >> "$LOG"
```

```bash
# Cronjob auf dem HPC:
0 2 * * * /home/hpcuser/aiub_azure_sync.sh
```

---

## Azure-Einrichtung: Schritt für Schritt

### Schritt 1 — Azure-Konto und Ressourcengruppe

```bash
# Azure CLI installieren und einloggen
az login
az group create --name rg-aiub-download --location switzerlandnorth
```

### Schritt 2 — Storage Account erstellen

```bash
az storage account create \
  --name staiubdownload \
  --resource-group rg-aiub-download \
  --location switzerlandnorth \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access true

# Static Website aktivieren
az storage blob service-properties update \
  --account-name staiubdownload \
  --static-website \
  --index-document index.html \
  --404-document 404.html
```

### Schritt 3 — Azure Front Door Standard erstellen

```bash
# Front Door-Profil erstellen (Standard-Tier)
az afd profile create \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --sku Standard_AzureFrontDoor

# Origin Group erstellen
az afd origin-group create \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --origin-group-name og-blob-static \
  --probe-request-type GET \
  --probe-protocol Https \
  --probe-interval-in-seconds 60 \
  --probe-path "/" \
  --sample-size 4 \
  --successful-samples-required 3

# Origin (Blob Static Website) hinzufügen
ORIGIN_HOST=$(az storage account show \
  --name staiubdownload \
  --resource-group rg-aiub-download \
  --query "primaryEndpoints.web" -o tsv | sed 's|https://||' | sed 's|/||')

az afd origin create \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --origin-group-name og-blob-static \
  --origin-name origin-blob \
  --host-name "$ORIGIN_HOST" \
  --origin-host-header "$ORIGIN_HOST" \
  --http-port 80 \
  --https-port 443 \
  --priority 1 \
  --weight 1000 \
  --enabled-state Enabled

# Endpoint erstellen
az afd endpoint create \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --endpoint-name ep-aiub-download \
  --enabled-state Enabled

# Route erstellen (leitet alle Anfragen an die Origin Group)
az afd route create \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --endpoint-name ep-aiub-download \
  --route-name route-default \
  --origin-group og-blob-static \
  --supported-protocols Https Http \
  --https-redirect Enabled \
  --patterns-to-match "/*" \
  --forwarding-protocol HttpsOnly \
  --link-to-default-domain Enabled \
  --enable-caching false
# Caching-Strategie: Standardmässig deaktiviert (siehe Abschnitt «Caching-Strategie»)
# Auf «true» wechseln, sobald die Forschungsgruppe bestätigt, dass Dateien unveränderlich sind
```

### Schritt 4 — Benutzerdefinierte Domain und HTTPS

```bash
# DNS konfigurieren:
# 1. TXT-Eintrag zur Domain-Validierung (wird von Azure vorgegeben)
#    _dnsauth.download.aiub.unibe.ch  TXT  <validierungscode>
# 2. CNAME-Eintrag:
#    download.aiub.unibe.ch  CNAME  <endpoint>.z01.azurefd.net
# (Beide Schritte erfolgen beim Hostmaster der Uni Bern)

# Custom Domain im Front Door-Profil registrieren
az afd custom-domain create \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --custom-domain-name download-aiub-unibe-ch \
  --host-name download.aiub.unibe.ch \
  --certificate-type ManagedCertificate \
  --minimum-tls-version TLS12

# Custom Domain mit der Route verknüpfen
az afd route update \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --endpoint-name ep-aiub-download \
  --route-name route-default \
  --custom-domains download-aiub-unibe-ch
```

---

## HTTPS und Zertifikatsverwaltung

Azure Front Door verwaltet das TLS-Zertifikat für die benutzerdefinierte Domain vollständig
automatisch. Die Domain-Validierung erfolgt einmalig per DNS-TXT-Eintrag. Danach wird das
Zertifikat ohne manuelles Eingreifen ausgestellt und erneuert.

---

## Caching-Strategie

Azure Front Door speichert Kopien der Dateien an weltweit verteilten Standorten (Points of
Presence), um Anfragen schneller zu beantworten. Da Forschungsdaten mehrmals täglich
aktualisiert werden können, muss die Caching-Strategie sorgfältig gewählt werden, damit
Nutzer **keine veralteten Daten** erhalten.

### Drei Optionen im Vergleich

#### Option A: Kein Caching (konservativ)

Alle Anfragen werden an Blob Storage durchgereicht. Nutzer erhalten **immer** die aktuellste
Version.

```bash
# Caching in der Route deaktivieren
az afd route update \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --endpoint-name ep-aiub-download \
  --route-name route-default \
  --enable-caching false
```

| Eigenschaft | Bewertung |
| --- | --- |
| Datenaktualität | ✅ Immer aktuell |
| Konfiguration | ✅ Einfach (eine Einstellung) |
| Latenz | ⚠️ Jede Anfrage geht nach Zürich (Blob Storage Origin) |
| Blob-Lesekosten | ⚠️ Leicht erhöht (kein Cache-Hit) — vernachlässigbar |
| Kostendifferenz | ~$1–2/Monat zusätzliche Blob-Lesekosten |

> **Empfohlen, wenn:** Dateien unter demselben Namen überschrieben werden oder die
> Forschungsgruppe explizit kein Caching wünscht.

#### Option B: Differenziertes Caching (empfohlen)

Verzeichnislisten (`index.html`) werden **nie** gecacht, Datendateien werden **lange** gecacht.
Dies erfordert **zwei Routen** im Front Door-Profil mit unterschiedlichen Caching-Regeln.

```bash
# Route 1: Verzeichnislisten — kein Caching
az afd route create \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --endpoint-name ep-aiub-download \
  --route-name route-index \
  --origin-group og-blob-static \
  --supported-protocols Https Http \
  --https-redirect Enabled \
  --patterns-to-match "*/index.html" \
  --forwarding-protocol HttpsOnly \
  --link-to-default-domain Enabled \
  --enable-caching false

# Route 2: Alles andere — Caching aktiviert
az afd route update \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --endpoint-name ep-aiub-download \
  --route-name route-default \
  --enable-caching true
# Standard-TTL: wird vom Origin bestimmt oder auf 7 Tage gesetzt
```

| Eigenschaft | Bewertung |
| --- | --- |
| Datenaktualität | ✅ Listings immer aktuell, Dateien aus Cache (unveränderlich) |
| Konfiguration | ⚠️ Zwei Routen nötig |
| Latenz | ✅ Dateidownloads profitieren vom Edge-Cache |
| Kosten | ✅ Weniger Blob-Lesezugriffe durch Cache-Hits |

> **Empfohlen, wenn:** Datendateien nach der Publikation nicht mehr verändert werden
> (neuer Inhalt = neuer Dateiname). Dies ist bei wissenschaftlichen Datenprodukten mit
> Zeitstempel im Dateinamen typischerweise der Fall.

#### Option C: Kurze TTL (Kompromiss)

Alle Inhalte werden kurz gecacht (z.B. 5 Minuten). Nutzer sehen maximal 5 Minuten
alte Daten.

```bash
# Caching mit kurzer TTL aktivieren
az afd route update \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --endpoint-name ep-aiub-download \
  --route-name route-default \
  --enable-caching true \
  --query-string-caching-behavior IgnoreQueryString

# Rule Set erstellen für kurze TTL
az afd rule-set create \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --rule-set-name CacheRules

az afd rule create \
  --profile-name fd-aiub-download \
  --resource-group rg-aiub-download \
  --rule-set-name CacheRules \
  --rule-name ShortTTL \
  --order 1 \
  --match-variable RequestUri \
  --operator Any \
  --action-name CacheExpiration \
  --cache-behavior OverrideAlways \
  --cache-duration 00:05:00
```

| Eigenschaft | Bewertung |
| --- | --- |
| Datenaktualität | ⚠️ Maximal 5 Minuten verzögert |
| Konfiguration | ⚠️ Rule Set nötig |
| Latenz | ✅ Die meisten Anfragen aus dem Cache bedient |
| Kosten | ✅ Weniger Blob-Lesezugriffe als ohne Caching |

> **Empfohlen, wenn:** Dateien gelegentlich überschrieben werden, aber eine kurze Verzögerung
> akzeptabel ist.

### Empfehlung

Bis die Forschungsgruppe bestätigt hat, ob Dateien in-place überschrieben werden (siehe
offene Fragen im Management Summary), empfehlen wir **Option A (kein Caching)** als sichere
Standardkonfiguration. Ein späterer Wechsel auf Option B oder C ist jederzeit ohne
Datenverlust oder Dienstunterbrechung möglich.

> **Hinweis:** Auch ohne Caching bleibt der Datenausgang von Blob Storage zu Front Door
> **innerhalb von Azure kostenlos**. Die Caching-Strategie hat keinen Einfluss auf die
> Egress-Kosten, sondern nur auf die Blob-Lesezugriffe (Unterschied: wenige Dollar pro Monat).

---

## Kostenschätzung (monatlich, geschätzt)

Basis: 500 GB Dateibestand, ~1 TB monatliche Downloads, Region Switzerland North

| Posten | Berechnung | Kosten (ca.) |
| --- | -- | --- |
| Blob Storage (Hot Tier, 500 GB) | 500 GB × $0.018/GB | ~$9.– |
| Blob-Schreibvorgänge (Sync, ~10k ops/Tag) | Vernachlässigbar | ~$1.– |
| Blob-Lesevorgänge (durch Front Door) | Vernachlässigbar | ~$1.– |
| Azure Front Door Standard Grundgebühr | Fixkosten pro Profil | **$35.–** |
| Datenausgang (Front Door → Internet, Europa) | 1'000 GB × $0.083/GB | ~$83.– |
| Datenausgang (Origin → Front Door, Azure-intern) | Kostenlos | $0.– |
| Anfragen (Front Door, ~10M requests) | 10M × $0.009/10k | ~$9.– |
| **Total** | | **~$138.–/Monat** |

> **Hinweis:** Im Vergleich zum eingestellten Azure CDN Classic (keine Grundgebühr, ~$96/Monat)
> ist Azure Front Door Standard ca. $40.–/Monat teurer, hauptsächlich durch die Grundgebühr
> von $35/Monat und die zusätzliche Abrechnung pro Anfrage.
>
> Bei geringerem Download-Aufkommen:
>
> - 250 GB/Monat: ~$66.–/Monat
> - 500 GB/Monat: ~$87.–/Monat
>
> **Wichtig:** Preise in USD, basierend auf publizierten Azure-Listenpreisen (Stand 2025/2026).
> Endgültige Preise über den [Azure-Preisrechner](https://azure.microsoft.com/en-us/pricing/calculator/)
> ermitteln.

---

## Vorteile und Nachteile

| ✅ Vorteile | ❌ Nachteile |
| --- | --- |
| Kein Server zu verwalten | ~$138.–/Monat laufende Kosten (bei 1 TB Egress) |
| Vollständig verwaltete Infrastruktur | Verzeichnislisten erfordern index.html-Generator |
| Automatische Skalierung | Verzeichnislisten erfordern index.html-Generator |
| Azure Front Door verbessert globale Download-Geschwindigkeit | Azure-Konto und Lernkurve erforderlich |
| Automatisches HTTPS-Zertifikat via Front Door | Egress-Kosten + Front Door Grundgebühr dominieren die Kosten |
| Hohe Verfügbarkeit (Azure SLA 99.99%) | Region Switzerland North teurer als West Europe |
| Keine Betriebssystem-Updates | Front Door Standard ist Nachfolger des günstigeren CDN Classic |

---

## Betrieb und Wartung

| Aufgabe | Häufigkeit | Aufwand |
| --- | --- | --- |
| Cronjob-Logfile prüfen | Wöchentlich | 5 min |
| Azure-Kosten im Portal prüfen | Monatlich | 5 min |
| azcopy-Version aktualisieren | Halbjährlich | 10 min |
| Service-Principal-Secret erneuern | Jährlich | 15 min |

---

*Dieses Dokument beschreibt Lösung 2 von 4. Für einen vollständigen Vergleich aller Lösungen
siehe **README.md**.*
