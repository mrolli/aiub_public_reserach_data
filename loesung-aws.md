# Lösung 3: Amazon Web Services (S3 + CloudFront)

> **Status:** Vollständig verwaltete Cloud-Lösung ohne eigene Server

---

## Zusammenfassung

Diese Lösung nutzt **Amazon S3** als zentralen Datenspeicher mit aktiviertem Static-Website-
Hosting. Die Dateien werden täglich per `rclone` vom HPC-Cluster nach S3 übertragen.
Statische `index.html`-Dateien werden während der Synchronisation automatisch erzeugt und
ermöglichen die Verzeichnisnavigation im Browser. **Amazon CloudFront** (Content Delivery
Network) stellt die Dateien unter dem Domain-Namen `download.aiub.unibe.ch` mit HTTPS bereit.

Der Datenausgang von S3 zu CloudFront ist **kostenlos** – ein wesentlicher Vorteil der AWS-
Architektur. Es werden **keine virtuellen Maschinen** betrieben.

---

## Architekturübersicht

```
┌─────────────────────────────────────────────────────────┐
│  HPC-Cluster (gesichertes Netz)                         │
│                                                         │
│  Schritt 1: rclone sync /data/gnss/ → S3-Bucket        │
│  Schritt 2: Python-Skript generiert index.html-Dateien  │
│  Schritt 3: rclone copy index.html-Dateien → S3         │
└─────────────────────────────────────────────────────────┘
                          │
                          │  HTTPS (rclone / AWS CLI)
                          ▼
         ┌────────────────────────────────────┐
         │  Amazon S3                         │
         │  Region: eu-central-1 (Frankfurt)  │
         │  Bucket: aiub-download             │
         │  Static Website Hosting: aktiviert │
         │  Bucket Policy: öffentlich lesbar  │
         └──────────────┬─────────────────────┘
                        │ Origin (kostenlos)
                        ▼
         ┌────────────────────────────────────┐
         │  Amazon CloudFront                 │
         │  Distribution mit Custom Domain    │
         │  HTTPS: AWS Certificate Manager    │
         │  (kostenloses TLS-Zertifikat)      │
         └──────────────┬─────────────────────┘
                        │
                        ▼
         ┌────────────────────────────────────┐
         │  Öffentliche Nutzer                │
         │  https://download.aiub.unibe.ch/   │
         └────────────────────────────────────┘
```

---

## AWS-Dienste im Überblick

| AWS-Dienst | Zweck | Tarif |
|---|---|---|
| Amazon S3 | Datenspeicher, Static Website Hosting | Standard |
| Amazon CloudFront | Benutzerdefinierte Domain, HTTPS, CDN | Standard |
| AWS Certificate Manager (ACM) | TLS-Zertifikat für Custom Domain | Kostenlos |
| Amazon Route 53 (optional) | DNS-Verwaltung in AWS | $0.50/Monat |

---

## Verzeichnisauflistung: Lösung des zentralen Problems

Amazon S3 bietet **keine automatische Verzeichnisnavigation** (kein `autoindex`). Das Problem
wird identisch zur Azure-Lösung durch statische `index.html`-Dateien gelöst, die bei jeder
Synchronisation neu generiert werden.

### Ablauf

```
1. rclone sync     → Datendateien nach S3 übertragen
2. Python-Skript   → index.html pro Verzeichnis generieren
3. rclone copy     → index.html-Dateien nach S3 übertragen
```

Das Python-Skript zur Index-Generierung ist **identisch** mit dem in Lösung 2 (Azure)
beschriebenen Skript (`generate_index.py`) und kann für beide Cloud-Lösungen verwendet werden.
Bitte das entsprechende Skript aus `loesung-azure.md` entnehmen.

---

## Synchronisation vom HPC-Cluster

### Installation von rclone auf dem HPC

`rclone` ist ein vielseitiges Kommandozeilenwerkzeug, das mit S3, Azure Blob, Google Cloud und
vielen weiteren Speicherdiensten kompatibel ist. Es ist in den meisten Linux-Distributionen
verfügbar:

```bash
# Installation (als normaler Benutzer, ohne Root)
curl https://rclone.org/install.sh | sudo bash
rclone version
```

### rclone-Konfiguration für AWS S3

```bash
rclone config
# → New remote → Name: aiub-s3
# → Type: s3
# → Provider: AWS
# → Access Key ID: <IAM Access Key>
# → Secret Access Key: <IAM Secret Key>
# → Region: eu-central-1
# → Endpoint: (leer lassen)
# → Location constraint: eu-central-1
# → ACL: public-read
```

Die Konfiguration wird in `~/.config/rclone/rclone.conf` gespeichert.
Für automatisierte Cronjobs empfiehlt sich die Ablage der Credentials als Umgebungsvariablen:

```bash
export AWS_ACCESS_KEY_ID="<access-key>"
export AWS_SECRET_ACCESS_KEY="<secret-key>"
```

### Sync-Skript (täglich via Cronjob)

```bash
#!/bin/bash
# /home/hpcuser/aiub_aws_sync.sh

set -euo pipefail

DATA_DIR="/data/gnss"
INDEX_TMP="/tmp/aiub_index"
S3_BUCKET="s3://aiub-download"
LOG="/home/hpcuser/aws_sync.log"

echo "$(date): Sync gestartet" >> "$LOG"

# Schritt 1: Datendateien synchronisieren (index.html ausschliessen)
rclone sync "$DATA_DIR" "aiub-s3:aiub-download" \
    --fast-list \
    --transfers=16 \
    --checkers=32 \
    --delete-after \
    --exclude="index.html" \
    --log-file="$LOG" \
    --log-level=INFO

# Schritt 2: index.html-Dateien generieren
rm -rf "$INDEX_TMP"
python3 /home/hpcuser/generate_index.py "$DATA_DIR" "$INDEX_TMP"

# Schritt 3: index.html-Dateien hochladen
rclone copy "$INDEX_TMP/" "aiub-s3:aiub-download" \
    --include="index.html" \
    --header-upload "Content-Type:text/html" \
    --log-file="$LOG" \
    --log-level=INFO

echo "$(date): Sync abgeschlossen" >> "$LOG"
```

```bash
# Cronjob auf dem HPC:
0 2 * * * /home/hpcuser/aiub_aws_sync.sh
```

### rclone-Vorteile gegenüber aws cli

- Unterstützt alle gängigen Cloud-Speicherdienste (S3, Azure, GCS, Nextcloud, …)
- `--fast-list` reduziert API-Aufrufe stark bei vielen Dateien (100 000+)
- Einfaches Wechseln des Anbieters (z.B. von AWS auf Azure) durch Konfigurationsänderung
- Parallele Übertragung (`--transfers=16`) für schnellere Syncs

---

## AWS-Einrichtung: Schritt für Schritt

### Schritt 1 — AWS-Konto und IAM-Benutzer

```bash
# AWS CLI installieren
pip install awscli
aws configure  # Access Key, Secret Key, Region: eu-central-1

# IAM-Richtlinie für S3-Zugriff erstellen (minimale Berechtigungen)
aws iam create-policy \
  --policy-name AiubS3SyncPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:PutObject","s3:DeleteObject","s3:ListBucket","s3:GetObject"],
      "Resource": ["arn:aws:s3:::aiub-download","arn:aws:s3:::aiub-download/*"]
    }]
  }'
```

### Schritt 2 — S3-Bucket erstellen und konfigurieren

```bash
# Bucket erstellen
aws s3api create-bucket \
  --bucket aiub-download \
  --region eu-central-1 \
  --create-bucket-configuration LocationConstraint=eu-central-1

# Öffentlichen Zugriff erlauben (Public Access Block deaktivieren)
aws s3api put-public-access-block \
  --bucket aiub-download \
  --public-access-block-configuration \
    "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Bucket Policy: öffentliche Lesezugriffe erlauben
aws s3api put-bucket-policy \
  --bucket aiub-download \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::aiub-download/*"
    }]
  }'

# Static Website Hosting aktivieren
aws s3 website s3://aiub-download/ \
  --index-document index.html \
  --error-document error.html
```

### Schritt 3 — TLS-Zertifikat über AWS Certificate Manager

```bash
# Zertifikat für die Custom Domain beantragen (MUSS in Region us-east-1 sein für CloudFront!)
aws acm request-certificate \
  --domain-name download.aiub.unibe.ch \
  --validation-method DNS \
  --region us-east-1
```

Das Zertifikat wird per DNS-Validierung bestätigt. ACM generiert einen CNAME-Eintrag,
der beim DNS-Anbieter der Universität eingetragen werden muss. Nach Validierung wird das
Zertifikat automatisch erneuert.

### Schritt 4 — CloudFront-Distribution erstellen

```bash
# S3 Static Website Origin URL ermitteln
ORIGIN="aiub-download.s3-website.eu-central-1.amazonaws.com"

# CloudFront-Distribution erstellen
# CachePolicyId: CachingDisabled — siehe Abschnitt «Caching-Strategie» für Alternativen
# Vollständige Konfiguration via AWS-Konsole empfohlen
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "aiub-2025",
    "Origins": {
      "Quantity": 1,
      "Items": [{
        "Id": "S3-aiub-download",
        "DomainName": "'"$ORIGIN"'",
        "CustomOriginConfig": {
          "HTTPPort": 80, "HTTPSPort": 443,
          "OriginProtocolPolicy": "http-only"
        }
      }]
    },
    "DefaultCacheBehavior": {
      "TargetOriginId": "S3-aiub-download",
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
      "Compress": true
    },
    "Aliases": {"Quantity": 1, "Items": ["download.aiub.unibe.ch"]},
    "ViewerCertificate": {
      "ACMCertificateArn": "<certificate-arn>",
      "SSLSupportMethod": "sni-only",
      "MinimumProtocolVersion": "TLSv1.2_2021"
    },
    "DefaultRootObject": "index.html",
    "Enabled": true,
    "Comment": "AIUB Public Research Data"
  }'
```

### Schritt 5 — DNS-Eintrag setzen

Beim DNS-Anbieter der Universität Bern folgenden Eintrag erstellen:

```
download.aiub.unibe.ch  CNAME  <distribution-id>.cloudfront.net.
```

---

## HTTPS und Zertifikatsverwaltung

AWS Certificate Manager (ACM) stellt das TLS-Zertifikat kostenlos bereit und erneuert es
vollautomatisch. Kein manueller Eingriff erforderlich.

---

## Caching-Strategie

CloudFront speichert Kopien der Dateien an weltweit verteilten Standorten (Edge Locations),
um Anfragen schneller zu beantworten. Da Forschungsdaten mehrmals täglich aktualisiert werden
können, muss die Caching-Strategie sorgfältig gewählt werden, damit Nutzer **keine veralteten
Daten** erhalten.

### Drei Optionen im Vergleich

#### Option A: Kein Caching (konservativ)

Alle Anfragen werden an S3 durchgereicht. Nutzer erhalten **immer** die aktuellste Version.

```bash
# Managed Policy "CachingDisabled" verwenden
"DefaultCacheBehavior": {
  "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
  ...
}
```

| Eigenschaft | Bewertung |
| --- | --- |
| Datenaktualität | ✅ Immer aktuell |
| Konfiguration | ✅ Einfach (eine Einstellung) |
| Latenz | ⚠️ Jede Anfrage geht nach Frankfurt (S3 Origin) |
| S3 GET-Kosten | ⚠️ Leicht erhöht (kein Cache-Hit, jede Anfrage = S3 GET) |
| Kostendifferenz | ~$1–3/Monat zusätzliche S3 GET-Kosten — vernachlässigbar |

> **Empfohlen, wenn:** Dateien unter demselben Namen überschrieben werden oder die
> Forschungsgruppe explizit kein Caching wünscht.

#### Option B: Differenziertes Caching (empfohlen)

Verzeichnislisten (`index.html`) werden **nie** gecacht, Datendateien werden **lange** gecacht.
Dies funktioniert optimal, wenn Datendateien nach der Publikation unveränderlich sind
(neuer Inhalt = neuer Dateiname).

```bash
# Zwei Cache Behaviors in der CloudFront-Distribution:

# 1. Verzeichnislisten: kein Caching
"CacheBehaviors": [{
  "PathPattern": "*/index.html",
  "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
  ...
}]

# 2. Alles andere (Default): CachingOptimized (TTL 24h)
"DefaultCacheBehavior": {
  "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
  ...
}
```

| Eigenschaft | Bewertung |
| --- | --- |
| Datenaktualität | ✅ Listings immer aktuell, Dateien aus Cache (unveränderlich) |
| Konfiguration | ⚠️ Zwei Cache Behaviors nötig |
| Latenz | ✅ Dateidownloads profitieren vom Edge-Cache |
| Kosten | ✅ Weniger S3 GET-Anfragen durch Cache-Hits |

> **Empfohlen, wenn:** Datendateien nach der Publikation nicht mehr verändert werden
> (neuer Inhalt = neuer Dateiname). Dies ist bei wissenschaftlichen Datenprodukten mit
> Zeitstempel im Dateinamen typischerweise der Fall.

#### Option C: Kurze TTL (Kompromiss)

Alle Inhalte werden kurz gecacht (z.B. 5 Minuten). Nutzer sehen maximal 5 Minuten
alte Daten.

```bash
# Eigene Cache Policy mit kurzer TTL erstellen
aws cloudfront create-cache-policy \
  --cache-policy-config '{
    "Name": "AIUB-ShortTTL",
    "DefaultTTL": 300,
    "MaxTTL": 300,
    "MinTTL": 0,
    "ParametersInCacheKeyAndForwardedToOrigin": {
      "EnableAcceptEncodingGzip": true,
      "EnableAcceptEncodingBrotli": true,
      "HeadersConfig": {"HeaderBehavior": "none"},
      "CookiesConfig": {"CookieBehavior": "none"},
      "QueryStringsConfig": {"QueryStringBehavior": "none"}
    }
  }'
```

| Eigenschaft | Bewertung |
| --- | --- |
| Datenaktualität | ⚠️ Maximal 5 Minuten verzögert |
| Konfiguration | ⚠️ Eigene Cache Policy nötig |
| Latenz | ✅ Die meisten Anfragen aus dem Cache bedient |
| Kosten | ✅ Weniger S3 GET-Anfragen als ohne Caching |

> **Empfohlen, wenn:** Dateien gelegentlich überschrieben werden, aber eine kurze Verzögerung
> akzeptabel ist.

### Empfehlung

Bis die Forschungsgruppe bestätigt hat, ob Dateien in-place überschrieben werden (siehe
offene Fragen im Management Summary), empfehlen wir **Option A (kein Caching)** als sichere
Standardkonfiguration. Ein späterer Wechsel auf Option B oder C ist jederzeit ohne
Datenverlust oder Dienstunterbrechung möglich.

> **Wichtig:** Unabhängig von der gewählten Option bleibt der Datenausgang von S3 zu
> CloudFront **kostenlos**. Die Caching-Strategie hat keinen Einfluss auf die Egress-Kosten,
> sondern nur auf die S3 GET-Anfragen (Unterschied: wenige Dollar pro Monat).

---

## Kostenschätzung (monatlich, geschätzt)

Basis: 500 GB Dateibestand, ~1 TB monatliche Downloads, Region eu-central-1 / Frankfurt

| Posten | Berechnung | Kosten (ca.) |
|---|---|---|
| S3 Standard Storage (500 GB) | 500 GB × $0.023/GB | ~$11.50 |
| S3 PUT/COPY-Anfragen (Sync) | ~300k Anfragen × $0.005/1000 | ~$1.50 |
| S3 GET-Anfragen | Ohne Caching: alle Anfragen treffen S3 | ~$2–4.– |
| Datenausgang S3 → CloudFront | **Kostenlos** | $0.– |
| CloudFront Datenausgang (Europa) | 1'000 GB × $0.085/GB | ~$85.– |
| AWS Certificate Manager (ACM) | Kostenlos für CloudFront | $0.– |
| Route 53 (optional) | Hosted Zone + Anfragen | ~$1.– |
| **Total** | | **~$100.–/Monat** |

> **Hinweis:** Der grösste Kostenblock ist der CloudFront-Ausgangsverkehr. Bei geringerem
> Download-Aufkommen (< 500 GB/Monat) sinken die Kosten auf ~$55.–/Monat.

> **Wichtig:** S3 → CloudFront-Datenausgang ist kostenlos (innerhalb derselben Region), was
> gegenüber einer direkten S3-Anbindung erhebliche Einsparungen bringt.

> Preise in USD, basierend auf publizierten AWS-Listenpreisen (Stand 2025/2026). Endgültige
> Preise über den [AWS Pricing Calculator](https://calculator.aws/) ermitteln.

---

## Vorteile und Nachteile

| ✅ Vorteile | ❌ Nachteile |
|---|---|
| Kein Server zu verwalten | ~$100.–/Monat laufende Kosten |
| Vollständig verwaltete Infrastruktur | Verzeichnislisten erfordern index.html-Generator |
| S3 → CloudFront-Datenausgang kostenlos | AWS-Konto und Lernkurve erforderlich |
| rclone: einfaches, flexibles Sync-Werkzeug | CloudFront-Konfiguration initial etwas komplex |
| Automatisches HTTPS via ACM (kostenlos) | IAM-Berechtigungen müssen gepflegt werden |
| Hervorragende globale Verfügbarkeit | |
| rclone unterstützt auch Azure/andere Anbieter | |
| Automatische Skalierung | |

---

## Betrieb und Wartung

| Aufgabe | Häufigkeit | Aufwand |
|---|---|---|
| Cronjob-Logfile prüfen | Wöchentlich | 5 min |
| AWS-Kosten im Cost Explorer prüfen | Monatlich | 5 min |
| rclone-Version aktualisieren | Halbjährlich | 5 min |
| IAM-Access-Key rotieren | Jährlich | 15 min |

---

## Empfehlung: rclone auch für Azure verwenden

Das Werkzeug `rclone` unterstützt sowohl AWS S3 als auch Azure Blob Storage und kann als
einheitliches Sync-Werkzeug für beide Cloud-Lösungen eingesetzt werden. Bei einem späteren
Wechsel zwischen Anbietern muss lediglich die rclone-Konfiguration angepasst werden, nicht
das gesamte Sync-Skript.

```bash
# rclone funktioniert für AWS S3:
rclone sync /data/gnss/ aiub-s3:aiub-download --fast-list

# …und für Azure Blob (identische Syntax):
rclone sync /data/gnss/ aiub-azure:$web --fast-list
```

---

*Dieses Dokument beschreibt Lösung 3 von 4. Für einen vollständigen Vergleich aller Lösungen
siehe **README.md**.*
