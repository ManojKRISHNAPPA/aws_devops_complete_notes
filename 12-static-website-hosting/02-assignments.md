# Static Website Hosting — Assignments

## Assignment 1: Deploy itkannadigaru.com

### Goal
Host your website at `https://itkannadigaru.com` using S3 + CloudFront + ACM + Route 53.

---

### Step 1 — Create S3 Bucket and Upload Code

```bash
# Create bucket (replace region if needed)
aws s3api create-bucket \
  --bucket itkannadigaru-website \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Block all public access
aws s3api put-public-access-block \
  --bucket itkannadigaru-website \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,\
    BlockPublicPolicy=true,RestrictPublicBuckets=true

# Upload your built code
aws s3 sync ./dist s3://itkannadigaru-website/
# OR
aws s3 cp ./dist/ s3://itkannadigaru-website/ --recursive
```

**Verify:**
```
AWS Console → S3 → itkannadigaru-website → Objects tab
You should see index.html and your other files
```

---

### Step 2 — Request ACM Certificate (us-east-1)

> IMPORTANT: Switch region to us-east-1 before doing this step.

```
AWS Console → ACM → Switch region to us-east-1
→ Request a certificate
→ Public certificate
→ Add domain names:
    itkannadigaru.com
    *.itkannadigaru.com
→ Validation: DNS validation
→ Request
```

After requesting, you see a **CNAME record** to validate. Keep this page open — you need it in Step 3.

---

### Step 3 — Set Up Route 53 Hosted Zone

```
Route 53 → Hosted zones → Create hosted zone
Domain name: itkannadigaru.com
Type:        Public hosted zone
→ Create
```

Route 53 creates the hosted zone and auto-adds:
- **SOA record** (Start of Authority)
- **NS record** with 4 nameservers

**Copy these 4 NS values** — example:
```
ns-1234.awsdns-12.org
ns-567.awsdns-34.co.uk
ns-890.awsdns-56.com
ns-12.awsdns-78.net
```

**Go to your domain registrar** (GoDaddy / Namecheap / wherever you bought itkannadigaru.com):
```
Domain settings → Nameservers → Custom nameservers
→ Paste all 4 NS values above
→ Save
```

DNS propagation: takes a few minutes to 48 hours (usually under 1 hour).

**Add ACM Validation Record to Route 53:**
```
Go back to ACM (us-east-1) → Your certificate → Domains section
Click "Create records in Route 53" (if domain is in Route 53, one click works)
```
Wait for certificate status to change from **Pending validation** → **Issued** ✓

---

### Step 4 — Create CloudFront Distribution

```
CloudFront → Create distribution

ORIGIN section:
  Origin domain:    itkannadigaru-website.s3.ap-south-1.amazonaws.com
  Origin path:      (leave blank)
  Name:             S3-itkannadigaru-website
  Origin access:    Origin access control settings (recommended)
                    → Create new OAC → name: itkannadigaru-oac → Create
  
DEFAULT CACHE BEHAVIOR:
  Viewer protocol policy:  Redirect HTTP to HTTPS
  Allowed HTTP methods:    GET, HEAD
  Cache policy:            CachingOptimized
  
SETTINGS:
  Alternate domain names:  itkannadigaru.com
                           www.itkannadigaru.com
  Custom SSL certificate:  → select your ACM cert (itkannadigaru.com)
  Default root object:     index.html
  
→ Create distribution
```

**Update S3 bucket policy** (CloudFront will show a banner with the policy — copy and apply):
```
S3 → itkannadigaru-website → Permissions → Bucket policy → Edit → Paste OAC policy
```

**For SPA (React/Vue/Angular) — Add custom error response:**
```
CloudFront → your distribution → Error pages → Create custom error response
  HTTP error code:       403
  Response page path:    /index.html
  HTTP response code:    200
  
Repeat for:
  HTTP error code:       404
  Response page path:    /index.html
  HTTP response code:    200
```

Wait for distribution **Status: Deployed** (5–10 min).

Note your distribution domain: `d1a2b3c4d5e6.cloudfront.net`

---

### Step 5 — Add DNS Records in Route 53

```
Route 53 → Hosted zones → itkannadigaru.com → Create record

Record 1 (apex):
  Record name:  (leave blank — this is itkannadigaru.com)
  Record type:  A
  Alias:        ON
  Route traffic to: Alias to CloudFront distribution
  Distribution: d1a2b3c4d5e6.cloudfront.net
  → Create records

Record 2 (www):
  Record name:  www
  Record type:  A
  Alias:        ON
  Route traffic to: Alias to CloudFront distribution
  Distribution: d1a2b3c4d5e6.cloudfront.net
  → Create records
```

---

### Step 6 — Test

```bash
# Test DNS resolution
dig itkannadigaru.com
nslookup itkannadigaru.com

# Test HTTPS
curl -I https://itkannadigaru.com
# Should see: HTTP/2 200 and x-cache: Hit from cloudfront (after first load)

# Test redirect
curl -I http://itkannadigaru.com
# Should see: 301 redirect to https://itkannadigaru.com
```

Open in browser: `https://itkannadigaru.com` ✓

---

### Step 7 — Deploy Code Updates

Whenever you push new code:
```bash
# Upload new build
aws s3 sync ./dist s3://itkannadigaru-website/ --delete

# Invalidate CloudFront cache so users get new version
aws cloudfront create-invalidation \
  --distribution-id YOUR_DIST_ID \
  --paths "/*"
```

---

## Assignment 2: Verify Each Layer

**Test S3 directly** (should fail — private bucket):
```bash
curl https://itkannadigaru-website.s3.ap-south-1.amazonaws.com/index.html
# Expected: AccessDenied — bucket is private ✓
```

**Test CloudFront directly** (should work):
```bash
curl https://d1a2b3c4d5e6.cloudfront.net/
# Expected: 200 with your index.html content ✓
```

**Test your domain** (should work via Route 53 + CloudFront):
```bash
curl https://itkannadigaru.com/
# Expected: 200 with your index.html content ✓
```

**Check certificate**:
```bash
curl -v https://itkannadigaru.com/ 2>&1 | grep -i "subject\|issuer\|SSL"
# Should show: subject: CN=itkannadigaru.com, issuer: Amazon
```

---

## Assignment 3: Verify Nameserver Setup

```bash
# Check what nameservers the world sees for your domain
dig NS itkannadigaru.com

# Check your Route 53 hosted zone NS records
aws route53 list-resource-record-sets \
  --hosted-zone-id YOUR_ZONE_ID \
  --query "ResourceRecordSets[?Type=='NS']"

# The dig output NS values should match Route 53 NS records
# If they don't match → you haven't updated nameservers at your registrar yet
```

---

## Troubleshooting Checklist

```
Problem: https://itkannadigaru.com shows "This site can't be reached"
Fix:     Check NS records at registrar match Route 53 hosted zone
         Run: dig NS itkannadigaru.com

Problem: Browser shows "Your connection is not private" (SSL error)
Fix:     Check ACM cert is Issued (not Pending)
         Check cert is in us-east-1
         Check CloudFront CNAME matches cert domain

Problem: Site loads but shows XML (S3 error page)
Fix:     Set Default root object = index.html in CloudFront

Problem: React routes work on direct load but 403 after page refresh
Fix:     Add custom error response: 403 → /index.html with 200

Problem: Deployed new code but site still shows old version
Fix:     Run CloudFront invalidation: --paths "/*"

Problem: www.itkannadigaru.com doesn't work
Fix:     Add www CNAME record in Route 53 pointing to CloudFront
         Add www to CloudFront Alternate domain names list
```
