---
toc: true
toc_label: "Table of Contents"
toc_icon: "search"
toc_sticky: true
title: "Méthodologie de Recon A à Z pour le Bug Bounty"
header:
  image: /assets/images/recon-a-z/recon_header.png
  teaser: /assets/images/recon-a-z/recon_teaser.png
---

**Introduction**

La reconnaissance (recon) est la phase la plus critique du bug bounty hunting. Une recon solide peut faire la différence entre trouver un P1 critique ou passer des heures à tourner en rond. Dans cet article, je vais détailler une méthodologie complète, de A à Z, que j'utilise sur chaque nouveau programme.

Cette méthodologie est structurée en 7 phases distinctes, chacune nourrissant la suivante.

---

## Phase 1 : Acquisition du Scope

Avant même de lancer le moindre scan, vous devez **comprendre exactement ce que vous avez le droit de tester**.

### 1.1 Lire la Policy du Programme

- Lisez la page du programme **en entier**. Pas juste les domaines.
- Notez les **exclusions explicites** (hors-scope).
- Vérifiez les types de vulnérabilités **acceptées et refusées**.
- Identifiez si le programme a des **règles spéciales** (ex: pas d'automated scanning, pas de nuclei).

### 1.2 Extraire les Assets

- **Wildcards** : `*.target.com` → tout ce qui match.
- **Domaines explicites** : `api.target.com`, `admin.target.com`.
- **Applications mobiles** : Si mentionnées, téléchargez l'APK.
- **GitHub/GitLab** : Certains programmes incluent leurs repos.
- **Propriétés tierces** : Parfois incluses (ex: `target.zendesk.com`).

### 1.3 Vérifier la Date de Mise à Jour du Scope

Un scope mis à jour récemment = nouvelles features = **potentiel goldmine**. Un scope stagnant depuis 2 ans = probablement déjà bien testé.

**Outils utiles :**
- `bbscope` pour extraire le scope automatiquement
- Page du programme sur HackerOne / Bugcrowd / Intigriti
- `chaos.projectdiscovery.io` pour les programmes publics

---

## Phase 2 : Subdomain Enumeration

C'est le cœur de la recon. Plus vous trouvez de sous-domaines, plus votre surface d'attaque est large.

### 2.1 Sources Passives (sans toucher la cible)

| Outil | Commande | Source |
|-------|----------|--------|
| **crt.sh** | `curl -s "https://crt.sh/?q=%25.target.com&output=json"` | Certificats TLS |
| **Subfinder** | `subfinder -d target.com -o subs.txt` | Multi-sources |
| **Assetfinder** | `assetfinder target.com >> subs.txt` | Multi-sources |
| **Amass (passive)** | `amass enum -passive -d target.com -o subs.txt` | Multi-sources |
| **Chaos** | `chaos -d target.com -o subs.txt` | ProjectDiscovery DB |
| **GitHub** | `github-subdomains.py -t TOKEN -d target.com` | Repos GitHub |
| **Shodan** | `shodan search hostname:target.com` | Moteur de recherche |
| **Censys** | `censys search "target.com"` | Certificats |
| **AlienVault OTX** | `curl -s "https://otx.alienvault.com/api/v1/indicators/domain/target.com/passive_dns"` | Passive DNS |
| **URLScan.io** | `curl -s "https://urlscan.io/api/v1/search/?q=domain:target.com"` | Scans publics |
| **Wayback Machine** | `curl -s "http://web.archive.org/cdx/search/cdx?url=*.target.com&output=text&fl=original&collapse=urlkey"` | Archives web |

### 2.2 Sources Actives (bruteforce DNS)

**⚠️ Attention :** Vérifiez que le programme autorise le bruteforce DNS.

```bash
# Avec une wordlist légère
puredns bruteforce best-dns-wordlist.txt target.com -r resolvers.txt -w active-subs.txt

# Avec ShuffleDNS (plus rapide)
shuffledns -d target.com -w wordlist.txt -r resolvers.txt -o active.txt
```

**Wordlists recommandées :**
- `jhaddix_all.txt` (la référence)
- `best-dns-wordlist.txt` de Assetnote
- `subdomains-top1million-5000.txt` de Daniel Miessler
- `commonspeak2-wordlists` pour les environnements cloud

### 2.3 Permutations et Alterations

Générez des variations à partir des sous-domaines déjà trouvés :

```bash
# Avec gotator
gotator -sub subs.txt -perm permutations.txt -depth 2 -numbers 10 -o permuted.txt

# Avec altdns
altdns -i subs.txt -o alt-output.txt -w words.txt
```

**Patterns de permutation à tester :**
- `{sub}-dev`, `{sub}-staging`, `{sub}-prod`
- `dev-{sub}`, `api-{sub}`, `admin-{sub}`
- `{sub}1`, `{sub}2`, `{sub}01`
- `{sub}.internal`, `{sub}.local`

### 2.4 Déduplication et Résolution

```bash
# Trier, dédupliquer et résoudre
sort -u all-subs.txt > unique-subs.txt
dnsx -l unique-subs.txt -r resolvers.txt -o resolved-subs.txt -a -aaaa -cname -ns
```

---

## Phase 3 : Live Host Discovery

Maintenant que vous avez vos sous-domaines, découvrez lesquels sont vivants et ce qu'ils hébergent.

### 3.1 HTTP Probing

```bash
# httpx - le couteau suisse
cat resolved-subs.txt | httpx -ports 80,443,8080,8443,8000,8081,3000,5000,9000,9443 -sc -title -server -tech-detect -o live-hosts.txt

# Vérifiez aussi les ports atypiques
cat resolved-subs.txt | httpx -ports 80,443,3000,5000,8000,8080,8443,9000,9090,9443,10443 -o live-all.txt
```

### 3.2 Catégorisation par Technologie

```bash
# Détection des technologies
httpx -l resolved-subs.txt -tech-detect -o tech-hosts.txt

# Filtrage par technologie spécifique
grep -i "nginx\|apache\|cloudflare\|aws\|gcp\|azure" tech-hosts.txt
grep -i "react\|angular\|vue\|django\|rails\|laravel\|spring" tech-hosts.txt
grep -i "wordpress\|joomla\|drupal\|magento\|shopify" tech-hosts.txt
```

### 3.3 Screenshots (optionnel mais utile)

```bash
# gowitness
gowitness file -f live-hosts.txt --destination screenshots/

# Aquatone
cat live-hosts.txt | aquatone -out aquatone-output/
```

**Ce qu'il faut chercher dans les screenshots :**
- Pages de login (surtout panel admin)
- Pages d'erreur exposant des versions
- Interfaces API (Swagger, GraphiQL)
- Pages de statut / monitoring
- Portails développeurs

### 3.4 Analyse des Codes de Statut

Priorisez vos cibles selon les codes HTTP :

| Code | Signification | Priorité |
|------|--------------|----------|
| **200** | OK - Accessible | ⭐⭐⭐ |
| **301/302** | Redirection - Intéressant | ⭐⭐ |
| **401** | Non autorisé - À tester (bypass) | ⭐⭐⭐⭐ |
| **403** | Interdit - Bypass à tenter | ⭐⭐⭐⭐⭐ |
| **404** | Non trouvé | ⭐ |
| **500** | Erreur serveur - À investiguer | ⭐⭐⭐ |

---

## Phase 4 : URL Discovery & Crawling

### 4.1 Wayback Machine & Archives

```bash
# gau (Get All URLs)
gau target.com | sort -u > gau-urls.txt
gau --subs target.com | sort -u > all-gau-urls.txt

# waybackurls
waybackurls target.com > wayback-urls.txt

# katana (crawler moderne de ProjectDiscovery)
katana -u https://target.com -d 3 -o katana-urls.txt

# ParamSpider
paramspider -d target.com
```

### 4.2 Filtrage Intelligent des URLs

```bash
# URLs avec paramètres (potentiel pour IDOR, XSS, SQLi, etc.)
grep -E '\?.*=' all-urls.txt > urls-with-params.txt

# Fichiers JavaScript
grep -E '\.js(\?|$)' all-urls.txt > js-files.txt

# Endpoints d'API
grep -E '/api/|/v[0-9]+/|/graphql|/rest/' all-urls.txt > api-endpoints.txt

# Fichiers de configuration potentiels
grep -E '\.json$|\.yaml$|\.yml$|\.xml$|\.config$|\.conf$' all-urls.txt > config-files.txt

# Endpoints d'admin
grep -Ei 'admin|dashboard|manager|panel|console|control' all-urls.txt > admin-endpoints.txt

# Endpoints d'upload
grep -Ei 'upload|file|import|export|attachment|avatar' all-urls.txt > upload-endpoints.txt

# Endpoints d'authentification
grep -Ei 'login|signup|register|auth|sso|oauth|saml|forgot|reset' all-urls.txt > auth-endpoints.txt
```

### 4.3 Découverte de Paramètres Cachés

Les paramètres cachés sont une **mine d'or** pour les IDORs, LFI, SSRF et contournements d'auth.

```bash
# Arjun
arjun -u https://target.com/endpoint -o arjun-output.txt

# x8
x8 -u https://target.com/endpoint -w param-wordlist.txt

# Param Miner (extension Burp)
# Utilisez-le pour le fuzzing passif et actif
```

**Wordlists de paramètres :**
- `params.txt` (SecLists)
- `burp-parameter-names.txt`
- `arjun-parameters.txt`

---

## Phase 5 : Analyse JavaScript

L'analyse JS est souvent négligée mais peut révéler des **secrets, endpoints cachés, et clés API**.

### 5.1 Téléchargement des Fichiers JS

```bash
# Télécharger tous les JS
mkdir js-files && cd js-files
cat ../js-files.txt | while read url; do
  curl -s "$url" -o "$(echo $url | md5sum | cut -d' ' -f1).js"
done
```

### 5.2 Extraction de Secrets

```bash
# SecretFinder (souvent intégré dans les pipelines)
# Via LinkFinder
python3 linkfinder.py -i target.js -o cli

# Mantra
python3 mantra.py -d target.com

# TruffleHog sur les repos JS
trufflehog filesystem js-files/
```

### 5.3 Recherche de Patterns Clés

```bash
# Endpoints d'API
grep -roE '"(/[a-zA-Z0-9_\-./]+)+"' *.js | sort -u

# Clés API potentielles
grep -roE '[a-zA-Z0-9_\-]{20,50}' *.js | sort -u

# URLs internes
grep -roE 'https?://[a-zA-Z0-9.\-]+' *.js | sort -u

# Secrets AWS/GCP
grep -roE 'AKIA[0-9A-Z]{16}' *.js
grep -roE 'AIza[0-9A-Za-z\-_]{35}' *.js
```

### 5.4 Découverte de Routes (SPA)

Pour les applications React, Vue, Angular :

```bash
# Analyser les fichiers JS pour les routes SPA
grep -roE 'path:\s*["'\''][/][a-zA-Z0-9_\-/]+["'\'']' *.js
grep -roE 'route:\s*["'\''][/][a-zA-Z0-9_\-/]+["'\'']' *.js
```

---

## Phase 6 : Analyse d'Infrastructure

### 6.1 DNS Records

```bash
# Tous les enregistrements DNS
dig target.com ANY
dig target.com A
dig target.com AAAA
dig target.com MX
dig target.com TXT
dig target.com NS
dig target.com CNAME
dig target.com SOA

# Zone transfer (rare mais à tester)
dig axfr target.com @ns1.target.com
```

### 6.2 WHOIS & ASN

```bash
# WHOIS
whois target.com

# ASN Lookup
amass intel -asn <ASN_NUMBER>
nmap --script whois-* target.com
```

### 6.3 Scan de Ports

```bash
# nmap rapide (top 1000 ports)
nmap -sV -sC -T4 -iL live-ips.txt -oA nmap-scan

# nmap complet (tous les ports)
nmap -p- -T4 -iL live-ips.txt -oA nmap-full

# RustScan (très rapide)
rustscan -a target.com -- -sV -sC -oA rustscan
```

### 6.4 Découverte Cloud

```bash
# S3Scanner pour les buckets AWS
s3scanner scan --bucket-file buckets-to-test.txt

# cloud_enum pour Azure/GCP
cloud_enum.py -k target -k target-dev -k target-prod

# CloudFlare origin IP bypass
# Cherchez les vieux enregistrements DNS, les leaks dans certs
```

### 6.5 Subdomain Takeover

```bash
# dnsReaper (meilleur signal)
python dnsreaper.py file --filename subs.txt

# subjack
subjack -w subs.txt -t 100 -timeout 30 -o takeover-results.txt -ssl

# Vérification manuelle des CNAME
dnsx -l subs.txt -cname -o cname-records.txt
```

**Services à vérifier pour takeover :**
- GitHub Pages (`*.github.io`)
- Amazon S3 (`*.s3.amazonaws.com`)
- Heroku (`*.herokuapp.com`)
- Shopify (`*.myshopify.com`)
- Azure (`*.azurewebsites.net`, `*.cloudapp.net`)
- Fastly (`*.fastly.net`)
- Zendesk (`*.zendesk.com`)

### 6.6 Vérification des CORS

```bash
# Testez chaque domaine pour des CORS mal configurées
curl -s -H "Origin: https://evil.com" https://target.com/api/test -I | grep -i "access-control"
```

---

## Phase 7 : Surveillance Continue (Continuous Monitoring)

La recon n'est **jamais terminée**. Les programmes évoluent constamment.

### 7.1 Nouveaux Sous-Domaines

```bash
# Ajoutez un cron job quotidien
0 7 * * * /path/to/recon-script.sh target.com

# Script de monitoring
subfinder -d target.com -o new-subs.txt
# Comparez avec la liste précédente
comm -13 old-subs.txt new-subs.txt > fresh-subs.txt
# Alerte si nouveaux sous-domaines
notify -data fresh-subs.txt -bulk
```

### 7.2 Changements dans les Fichiers JavaScript

```bash
# Surveillez les changements de contenu JS
# Script qui télécharge les JS connus et compare les hashes
while read url; do
  hash=$(curl -s "$url" | md5sum)
  old_hash=$(grep "$url" js-hashes.txt)
  if [ "$hash" != "$old_hash" ]; then
    echo "[CHANGED] $url" | notify
  fi
done < js-files.txt
```

### 7.3 GitHub Watch

```bash
# Surveillez les repos GitHub de l'organisation
# Nouveaux commits = potentielles nouvelles fonctionnalités = nouveaux bugs
# Utilisez gitdorks ou des alertes GitHub natives
```

### 7.4 Outils de Monitoring

| Outil | Utilité |
|-------|---------|
| **notify** (ProjectDiscovery) | Notifications Discord/Slack/Telegram |
| **Sitadel** | Monitoring de surface d'attaque |
| **reNgine** | Plateforme de recon automatisée |
| **ReconFTW** | Pipeline de recon complet |
| **ProjectDiscovery Cloud Platform** | Monitoring continu (payant) |

---

## Workflow Complet : Script Tout-en-Un

Voici un script bash simplifié qui enchaîne les phases principales :

```bash
#!/bin/bash
# recon-pipeline.sh - Pipeline de recon automatisé
TARGET=$1
OUTDIR="recon/$TARGET"
mkdir -p $OUTDIR

echo "[*] Phase 1: Subdomain Enumeration"
subfinder -d $TARGET -o $OUTDIR/subs-subfinder.txt
assetfinder --subs-only $TARGET > $OUTDIR/subs-assetfinder.txt
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" | jq -r '.[].name_value' 2>/dev/null | sed 's/\*\.//g' | sort -u > $OUTDIR/subs-crtsh.txt

echo "[*] Phase 2: Dedup & Resolve"
cat $OUTDIR/subs-*.txt | sort -u > $OUTDIR/all-subs.txt
dnsx -l $OUTDIR/all-subs.txt -r resolvers.txt -o $OUTDIR/resolved.txt

echo "[*] Phase 3: Live Host Discovery"
cat $OUTDIR/resolved.txt | httpx -sc -title -tech-detect -o $OUTDIR/live-hosts.txt

echo "[*] Phase 4: URL Discovery"
gau --subs $TARGET | sort -u > $OUTDIR/gau-urls.txt
waybackurls $TARGET | sort -u > $OUTDIR/wayback-urls.txt
katana -u "https://$TARGET" -d 3 -silent -o $OUTDIR/katana-urls.txt

echo "[*] Phase 5: JS Analysis"
cat $OUTDIR/*-urls.txt | grep -E '\.js(\?|$)' | sort -u > $OUTDIR/js-files.txt
cat $OUTDIR/js-files.txt | python3 linkfinder.py -i - -o cli 2>/dev/null > $OUTDIR/js-endpoints.txt

echo "[*] Done! Results in $OUTDIR/"
```

---

## Priorisation de la Surface d'Attaque

Une fois la recon terminée, voici comment **prioriser** ce que vous devez tester en premier :

### Priorité Très Élevée 🔴
- Endpoints `/api/` avec paramètres
- Pages d'authentification (login, register, SSO, OAuth)
- Endpoints exposant des UUIDs
- Pages admin/dashboard accessibles sans auth
- Sous-domaines de staging/dev (moins sécurisés que la prod)

### Priorité Élevée 🟠
- Webhooks et callbacks
- Endpoints d'upload de fichiers
- Fonctionnalités d'invitation d'utilisateurs
- APIs GraphQL
- Pages de reset de mot de passe

### Priorité Moyenne 🟡
- Fonctionnalités d'export (PDF, CSV, XML)
- Recherche (potentiel pour injection)
- Endpoints de téléchargement
- Pages de profil utilisateur
- Fonctionnalités de paiement/checkout

### Priorité Basse 🟢
- Pages statiques
- Blogs
- Pages marketing
- Documentation

---

## Erreurs Courantes à Éviter

1. **Scanner sans lire le scope** → Vous risquez de tester du hors-scope et de vous faire bannir.
2. **Ignorer les sous-domaines "inintéressants"** → Les apps staging/dev sont souvent moins sécurisées.
3. **Négliger l'analyse JavaScript** → Vous passez à côté d'endpoints cachés et de secrets.
4. **Utiliser une seule source de sous-domaines** → Chaque source a ses angles morts.
5. **Ne pas faire de suivi continu** → Les nouveaux assets = nouvelles opportunités.
6. **Oublier les ports non-standards** → Beaucoup de services tournent sur 8443, 8080, 3000, 5000, etc.
7. **Scanner trop agressivement** → Respectez les rate limits, utilisez des délais.
8. **Ne pas documenter sa recon** → Vous allez vous perdre si vous testez plusieurs programmes.

---

## Conclusion

La recon est un **processus itératif et continu**. Plus vous y passez de temps, plus votre surface d'attaque s'élargit et plus vous avez de chances de trouver des bugs que personne n'a encore découverts.

Rappelez-vous : **la plupart des hunters passent 70% de leur temps en recon et 30% en hunting**. Une recon solide est le meilleur investissement que vous puissiez faire.

**Ressources complémentaires :**
- [ReconFTW](https://github.com/six2dez/reconftw) - Pipeline automatisé complet
- [Bug Bounty Recon (Book)](https://recon.bookies.pro/) - Livre de référence par @ofjaaah
- [NahamSec Recon Playlist](https://www.youtube.com/playlist?list=PLKAaMVNxvLmAkqBkzAoOxcNTKNLMQzR-R) - Vidéos tutorielles
- [jhaddix methodology](https://www.youtube.com/watch?v=MIujHcGhCUc) - Conférence de référence

---

*Dernière mise à jour : 8 Juin 2026 — Happy hunting ! 🎯*
