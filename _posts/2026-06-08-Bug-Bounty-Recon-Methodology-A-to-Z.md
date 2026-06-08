---
toc: true
toc_label: "Table of Contents"
toc_icon: "search"
toc_sticky: true
title: "Bug Bounty Recon Methodology — From A to Z"
header:
  image: /assets/images/recon-a-z/recon_header.png
  teaser: /assets/images/recon-a-z/recon_teaser.png
---

**Introduction**

Reconnaissance is the single most important phase of bug bounty hunting. A solid recon pipeline can be the difference between finding a critical P1 and spending weeks spinning your wheels. In this article, I'm breaking down the complete A-to-Z recon methodology I use on every single program I join.

This methodology is structured into 7 distinct phases — each one feeding into the next like an assembly line of data.

---

## Phase 1: Scope Acquisition

Before you fire off a single scan, you need to know **exactly what you are allowed to touch**.

### 1.1 Read the Program Policy — All of It

- Read the entire program page. Not just the domain table.
- Note every **explicit exclusion**.
- Check which vulnerability types are **accepted and which are out of scope**.
- Identify special rules: no automated scanning, no nuclei, nuclei allowed but only with rate limiting, manual only, etc.

### 1.2 Extract Every Asset

- **Wildcards**: `*.target.com` — everything that resolves under it.
- **Explicit domains**: `api.target.com`, `admin.target.com`, `dashboard.target.com`.
- **Mobile apps**: If listed, download the APK or IPA. Mobile APIs often expose internal endpoints.
- **GitHub/GitLab orgs**: Some programs include their source repos.
- **Third-party properties**: `target.zendesk.com`, `target.statuspage.io` — sometimes in scope.

### 1.3 Check Scope Freshness

A scope updated last week = new features = **potential goldmine**. A scope that hasn't changed in 2 years = likely well-tested already.

**Useful tools:**
- `bbscope` — pull scope from HackerOne, Bugcrowd, Intigriti programmatically
- [chaos.projectdiscovery.io](https://chaos.projectdiscovery.io) — public program data
- The program page itself — always the source of truth

---

## Phase 2: Subdomain Enumeration

This is the heart of recon. Every new subdomain you discover expands your attack surface.

### 2.1 Passive Sources (No Touching the Target)

| Source | Command / URL | What You Get |
|--------|--------------|--------------|
| **crt.sh** | `curl -s "https://crt.sh/?q=%25.target.com&output=json" \| jq -r '.[].name_value'` | TLS certificate transparency |
| **Subfinder** | `subfinder -d target.com -o subs.txt` | Multi-source aggregate |
| **Assetfinder** | `assetfinder --subs-only target.com` | Multi-source aggregate |
| **Amass (passive)** | `amass enum -passive -d target.com` | Multi-source aggregate |
| **Chaos** | `chaos -d target.com` | ProjectDiscovery database |
| **GitHub** | `github-subdomains.py` | Org repos, gists, commits |
| **Shodan** | `shodan domain target.com` | Internet scan data |
| **Censys** | `censys search "target.com"` | Certificate corpus |
| **AlienVault OTX** | `curl -s "https://otx.alienvault.com/api/v1/indicators/domain/target.com/passive_dns"` | Passive DNS |
| **URLScan.io** | `curl -s "https://urlscan.io/api/v1/search/?q=domain:target.com"` | Public scan results |
| **Wayback Machine** | `curl -s "http://web.archive.org/cdx/search/cdx?url=*.target.com&output=text&fl=original&collapse=urlkey"` | Historical URLs |

### 2.2 Active DNS Bruteforcing

**⚠️ Warning:** Check that the program allows DNS bruteforcing before doing this.

```bash
# puredns — uses a trusted resolver list, avoids rate limits
puredns bruteforce best-dns-wordlist.txt target.com -r resolvers.txt -w active-subs.txt

# shuffledns — blazing fast, massdns wrapper
shuffledns -d target.com -w wordlist.txt -r resolvers.txt -o active.txt
```

**Recommended wordlists:**
- `jhaddix_all.txt` — the gold standard
- `best-dns-wordlist.txt` — Assetnote's curated list
- `subdomains-top1million-5000.txt` — Daniel Miessler
- `commonspeak2` — cloud-heavy environments

### 2.3 Permutation and Alteration Scanning

Generate variations from already-discovered subdomains:

```bash
# gotator — generates permutations
gotator -sub subs.txt -perm permutations.txt -depth 2 -numbers 10 -o permuted.txt

# altdns — alteration-based discovery
altdns -i subs.txt -o alt-output.txt -w words.txt
```

**Permutation patterns worth testing:**
- `{sub}-dev`, `{sub}-staging`, `{sub}-prod`
- `dev-{sub}`, `api-{sub}`, `admin-{sub}`
- `{sub}1`, `{sub}2`, `{sub}01`
- `{sub}.internal`, `{sub}.local`, `{sub}.corp`

### 2.4 Deduplicate and Resolve

```bash
# Sort, deduplicate, and resolve all at once
sort -u all-subs.txt > unique-subs.txt
dnsx -l unique-subs.txt -r resolvers.txt -o resolved-subs.txt -a -aaaa -cname -ns
```

---

## Phase 3: Live Host Discovery

Now that you have subdomains, figure out which ones are alive and what they're running.

### 3.1 HTTP Probing

```bash
# httpx — the Swiss Army knife of recon
cat resolved-subs.txt | httpx -ports 80,443,8080,8443,8000,8081,3000,5000,9000,9443 \
    -sc -title -server -tech-detect -o live-hosts.txt

# Don't forget non-standard ports
cat resolved-subs.txt | httpx -ports 80,443,3000,5000,8000,8080,8443,9000,9090,9443,10443 \
    -o live-all-ports.txt
```

### 3.2 Technology Fingerprinting

```bash
# Detect technologies across all live hosts
httpx -l resolved-subs.txt -tech-detect -o tech-hosts.txt

# Filter by specific tech — focus your hunting
grep -i "nginx\|apache\|cloudflare\|aws\|gcp\|azure" tech-hosts.txt   # Infrastructure
grep -i "react\|angular\|vue\|django\|rails\|laravel\|spring" tech-hosts.txt  # Frameworks
grep -i "wordpress\|joomla\|drupal\|magento\|shopify" tech-hosts.txt  # CMS/eCommerce
```

### 3.3 Screenshotting (Optional but Powerful)

```bash
# gowitness
gowitness file -f live-hosts.txt --destination screenshots/

# Aquatone
cat live-hosts.txt | aquatone -out aquatone-output/
```

**What to look for in screenshots:**
- Login pages (especially admin panels)
- Error pages exposing version numbers or stack traces
- API interfaces (Swagger UI, GraphiQL, /graphql playgrounds)
- Status / monitoring dashboards (Grafana, Kibana, Prometheus)
- Developer portals or staging environments left exposed

### 3.4 Status Code Triage

Prioritize your targets by HTTP response code:

| Status Code | Meaning | Priority |
|-------------|---------|----------|
| **200** | OK — Fully accessible | ⭐⭐⭐ |
| **301/302** | Redirect — Follow it, see where it leads | ⭐⭐ |
| **401** | Unauthorized — **Bypass target** | ⭐⭐⭐⭐ |
| **403** | Forbidden — **Bypass target** | ⭐⭐⭐⭐⭐ |
| **404** | Not Found | ⭐ |
| **500** | Internal Server Error — **Investigate** | ⭐⭐⭐ |

---

## Phase 4: URL Discovery and Crawling

### 4.1 Archive and Crawl for URLs

```bash
# gau (Get All URLs) — fetches from Wayback, URLScan, AlienVault OTX
gau target.com | sort -u > gau-urls.txt
gau --subs target.com | sort -u > all-gau-urls.txt

# waybackurls — Wayback Machine only
waybackurls target.com > wayback-urls.txt

# katana — modern headless crawler from ProjectDiscovery
katana -u https://target.com -d 3 -o katana-urls.txt

# ParamSpider — parameter-focused crawling
paramspider -d target.com
```

### 4.2 Smart URL Filtering

This is where you separate signal from noise:

```bash
# URLs with parameters — prime targets for IDOR, XSS, SQLi, Open Redirect
grep -E '\?.*=' all-urls.txt > urls-with-params.txt

# JavaScript files — hidden endpoints, secrets, API keys
grep -E '\.js(\?|$)' all-urls.txt > js-files.txt

# API endpoints
grep -E '/api/|/v[0-9]+/|/graphql|/rest/' all-urls.txt > api-endpoints.txt

# Configuration files — sometimes exposed
grep -E '\.json$|\.yaml$|\.yml$|\.xml$|\.config$|\.env$' all-urls.txt > config-files.txt

# Admin endpoints
grep -Ei 'admin|dashboard|manager|panel|console|control' all-urls.txt > admin-endpoints.txt

# File upload endpoints
grep -Ei 'upload|file|import|export|attachment|avatar' all-urls.txt > upload-endpoints.txt

# Authentication endpoints
grep -Ei 'login|signup|register|auth|sso|oauth|saml|forgot|reset' all-urls.txt > auth-endpoints.txt
```

### 4.3 Hidden Parameter Discovery

Hidden parameters are **gold** for IDORs, LFI, SSRF, and authorization bypasses.

```bash
# Arjun — the reference tool for parameter discovery
arjun -u https://target.com/endpoint -o arjun-output.txt

# x8 — fast, lightweight alternative
x8 -u https://target.com/endpoint -w param-wordlist.txt

# Param Miner — Burp Suite extension
# Use it for both passive and active parameter fuzzing inside Burp
```

**Parameter wordlists:**
- `params.txt` (SecLists)
- `burp-parameter-names.txt`
- `arjun-parameters.txt`

---

## Phase 5: JavaScript Analysis

JS analysis is the most underrated phase of recon. It regularly reveals **secrets, hidden API routes, internal hostnames, and hardcoded credentials**.

### 5.1 Download All JS Files

```bash
mkdir js-files && cd js-files
cat ../js-files.txt | while read url; do
  curl -s "$url" -o "$(echo $url | md5sum | cut -d' ' -f1).js"
done
```

### 5.2 Extract Secrets

```bash
# LinkFinder — endpoint and URL extraction
python3 linkfinder.py -i target.js -o cli

# SecretFinder — API key and secret detection
# Often integrated into recon pipelines

# Mantra — JS analysis framework
python3 mantra.py -d target.com

# TruffleHog — scan downloaded JS for verified secrets
trufflehog filesystem js-files/
```

### 5.3 Grep for High-Value Patterns

```bash
# Internal API paths — often reveal undocumented endpoints
grep -roE '"(/[a-zA-Z0-9_\-./]+)+"' *.js | sort -u

# Potential API keys — long alphanumeric strings
grep -roE '[a-zA-Z0-9_-]{20,60}' *.js | sort -u

# Internal hostnames and IPs
grep -roE 'https?://[a-zA-Z0-9.\-]+' *.js | sort -u

# AWS keys
grep -roE 'AKIA[0-9A-Z]{16}' *.js

# GCP keys
grep -roE 'AIza[0-9A-Za-z\-_]{35}' *.js

# GitHub tokens
grep -roE 'ghp_[a-zA-Z0-9]{36}' *.js

# Stripe keys
grep -roE 'sk_live_[0-9a-zA-Z]{24}' *.js
```

### 5.4 SPA Route Discovery

For React, Vue, and Angular apps, JS files contain the client-side routing table:

```bash
# React / Vue / Angular route definitions
grep -roE 'path:\s*["'\''][/][a-zA-Z0-9_\-/]+["'\'']' *.js
grep -roE 'route:\s*["'\''][/][a-zA-Z0-9_\-/]+["'\'']' *.js

# This often surfaces /admin, /internal, /debug panels hidden from the UI
```

---

## Phase 6: Infrastructure Analysis

### 6.1 DNS Record Enumeration

```bash
dig target.com ANY
dig target.com A
dig target.com AAAA
dig target.com MX
dig target.com TXT   # SPF, DMARC, verification tokens
dig target.com NS
dig target.com CNAME
dig target.com SOA

# Zone transfer — rare but always worth testing
dig axfr target.com @ns1.target.com
```

### 6.2 WHOIS and ASN Enumeration

```bash
# WHOIS lookup — find registrant org, IP ranges, related domains
whois target.com

# ASN-based discovery — find neighboring assets on the same infrastructure
amass intel -asn <ASN_NUMBER>
nmap --script whois-* target.com
```

### 6.3 Port Scanning

```bash
# Quick sweep — top 1000 ports
nmap -sV -sC -T4 -iL live-ips.txt -oA nmap-scan

# Full port scan — all 65535 ports
nmap -p- -T4 -iL live-ips.txt -oA nmap-full

# RustScan — very fast port discovery, pipe into nmap for service detection
rustscan -a target.com -- -sV -sC -oA rustscan
```

### 6.4 Cloud Asset Discovery

```bash
# S3Scanner — find and test AWS S3 buckets
s3scanner scan --bucket-file buckets-to-test.txt

# cloud_enum — enumerate Azure, GCP, and AWS resources
cloud_enum.py -k target -k target-dev -k target-prod -k target-staging

# GreyHatWarfare — search for public S3 buckets
# https://buckets.grayhatwarfare.com
```

**Bucket naming patterns to test:**
- `target`, `target-dev`, `target-prod`, `target-staging`
- `target-data`, `target-assets`, `target-backup`
- `target-media`, `target-uploads`, `target-logs`

### 6.5 Subdomain Takeover Check

```bash
# dnsReaper — best signal-to-noise ratio
python dnsreaper.py file --filename subs.txt

# subjack — fast Go-based checker
subjack -w subs.txt -t 100 -timeout 30 -o takeover-results.txt -ssl

# Manual CNAME inspection
dnsx -l subs.txt -cname -o cname-records.txt
```

**High-value takeover targets (dangling CNAMEs pointing to):**
- GitHub Pages (`*.github.io`)
- Amazon S3 (`*.s3.amazonaws.com`)
- Heroku (`*.herokuapp.com`)
- Shopify (`*.myshopify.com`)
- Azure (`*.azurewebsites.net`, `*.cloudapp.net`)
- Fastly (`*.fastly.net`)
- Zendesk (`*.zendesk.com`)
- Intercom (`*.intercom.io`)

### 6.6 CORS Misconfiguration Check

```bash
# Test each live domain for CORS misconfigurations
curl -s -H "Origin: https://evil.com" https://target.com/api/test -I | grep -i "access-control"

# If Access-Control-Allow-Origin: https://evil.com comes back — you have a finding.
```

---

## Phase 7: Continuous Monitoring

Recon is **never done**. Target organizations ship code constantly. Every deploy is an opportunity.

### 7.1 New Subdomain Alerts

```bash
# Daily cron job for subdomain monitoring
0 7 * * * /path/to/recon-script.sh target.com

# Diff-based alerting
subfinder -d target.com -o new-subs.txt
comm -13 old-subs.txt new-subs.txt > fresh-subs.txt
notify -data fresh-subs.txt -bulk   # Discord / Slack / Telegram
```

### 7.2 JavaScript Change Detection

```bash
# Monitor JS files for content changes — code diffs = new features = new bugs
while read url; do
  hash=$(curl -s "$url" | md5sum)
  old_hash=$(grep "$url" js-hashes.txt)
  if [ "$hash" != "$old_hash" ]; then
    echo "[CHANGED] $url — go check what changed" | notify
  fi
done < js-files.txt
```

### 7.3 GitHub Organization Watching

- Watch the target's GitHub org for new repos, commits, and PRs.
- New commits to staging branches often leak details about upcoming features.
- Use `gitdorks` or native GitHub watch/notification features.

### 7.4 Monitoring Tools

| Tool | Purpose |
|------|---------|
| **notify** (ProjectDiscovery) | Discord / Slack / Telegram notifications |
| **Sitadel** | Attack surface monitoring platform |
| **reNgine** | Automated recon framework with continuous monitoring |
| **ReconFTW** | Full pipeline automation |
| **ProjectDiscovery Cloud** | Paid continuous monitoring with dashboard |

---

## Full Pipeline: All-in-One Script

Here's a simplified bash script that chains the core phases together:

```bash
#!/bin/bash
# recon-pipeline.sh — Automated Recon Pipeline
TARGET=$1
OUTDIR="recon/$TARGET"
WORDLIST="${2:-/opt/wordlists/best-dns-wordlist.txt}"
RESOLVERS="${3:-/opt/resolvers.txt}"

mkdir -p $OUTDIR

echo "[*] Phase 1: Passive Subdomain Enumeration"
subfinder -d $TARGET -all -o $OUTDIR/subs-subfinder.txt 2>/dev/null
assetfinder --subs-only $TARGET > $OUTDIR/subs-assetfinder.txt
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" | jq -r '.[].name_value' 2>/dev/null | sed 's/\*\.//g' | sort -u > $OUTDIR/subs-crtsh.txt
curl -s "https://otx.alienvault.com/api/v1/indicators/domain/$TARGET/passive_dns" | jq -r '.passive_dns[].hostname' 2>/dev/null | sort -u > $OUTDIR/subs-otx.txt

echo "[*] Phase 2: Deduplicate and Resolve"
cat $OUTDIR/subs-*.txt | sort -u > $OUTDIR/all-subs.txt
dnsx -l $OUTDIR/all-subs.txt -r $RESOLVERS -o $OUTDIR/resolved.txt -a -cname

echo "[*] Phase 3: Live Host Discovery"
cat $OUTDIR/resolved.txt | httpx -ports 80,443,8080,8443,3000,5000,8000,9000 -sc -title -server -tech-detect -o $OUTDIR/live-hosts.txt
grep -E '\[2[0-9]{2}\]' $OUTDIR/live-hosts.txt > $OUTDIR/live-200.txt
grep -E '\[40[13]\]' $OUTDIR/live-hosts.txt > $OUTDIR/live-403-401.txt

echo "[*] Phase 4: URL Discovery"
gau --subs $TARGET 2>/dev/null | sort -u > $OUTDIR/gau-urls.txt
waybackurls $TARGET 2>/dev/null | sort -u > $OUTDIR/wayback-urls.txt
katana -u "https://$TARGET" -d 3 -silent -o $OUTDIR/katana-urls.txt 2>/dev/null

echo "[*] Phase 5: JS Extraction and Analysis"
cat $OUTDIR/*-urls.txt | grep -E '\.js(\?|$)' | sort -u > $OUTDIR/js-files.txt
mkdir -p $OUTDIR/js-downloads
cat $OUTDIR/js-files.txt | while read url; do
  curl -s -L "$url" -o "$OUTDIR/js-downloads/$(echo $url | md5sum | cut -d' ' -f1).js"
done
cat $OUTDIR/js-downloads/*.js 2>/dev/null | grep -oE '"(/[a-zA-Z0-9_\-./]+)+"' | sort -u > $OUTDIR/js-endpoints.txt

echo "[*] Done! Results in $OUTDIR/"
echo "[*] Live hosts: $(wc -l < $OUTDIR/live-hosts.txt)"
echo "[*] URLs collected: $(cat $OUTDIR/*-urls.txt 2>/dev/null | sort -u | wc -l)"
echo "[*] JS files: $(wc -l < $OUTDIR/js-files.txt)"
echo "[*] JS endpoints found: $(wc -l < $OUTDIR/js-endpoints.txt)"
```

---

## Attack Surface Prioritization

Once recon is done, here's how to **stack-rank** what you test first:

### Critical Priority 🔴
- `/api/` endpoints with parameters (especially UUIDs / numeric IDs)
- Authentication pages (login, register, SSO, OAuth flows)
- Endpoints exposing user-specific identifiers
- Admin/dashboard panels accessible without auth
- Staging/development subdomains (almost always less hardened than prod)

### High Priority 🟠
- Webhooks and callbacks
- File upload endpoints
- User invitation / onboarding flows
- GraphQL endpoints
- Password reset / account recovery flows

### Medium Priority 🟡
- Export functionality (PDF, CSV, XML — XXE and injection vectors)
- Search features (SQLi, XSS, command injection)
- File download endpoints (path traversal)
- User profile pages
- Payment / checkout flows

### Lower Priority 🟢
- Static pages
- Blogs and marketing sites
- Documentation portals
- Public landing pages

---

## Common Mistakes That Kill Recon

1. **Scanning before reading the scope** — You'll hit out-of-scope assets and get banned.
2. **Ignoring "boring" subdomains** — Staging and dev apps are the least secured.
3. **Skipping JavaScript analysis** — You're leaving hidden endpoints and secrets on the table.
4. **Using only one subdomain source** — Every source has blind spots. Layer them.
5. **Not doing continuous monitoring** — New assets = new opportunities. One-and-done recon is dead.
6. **Forgetting non-standard ports** — Services run on 8443, 8080, 3000, 5000, 9090, etc.
7. **Scanning too aggressively** — Respect rate limits, use delays, don't burn your IP.
8. **Not documenting your work** — You WILL get lost. Keep structured notes per program.

---

## The Recon Mindset

The best hunters spend **70% of their time on recon and 30% on actual hunting**. Why? Because when you thoroughly map the attack surface, vulnerabilities almost find themselves.

Think like a developer who shipped the feature at 5 PM on a Friday. They pushed an internal API endpoint to staging for testing, forgot to clean it up, and the endpoint exposes user data with no auth check. That endpoint exists somewhere in your recon data — you just need to find it.

Recon is an **iterative, compounding process**. Every new subdomain, every JS endpoint, every parameter you discover feeds back into the loop and uncovers more.

---

## Further Resources

- [ReconFTW](https://github.com/six2dez/reconftw) — Full automated recon pipeline
- [Bug Bounty Recon (Book)](https://recon.bookies.pro/) — Definitive guide by @ofjaaah
- [NahamSec Recon Playlist](https://www.youtube.com/playlist?list=PLKAaMVNxvLmAkqBkzAoOxcNTKNLMQzR-R) — Video tutorials
- [jhaddix — The Bug Hunter's Methodology](https://www.youtube.com/watch?v=MIujHcGhCUc) — Landmark talk
- [ProjectDiscovery Tools](https://projectdiscovery.io) — httpx, subfinder, nuclei, katana, dnsx, notify
- [SecLists](https://github.com/danielmiessler/SecLists) — Essential wordlists
- [Assetnote Wordlists](https://wordlists.assetnote.io/) — High-quality DNS and content wordlists

---

*Last updated: June 8, 2026 — Go hunt! 🎯*
