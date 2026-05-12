# Static Website Hosting — S3 + CloudFront + Route 53 + ACM

## Table of Contents
1. [Architecture Overview](#1-architecture-overview)
2. [S3 Bucket — Static Hosting](#2-s3-bucket--static-hosting)
3. [ACM — SSL/TLS Certificate](#3-acm--ssltls-certificate)
4. [CloudFront — CDN Distribution](#4-cloudfront--cdn-distribution)
5. [Route 53 — DNS Management](#5-route-53--dns-management)
6. [End-to-End Flow: How a Request Works](#6-end-to-end-flow-how-a-request-works)
7. [Cheat Sheet](#7-cheat-sheet)

---

## 1. Architecture Overview

```
User Browser
     │
     │  https://itkannadigaru.com
     ▼
┌─────────────┐
│  Route 53   │  ← DNS: maps domain → CloudFront
└──────┬──────┘
       │  CNAME / A (Alias) record → d1234abcd.cloudfront.net
       ▼
┌─────────────────────┐
│    CloudFront CDN   │  ← Serves cached content globally
│  (Edge Locations)   │     HTTPS via ACM Certificate
└──────────┬──────────┘
           │  Origin request (cache miss)
           ▼
┌─────────────────────┐
│     S3 Bucket       │  ← Stores your HTML/CSS/JS files
│  (Origin Server)    │     Private bucket (OAC protects it)
└─────────────────────┘
           ▲
┌─────────────────────┐
│   ACM Certificate   │  ← Free SSL cert for itkannadigaru.com
│  (us-east-1 only)   │     Attached to CloudFront
└─────────────────────┘
```

**Why each piece?**

| Component   | Job                                            |
|-------------|------------------------------------------------|
| S3          | Store your website files (HTML, CSS, JS, imgs) |
| CloudFront  | Deliver files fast from edge, enforce HTTPS    |
| ACM         | Free TLS certificate so browser shows padlock  |
| Route 53    | Maps your domain name to CloudFront            |

---

## 2. S3 Bucket — Static Hosting

### What is it here?
S3 holds your website files. It is the **origin** — the source of truth. CloudFront pulls from it.

### Key settings for a static website

```
Bucket name:  itkannadigaru-website   (any name, does not need to match domain)
Region:       ap-south-1              (or any region, CloudFront is global anyway)
```

### Two approaches: Public vs Private (OAC)

**Old way — Public bucket**
```
Bucket → Permissions → Block Public Access → OFF
Bucket policy → Allow s3:GetObject to *
```
Problem: anyone can bypass CloudFront and hit S3 URL directly.

**New way — Private bucket + OAC (recommended)**
```
Block Public Access = ON  (default)
CloudFront Origin Access Control (OAC) gets exclusive read permission
```
S3 bucket policy only allows CloudFront's OAC identity:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFront",
    "Effect": "Allow",
    "Principal": {
      "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::itkannadigaru-website/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT_ID:distribution/DIST_ID"
      }
    }
  }]
}
```

### Static Website Hosting setting
```
Bucket → Properties → Static website hosting → Enable
Index document: index.html
Error document: index.html   ← for SPA (React/Vue) this handles client-side routing
```

> Note: When using OAC + private bucket, you do NOT need to enable "Static Website Hosting".
> You can serve files directly through CloudFront pointing at the S3 REST endpoint.

### Upload your code
```bash
aws s3 sync ./dist s3://itkannadigaru-website/
# OR
aws s3 cp ./dist s3://itkannadigaru-website/ --recursive
```

---

## 3. ACM — SSL/TLS Certificate

### What is ACM?
**AWS Certificate Manager** issues free SSL/TLS certificates.  
The certificate proves to browsers that your site is really `itkannadigaru.com`.  
Without it: `http://` only, browser shows "Not Secure".  
With it: `https://` with padlock.

### CRITICAL RULE: Region must be us-east-1

```
CloudFront only accepts ACM certificates from us-east-1 (N. Virginia).
Even if your S3 bucket is in ap-south-1, the certificate MUST be in us-east-1.
```

### Request a certificate

```
ACM → Request certificate → Public certificate
Domain names:
  itkannadigaru.com          ← apex domain
  *.itkannadigaru.com        ← wildcard covers www, api, etc.

Validation method: DNS validation (recommended)
```

### DNS Validation
ACM gives you a **CNAME record** to add to your DNS. This proves you own the domain.

```
ACM gives you:
  Name:   _abc123xyz.itkannadigaru.com
  Value:  _def456.acm-validations.aws.

You add this CNAME to Route 53 (or your domain registrar).
ACM checks for it → validates → issues certificate.
```

If your domain is already in Route 53:
```
ACM → Certificate → "Create records in Route 53" button → one click done
```

Validation takes **a few minutes to a few hours** (usually under 5 min if DNS is already in Route 53).

### Certificate status
```
Pending validation → Issued  ✓
```
Once **Issued**, copy the certificate ARN — you'll attach it to CloudFront.

---

## 4. CloudFront — CDN Distribution

### What is CloudFront?

**Amazon CloudFront** is a **Content Delivery Network (CDN)**.  
It has **edge locations** (400+ worldwide) — mini data centers close to users.

```
Without CloudFront:
  User in London → S3 in Mumbai → 200ms latency

With CloudFront:
  User in London → Edge in London (cached) → 10ms latency
                              ↑
               Only fetches from S3 once, then caches
```

### Core Concepts

#### Distribution
A **distribution** is one CloudFront deployment. It gets a domain like:
```
d1a2b3c4d5e6f7.cloudfront.net
```
You point your Route 53 domain to this.

#### Origin
The **origin** is where CloudFront fetches the real content from.  
In our case: the S3 bucket.
```
Origin domain: itkannadigaru-website.s3.ap-south-1.amazonaws.com
```

#### Behavior
A **cache behavior** defines rules for how CloudFront handles requests.
```
Path pattern: /*  (all paths)
Viewer Protocol Policy: Redirect HTTP to HTTPS
Allowed HTTP Methods: GET, HEAD
Cache Policy: CachingOptimized (for static sites)
```

#### Edge Location vs Region
```
AWS Region         → Where your actual data lives (S3, EC2, etc.)
Edge Location      → CloudFront's cache node near users (150+ cities)
Point of Presence  → Another name for Edge Location
```

#### Cache Behavior
```
1. User requests itkannadigaru.com/index.html
2. Request hits nearest Edge Location
3. Edge checks cache:
   HIT  → Returns cached file instantly
   MISS → Fetches from S3 origin, caches it, returns to user
4. Next user in same region gets HIT
```

**TTL (Time To Live)** — how long files stay cached:
```
Default TTL:  86400 seconds (24 hours)
Min TTL:      0
Max TTL:      31536000 (1 year)
```

For frequently updated sites, set short TTL or use **cache invalidation**:
```bash
aws cloudfront create-invalidation \
  --distribution-id DIST_ID \
  --paths "/*"
```

#### OAC — Origin Access Control
Modern way to let CloudFront read your private S3 bucket.

```
CloudFront → Security → Origin Access Control → Create
Name: itkannadigaru-oac
Origin type: S3
Signing behavior: Sign requests (recommended)
```
Then attach it to the origin in your distribution.

### Creating a Distribution — Step by Step

```
CloudFront → Create distribution

Origin:
  Origin domain:      itkannadigaru-website.s3.ap-south-1.amazonaws.com
  Origin access:      Origin Access Control (OAC) → select your OAC
  
Default cache behavior:
  Viewer Protocol Policy: Redirect HTTP to HTTPS
  Allowed methods:        GET, HEAD
  Cache policy:           CachingOptimized

Settings:
  Alternate domain names (CNAMEs): itkannadigaru.com
                                   www.itkannadigaru.com
  Custom SSL certificate:          → select your ACM cert (us-east-1)
  
  Default root object: index.html
  
  Price class: Use all edge locations (best perf)
               OR Use only North America and Europe (cheaper)
```

After creating, CloudFront gives you:
```
Distribution domain name: d1a2b3c4d5e6.cloudfront.net
Distribution ID:          ABCDE1FGH2IJKL
Status:                   Deploying → Deployed (takes ~5-10 min)
```

### CloudFront Error Pages (Custom Error Responses)
Important for Single Page Apps (React/Vue/Angular):
```
CloudFront → Error pages → Create custom error response
HTTP error code:     403 or 404
Response page path:  /index.html
HTTP response code:  200

Why: S3 returns 403 for /about (file doesn't exist as /about)
     But React Router handles /about client-side
     So we redirect all 403/404 → index.html with 200
```

### CloudFront Functions / Lambda@Edge
For advanced logic at the edge (URL rewrites, auth headers, redirects).  
Not needed for basic static hosting.

---

## 5. Route 53 — DNS Management

### What is Route 53?

**Amazon Route 53** is AWS's **DNS (Domain Name System)** service.  
DNS translates human-readable domain names into IP addresses (or in CloudFront's case, into a CNAME).

```
Without DNS:  User types d1a2b3c4d5e6.cloudfront.net (ugly, hard to remember)
With DNS:     User types itkannadigaru.com → Route 53 → CloudFront
```

### Core Concepts

#### Hosted Zone
A **hosted zone** is a container for DNS records for one domain.
```
Hosted zone for: itkannadigaru.com
Contains:        All DNS records (A, CNAME, MX, TXT, NS, SOA, etc.)
```
There are two types:
```
Public hosted zone  → Answers DNS queries from the internet (your website)
Private hosted zone → Answers queries only from within a VPC (internal services)
```

#### What is a Nameserver (NS)?

This is very important to understand.

```
When you register a domain (on GoDaddy, Namecheap, or Route 53 Registrar),
the registrar keeps a record of which nameservers are authoritative for your domain.

Nameservers are the servers that KNOW all the DNS records for your domain.
```

**Flow:**
```
1. User types itkannadigaru.com in browser
2. Browser asks its local DNS resolver (ISP or 8.8.8.8)
3. Resolver asks Root DNS: "who knows about .com?"
4. Root DNS says: "ask Verisign (TLD for .com)"
5. Verisign says: "ask ns-123.awsdns-12.com" ← this is the nameserver
6. ns-123.awsdns-12.com (Route 53) says: "itkannadigaru.com → d1a2b3.cloudfront.net"
7. Browser connects to CloudFront
```

**Route 53 gives you 4 nameservers when you create a hosted zone:**
```
ns-1234.awsdns-12.org
ns-567.awsdns-34.co.uk
ns-890.awsdns-56.com
ns-12.awsdns-78.net
```

**CRITICAL:** You must copy these 4 NS records to your **domain registrar**.
If you bought `itkannadigaru.com` on GoDaddy/Namecheap, go there and set these nameservers.
If you registered via Route 53 Registrar, it is done automatically.

```
Domain Registrar (GoDaddy)        Route 53 Hosted Zone
────────────────────────         ──────────────────────
Set nameservers to:     ──────►  ns-1234.awsdns-12.org
  ns-1234.awsdns-12.org           ns-567.awsdns-34.co.uk
  ns-567.awsdns-34.co.uk          ns-890.awsdns-56.com
  ns-890.awsdns-78.net            ns-12.awsdns-78.net
                                       │
                                  Hosts all DNS records
                                  for itkannadigaru.com
```

#### DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| A      | Maps domain → IPv4 address | `itkannadigaru.com → 1.2.3.4` |
| AAAA   | Maps domain → IPv6 address | `itkannadigaru.com → 2001::1` |
| CNAME  | Maps domain → another domain | `www → itkannadigaru.com` |
| NS     | Nameserver records (who manages this domain) | `itkannadigaru.com → ns-1234.awsdns-12.org` |
| SOA    | Start of Authority (auto-created, metadata) | — |
| MX     | Mail server | `mail.itkannadigaru.com` |
| TXT    | Text records (domain verification, SPF) | ACM validation CNAME |

#### Alias Record (AWS Special Feature)

A CNAME cannot be used on the **apex domain** (root domain — `itkannadigaru.com`).  
DNS rules say the apex must be an A record.

AWS solves this with **Alias records** — they look like A records but point to AWS resources.

```
Record type: A (Alias)
Name:        itkannadigaru.com
Alias target: CloudFront distribution → d1a2b3.cloudfront.net

Advantages:
  - Works on apex domain
  - No extra DNS lookup (resolved internally by AWS)
  - Free — Route 53 does not charge for alias queries to AWS resources
```

#### TTL (Time To Live)
```
TTL = how long DNS resolvers cache your record

Low TTL (60s):   Changes propagate fast, more DNS queries (cost)
High TTL (86400s): Cached longer, changes take longer to propagate

For CloudFront Alias records: TTL is managed by AWS, not configurable
```

### DNS Records to Create for Your Site

**Record 1 — Apex domain**
```
Type:         A (Alias)
Name:         itkannadigaru.com   (or leave blank — means apex)
Alias target: CloudFront distribution
              → d1a2b3c4d5e6.cloudfront.net
```

**Record 2 — www subdomain**
```
Type:         CNAME  (or A Alias)
Name:         www.itkannadigaru.com
Value:        itkannadigaru.com   (redirect to apex)
              OR
Alias target: same CloudFront distribution
```

**Record 3 — ACM Validation (auto-created by ACM)**
```
Type:  CNAME
Name:  _abc123.itkannadigaru.com
Value: _def456.acm-validations.aws.
```

### Route 53 Routing Policies
For your static website you use **Simple routing**.  
But Route 53 supports advanced policies:

| Policy | Use Case |
|--------|----------|
| Simple | Single resource (your case — one CloudFront dist) |
| Weighted | A/B testing (10% → new version, 90% → old) |
| Latency | Route to lowest-latency region |
| Failover | Active-passive: switch to backup if primary fails |
| Geolocation | Route users by country/continent |
| Geoproximity | Route by location with bias adjustments |
| Multivalue | Like Simple but returns multiple IPs (basic load balance) |

---

## 6. End-to-End Flow: How a Request Works

```
User types: https://itkannadigaru.com/about

Step 1 — DNS Resolution
  Browser → asks DNS resolver
  DNS resolver → asks Route 53
  Route 53 → "itkannadigaru.com is A Alias → d1a2b3.cloudfront.net"

Step 2 — TLS Handshake
  Browser connects to nearest CloudFront Edge Location
  CloudFront presents ACM certificate for itkannadigaru.com
  Browser validates cert → HTTPS established ✓

Step 3 — CloudFront Cache Check
  Edge Location checks: do I have /about cached?
  
  CACHE HIT  → Return cached file immediately
  CACHE MISS → Go to Step 4

Step 4 — Origin Fetch (S3)
  CloudFront → S3 bucket (authenticated via OAC)
  S3 returns index.html (since /about is React-routed)
  CloudFront caches it → returns to browser

Step 5 — Browser Renders
  Browser loads index.html
  React Router reads /about → renders About component
```

---

## 7. Cheat Sheet

```
COMPONENT SUMMARY
─────────────────────────────────────────────────────────────
S3 Bucket
  Purpose:     Store website files (origin)
  Setting:     Block public access ON, use OAC
  Key files:   index.html (root), index.html (error for SPA)

ACM Certificate
  Purpose:     HTTPS / SSL for your domain
  MUST be in:  us-east-1 (N. Virginia) for CloudFront
  Validation:  DNS (CNAME record in Route 53)
  Covers:      itkannadigaru.com + *.itkannadigaru.com

CloudFront Distribution
  Purpose:     CDN — cache + deliver content globally
  Origin:      S3 bucket via OAC
  CNAMEs:      itkannadigaru.com, www.itkannadigaru.com
  SSL cert:    ACM cert (us-east-1)
  Root object: index.html
  Error pages: 403/404 → /index.html with 200 (for SPA)

Route 53
  Purpose:     DNS — map domain to CloudFront
  Hosted zone: Public zone for itkannadigaru.com
  NS records:  4 nameservers → set these at your registrar
  A record:    itkannadigaru.com → Alias → CloudFront dist
  CNAME:       www → CloudFront dist (or same Alias)

─────────────────────────────────────────────────────────────
DEPLOYMENT ORDER
─────────────────────────────────────────────────────────────
1. Create S3 bucket + upload code
2. Request ACM certificate (us-east-1) + validate via DNS
3. Create Route 53 hosted zone + copy NS to registrar
4. Create CloudFront distribution (attach OAC + ACM cert)
5. Add A Alias record in Route 53 → CloudFront
6. Wait for CloudFront to deploy (~10 min)
7. Test: https://itkannadigaru.com ✓

─────────────────────────────────────────────────────────────
COMMON MISTAKES
─────────────────────────────────────────────────────────────
✗  ACM cert in wrong region (must be us-east-1 for CloudFront)
✗  Forgot to set NS records at domain registrar
✗  CNAME on apex domain (use A Alias instead)
✗  Public S3 bucket (use OAC + private bucket)
✗  Forgot custom error response for SPA (403/404 → index.html)
✗  CloudFront CNAME not set (cert validation fails)
✗  Deploying new code but CloudFront serving old cached version
   → Fix: aws cloudfront create-invalidation --paths "/*"
```
