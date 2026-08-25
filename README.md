# Portfolio-build-deployment-Doc
Its a description of How I build and deployed my portfolio website on AWS S3 + CloudFront

`project live at - https://d1ua0n7d5gbns6.cloudfront.net/`

Link to private repo for authorized users : `github.com/Ashut2/Ashu-portfolio`
# Ashutosh Shukla — DevOps Portfolio

> **"Be less Impressed & More Involved!"**

A retro terminal-themed, single-page portfolio built with **Next.js 16 · TypeScript · Tailwind CSS v4 · Framer Motion**.  
Statically exported — zero server needed. Drop it on S3 and go.

---

## Table of Contents

1. [Quick Start (Local Dev)](#-quick-start)
2. [Build for Production](#-build-for-production)
3. [Deploy to AWS S3 + CloudFront](#-deploy-to-aws-s3--cloudfront)
4. [Custom Domain Setup](#-custom-domain-setup)
   - [Option A — Cloudflare](#option-a--cloudflare-recommended)
   - [Option B — Namecheap](#option-b--namecheap)
5. [GitHub Actions CI/CD](#-github-actions-cicd)
6. [Project Structure](#-project-structure)
7. [Tech Stack](#-tech-stack)
8. [Customization Guide](#-customization)
9. [Troubleshooting](#-troubleshooting)

---

## 🚀 Quick Start

```bash
# Prerequisites: Node.js 18+ and Yarn
node -v   # should print v18.x or higher

# 1. Install dependencies
cd portfolio
yarn install

# 2. Start dev server
yarn dev

# 3. Open browser
#    http://localhost:3000
```

---

## 📦 Build for Production

```bash
yarn build
```

This creates a fully static site in the **`out/`** directory.  
No Node server required — just HTML, CSS, JS, and assets.

```
out/
├── index.html          # Your portfolio
├── _next/              # JS bundles, CSS, fonts
├── profile.jpeg        # Your photo
├── 404.html            # Custom 404
└── ...
```

Test the build locally before deploying:

```bash
npx serve out
# → http://localhost:3000
```

---

## ☁️ Deploy to AWS S3 + CloudFront

### Prerequisites

| Tool | Install |
|---|---|
| AWS CLI v2 | `brew install awscli` or [docs.aws.amazon.com/cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) |
| AWS Account | [aws.amazon.com](https://aws.amazon.com/) |
| IAM User | With `S3FullAccess` + `CloudFrontFullAccess` permissions |

Configure your CLI:

```bash
aws configure
# AWS Access Key ID:     <your-key>
# AWS Secret Access Key: <your-secret>
# Default region:        ap-south-1    # Mumbai, closest to India
# Default output format: json
```

---

### Step 1 — Create an S3 Bucket

Pick a globally unique name. If you plan to use a custom domain, the bucket name **must match your domain** (e.g., `ashutosh.dev`).

```bash
BUCKET_NAME="ashutosh-portfolio-2026"
REGION="ap-south-1"

# Create the bucket
aws s3 mb s3://$BUCKET_NAME --region $REGION
```

---

### Step 2 — Enable Static Website Hosting

```bash
aws s3 website s3://$BUCKET_NAME \
  --index-document index.html \
  --error-document 404.html
```

Your S3 website endpoint will be:  
`http://<BUCKET_NAME>.s3-website.<REGION>.amazonaws.com`

---

### Step 3 — Set Bucket Policy (Public Read)

Create a file called `bucket-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    }
  ]
}
```

> **Important:** Replace `BUCKET_NAME` with your actual bucket name in the JSON above.

Before applying, disable the "Block Public Access" settings:

```bash
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

Apply the policy:

```bash
# Replace BUCKET_NAME in the file first, then:
aws s3api put-bucket-policy \
  --bucket $BUCKET_NAME \
  --policy file://bucket-policy.json
```

---

### Step 4 — Upload Your Build

```bash
# From the portfolio root directory
aws s3 sync out/ s3://$BUCKET_NAME --delete

# Verify
aws s3 ls s3://$BUCKET_NAME
```

At this point, your site is live at:  
`http://<BUCKET_NAME>.s3-website-<REGION>.amazonaws.com`

---

### Step 5 — Create a CloudFront Distribution

CloudFront gives you HTTPS, global CDN, and caching.

```bash
aws cloudfront create-distribution \
  --origin-domain-name "$BUCKET_NAME.s3-website-$REGION.amazonaws.com" \
  --default-root-object index.html \
  --query 'Distribution.{Id:Id,DomainName:DomainName,Status:Status}' \
  --output table
```

> **Note:** Use the S3 *website endpoint* (with `-website-` in the URL), **not** the plain S3 bucket URL. This ensures proper routing of `index.html` in subdirectories.

You'll get output like:

```
---------------------------------------------------------
|                  CreateDistribution                    |
+--------------+----------------------------------------+
| DomainName   | d1234abcdef8.cloudfront.net            |
| Id           | E1A2B3C4D5E6F7                         |
| Status       | InProgress                              |
+--------------+----------------------------------------+
```

Save the **Distribution ID** — you'll need it for cache invalidation and domain setup.

Wait for deployment (takes 5-15 minutes):

```bash
aws cloudfront wait distribution-deployed --id E1A2B3C4D5E6F7
echo "CloudFront is ready!"
```

Your site is now live at:  
`https://d1234abcdef8.cloudfront.net`

---

### Step 6 — Configure Error Pages (SPA Routing)

Since this is a single-page app, configure CloudFront to handle 404s gracefully:

```bash
aws cloudfront update-distribution \
  --id E1A2B3C4D5E6F7 \
  --if-match $(aws cloudfront get-distribution --id E1A2B3C4D5E6F7 --query 'ETag' --output text) \
  --distribution-config file://cloudfront-config.json
```

Or do this in the **AWS Console**:
1. Go to **CloudFront** → Your distribution → **Error Pages**
2. Create custom error response:
   - HTTP Error Code: **403**
   - Response Page Path: `/index.html`
   - HTTP Response Code: **200**
3. Repeat for error code **404**

---

### Step 7 — Redeploy (After Making Changes)

Every time you update the site:

```bash
# 1. Rebuild
yarn build

# 2. Sync to S3
aws s3 sync out/ s3://$BUCKET_NAME --delete

# 3. Invalidate CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id E1A2B3C4D5E6F7 \
  --paths "/*"
```

Or save this as a script — **`deploy.sh`**:

```bash
#!/bin/bash
set -e

BUCKET_NAME="ashutosh-portfolio-2026"
DISTRIBUTION_ID="E1A2B3C4D5E6F7"

echo "Building..."
yarn build

echo "Uploading to S3..."
aws s3 sync out/ s3://$BUCKET_NAME --delete

echo "Invalidating CloudFront cache..."
aws cloudfront create-invalidation \
  --distribution-id $DISTRIBUTION_ID \
  --paths "/*"

echo "Deployed! 🚀"
```

```bash
chmod +x deploy.sh
./deploy.sh
```

---

## 🌐 Custom Domain Setup

### Option A — Cloudflare (Recommended)

Cloudflare gives you free SSL, DDoS protection, and excellent DNS with global anycast.

#### A1. Add Your Domain to Cloudflare

1. Sign up at [cloudflare.com](https://www.cloudflare.com/)
2. Click **"Add a Site"** → enter your domain (e.g., `ashutosh.dev`)
3. Select the **Free** plan
4. Cloudflare will scan your existing DNS records

#### A2. Update Nameservers

Cloudflare will give you two nameservers like:

```
lila.ns.cloudflare.com
todd.ns.cloudflare.com
```

Go to your domain registrar (Namecheap, GoDaddy, etc.) and **replace the nameservers** with the Cloudflare ones.

- **Namecheap**: Domain List → Manage → Nameservers → Custom DNS → paste both
- **GoDaddy**: My Domains → DNS → Nameservers → Change → Custom

> DNS propagation takes 5 minutes to 48 hours. Usually under 30 minutes.

#### A3. Request an SSL Certificate from AWS ACM

CloudFront requires an SSL certificate from **AWS Certificate Manager (ACM)**.  
**Critical:** The certificate **must** be created in the **`us-east-1`** (N. Virginia) region.

```bash
aws acm request-certificate \
  --domain-name ashutosh.dev \
  --subject-alternative-names "*.ashutosh.dev" \
  --validation-method DNS \
  --region us-east-1 \
  --query 'CertificateArn' \
  --output text
```

This returns a Certificate ARN like:  
`arn:aws:acm:us-east-1:123456789:certificate/abc-def-ghi`

#### A4. Validate the Certificate via DNS

Get the CNAME validation record:

```bash
aws acm describe-certificate \
  --certificate-arn <YOUR_CERT_ARN> \
  --region us-east-1 \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
```

Output:

```json
{
  "Name": "_abc123.ashutosh.dev.",
  "Type": "CNAME",
  "Value": "_xyz789.acm-validations.aws."
}
```

Go to **Cloudflare DNS** → **Add Record**:

| Type | Name | Target | Proxy |
|---|---|---|---|
| CNAME | `_abc123` | `_xyz789.acm-validations.aws.` | **DNS Only** (grey cloud) |

Wait for validation (usually 5-15 minutes):

```bash
aws acm wait certificate-validated \
  --certificate-arn <YOUR_CERT_ARN> \
  --region us-east-1
echo "Certificate validated!"
```

#### A5. Add Domain to CloudFront

In the AWS Console:
1. **CloudFront** → Your distribution → **General** → **Edit**
2. **Alternate domain name (CNAME):** add `ashutosh.dev`
3. **Custom SSL certificate:** select the ACM certificate you just created
4. Save changes

Or via CLI:

```bash
# You'll need to update the distribution config JSON
# Add "Aliases" and "ViewerCertificate" fields
```

#### A6. Point Your Domain to CloudFront

Back in **Cloudflare DNS**, add:

| Type | Name | Target | Proxy |
|---|---|---|---|
| CNAME | `@` (or `ashutosh.dev`) | `d1234abcdef8.cloudfront.net` | **DNS Only** (grey cloud) |
| CNAME | `www` | `d1234abcdef8.cloudfront.net` | **DNS Only** (grey cloud) |

> **Important:** Set proxy status to **"DNS Only"** (grey cloud icon). Using Cloudflare's proxy (orange cloud) with CloudFront can cause SSL conflicts.

#### A7. Configure Cloudflare SSL Settings

In Cloudflare dashboard:
1. **SSL/TLS** → Set mode to **Full (Strict)**
2. **Edge Certificates** → Enable **Always Use HTTPS**

Your site is now live at `https://ashutosh.dev`

---

### Option B — Namecheap

If you bought your domain on Namecheap and want to use their DNS directly (without Cloudflare).

#### B1. Request ACM Certificate

Same as Step A3 above — create an ACM certificate in `us-east-1`.

#### B2. Validate via Namecheap DNS

1. Go to **Namecheap** → Domain List → Manage → **Advanced DNS**
2. Add a new **CNAME Record**:
   - Host: `_abc123` (the part before your domain)
   - Value: `_xyz789.acm-validations.aws.`
   - TTL: Automatic

Wait for AWS to validate (check ACM console or CLI).

#### B3. Add Domain to CloudFront

Same as Step A5 — add your domain as an alternate CNAME in CloudFront and attach the ACM certificate.

#### B4. Point Domain to CloudFront

In **Namecheap Advanced DNS**, add:

| Type | Host | Value | TTL |
|---|---|---|---|
| CNAME | `www` | `d1234abcdef8.cloudfront.net.` | Automatic |
| URL Redirect (301) | `@` | `https://www.ashutosh.dev` | — |

> **Why a redirect for `@`?** Namecheap doesn't support CNAME on the root domain (`@`). The redirect sends `ashutosh.dev` → `www.ashutosh.dev` which then resolves via CNAME to CloudFront. If you want the root domain to work directly, consider switching to Cloudflare (Option A) which supports CNAME flattening.

---

## 🔄 GitHub Actions CI/CD

Auto-deploy on every push to `main`. Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy Portfolio to AWS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'yarn'

      - name: Install dependencies
        run: yarn install --frozen-lockfile

      - name: Build
        run: yarn build

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1

      - name: Deploy to S3
        run: aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

**Add these secrets** in your GitHub repo → Settings → Secrets:

| Secret | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your IAM access key |
| `AWS_SECRET_ACCESS_KEY` | Your IAM secret key |
| `S3_BUCKET_NAME` | Your bucket name |
| `CLOUDFRONT_DISTRIBUTION_ID` | Your CloudFront distribution ID |

Now every `git push origin main` auto-deploys. DevOps deploying DevOps.

---

## 📁 Project Structure

```
portfolio/
├── app/
│   ├── layout.tsx            # Root layout — fonts, metadata, OG tags
│   ├── page.tsx              # Single page composing all sections
│   └── globals.css           # Tailwind v4 @theme + custom component styles
├── components/
│   ├── Navbar.tsx             # File-explorer tab navigation + scroll spy
│   ├── Hero.tsx               # Terminal boot sequence + typewriter effect
│   ├── About.tsx              # Bio, timeline, profile photo
│   ├── Skills.tsx             # 4 skill categories + certifications
│   ├── Projects.tsx           # Project cards with GitHub links
│   ├── Experience.tsx         # Git-log styled timeline
│   ├── Education.tsx          # JSON syntax-highlighted display
│   ├── Contact.tsx            # Social links + future form placeholder
│   ├── Footer.tsx             # Tech stack badges + social icons
│   └── ui/
│       ├── TerminalWindow.tsx # Reusable terminal card (title bar + dots)
│       ├── Typewriter.tsx     # Character-by-character typing animation
│       └── Icons.tsx          # Custom SVG brand icons (GitHub, LinkedIn, etc.)
├── lib/
│   └── utils.ts              # cn() utility for className merging
├── public/
│   └── profile.jpeg          # Profile photo
├── next.config.ts            # Static export + image config
├── tailwind.config.ts        # Tailwind theme extensions
├── tsconfig.json
├── package.json
└── README.md                 # You are here
```

---

## 🏗️ Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Framework | Next.js 16 (App Router) | SSG, file-based routing, React Server Components |
| Language | TypeScript | Type safety, better DX |
| Styling | Tailwind CSS v4 | Utility-first, `@theme` directive for tokens |
| Animations | Framer Motion 12 | Scroll reveals, hover effects, typewriter |
| Icons | Lucide React + Simple Icons | UI icons + DevOps tool logos |
| Fonts | Space Grotesk · JetBrains Mono · Inter | Headings · Code · Body |

---

## 🎨 Customization

### Colors

All theme colors live in `app/globals.css` under `@theme`:

```css
@theme {
  --color-terminal-bg: #1a1a2e;         /* Main background */
  --color-terminal-bg-light: #1e1e2f;   /* Card backgrounds */
  --color-terminal-bg-dark: #0d0d0d;    /* Darker sections */
  --color-terminal-accent: #f9bd2b;     /* Amber — primary CTA */
  --color-terminal-accent-2: #e05c2d;   /* Burnt orange — hover */
  --color-terminal-text: #f1f1f1;       /* Main text */
  --color-terminal-muted: #888888;      /* Secondary text */
  --color-terminal-border: #2a2a3e;     /* Borders */
}
```

### Content

Each section is a self-contained component. Edit directly:

| What to Change | File |
|---|---|
| Name, bio, GPA, timeline | `components/About.tsx` |
| Skill categories & levels | `components/Skills.tsx` |
| Projects, repos, tags | `components/Projects.tsx` |
| Work experience | `components/Experience.tsx` |
| Degree details | `components/Education.tsx` |
| Social links | `components/Contact.tsx` |
| Page title, SEO meta | `app/layout.tsx` |

### Adding a New Project

In `components/Projects.tsx`, add to the `projects` array:

```tsx
{
  title: "Your New Project",
  description: "What it does...",
  repo: "https://github.com/Ashut2/your-repo",
  tags: ["aws", "terraform", "docker"],
  status: "Active",
  period: "2026 - Present",
  highlights: [
    "Key achievement 1",
    "Key achievement 2",
  ],
},
```

---

## 🐛 Troubleshooting

| Problem | Solution |
|---|---|
| `yarn build` fails | Delete `.next/` and `node_modules/`, then `yarn install && yarn build` |
| Fonts not loading | Check `app/layout.tsx` — fonts load via `next/font/google` |
| Images broken on S3 | Ensure `next.config.ts` has `images: { unoptimized: true }` |
| CloudFront showing old content | Run `aws cloudfront create-invalidation --distribution-id <ID> --paths "/*"` |
| 404 on page refresh | Add custom error responses in CloudFront (403/404 → `/index.html` → 200) |
| ACM certificate stuck "Pending" | Verify CNAME record matches exactly. Check in `us-east-1` region only. |
| Namecheap root domain not working | Use URL redirect `@ → https://www.yourdomain.com` (CNAME not supported on root) |

---

## 📝 Scripts

```bash
yarn dev       # Start development server (hot reload)
yarn build     # Production build → out/ directory
yarn start     # Start production server (not needed for static)
yarn lint       # Run ESLint checks
```

---

## 📊 Cost Estimate (AWS)

| Service | Free Tier | After Free Tier |
|---|---|---|
| S3 | 5 GB storage, 20K GET/month | ~$0.02/month |
| CloudFront | 1 TB transfer, 10M requests/month | ~$0.085/GB after |
| ACM (SSL) | Free forever | Free forever |
| Route 53 (if used) | — | $0.50/hosted zone/month |

**For a portfolio site: effectively $0/month** within the free tier.

---

## 📄 License

Open source for personal and educational use.

---

**Built by Ashutosh Shukla** · CS Engineering · Abdul Kalam Technical University  
DevOps portfolio, built with DevOps practices.
