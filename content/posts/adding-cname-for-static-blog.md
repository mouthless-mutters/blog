+++
title = "Adding a CNAME for Static Blog"
date = '2026-02-16T07:24:51-05:00'
draft = false
+++

In an earlier post I used Hugo and GitHub pages to create a static blog. I purchased a domain (mouthlessmutters.com) in GCP via Cloud Domains, and wanted to create a 'blog' subdomain and map it to the GitHub pages blog. Here are the steps I took:

## Step 1: Create a Public DNS Zone in Cloud DNS

After purchasing the domain in Google Cloud Domains, I needed to ensure that DNS was managed by Cloud DNS.

In the Google Cloud Console:

- Go to **Network Services → Cloud DNS**
- Click **Create Zone**
- Choose:
  - Type: Public
  - DNS name: `mouthlessmutters.com.`
  - DNSSEC: Off (important — see note below)

Once created, Cloud DNS generated four authoritative nameservers.

Then I went back to:

Cloud Domains → mouthlessmutters.com → Edit Nameservers

And selected **Use Cloud DNS** so that the domain would use the Cloud DNS zone I just created.

This step is critical. Without it, DNS records in Cloud DNS will not actually be used.

---

## Step 2: Add a CNAME Record for the Blog Subdomain

Inside the Cloud DNS zone, I added a new record:

- Type: `CNAME`
- Name: `blog`
- TTL: `300`
- Record data: `mouthless-mutters.github.io.`

Important details:

- The trailing dot on `github.io.` matters.
- Do not include `https://`
- The name `blog` creates `blog.mouthlessmutters.com`

This maps:

blog.mouthlessmutters.com → mouthless-mutters.github.io

---

## Step 3: Configure GitHub Pages Custom Domain

Next, I went to my repository:

Settings → Pages

Under **Custom domain**, I entered:

blog.mouthlessmutters.com

GitHub then performs a DNS check to verify that the CNAME is correctly configured.

If DNS is correct, it shows:

DNS check successful

---

## Step 4: Update Hugo baseURL

Since the blog was originally deployed under the GitHub Pages URL, I needed to update the Hugo configuration.

In `hugo.toml`, I changed:

baseURL = "https://mouthless-mutters.github.io/blog/"

to:

baseURL = "https://blog.mouthlessmutters.com/"

Then I committed and pushed the change so GitHub Actions would rebuild the site.

---

## Step 5: Wait for HTTPS Provisioning

Once DNS and GitHub were aligned, GitHub automatically requested a TLS certificate (via Let's Encrypt).

After a few minutes, the **Enforce HTTPS** option became available in the Pages settings.

Enabling this removed the certificate warning and completed the setup.

---

## A Note About DNSSEC

At one point, DNS validation failed with an `InvalidDNSError`.

The cause was DNSSEC being enabled in Cloud DNS without a corresponding DS record configured at the registrar level.

Disabling DNSSEC resolved the issue.

Unless you specifically need DNSSEC and understand the full trust chain configuration, it’s safer to leave it off for a small personal site.

---

## Final Result

The blog is now accessible at:

https://blog.mouthlessmutters.com

without redirects, without GitHub URLs in the address bar, and with HTTPS enforced.

---

This setup provides:

- Free hosting
- Automatic HTTPS
- Git-based publishing workflow
- Full control over the domain


TODO

