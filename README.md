# 🚀 Project 2

🚀 Day 2 of 30 — #30DaysOfAWS


Amazon S3 paired with CloudFront is how AWS recommends hosting static content — and after building it hands-on, I completely understand why.

---

## Architecture overview

```
Browser (anywhere on earth)
    │
    ▼  HTTPS
CloudFront edge location  ──(cache HIT)──▶  serves instantly
    │
    │ cache MISS (first request or TTL expired)
    ▼
Origin Access Control (OAC)
    │  only this distribution can read the bucket
    ▼
S3 bucket (private)
    └── index.html, styles.css, app.js ...
```

The beauty of this architecture: once a file is cached at a CloudFront edge location, every subsequent request for it is served in single-digit milliseconds — no round-trip to S3, no server computation, nothing. It just flies.

---

## What I built

A fully animated, single-page site served from S3 via CloudFront with:

- Private S3 bucket (public access fully blocked)
- Origin Access Control (OAC) — only the CloudFront distribution can read from the bucket
- HTTP → HTTPS automatic redirect
- Free TLS certificate via AWS Certificate Manager
- Default root object (`index.html`) configured on the distribution
- Custom error pages mapped for 403 → 404

The site displays live stats: actual page load time from the Performance API, UTC clock, session uptime counter, and the real OAC bucket policy I used.

---

## Step-by-step build

### Step 1 — Create the S3 bucket

Go to **S3 → Create bucket**:

- Name: globally unique (e.g. `yourname-day2-website`)
- Region: `us-east-1`
- **Block all public access: leave ON** — this is correct. CloudFront reads via OAC, not via public access.

Upload your `index.html` and any other assets to the bucket root.

> ⚠️ **Common mistake:** Uploading files into a subfolder (e.g. `site/index.html`). CloudFront looks for `index.html` at the bucket root unless you configure a path prefix. Keep it flat.

### Step 2 — Create the CloudFront distribution

Go to **CloudFront → Create distribution**:

- **Origin domain:** select your S3 bucket from the dropdown (the S3 bucket option, NOT the static website endpoint URL)
- **Origin access:** Origin Access Control → Create new OAC (accept all defaults)
- **Viewer protocol policy:** Redirect HTTP to HTTPS
- **Default root object:** `index.html`
- **Price class:** North America and Europe (cheapest, fine for learning)

After clicking Create, AWS shows a yellow banner: *"You must update the S3 bucket policy."* Copy the generated policy — you need it in the next step.

> ⚠️ **The #1 reason people get AccessDenied:** Skipping this banner and never applying the bucket policy.

### Step 3 — Apply the bucket policy

Go to **S3 → your bucket → Permissions → Bucket policy → Edit**. Paste the policy CloudFront generated:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::YOUR-ACCOUNT-ID:distribution/YOUR-DIST-ID"
        }
      }
    }
  ]
}
```

The `Condition` block is critical — it locks access to *your specific distribution* only. Even another CloudFront distribution in your own account cannot read this bucket.

### Step 4 — Configure custom error pages

In your distribution → **Error pages → Create custom error response**:

<table>
  <thead>
    <tr><th>HTTP error code</th><th>Response page path</th><th>HTTP response code</th></tr>
  </thead>
  <tbody>
    <tr><td>403</td><td>/error.html</td><td>404</td></tr>
    <tr><td>404</td><td>/error.html</td><td>404</td></tr>
  </tbody>
</table>

S3 returns a **403** (not 404) when a file doesn't exist in a private bucket. Mapping it to a friendly 404 page matters for UX and SEO.

### Step 5 — Wait and test

CloudFront deployment takes 3–5 minutes. Once the distribution status shows **Enabled**, open the `.cloudfront.net` domain in your browser. You should see your page over HTTPS with a valid certificate.

### Step 6 — Update files and invalidate cache

After uploading new files to S3, CloudFront still serves the old cached version until the TTL expires. Force an immediate refresh:

```bash
# Via AWS CLI
aws cloudfront create-invalidation \
  --distribution-id YOUR_DIST_ID \
  --paths "/*"
```

Or via the console: **CloudFront → your distribution → Invalidations → Create invalidation → `/*`**

> 💡 **Cost note:** The first 1,000 invalidation paths per month are free. After that it's $0.005 per path. For production, use versioned filenames (`app.v2.js`) instead of invalidating — that's the real-world pattern.

### Step 7 — Clean up

Disable the CloudFront distribution first (Actions → Disable), wait ~5 minutes for it to fully disable, then delete it. Then empty and delete the S3 bucket.

---

## The AccessDenied debugging checklist

I hit the `AccessDenied` XML error myself. Here's the exact checklist to fix it:

**1. Bucket policy missing** — Go to S3 → Permissions → Bucket policy. If it's empty, paste in the OAC policy from Step 3 above.

**2. Default root object not set** — CloudFront → General → Settings → Edit → set `index.html`. Without it, `/` maps to nothing in S3.

**3. OAC not selected** — CloudFront → Origins → Edit. Confirm "Origin access control settings" is selected, not "Public" or "Legacy OAI."

**4. File uploaded to wrong path** — Run `aws s3 ls s3://your-bucket/` and confirm `index.html` is at the root.

**5. Distribution still deploying** — Status must show "Enabled" (not "Deploying") before it works.

---

## SAA-C03 exam concepts

### OAC vs OAI

Origin Access Identity (OAI) is the legacy method for securing S3 origins. Origin Access Control (OAC) is the modern replacement — it supports SSE-KMS encrypted buckets, all S3 regions, and POST/PUT/DELETE methods. The exam still tests OAI but learn both. In real implementations, always use OAC.

### CloudFront cache tiers

CloudFront has two cache layers. **Edge locations** (400+) sit close to users globally. **Regional Edge Caches** sit between edge locations and the origin. A cache miss at an edge location hits the Regional Edge Cache first before reaching S3 — significantly reducing origin load at scale.

### ACM certificate must be in us-east-1

If you attach a custom domain to CloudFront, the ACM certificate must be provisioned in **us-east-1** regardless of where your S3 bucket or users are. This is a favourite SAA-C03 trap — certificates in other regions won't appear in the CloudFront console.

### S3 static website endpoint vs S3 REST API endpoint

There are two ways to reference an S3 bucket in CloudFront. The **REST API endpoint** (`bucket.s3.region.amazonaws.com`) works with OAC and private buckets. The **static website endpoint** (`bucket.s3-website-region.amazonaws.com`) requires public bucket access and only supports HTTP. Always use the REST API endpoint with OAC.

### CloudFront vs S3 Transfer Acceleration

Both use CloudFront's edge network but in opposite directions. **CloudFront** optimises *downloads* — content flows from S3 to users via edge caches. **S3 Transfer Acceleration** optimises *uploads* — data flows from users to S3 via edge locations. Know the direction each one serves.

### Cache-Control headers and TTL

By default CloudFront respects `Cache-Control` headers set on S3 objects. Without them it applies a default TTL of 24 hours. For static sites, a common pattern is: `Cache-Control: max-age=31536000` (1 year) for versioned assets like `app.abc123.js`, and `Cache-Control: no-cache` for `index.html` so the entry point always fetches fresh.

---

## Estimated cost

<table>
  <thead>
    <tr><th>Resource</th><th>Free tier</th><th>Beyond free tier</th></tr>
  </thead>
  <tbody>
    <tr><td>S3 storage</td><td>5 GB free (12 months)</td><td>$0.023/GB/month</td></tr>
    <tr><td>S3 GET requests</td><td>20,000 free/month</td><td>$0.0004 per 1,000</td></tr>
    <tr><td>CloudFront data transfer</td><td>1 TB/month free (12 months)</td><td>$0.0085/GB</td></tr>
    <tr><td>CloudFront HTTPS requests</td><td>10M free/month (12 months)</td><td>$0.0100 per 10,000</td></tr>
    <tr><td>ACM certificate</td><td>Free forever</td><td>Free forever</td></tr>
  </tbody>
</table>

For a personal portfolio or project site with modest traffic, the monthly bill after the free tier is typically **under $1**.

---

## Day 1 vs Day 2 — the key difference

| | Day 1 (EC2) | Day 2 (S3 + CloudFront) |
|---|---|---|
| Server to manage | Yes — EC2 instance | None |
| OS patching | Yes | Never |
| Scaling | Manual (or ASG) | Automatic, infinite |
| HTTPS setup | Manual (Certbot etc.) | Automatic via ACM |
| Global performance | Single region | 400+ edge locations |
| Monthly cost | ~$8 (t2.micro) | ~$0–1 |
| Best for | Dynamic apps, APIs | Static HTML/CSS/JS |

The right tool for the right job. Neither is universally better — but for static content, S3 + CloudFront wins on every axis.

---

## What's next

**Day 3** goes deep on networking — building a VPC from scratch with public and private subnets, route tables, an internet gateway, and a NAT gateway. The foundational network layer that everything in AWS runs on top of.
