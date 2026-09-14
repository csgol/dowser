# Dowser

```
██████╗  ██████╗ ██╗    ██╗███████╗███████╗██████╗ 
██╔══██╗██╔═══██╗██║    ██║██╔════╝██╔════╝██╔══██╗
██║  ██║██║   ██║██║ █╗ ██║███████╗█████╗  ██████╔╝
██║  ██║██║   ██║██║███╗██║╚════██║██╔══╝  ██╔══██╗
██████╔╝╚██████╔╝╚███╔███╔╝███████║███████╗██║  ██║
╚═════╝  ╚═════╝  ╚══╝╚══╝ ╚══════╝╚══════╝╚═╝  ╚═╝
       \    /                                       
        \  /         
         \/
```

**Purpose:** Take a noisy list of external-facing IPs (e.g. from Censys) and produce a clean, in-scope, DNS-validated target set for an authorized engagement — by cross-checking the IPs against the organization's *own* DNS footprint rather than trusting a single source.

---

## The idea

A broad IP source (Censys) gives you *candidates*, but some get pulled in on loose matches — shared-hosting or CDN IPs where only one stray name grazed the target domain. To separate real from junk, you build an **independent** picture of what the org actually owns: harvest their real subdomains from certificate transparency, resolve those names to the IPs they currently point at, and keep the IPs that appear in **both** sets. The overlap — minus any out-of-scope ranges — is your high-confidence target list.

**Flow:** Censys IPs (candidates) → crt.sh subdomains + dnsx resolution (independent proof) → `comm` intersection → CIDR scope filter → validated list.

---

## Prerequisites (one-time setup)

Run on a Linux box (Kali or Ubuntu). Tools needed: `curl`, `jq`, `dnsx`, `grepcidr`, `whois`.

```bash
# curl, jq, whois, grepcidr come from apt
sudo apt update && sudo apt install -y curl jq whois grepcidr
```

### Installing dnsx

`dnsx` is ProjectDiscovery's bulk DNS resolver. **Do not `pip install dnsx`** — that is a different, unrelated tool with the same name.

**Preferred: Go install**
```bash
sudo apt install -y golang-go
go install github.com/projectdiscovery/dnsx/cmd/dnsx@latest
export PATH="$PATH:$HOME/go/bin"
echo 'export PATH="$PATH:$HOME/go/bin"' >> ~/.bashrc
```

**Fallback: prebuilt binary** (use this if the box is behind a proxy — see Troubleshooting)
1. Browser → `https://github.com/projectdiscovery/dnsx/releases`
2. Download the latest **`dnsx_<version>_linux_amd64.zip`** (the `linux_amd64` build for a normal 64-bit box — confirm with `uname -m` → `x86_64` = amd64).
3. Install it:
```bash
unzip dnsx_*_linux_amd64.zip
sudo mv dnsx /usr/local/bin/     # optional: makes it callable as plain `dnsx` from anywhere
dnsx -version                    # confirm: prints a ProjectDiscovery banner
```
> If you skip the `sudo mv`, run it as `./dnsx` from the folder it lives in (after `chmod +x dnsx`).

### Verify everything
```bash
jq --version && curl --version | head -1 && whois --version 2>/dev/null | head -1 && dnsx -version
```

---

## Inputs you need before starting

1. **Your candidate IP list** — the "huge list" of external IPs from Censys, saved as a text file.
2. **The target domain** — e.g. `company.org`. Use the registrable domain (for multi-level TLDs, the full name, e.g. `company.co.uk`).
3. **The written scope / rules of engagement** — which ranges are in scope, which are excluded. You'll need the excluded CIDRs for the filter step.

---

## The workflow

### Step 0 — Set up a working folder

```bash
mkdir -p ~/recon && cd ~/recon
```

All files below live here. Move your Censys IP export into this folder and name it `big_ip_list.txt`:
```bash
ls ~ ~/Downloads                              # find where your export landed
mv ~/Downloads/YOUR_EXPORT.txt ~/recon/big_ip_list.txt
```

### Step 1 — Clean the candidate IP list → `ips.txt`

The list must end up as **one bare IPv4 per line** — no ports, CIDRs, hostnames, or blank lines. This scrub extracts only IPv4 addresses, dedupes, and sorts:

```bash
grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' big_ip_list.txt | sort -u > ips.txt
wc -l ips.txt
```

> **IPv6 note:** this scrub drops IPv6 automatically. If v6 is in scope, check with `grep -c ':' big_ip_list.txt` and run a parallel track (see Appendix).

### Step 2 — Harvest the org's real subdomains → `org_names.txt`

Certificate transparency logs (via crt.sh) list every publicly issued cert and the hostnames it covers — a free, authoritative source of subdomains. Set the domain once as a variable so it can't get out of sync across the command:

```bash
DOMAIN=company.org
curl -s "https://crt.sh/?q=%25.$DOMAIN&output=json" \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' \
  | grep -E "\.$DOMAIN$" \
  | sort -u > org_names.txt
wc -l org_names.txt
```

What each stage does:
- `curl` — fetches all certs under the domain as JSON. `%25.` is the URL-encoded wildcard (`%` → `%25`), so it means "anything ending in .$DOMAIN".
- `jq -r '.[].name_value'` — pulls the hostname field from each record.
- `sed 's/\*\.//g'` — strips leading `*.` from wildcard entries so they're resolvable.
- `grep -E "\.$DOMAIN$"` — keeps only names that truly end in the target domain (certs sometimes list unrelated names).
- `sort -u` — dedupe.

> If this returns **0**: it's almost always (a) crt.sh being slow/rate-limited — rerun the `curl`; or (b) a domain mismatch — the `DOMAIN` variable fixes this since it's used in all three spots. Sanity-check the raw response with `curl -s "https://crt.sh/?q=%25.$DOMAIN&output=json" | head -c 500`.

> **Alternative source:** if you already have a names+IPs list from Censys, save it as `censys_ai.txt` and substitute:
> `grep -oE "[A-Za-z0-9._-]+\.$DOMAIN" censys_ai.txt | sort -u > org_names.txt`

### Step 3 — Resolve names to live IPs → `org_ips.txt`

`dnsx` does a live A-record lookup on every name — asking real DNS servers *right now*, which is what makes this a fresh, independent check rather than cached data:

```bash
dnsx -l org_names.txt -a -resp-only -silent | sort -u > org_ips.txt
wc -l org_ips.txt
```

- `-l` input list · `-a` IPv4 A-record lookup · `-resp-only` output IPs only · `-silent` no banner.
- Expect **fewer** IPs than names — many subdomains share one IP (load balancers/CDNs), and dead or internal-only names resolve to nothing and drop out.

> **Proxied/hardened box:** if DNS must egress through sanctioned resolvers, add `-r resolvers.txt` so lookups don't leak out uncontrolled.

### Step 4 — Filter out-of-scope ranges → `org_ips_filtered.txt`

Strip any excluded CIDRs. Use `grepcidr` — a text `grep` is **wrong** for this because ranges like `/21` aren't octet-aligned and won't match as a string. `-v` = exclude:

```bash
grepcidr -v "137.200.0.0/16,199.173.224.0/21" org_ips.txt > org_ips_filtered.txt
wc -l org_ips_filtered.txt
```

- Replace the CIDRs with your actual excluded ranges (comma-separated, no spaces).
- **Direction matters:** `-v` removes those ranges (use when they're out-of-scope/third-party). **Drop `-v`** to keep *only* those ranges (use when they're the org's own blocks you're zeroing in on).

> No `grepcidr` available? Manual fallback: `grep -E '^137\.200\.|^199\.173\.' org_ips.txt` to list suspects, then delete by hand in an editor. For `/16` cut everything matching the first two octets; for `199.173.224.0/21` cut only third octet **224–231**.

### Step 5 — Correlate against the candidate list → `confirmed.txt`

`comm -12` prints only lines present in **both** sorted files. This intersects your Censys candidates with the in-scope org-resolved IPs:

```bash
comm -12 <(sort -u ips.txt) <(sort -u org_ips_filtered.txt) > confirmed.txt
wc -l confirmed.txt
```

Also capture what the intersection *drops* — org IPs the org points a name at but that weren't in your Censys pull (potentially real assets Censys missed, not junk):

```bash
comm -23 <(sort -u org_ips_filtered.txt) <(sort -u ips.txt) > org_not_in_censys.txt
wc -l org_not_in_censys.txt
```

> `<( ... )` is process substitution — it sorts each file on the fly so `comm` gets two sorted inputs (which it requires, or it silently gives wrong results).

### Step 6 — Keep the name↔IP mapping (for notes and vhost targeting)

Bare IPs lose the "which subdomain is this" context. Re-resolve with names attached:

```bash
dnsx -l org_names.txt -a -resp -silent > resolved_with_names.txt   # lines like: app.company.org [1.2.3.4]
```

### Step 7 — Scope-confirm before scanning (the gate)

**This is a gate, not a formality.** Before pointing any scanner at the list, verify every IP against the written scope. The specific trap: some names resolve to **shared/SaaS/CDN infrastructure** — the IP is the org's *pointer*, but the box belongs to a third party you're not cleared to touch. Check ownership on anything suspect:

```bash
whois <ip> | grep -iE 'netname|orgname|descr'
```

- Org's name on the allocation → in-bounds.
- Amazon / Cloudflare / GoDaddy / a SaaS vendor → flag for a scope check **before** scanning; don't assume it's in scope just because a target-domain name points there.

Loop it over the leftover/unconfirmed pile to triage by owner:
```bash
while read ip; do echo "== $ip"; whois "$ip" | grep -iE 'netname|orgname|descr'; done < org_not_in_censys.txt
```

## Step 8 — Classify targets before scanning (CDN / cloud / owner)

Once you have a validated list (`confirmed.txt` or `org_ips_filtered.txt`), not every
IP is a box you should point a scanner at. Some resolve onto **CDN edges**
(Cloudflare, Akamai, Fastly), **cloud hosting** (Azure, AWS, GCP), or **SaaS**
(Microsoft 365, etc.). These need classifying first, because:

- A **CDN edge** is a shared cache in front of the real origin — scanning it tests the
  provider, not the org, and won't reveal the origin anyway.
- **SaaS** infrastructure (e.g. `*.protection.outlook.com`) belongs to the vendor, not
  the org — out of bounds regardless of engagement scope.
- **Cloud-hosted org apps** (e.g. the org's own Azure App Service) *are* their asset and
  may be scannable — but only if in scope **and** the provider's testing rules are met.

### Check 1 — Reverse DNS (PTR)

Quick, passive, but usually blank — PTR is optional and often unset. It's a **one-way**
signal: it can *confirm* a provider when a record exists, but a blank result rules
nothing out.

```bash
# full nslookup output per IP
while read ip; do echo "===== $ip ====="; nslookup "$ip"; echo; done < confirmed.txt

# compact: IP -> PTR name (dig is cleanest; needs dnsutils)
while read ip; do echo "$ip -> $(dig +short -x "$ip")"; done < confirmed.txt
```

Windows (PowerShell):

```powershell
Get-Content confirmed.txt | ForEach-Object { "===== $_ ====="; nslookup $_; "" }
```

### Check 2 — Owner / ASN (the reliable one)

Every routed IP belongs to a registered netblock, whether or not PTR is set. This is the
check that actually answers "CDN/cloud vs. org."

```bash
# Linux
while read ip; do echo "== $ip"; whois "$ip" | grep -iE 'orgname|org-name|netname|descr' | head -3; done < confirmed.txt
```

Windows has no built-in whois — use the browser: paste each IP into
`https://ipinfo.io/<ip>` or `https://bgp.he.net`. The owner and ASN are at the top.

### Reading the result

| Owner / name pattern | Meaning | Scan? |
|----------------------|---------|-------|
| Cloudflare, Akamai, Fastly, Imperva, Sucuri, Edgio | CDN edge | No — flag as third-party |
| `*.protection.outlook.com`, `autodiscover.*`, `*.mail.` | Microsoft 365 / SaaS | No — vendor infra |
| `*.cloudapp.azure.com`, `*.azurewebsites.net` | Org's Azure-hosted app | Maybe — if in scope + cloud rules met |
| `*.amazonaws.com`, `*.compute.` | AWS-hosted | Maybe — same conditions |
| Org's own name / ordinary hosting provider | Org infrastructure | Yes — if in scope |

---

## Understanding your output files

| File | Contains | Meaning |
|------|----------|---------|
| `big_ip_list.txt` | Raw Censys export | Untouched original / backup |
| `ips.txt` | Cleaned candidate IPs | Censys pile, one bare IP per line |
| `org_names.txt` | Real subdomains of the target | Independent "what they own" (from certs) |
| `org_ips.txt` | Live IPs those names resolve to | The org's real DNS footprint |
| `org_ips_filtered.txt` | Above, minus excluded ranges | In-scope org footprint |
| `confirmed.txt` | Candidate IPs ∩ in-scope org IPs | **Validated: in Censys AND org points a live name here AND in scope** |
| `org_not_in_censys.txt` | Org IPs not in the candidate list | Real org assets Censys missed — review, don't discard |
| `resolved_with_names.txt` | name [ip] pairs | Provenance / vhost targeting |

### `confirmed.txt` vs `org_ips_filtered.txt` — which is your target set?

`confirmed.txt` is a **subset** of `org_ips_filtered.txt` — the extra condition is "was it in Censys too?" The gap between them (`org_not_in_censys.txt`) is legitimate org infrastructure that Censys alone didn't surface.

- **Goal = "vet the Censys list"** → use `confirmed.txt`.
- **Goal = "map the org's real external footprint"** → `org_ips_filtered.txt` is usually the stronger set; intersecting with Censys would throw away real assets just because one source lacked them.

---

## What this method does and doesn't prove

- **Does:** confirm association via *live forward DNS* — a precise, defensible claim ("the org actively points a name at this IP, right now").
- **Doesn't:** exhaustively enumerate every org IP. An asset with no current DNS name, or whose name wasn't in the cert set, won't land in `confirmed.txt` even if it's genuinely theirs — that's what the ownership pass (Step 7) is for.

Represent it accordingly in write-ups. "Confirmed via forward DNS" is accurate; "this is all of their infrastructure" would be overclaiming.

---

## Troubleshooting

**crt.sh returns 0 names**
- Slow/rate-limited — rerun the `curl`; it often takes a few tries. Add `-m 60` for more time.
- Domain mismatch — use the `DOMAIN=` variable so all three references match. Confirm certs exist by opening `https://crt.sh/?q=%25.yourdomain.com` in a browser.

**`go install` → `GOPROXY list is not the empty string, but contains no entries`**
```bash
go env -w GOPROXY=https://proxy.golang.org,direct
```
then retry.

**`go install` → `proxyconnect tcp: tls: first record does not look like a TLS handshake`**
The box has an outbound HTTP proxy set with an `https://` scheme, but the proxy speaks plain HTTP. Check `env | grep -i proxy`, then either fix the scheme (`export HTTPS_PROXY="http://HOST:PORT"`) or, if the proxy isn't required, `unset HTTPS_PROXY https_proxy HTTP_PROXY http_proxy`. If neither works, use the **prebuilt binary** route — your browser already handles the proxy, so download the release ZIP and skip Go entirely.

**Moving the ZIP onto the box (no internet on the box directly)**
Serve from the machine that *has* the file, pull from the box:
- Simplest — `scp` from your machine: `scp dnsx_*_linux_amd64.zip user@BOX_IP:~/`
- Or Python server **on the machine with the file** (`python -m http.server 8000`), then on the box `wget http://SOURCE_IP:8000/dnsx_*_linux_amd64.zip`. (A plain Python server only *serves* files; it can't receive uploads — so it goes on the source, not the box.)

**`comm` giving weird results** — both inputs must be sorted the same way; keep the `sort -u` on each side (the `<( )` wrappers handle this).

---

## Quick reference — full run, copy-paste

```bash
# 0. setup
mkdir -p ~/recon && cd ~/recon
# (put your Censys export here as big_ip_list.txt)
DOMAIN=company.org
EXCLUDE="137.200.0.0/16,199.173.224.0/21"

# 1. clean candidate IPs
grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' big_ip_list.txt | sort -u > ips.txt

# 2. harvest subdomains from crt.sh
curl -s "https://crt.sh/?q=%25.$DOMAIN&output=json" \
  | jq -r '.[].name_value' | sed 's/\*\.//g' \
  | grep -E "\.$DOMAIN$" | sort -u > org_names.txt

# 3. resolve to live IPs
dnsx -l org_names.txt -a -resp-only -silent | sort -u > org_ips.txt

# 4. filter out-of-scope ranges
grepcidr -v "$EXCLUDE" org_ips.txt > org_ips_filtered.txt

# 5. correlate
comm -12 <(sort -u ips.txt) <(sort -u org_ips_filtered.txt) > confirmed.txt
comm -23 <(sort -u org_ips_filtered.txt) <(sort -u ips.txt) > org_not_in_censys.txt

# 6. keep name<->ip mapping
dnsx -l org_names.txt -a -resp -silent > resolved_with_names.txt

# 7. review counts, then scope-check before scanning
wc -l ips.txt org_names.txt org_ips.txt org_ips_filtered.txt confirmed.txt org_not_in_censys.txt
```

---

## Appendix — IPv6 track (only if v6 is in scope)

```bash
grep ':' big_ip_list.txt | sort -u > ips6.txt
dnsx -l org_names.txt -aaaa -resp-only -silent | sort -u > org_ips6.txt
comm -12 ips6.txt org_ips6.txt > confirmed6.txt
```
Keep v4 and v6 separate — `comm` matches text exactly and the formats never overlap.

---
---
