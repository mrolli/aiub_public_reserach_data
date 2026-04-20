# Lösung 4: Azure Static Web App + bestehender Switch S3-Bucket (Low-Cost-Variante)

> **Status:** Kostengünstigste Cloud-Lösung — setzt vorhandenen S3-Bucket bei Switch voraus

---

## Zusammenfassung

Diese Lösung nutzt den **bereits vorhandenen S3-Bucket bei Switch** (dem Schweizer
Hochschulnetzwerk) als Datenspeicher und ergänzt ihn mit einer **Azure Static Web App** als
Browsing-Portal unter `download.aiub.unibe.ch`. Eine kleine JavaScript-Anwendung (~50 KB)
generiert die Verzeichnislisten **dynamisch im Browser** des Nutzers, indem sie die
S3-ListObjects-API direkt aufruft.

**Kernentscheidung:** Die File-Downloads erfolgen **direkt vom Switch S3-Bucket** und nicht
über `download.aiub.unibe.ch`. Damit entfällt die Anforderung einer einheitlichen URL für
Browsing und Downloads — dafür sinken die Kosten auf nahezu null.

**Voraussetzungen:**

- Der Switch S3-Bucket existiert bereits und enthält die Daten
- Der Bucket ist **öffentlich zugänglich** (anonymous access) und hat **keine Egress-Kosten**
- **CORS** ist auf dem Bucket konfiguriert (damit der Browser die S3-API aufrufen kann)

---

## Architekturdiagramm

### Grundvariante (Browser-only)

```
┌─────────────────────────────────────────────────────────┐
│                   HPC-Cluster (AIUB)                    │
│                                                         │
│   rclone sync /data s3://switch-bucket                  │
└──────────────────────┬──────────────────────────────────┘
                       │ rclone / s3cmd
                       ▼
         ┌─────────────────────────────┐
         │   Switch S3-Bucket          │
         │   (öffentlich, kein Egress) │
         │                             │
         │   Dateien + Verzeichnisse   │
         └──────────┬──────────────────┘
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
┌──────────────────┐  ┌──────────────────────────────────┐
│  Azure Static    │  │  Direkter S3-Download             │
│  Web App         │  │  https://s3.switch.ch/bucket/...  │
│                  │  │                                    │
│  download.aiub.  │  │  (Nutzer-Browser oder wget/curl)  │
│  unibe.ch        │  │                                    │
│                  │  └──────────────────────────────────┘
│  → JS-App lädt   │
│    S3-Listings   │
│  → Links zeigen  │
│    auf S3-URLs   │
└──────────────────┘
```

### Erweiterte Variante (mit Azure Function API)

```
┌────────────────────────────────────────────────────────┐
│                   HPC-Cluster (AIUB)                   │
└──────────────────────┬─────────────────────────────────┘
                       │ rclone / s3cmd
                       ▼
         ┌─────────────────────────────┐
         │   Switch S3-Bucket          │
         │   (öffentlich, kein Egress) │
         └──────────┬──────────────────┘
                    │
          ┌─────────┴──────────┐
          │                    │
          ▼                    ▼
┌───────────────────────┐  ┌──────────────────────────┐
│  Azure Static Web App │  │  Direkter S3-Download    │
│  download.aiub.       │  │  https://s3.switch.ch/…  │
│  unibe.ch             │  └──────────────────────────┘
│                       │
│  /           → JS-App │  ← Browser-Nutzer
│  /api/list   → Func.  │  ← Skript-Nutzer (curl/wget)
│  /api/html   → Func.  │  ← Skript-Nutzer (HTML)
└───────────────────────┘
```

---

## Beteiligte Dienste

| Dienst | Funktion | Kosten |
| --- | --- | --- |
| **Switch S3-Bucket** | Datenspeicher (bereits vorhanden) | Bestehend, kein Egress |
| **Azure Static Web App** (Free-Plan) | Hosting der JS-Browsing-Anwendung | $0.–/Monat |
| **Azure Static Web App** (Standard-Plan) | Falls SLA, API-Backend oder >2 Custom Domains benötigt | $9.–/Monat |
| **Custom Domain + TLS** | `download.aiub.unibe.ch` | Inklusive (Free + Standard) |
| **Managed Azure Functions** (im Standard-Plan) | API-Endpunkt für skriptbasierten Zugriff | Im Standard-Plan enthalten |

---

## Funktionsweise

### Browsing (Verzeichnisnavigation im Browser)

1. Der Nutzer öffnet `https://download.aiub.unibe.ch/CODE/` im Browser.
2. Die Azure Static Web App liefert eine kleine JavaScript-Anwendung (~50 KB) aus.
3. Das JavaScript ruft die **S3 ListObjectsV2-API** des Switch-Buckets direkt auf:
   ```
   GET https://s3.switch.ch/bucket-name/?list-type=2&prefix=CODE/&delimiter=/
   ```
4. Die API gibt eine XML-Antwort mit den Dateien und Unterverzeichnissen zurück.
5. Das JavaScript rendert daraus eine **Apache-ähnliche Verzeichnisliste** im Browser.
6. Jeder Dateilink zeigt direkt auf die **S3-URL** der Datei.

### Dateidownload

Der Nutzer klickt auf einen Dateilink und wird direkt zum Switch S3-Bucket weitergeleitet:

```
https://s3.switch.ch/bucket-name/CODE/ORB/2024/COD0OPSFIN_20240101.SP3.gz
```

Der Download erfolgt **nicht** über `download.aiub.unibe.ch`, sondern direkt vom S3-Bucket.

### Synchronisation

Die Synchronisation vom HPC-Cluster zum Switch S3-Bucket erfolgt wie bisher (z.B. per
`rclone` oder `s3cmd`). **Es ist kein zusätzlicher Sync-Schritt nötig**, da die Static Web App
nur die kleine JS-Anwendung enthält, nicht die Forschungsdaten.

---

## CORS-Konfiguration auf dem Switch S3-Bucket

Damit die JavaScript-Anwendung die S3-API aufrufen kann, muss **CORS** (Cross-Origin Resource
Sharing) auf dem Switch S3-Bucket konfiguriert sein:

```xml
<CORSConfiguration>
  <CORSRule>
    <AllowedOrigin>https://download.aiub.unibe.ch</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedHeader>*</AllowedHeader>
    <MaxAgeSeconds>3600</MaxAgeSeconds>
  </CORSRule>
</CORSConfiguration>
```

Konfiguration über die S3-API:

```bash
# cors.xml erstellen (Inhalt wie oben)
s3cmd setcors cors.xml s3://aiub-data
```

---

## JavaScript-Anwendung (Skizze)

Die Kern-Logik der Browsing-Anwendung ist überschaubar:

```javascript
// Konfiguration
const S3_ENDPOINT = "https://s3.switch.ch";
const BUCKET_NAME = "aiub-data";

async function listDirectory(prefix) {
  const params = new URLSearchParams({
    "list-type": "2",
    "prefix": prefix,
    "delimiter": "/"
  });

  const response = await fetch(
    `${S3_ENDPOINT}/${BUCKET_NAME}?${params}`
  );
  const xml = await response.text();
  const parser = new DOMParser();
  const doc = parser.parseFromString(xml, "text/xml");

  // Unterverzeichnisse
  const prefixes = doc.querySelectorAll("CommonPrefixes > Prefix");
  // Dateien
  const contents = doc.querySelectorAll("Contents");

  renderListing(prefix, prefixes, contents);
}

function renderListing(prefix, dirs, files) {
  const table = document.getElementById("listing");
  table.innerHTML = "";

  // Elternverzeichnis-Link
  if (prefix) {
    const parent = prefix.replace(/[^/]+\/$/, "");
    addRow(table, "📁", `<a href="#${parent}">..</a>`, "–", "–");
  }

  // Verzeichnisse
  for (const dir of dirs) {
    const name = dir.textContent.replace(prefix, "").replace("/", "");
    addRow(table, "📁",
      `<a href="#${dir.textContent}">${name}/</a>`, "–", "–");
  }

  // Dateien (Link zeigt direkt auf S3)
  for (const file of files) {
    const key = file.querySelector("Key").textContent;
    const name = key.replace(prefix, "");
    const size = file.querySelector("Size").textContent;
    const date = file.querySelector("LastModified").textContent;
    const url = `${S3_ENDPOINT}/${BUCKET_NAME}/${key}`;
    addRow(table, "📄",
      `<a href="${url}">${name}</a>`, formatSize(size), date);
  }
}

// URL-Hash als Navigationspfad
window.addEventListener("hashchange", () => {
  listDirectory(location.hash.substring(1));
});
listDirectory(location.hash.substring(1) || "");
```

> **Hinweis:** Dies ist eine vereinfachte Darstellung. Eine produktionsreife Version benötigt
> Paginierung (S3 liefert maximal 1000 Objekte pro Aufruf), Fehlerbehandlung, einen
> Ladezustand und ggf. Sortierung.

---

## Variante: API-Endpunkt für skriptbasierten Zugriff (Azure Function)

### Problem

Die Grundvariante funktioniert ausschliesslich im Browser: `curl` oder `wget` auf
`https://download.aiub.unibe.ch/CODE/` liefert die leere HTML-Hülle der JS-Anwendung — **keine
Dateiliste**. Für die IGS-Community, deren Workflows auf skriptbasierten Batch-Downloads basieren,
ist das ein erhebliches Manko.

### Lösung: Managed Azure Function als API-Backend

Im **Standard-Plan** ($9/Monat) der Static Web App sind **managed Azure Functions** enthalten.
Damit lassen sich serverseitige API-Endpunkte bereitstellen, die die S3-API aufrufen und
die Ergebnisse als JSON oder HTML zurückgeben:

#### API-Endpunkt 1: JSON-Dateiliste

```
GET https://download.aiub.unibe.ch/api/list?prefix=CODE/
```

Antwort:

```json
{
  "prefix": "CODE/",
  "directories": ["CODE/ORB/", "CODE/CLK/", "CODE/ERP/"],
  "files": [
    {
      "name": "COD0OPSFIN_20240101.SP3.gz",
      "size": 142857,
      "lastModified": "2024-01-02T06:00:00Z",
      "url": "https://s3.switch.ch/aiub-data/CODE/COD0OPSFIN_20240101.SP3.gz"
    }
  ]
}
```

Nutzung in einem Skript:

```bash
# Alle Dateien eines Verzeichnisses herunterladen
curl -s "https://download.aiub.unibe.ch/api/list?prefix=CODE/" \
  | jq -r '.files[].url' \
  | xargs -n1 curl -O
```

#### API-Endpunkt 2: HTML-Listing (für einfaches wget)

```
GET https://download.aiub.unibe.ch/api/html?prefix=CODE/
```

Liefert klassisches HTML mit `<a href>`-Links, ähnlich dem Nginx-Autoindex:

```html
<html>
<body>
  <h1>Index of /CODE/</h1>
  <table>
    <tr><td><a href="/api/html?prefix=CODE/ORB/">ORB/</a></td><td>-</td></tr>
    <tr><td><a href="https://s3.switch.ch/aiub-data/CODE/file.SP3">file.SP3</a></td>
        <td>142 KB</td></tr>
  </table>
</body>
</html>
```

### Azure Function Implementierung (Skizze)

```javascript
// api/list/index.js (Node.js Azure Function)
const { S3Client, ListObjectsV2Command } = require("@aws-sdk/client-s3");

const s3 = new S3Client({
  endpoint: "https://s3.switch.ch",
  region: "ch-1",
  credentials: { accessKeyId: "anonymous", secretAccessKey: "" }
});

module.exports = async function (context, req) {
  const prefix = req.query.prefix || "";
  const command = new ListObjectsV2Command({
    Bucket: "aiub-data",
    Prefix: prefix,
    Delimiter: "/"
  });

  const result = await s3.send(command);

  context.res = {
    headers: { "Content-Type": "application/json" },
    body: {
      prefix: prefix,
      directories: (result.CommonPrefixes || []).map(p => p.Prefix),
      files: (result.Contents || []).map(obj => ({
        name: obj.Key.replace(prefix, ""),
        size: obj.Size,
        lastModified: obj.LastModified,
        url: `https://s3.switch.ch/aiub-data/${obj.Key}`
      }))
    }
  };
};
```

> **Hinweis:** Die Funktion muss S3-Paginierung handhaben (maximal 1000 Objekte pro Aufruf).
> Für Verzeichnisse mit mehr als 1000 Einträgen sind mehrere S3-API-Aufrufe nötig.

### Projektstruktur mit API

```
download-portal/
├── index.html                    # Haupt-HTML mit Listing-Container
├── app.js                        # S3-Listing-Logik (Browser)
├── style.css                     # Apache-ähnliches Styling
├── staticwebapp.config.json      # Routing-Konfiguration
└── api/
    ├── list/
    │   ├── index.js              # JSON-API-Endpunkt
    │   └── function.json         # Function-Konfiguration
    └── html/
        ├── index.js              # HTML-Listing-Endpunkt
        └── function.json         # Function-Konfiguration
```

Die Datei `staticwebapp.config.json` leitet Browser-Pfade auf `index.html` um, lässt aber
`/api/*`-Pfade an die Azure Functions durch:

```json
{
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/api/*", "/style.css", "/app.js"]
  }
}
```

---

## Einrichtung Schritt für Schritt

### Schritt 1: CORS auf dem Switch S3-Bucket konfigurieren

```bash
# cors.xml erstellen
cat > cors.xml << 'EOF'
<CORSConfiguration>
  <CORSRule>
    <AllowedOrigin>https://download.aiub.unibe.ch</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedHeader>*</AllowedHeader>
    <MaxAgeSeconds>3600</MaxAgeSeconds>
  </CORSRule>
</CORSConfiguration>
EOF

s3cmd setcors cors.xml s3://aiub-data
```

### Schritt 2: Azure Static Web App erstellen

```bash
# Ressourcengruppe erstellen
az group create \
  --name rg-aiub-download \
  --location switzerlandnorth

# Static Web App erstellen (Standard-Plan für API-Support)
az staticwebapp create \
  --name swa-aiub-download \
  --resource-group rg-aiub-download \
  --location westeurope \
  --sku Standard \
  --source https://github.com/aiub/download-portal \
  --branch main \
  --app-location "/" \
  --api-location "api" \
  --output-location "."
```

> **Hinweis:** Azure Static Web Apps ist nicht in allen Regionen verfügbar. `westeurope` ist
> die nächstgelegene verfügbare Region. Die Inhalte werden global via CDN verteilt — die
> Region bestimmt nur den Standort der Build-Pipeline und der Functions.

> **Free-Plan:** Reicht für die Grundvariante (ohne API). Für die API-Variante ist der
> Standard-Plan ($9/Monat) empfohlen, da er SLA und zuverlässigere Function-Ausführung bietet.

### Schritt 3: Custom Domain konfigurieren

```bash
az staticwebapp hostname set \
  --name swa-aiub-download \
  --resource-group rg-aiub-download \
  --hostname download.aiub.unibe.ch
```

DNS-Eintrag bei der Universität Bern:

```
download.aiub.unibe.ch.  CNAME  swa-aiub-download.azurestaticapps.net.
```

Das TLS-Zertifikat wird automatisch ausgestellt und erneuert.

### Schritt 4: Anwendung deployen

Die JavaScript-Anwendung (und optional die API-Functions) werden über ein Git-Repository
(z.B. GitHub) bereitgestellt. Bei jedem Push auf `main` wird automatisch gebaut und deployed.

---

## Kostenübersicht (monatlich)

| Posten | Grundvariante (Free) | API-Variante (Standard) |
| --- | --- | --- |
| Azure Static Web App | $0.– | $9.– |
| Enthaltene Bandbreite | 100 GB | 100 GB |
| Bandbreite (Überschuss) | nicht verfügbar | $0.20/GB |
| Managed Azure Functions | nicht enthalten | inklusive |
| Switch S3-Speicher | bestehend | bestehend |
| Switch S3-Egress | $0.– | $0.– |
| TLS-Zertifikat | inklusive | inklusive |
| **Monatliche Gesamtkosten** | **$0.–** | **$9.–** |

> **Warum ist die Bandbreite irrelevant?** Die Static Web App liefert nur die kleine
> JS-Anwendung (~50 KB) und die API-Antworten (wenige KB pro Aufruf) aus. Selbst bei
> 10 000 Seitenaufrufen pro Monat sind das nur ~500 MB. Die eigentlichen Dateidownloads
> fliessen direkt über Switch S3 und belasten die Static Web App nicht.

---

## Vorteile und Nachteile

| ✅ Vorteile | ❌ Nachteile |
| --- | --- |
| Nahezu kostenlos ($0–9/Monat) | Download-URLs zeigen auf Switch S3, nicht auf `download.aiub.unibe.ch` |
| Kein Server zu verwalten | Kein einheitlicher URL-Namespace |
| Kein index.html-Generator-Skript nötig | S3-API muss öffentlich und CORS-fähig sein |
| Kein zusätzlicher Sync-Schritt | JS-Anwendung muss entwickelt und gepflegt werden |
| Automatisches HTTPS-Zertifikat | Nutzer ohne JavaScript sehen keine Listings (Grundvariante) |
| Bestehender S3-Bucket wird weiterverwendet | Paginierung nötig bei >1000 Einträgen pro Verzeichnis |
| Globale Verteilung der App via CDN | Abhängigkeit von Verfügbarkeit des Switch S3-Dienstes |
| API-Variante ermöglicht skriptbasierten Zugriff | Cold-Start-Latenz bei Azure Functions (1–5 Sek. nach Leerlauf) |
| Minimaler Wartungsaufwand | Kleine JavaScript-Anwendung muss entwickelt und gepflegt werden |

---

## Ehrliche Einschätzung

Diese Lösung verdient eine differenzierte Bewertung, da sie auf einem **anderen Kompromiss**
basiert als die Lösungen 1–3.

### Was diese Lösung gut kann

- **Kosten:** Unschlagbar günstig. $0–9/Monat statt $29–138/Monat.
- **Betrieb:** Nahezu wartungsfrei — kein Server, kein Sync-Skript für die Web-Infrastruktur,
  kein index.html-Generator.
- **Vorhandenes nutzen:** Der Switch S3-Bucket existiert bereits. Diese Lösung baut auf dem
  auf, was vorhanden ist, statt die Daten ein weiteres Mal zu kopieren.

### Was diese Lösung nicht kann

- **Einheitliche URL:** Dateien liegen unter `s3.switch.ch/…`, nicht unter
  `download.aiub.unibe.ch/…`. Wer per Skript auf die Daten zugreift, muss S3-URLs verwenden.
  Die bisherigen Skripte gegen `ftp.aiub.unibe.ch` müssen ohnehin angepasst werden — ob auf
  `download.aiub.unibe.ch` oder `s3.switch.ch` ist gleichbedeutend aufwändig.

- **`wget -r` (rekursiver Download):** Auch mit der API-Variante funktioniert `wget -r` nicht
  über `download.aiub.unibe.ch`, da `wget` HTML-Seiten mit `<a href>`-Links auf derselben
  Domain erwartet. Die Dateilinks zeigen jedoch auf `s3.switch.ch`. Ein `wget -r` direkt
  auf den S3-Bucket ist möglich, sofern S3-Listing aktiviert ist — aber nicht über die
  benutzerdefinierte Domain.

- **Verzeichnisnavigation ohne JavaScript:** Die Grundvariante setzt JavaScript im Browser
  voraus. `curl https://download.aiub.unibe.ch/CODE/` liefert kein Listing. Die API-Variante
  löst dies teilweise — Nutzer können `/api/list` oder `/api/html` aufrufen, aber das ist
  ein nicht-standardmässiger Zugriffspfad.

### Vergleich der Zugriffsmuster

| Zugriffsmuster | Lösung 1 (Nginx) | Lösung 2/3 (Cloud) | Lösung 4 (Static App) |
| --- | :---: | :---: | :---: |
| Browser-Navigation | ✅ nativ | ✅ via index.html | ✅ via JS-App |
| `curl URL/dir/` → Listing | ✅ HTML | ✅ HTML | ❌ / ⚠️ nur via `/api/html` |
| `wget -r URL/dir/` | ✅ funktioniert | ✅ funktioniert | ❌ Domain-Wechsel |
| `wget URL/dir/file.sp3` | ✅ funktioniert | ✅ funktioniert | ⚠️ nur via S3-URL |
| Skriptbasierter Batch-Download | ✅ einfach | ✅ einfach | ⚠️ via JSON-API möglich |

### Für wen ist diese Lösung die richtige Wahl?

✅ **Geeignet**, wenn:

- Das Budget absolut minimal sein muss
- Die Nutzer primär über den **Browser** auf die Daten zugreifen
- Die IGS-Community bereit ist, S3-URLs in ihren Skripten zu verwenden
- Der Switch S3-Bucket langfristig stabil und verfügbar bleibt

❌ **Nicht geeignet**, wenn:

- Alle Downloads zwingend über `download.aiub.unibe.ch` laufen müssen
- Bestehende `wget -r`-basierte Workflows ohne Anpassung weiter funktionieren sollen
- Die Lösung vollständig unabhängig von JavaScript im Browser sein muss

### Fazit

Lösung 4 ist **keine Universallösung** wie Lösung 1–3, sondern ein **gezielter Kompromiss:**
maximale Kosteneinsparung gegen eingeschränkte Zugriffsflexibilität. Wenn die IGS-Community
hauptsächlich per Browser navigiert und einzelne Dateien herunterlädt, ist diese Lösung
hervorragend. Wenn automatisierte Batch-Downloads über eine einheitliche URL ein
Kernbedürfnis sind, bleiben Lösung 1 (Nginx) oder Lösung 3 (AWS CloudFront) die besseren
Alternativen.

---

## Betrieb und Wartung

| Aufgabe | Häufigkeit | Aufwand |
| --- | --- | --- |
| CORS-Konfiguration prüfen (bei Bucket-Änderungen) | Bei Bedarf | 10 min |
| JS-Anwendung aktualisieren (Bugfixes, S3-API-Änderungen) | Selten | 30 min |
| Azure-Portal/Kosten prüfen | Monatlich | 5 min |
| Switch S3-Bucket-Zugang testen | Monatlich | 5 min |
| Azure Function Logs prüfen (API-Variante) | Monatlich | 10 min |

---

*Dieses Dokument beschreibt Lösung 4 von 4. Für einen vollständigen Vergleich aller Lösungen
siehe **README.md**.*
