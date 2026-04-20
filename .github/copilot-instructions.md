# Agent Memo — AIUB Public Research Data Distribution

## Project: Replace ftp.aiub.unibe.ch with download.aiub.unibe.ch

---

## What this project is about

The Astronomical Institute of the University of Bern (AIUB) needs to replace an
aging FTP-server with a modern public data distribution system. The researcher
operates largely without IT support, so simplicity and low maintenance overhead
are top priorities.

**Working directory:** `/Users/mrolli/Developer/azure/public_research_data/`

**Deliverables (all German, all already created):**

- `README.md` — executive overview, comparison, recommendation
- `loesung-onpremises.md` — Solution 1: Nginx on Linux VM
- `loesung-azure.md` — Solution 2: Azure Blob Storage + CDN
- `loesung-aws.md` — Solution 3: AWS S3 + CloudFront

---

## Key facts gathered in dialogue

### The data

- ~500 GB total, ~100 000 files, ~5 000 directories
- Text files (GNSS/satellite orbit products in various geodetic formats)
- Daily new files produced on a secured HPC cluster
- Structure visible at: <http://ftp.aiub.unibe.ch> (FTP with Apache-style HTTP listing)
- Top-level dirs: CODE, REPRO, igsdata, mgex, slr, BSWUSER52, BSWUSER54, etc.
- Products referenced by DOI (Zenodo), used by IGS community worldwide

### The constraints

- **No authentication** — fully public read access, forever
- **HTTP/HTTPS required** — directory browsing like the current server
- **Minimal IT overhead** — "it's mostly me (the researcher)"
- **No cloud subscription yet** — open to either Azure or AWS, or on-prem
- **No hard data sovereignty requirement** — public data, probably fine anywhere
- **HPC connectivity:** can push via rsync/SSH AND can use cloud CLIs (azcopy, rclone)
- **Traffic estimate:** moderate — hundreds of users, regular automated pulls (IGS scripts)
  → modelled as ~1 TB/month download volume in cost calculations

### The central technical problem

Neither Azure Blob Storage nor AWS S3 have a built-in directory listing / autoindex feature.
**Agreed solution:** generate static `index.html` files as part of the daily sync script.
The Python script `generate_index.py` is documented in full in `loesung-azure.md` and
referenced from `loesung-aws.md`. It walks the local data tree and generates Apache-style
HTML listings for every directory.

### Pricing data used (as of 2025/2026)

| Provider | Storage/GB/month | Egress/GB |
|---|---|---|
| Azure (Switzerland North, Hot) | $0.018 | $0.081 (via CDN) |
| AWS S3 (eu-central-1, Standard) | $0.023 | $0.085 (CloudFront, first 10 TB) |
| AWS: S3 → CloudFront | — | **$0.00 (free)** |
| Hetzner VPS volume (600 GB) | ~CHF 24 flat | included in traffic allowance |

---

## Architecture decisions made

| Decision | Choice | Reason |
|---|---|---|
| Azure region | Switzerland North (Zürich) | Closest to AIUB, potential data sovereignty |
| AWS region | eu-central-1 (Frankfurt) | Closest EU region to Switzerland |
| Directory listing strategy (cloud) | Static index.html generated at sync time | No VM needed, simpler than dynamic |
| Sync tool for Azure | azcopy | Native Azure tool, well-supported on Linux |
| Sync tool for AWS | rclone | More universal, works with both AWS and Azure |
| TLS for on-prem | Let's Encrypt (certbot) | Free, auto-renewing |
| TLS for Azure | Azure Front Door Standard managed certificate | Automatic (CDN Classic retired) |
| TLS for AWS | AWS Certificate Manager (ACM) | Free for CloudFront |
| Monitoring | UptimeRobot (free tier) | Zero cost, email alerts |

---

## Cost summary (monthly, ~1 TB/month downloads)

| Solution | Monthly cost |
|---|---|
| On-prem (Uni VM) | CHF 0.– |
| On-prem (Hetzner VPS) | ~CHF 29.– |
| Azure Blob + Front Door Standard | ~$138.– |
| AWS S3 + CloudFront | ~$100.– |

**Primary cloud cost driver:** egress fees + Front Door base fee ($35/month).
Azure CDN Classic was retired Aug 2025; Front Door Standard is the successor and ~$40/month
more expensive due to its $35 base fee and per-request charges.

---

## Current recommendation

**Lösung 1 (On-Premises Nginx)** is recommended:

- Ask uni IT for a Linux VM with public IP + 600 GB storage first (free)
- Fallback: Hetzner Cloud CX11 + 600 GB volume (~CHF 29/month)
- Nginx `autoindex on` — native directory listing, no extra scripts needed
- rsync over SSH — most reliable, universally available on HPC clusters

Cloud solutions are documented as valid alternatives if budget allows or if
university cannot provide a VM.

---

## What could be refined / extended

- **Cloudflare R2** was also evaluated informally (zero egress, ~$8–15/month, rclone-compatible)
  but NOT included in the final 3 documents per the user's request. Could be added as
  Lösung 4 if the user asks.
- **Cost model refinement:** current estimates assume 1 TB/month egress. If the researcher
  can provide actual download stats from the old FTP server logs, costs can be modelled more
  accurately.
- **Hetzner vs. Infomaniak vs. OVH:** only briefly mentioned. A dedicated Swiss-hosted VPS
  comparison (Infomaniak is Swiss, GDPR-compliant) could be added to `loesung-onpremises.md`.
- **rsync bandwidth throttling:** on HPC clusters, outbound bandwidth may be restricted.
  The sync script could add `--bwlimit=` to rsync if needed.
- **Retention / archival policy:** not discussed. Old data appears to be kept indefinitely
  (the FTP server has data back to 1991). No cleanup strategy was requested.
- **index.html styling:** the Python script uses a minimal Apache-style table. Could be
  enhanced with search, sorting, or the university's branding.
- **robots.txt / sitemap:** not mentioned. Could be added to guide search engine crawling.
- **Access logging and analytics:** not requested. Could add Nginx access log analysis or
  AWS/Azure access metrics if the researcher wants download statistics.
- **Incremental index.html regeneration:** current approach regenerates ALL index.html files
  on every sync. For 5000 directories this is fast (<1 min), but could be optimised to only
  regenerate changed directories if needed.

---

## How to continue in a new session

1. Read this file first for context
2. The four deliverable `.md` files are in `/Users/mrolli/Developer/azure/public_research_data/`
3. The researcher communicates in English; documents are written in German (Swiss-German spelling
   conventions preferred: "ss" instead of "ß", e.g. "Strasse" not "Straße")
4. The researcher is non-technical — avoid jargon in document prose; keep CLI examples in
   code blocks so they are clearly separate from explanatory text
5. All architecture documents follow the same structure:
   - Summary → Architecture diagram (ASCII) → Component table → Core problem solution →
     Sync script → Step-by-step setup → Cost table → Pros/cons table → Maintenance table

---

*Memo created: April 2026*
