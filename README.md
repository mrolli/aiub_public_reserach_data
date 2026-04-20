# Management Summary: Öffentliche Bereitstellung von Forschungsdaten

## Astronomisches Institut der Universität Bern (AIUB)

---

## Ausgangslage

Das Astronomische Institut der Universität Bern (AIUB) produziert täglich Daten aus der
GNSS-Satellitenorbitanalyse auf einem gesicherten HPC-Cluster. Diese Daten sollen der
wissenschaftlichen Gemeinschaft und der Öffentlichkeit **kostenlos und ohne Authentifizierung**
zugänglich gemacht werden.

Der bestehende FTP-Server (`ftp.aiub.unibe.ch`) soll durch eine moderne, langfristig
wartbare Lösung ersetzt werden, die unter **<https://download.aiub.unibe.ch/>** erreichbar ist.

---

## Anforderungen

| Anforderung | Priorität |
|---|---|
| Öffentlicher Zugang ohne Authentifizierung | Zwingend |
| HTTPS-Zugang (Browsing und Dateidownload) | Zwingend |
| Verzeichnisnavigation im Browser | Zwingend |
| Tägliche automatische Synchronisation vom HPC | Zwingend |
| Domain: `download.aiub.unibe.ch` | Zwingend |
| Minimaler Betriebsaufwand (kein dediziertes IT-Team) | Wichtig |
| Hohe Verfügbarkeit / Ausfallsicherheit | Wünschenswert |

**Datenmenge:**

- Ca. 500 GB Gesamtgrösse
- Ca. 100 000 Dateien in ca. 5 000 Verzeichnissen
- Täglicher Zuwachs: einige hundert Megabyte
- Geschätzter monatlicher Download-Verkehr: ~1 TB (mittleres Nutzungsvolumen)

---

## Überblick der vier evaluierten Lösungen

### Lösung 1: On-Premises – Nginx auf Linux-VM

Ein schlanker Linux-Webserver (Nginx) auf einer virtuellen Maschine — entweder auf
Uni-Infrastruktur oder bei einem europäischen VPS-Anbieter (z.B. Hetzner, Infomaniak, OVH).
Die Synchronisation erfolgt täglich per `rsync` über SSH vom HPC-Cluster.

**Schlüsselmerkmal:** Nginx besitzt eine native Verzeichnislistenfunktion (`autoindex on`),
die ohne zusätzliche Skripte oder Workarounds eine identische Benutzererfahrung wie der
aktuelle FTP-Server erzeugt.

---

### Lösung 2: Microsoft Azure – Blob Storage + CDN

Die Daten werden in **Azure Blob Storage** (Region Switzerland North, Zürich) gespeichert.
Die Synchronisation erfolgt täglich per `azcopy`. Ein Python-Skript generiert statische
`index.html`-Dateien für die Verzeichnisnavigation. **Azure Front Door Standard** stellt die
benutzerdefinierte Domain mit automatisch verwaltetem HTTPS-Zertifikat bereit.

> **Hinweis:** Der frühere Dienst *Azure CDN Standard from Microsoft (classic)* wird per
> September 2027 eingestellt. Der Nachfolger **Azure Front Door Standard** beinhaltet eine
> Grundgebühr von $35/Monat.

**Schlüsselmerkmal:** Vollständig verwaltete Cloud-Infrastruktur ohne eigene Server.
Datenspeicherung in der Schweiz möglich (Region Zürich).

---

### Lösung 3: Amazon Web Services – S3 + CloudFront

Die Daten werden in einem **Amazon S3-Bucket** (Region eu-central-1, Frankfurt) gespeichert.
Die Synchronisation erfolgt täglich per `rclone`. Statische `index.html`-Dateien werden für
die Verzeichnisnavigation generiert. **Amazon CloudFront** übernimmt Custom Domain, HTTPS und
globale Verteilung. Der Datenausgang von S3 zu CloudFront ist **kostenlos**.

**Schlüsselmerkmal:** Bewährte, hochverfügbare Cloud-Infrastruktur. `rclone` ist ein
einheitliches Werkzeug, das auch für einen späteren Anbieterwechsel geeignet ist.

---

### Lösung 4: Azure Static Web App + bestehender Switch S3-Bucket (Low-Cost)

Die Daten verbleiben im **bereits vorhandenen S3-Bucket bei Switch** (Schweizer
Hochschulnetzwerk, kein Egress). Eine **Azure Static Web App** dient als Browsing-Portal
unter `download.aiub.unibe.ch`. Eine kleine JavaScript-Anwendung generiert Verzeichnislisten
dynamisch im Browser, indem sie die S3-API direkt aufruft. File-Downloads erfolgen direkt
vom S3-Bucket. Optional kann eine **Azure Function** als API-Backend skriptbasierten Zugriff
ermöglichen (JSON- oder HTML-Listings).

**Schlüsselmerkmal:** Nahezu kostenlos ($0–9/Monat), aber kein einheitlicher URL-Namespace —
Downloads laufen über die S3-URL, nicht über `download.aiub.unibe.ch`.

> ⚠️ **Wichtige Einschränkung:** Lösung 4 ist ein **gezielter Kompromiss** — maximale
> Kosteneinsparung gegen eingeschränkte Zugriffsflexibilität. Die klassischen Zugriffsmuster
> der IGS-Community (`wget -r`, `curl` auf Verzeichnis-URLs) funktionieren nicht über
> `download.aiub.unibe.ch`, sondern nur direkt gegen den S3-Bucket. Wer primär Browser-basiert
> arbeitet, profitiert; wer automatisierte Batch-Downloads über eine einheitliche URL benötigt,
> ist mit Lösung 1 oder 3 besser bedient. Eine ausführliche Einschätzung findet sich in
> `loesung-azure-staticwebapp.md` unter «Ehrliche Einschätzung».

---

## Kostenvergleich nach Download-Volumen

Die monatlichen Kosten hängen stark vom Ausgangsverkehr (Egress) ab. Die folgende Tabelle
zeigt drei Szenarien:

| Lösung | Monatliche Kosten, 250 GB/Monat | Monatliche Kosten, 500 GB/Monat | Monatliche Kosten, 1 TB/Monat |
| --- | --- | --- | --- |
| **Lösung 1 (On-Premises, Uni-VM)** | CHF 0.– | CHF 0.– | CHF 0.– |
| **Lösung 1 (On-Premises, VPS)** | ~CHF 29.– | ~CHF 29.– | ~CHF 29.– |
| **Lösung 2 (Azure Front Door Standard)** | ~$66.– | ~$87.– | ~$138.– |
| **Lösung 3 (AWS)** | ~$34.– | ~$56.– | ~$100.– |
| **Lösung 4 (Static Web App + Switch S3)** | $0–9.– | $0–9.– | $0–9.– |

> **Hauptkostentreiber in der Cloud:** Der Ausgangsverkehr (Egress) dominiert die monatlichen
> Kosten und steigt linear mit dem Download-Volumen. Die On-Premises-Lösung ist vom
> Transfervolumen unabhängig (Traffic bei VPS-Anbietern wie Hetzner im Tarif inbegriffen).
>
> **Empfehlung:** Falls FTP-Logdaten (`xferlog`) vom bestehenden Server verfügbar sind,
> lässt sich das tatsächliche monatliche Download-Volumen ermitteln und die Kosten präziser
> einschätzen.

---

## Vergleichsmatrix

| Kriterium | Lösung 1 (On-Premises) | Lösung 2 (Azure) | Lösung 3 (AWS) | Lösung 4 (Static App + S3) |
| --- | :---: | :---: | :---: | :---: |
| **Kosten (monatlich)** | ⭐⭐⭐ günstig | ⭐ teuer | ⭐ teuer | ⭐⭐⭐ nahezu gratis |
| **Betriebsaufwand** | ⭐⭐ mittel | ⭐⭐⭐ minimal | ⭐⭐⭐ minimal | ⭐⭐⭐ minimal |
| **Erforderliche Fachkenntnisse** | ⭐ hoch (Sysadmin, Härtung) | ⭐⭐ mittel (Cloud-Portal) | ⭐⭐ mittel (Cloud-Portal) | ⭐⭐ mittel (JS-Entwicklung) |
| **Governance / Compliance** | ⭐ aufwändig (Härtung, Rapid7, Snow) | ⭐⭐⭐ entfällt | ⭐⭐⭐ entfällt | ⭐⭐⭐ entfällt |
| **Verzeichnisnavigation** | ⭐⭐⭐ nativ | ⭐⭐ via Skript | ⭐⭐ via Skript | ⭐⭐ via JS im Browser |
| **Einheitliche URL** | ⭐⭐⭐ ja | ⭐⭐⭐ ja | ⭐⭐⭐ ja | ❌ nein (S3-URLs) |
| **Skriptbasierter Zugriff (wget -r)** | ⭐⭐⭐ nativ | ⭐⭐⭐ via index.html | ⭐⭐⭐ via index.html | ❌ / ⚠️ nur via API |
| **Datensouveränität** | ⭐⭐⭐ Uni/EU | ⭐⭐ Zürich (CH) | ⭐ Frankfurt (DE) | ⭐⭐⭐ Switch (CH) |
| **Ausfallsicherheit** | ⭐⭐ mittel | ⭐⭐⭐ hoch (SLA 99.99%) | ⭐⭐⭐ hoch (SLA 99.99%) | ⭐⭐ abhängig von Switch S3 |
| **Skalierbarkeit** | ⭐⭐ manuell | ⭐⭐⭐ automatisch | ⭐⭐⭐ automatisch | ⭐⭐⭐ automatisch |
| **Anbieterunabhängigkeit** | ⭐⭐⭐ hoch | ⭐ Azure-Bindung | ⭐⭐ mittel (rclone) | ⭐⭐ Azure + Switch |
| **Einrichtungsaufwand** | ⭐ hoch (inkl. Härtung) | ⭐⭐ mittel | ⭐⭐ mittel | ⭐⭐ mittel (JS-App nötig) |
| **Sync-Werkzeug** | rsync (Standard) | azcopy | rclone (universell) | rclone / s3cmd (bestehend) |

---

## Empfehlung

### Empfehlung: Lösung 1 – On-Premises Nginx (kostengünstigste Lösung)

**Vorausgesetzt, die erforderlichen Kompetenzen sind vorhanden**, ist Lösung 1 die
kostengünstigste Wahl:

- ✅ Die **Kosten sind minimal** — bei Bereitstellung durch die Universität sogar null
- ✅ **Kein Index-Generator-Skript nötig** — Nginx liefert Verzeichnislisten nativ
- ✅ **rsync** ist das bewährteste Werkzeug für inkrementelle Dateisyncs und auf allen
  HPC-Clustern verfügbar
- ✅ Die Lösung ist vollständig **unabhängig von Cloud-Anbietern und deren Preisänderungen**

> ⚠️ **Wichtiger Vorbehalt:** Der Betrieb einer Linux-VM auf der Universitätsinfrastruktur
> erfordert die Einhaltung des **IT-Governance-Rahmens und des Härtungsleitfadens** der
> Universität Bern. Dazu gehören Server-Härtung, Firewall-Konfiguration, die Installation
> vorgeschriebener Agenten (**Rapid7 InsightVM** für Schwachstellen-Scanning, **Snow Agent**
> für Software-Inventarisierung) sowie regelmässiges Patch-Management. Diese Aufgaben
> setzen **Linux-Systemadministrationskenntnisse** voraus, die über den Betrieb der
> eigentlichen Anwendung (Nginx, rsync) hinausgehen.
>
> Wenn diese Kompetenzen weder beim Forscher noch bei der IT-Abteilung verfügbar sind,
> ist eine **Cloud-Lösung (Lösung 2 oder 3) die bessere Wahl**, da dort keine
> Server-Härtung, kein OS-Patching und keine Agenten-Installation anfallen.

**Empfohlene Hosting-Strategie (falls Kompetenzen vorhanden):** Zunächst bei der IT der
Universität Bern anfragen, ob eine Linux-VM mit öffentlicher IP-Adresse und 600+ GB Speicher
bereitgestellt werden kann. Falls nicht, ist **Hetzner Cloud** (Deutschland/EU, Cloud-Volumen
ab ~CHF 29/Monat) eine günstige, einfach zu bedienende Alternative.

---

### Alternative: Lösung 2 oder 3 – Cloud (geringster Fachkenntnisbedarf)

Falls die Linux-Sysadmin-Kompetenzen und die Governance-Anforderungen eine Hürde darstellen,
bieten die Cloud-Lösungen wesentliche Vorteile:

- ✅ **Keine Server-Härtung** — kein Rapid7, kein Snow Agent, kein OS-Patching
- ✅ **Keine Firewall-Konfiguration** — die Infrastruktur ist vollständig verwaltet
- ✅ **Geringere Einstiegshürde** — Azure- oder AWS-Portal statt Linux-Kommandozeile
- ❌ **Höhere laufende Kosten** (~$34–138/Monat je nach Anbieter und Download-Volumen)
- ❌ **Verzeichnislisten** erfordern ein zusätzliches Python-Skript (index.html-Generator)

---

### Wann welche Lösung wählen?

| Situation | Empfehlung |
| --- | --- |
| Sysadmin-Kenntnisse vorhanden, Uni stellt VM bereit | **Lösung 1** (On-Premises, kostenlos) |
| Sysadmin-Kenntnisse vorhanden, keine Uni-VM | **Lösung 1** (Hetzner VPS, ~CHF 29/Monat) |
| Keine Sysadmin-Kenntnisse, Daten sollen in der Schweiz bleiben | **Lösung 2** (Azure, Switzerland North) |
| Keine Sysadmin-Kenntnisse, maximale Verfügbarkeit gewünscht | **Lösung 3** (AWS + CloudFront) |
| Budget für Cloud-Ausgaben (~$34–138/Monat) vorhanden | Lösung 2 oder 3 |
| Kosten minimieren bei Cloud-Lösung | **Lösung 3** (AWS, günstiger als Azure Front Door) |
| Kosten absolut minimieren, Switch S3 vorhanden | **Lösung 4** (Static Web App, $0–9/Monat) |
| Einheitliche URL nicht zwingend, Browser-Zugriff reicht | **Lösung 4** (Low-Cost-Variante) |
| Späterer Anbieterwechsel soll einfach sein | **Lösung 3** (rclone als einheitliches Werkzeug) |

---

## Gemeinsame Elemente aller Lösungen

Unabhängig von der gewählten Lösung gelten folgende Gemeinsamkeiten:

1. **Synchronisation** erfolgt mehrmals täglich manuell oder per Cronjob vom HPC-Cluster
2. **Nur geänderte Dateien** werden übertragen (inkrementeller Sync)
3. **HTTPS** mit automatisch verwalteten, kostenlosen TLS-Zertifikaten
4. **DNS-Eintrag** `download.aiub.unibe.ch` bei der Universität Bern konfigurieren
5. **Kein Authentifizierungs-Overhead** — vollständig öffentlicher Lesezugang
6. **Monitoring** über UptimeRobot (kostenloser Dienst) mit E-Mail-Benachrichtigung

---

## Offene Fragen an die Forschungsgruppe

Die folgenden Fragen haben direkten Einfluss auf die Architekturentscheidung und sollten
vor der Umsetzung geklärt werden.

### 1. Werden Dateien in-place überschrieben oder sind sie unveränderlich?

Werden bestehende Dateien (z.B. `COD0OPSRAP_20240420.SP3`) nachträglich aktualisiert oder
ersetzt? Oder erzeugt jede neue Berechnung eine **neue Datei** mit einem neuen Namen bzw.
Zeitstempel?

**Warum ist das wichtig?** Bei den Cloud-Lösungen (Lösung 2 und 3) wird ein Content Delivery
Network (CDN) eingesetzt. Dieses speichert Kopien der Dateien an verschiedenen Standorten
weltweit zwischen (Caching), um Downloads zu beschleunigen. Wenn eine Datei unter demselben
Namen mit neuem Inhalt überschrieben wird, sehen Nutzer unter Umständen für Stunden die
**alte Version** — bis der Cache abläuft.

| Antwort | Auswirkung |
| --- | --- |
| Dateien sind **unveränderlich** (neuer Inhalt = neuer Dateiname) | ✅ CDN-Caching kann aktiviert bleiben → schnellere Downloads, kein Risiko |
| Dateien werden **überschrieben** (gleicher Name, neuer Inhalt) | ⚠️ CDN-Caching muss deaktiviert oder sehr kurz eingestellt werden |

### 2. Wie häufig werden Daten synchronisiert?

Wie oft werden neue oder aktualisierte Produkte auf den Download-Server übertragen?

| Produkttyp | Sync-Häufigkeit |
| --- | --- |
| Rapid-Produkte | Alle __ Stunden? |
| Final-Produkte | Einmal täglich? |
| Andere | __ |

**Warum ist das wichtig?** Die Sync-Häufigkeit bestimmt, wie oft die Verzeichnislisten
(`index.html`) aktualisiert werden müssen. Bei Cloud-Lösungen beeinflusst sie auch die
Cache-Strategie: Verzeichnislisten sollten **nie** zwischengespeichert werden, wenn sie sich
mehrmals täglich ändern.

### 3. Wie hoch ist das tatsächliche Download-Volumen?

Das bisherige monatliche Download-Volumen ist ein entscheidender Kostenfaktor für Cloud-
Lösungen. Falls Logdaten vom bestehenden Server verfügbar sind, liesse sich das Volumen
präzise ermitteln:

```bash
# Auf dem bestehenden FTP-Server ausführen:
# ProFTPd-Transferlog analysieren (typischer Pfad)
awk '{sum += $8} END {print sum/1024/1024/1024 " GB"}' /var/log/proftpd/xferlog

# Oder: Apache/Nginx Access Log (falls HTTP-Zugriff geloggt wird)
awk '{sum += $10} END {print sum/1024/1024/1024 " GB"}' /var/log/nginx/access.log
```

| Geschätztes Volumen | Empfohlene Lösung |
| --- | --- |
| < 250 GB/Monat | Cloud-Lösungen werden erschwinglich (~$34–66/Monat) |
| 250–500 GB/Monat | Cloud möglich, On-Premises deutlich günstiger |
| > 1 TB/Monat | On-Premises stark empfohlen (Cloud: $100–138/Monat) |

### 4. Ist der Switch S3-Bucket eine langfristige Option?

Falls die Forschungsdaten bereits in einem **S3-Bucket bei Switch** liegen:

- Ist der Bucket **dauerhaft** verfügbar (nicht an ein befristetes Projekt gebunden)?
- Fallen **Egress-Kosten** an oder ist der Datenausgang kostenlos?
- Kann **CORS** konfiguriert werden (nötig für Lösung 4)?
- Kann der Bucket **öffentlich lesbar** konfiguriert werden?

Falls ja, wird **Lösung 4** (Azure Static Web App + Switch S3) relevant — die mit Abstand
kostengünstigste Variante ($0–9/Monat), sofern die Einschränkungen beim URL-Namespace
akzeptabel sind.

---

## Nächste Schritte

1. **Entscheidung treffen:** Welche Lösung entspricht dem Budget und den organisatorischen
   Rahmenbedingungen der Universität?

2. **IT kontaktieren (Lösung 1):** Anfrage bei der IT der Universität Bern bezüglich
   Linux-VM mit öffentlicher IP und ~600 GB Speicher

3. **Cloud-Konto eröffnen (Lösung 2/3):** Azure- oder AWS-Konto erstellen und Budgetlimits
   konfigurieren (wichtig: Cost Alerts einrichten, um unerwartete Kosten zu vermeiden)

4. **Pilotbetrieb:** Neue Lösung parallel zum bestehenden FTP-Server betreiben und
   testen, bevor der DNS-Eintrag (`download.aiub.unibe.ch`) umgeschaltet wird

5. **Migration:** DNS-Umschaltung nach erfolgreichem Test; anschliessend alten FTP-Server
   dekommissionieren

---

## Dokumentenübersicht

| Dokument | Inhalt |
| --- | --- |
| `loesung-onpremises.md` | Detailbeschreibung Lösung 1: Nginx auf Linux-VM |
| `loesung-azure.md` | Detailbeschreibung Lösung 2: Azure Blob Storage + Front Door |
| `loesung-aws.md` | Detailbeschreibung Lösung 3: AWS S3 + CloudFront |
| `loesung-azure-staticwebapp.md` | Detailbeschreibung Lösung 4: Azure Static Web App + Switch S3 |
| `README.md` | Dieses Dokument: Übersicht und Vergleich |

---

*Erstellt: April 2026 | Universität Bern*
