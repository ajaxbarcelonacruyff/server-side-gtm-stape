# Server-Side GTM Setup Guide with Stape.io

A step-by-step guide for implementing server-side Google Tag Manager (sGTM) using Stape.io, with Custom Loader for maximum measurement accuracy and ITP mitigation.

---

## Table of Contents

1. [Prepare the Server Container (GTM)](#1-prepare-the-server-container-gtm)
2. [Infrastructure Setup and Custom Loader (Stape.io)](#2-infrastructure-setup-and-custom-loader-stapeio)
3. [Website Implementation Changes](#3-website-implementation-changes)
4. [Server Container Configuration](#4-server-container-configuration)
5. [What is Custom Loader?](#what-is-custom-loader)
6. [ITP Mitigation: Same-Domain Setup](#itp-mitigation-same-domain-setup)
7. [Verification Checklist](#verification-checklist)

---

## 1. Prepare the Server Container (GTM)

1. **Create a container**: In GTM, create a "New Container" and select **"Server"** as the target platform.
2. **Manual setup**: Select "Manually provision tagging server" and copy the displayed **"Container Config"** code.

## 2. Infrastructure Setup and Custom Loader (Stape.io)

1. **Register the container**: Log in to Stape.io, paste the copied code, and create the container.
2. **Custom domain configuration**:
   - Register a subdomain (e.g., `sgtm.example.com`) via **"Add Custom Domain"**.
   - Add the DNS record (A/CNAME) to your domain registrar or DNS provider.
3. **Enable Custom Loader**:
   - Go to the **"Power-ups"** tab in Stape and enable **"Custom Loader"**.
   - **Same Origin Path** (optional): Enter a path that blends in with your site (e.g., `metrics` or `lib`) for advanced obfuscation.
   - Copy the newly generated **installation code**.

## 3. Website Implementation Changes

1. **Replace the GTM code**: Swap out the standard GTM snippet on your site with the **Custom Loader code** obtained from Stape in Step 2.
2. **Update the Google Tag**:
   - Open the "Google Tag" in your Web container.
   - Add `server_container_url` to the configuration parameters.
   - Set the value to your **Stape custom domain URL** (e.g., `https://sgtm.example.com`).

## 4. Server Container Configuration

1. **Register the container URL**: Go to **Admin > Container Settings** and register the custom domain URL.
2. **Create a dynamic variable**: Under **Variables > Event Data**, set the path to `measurement_id` and save the variable.
3. **Create a trigger**: Under **Triggers > Custom**, create an `All Events` trigger selecting "All Events".
4. **Create a GA4 server tag**: Select "GA4" as the tag type, set the measurement ID to the variable created above, and assign the `All Events` trigger.

---

## What is Custom Loader?

Normally, the GTM script is loaded from `googletagmanager.com/gtm.js`. Custom Loader allows you to **serve it from your own subdomain** (e.g., `sgtm.example.com/xxxx.js`) instead.

### Key Benefits

- **Ad blocker bypass**: Since traffic goes to your own domain, GTM is far less likely to be blocked by tools like AdBlock.
- **ITP mitigation (extended cookie lifetime)**: Because the script origin and data destination share a first-party domain, browser cookie restrictions (e.g., Safari ITP) have less impact.
- **Fingerprint obfuscation**: GTM-specific file names and paths are hidden, improving data delivery reliability.

---

## ITP Mitigation: Same-Domain Setup

### Why Subdomains Are Not Enough

With the standard Stape setup, the sGTM server lives on a subdomain (e.g., `sgtm.example.com`). Safari's ITP treats subdomains as separate origins, so if the subdomain is classified as a tracker, cookies may still be restricted.

```
Browser → www.example.com       (main site)
        → sgtm.example.com      (Stape sGTM) ← treated as a separate origin
```

### True First-Party: Routing Through Your Main Domain

By routing sGTM traffic through a path on your **main domain**, the browser sees all requests as same-origin and grants full first-party cookie treatment.

```
Browser → www.example.com/analytics/collect  ──proxy──→  Stape sGTM
```

### Implementation Options

| Method | Difficulty | Requirement |
|--------|-----------|-------------|
| **Cloudflare Workers** (recommended) | ★☆☆ | Domain on Cloudflare |
| **Nginx/Apache reverse proxy** | ★★★ | Direct server access |
| **Other CDN reverse proxy** | ★★☆ | CDN with proxy rules (CloudFront, Vercel, Netlify, etc.) |

> For SaaS-hosted sites (HubSpot, Squarespace, etc.) without direct server access, Cloudflare Workers is the only practical option.

### Prerequisites & Cost

| Item | Detail |
|------|--------|
| Cloudflare plan | **Free** tier is sufficient |
| Site changes | GTM code swap only |
| DNS registrar changes | Nameserver update only (no domain transfer needed) |

### Step-by-Step: Cloudflare Workers Setup

#### ① Register Cloudflare and migrate DNS

1. Create a free account at [cloudflare.com](https://cloudflare.com).
2. Click "Add a Site" and enter your domain — Cloudflare **auto-imports** your existing DNS records.
3. Verify the imported records, especially the CNAME pointing to your hosting provider and any MX records for email.
4. At your domain registrar, change the nameservers to the two Cloudflare nameservers provided (e.g., `ns1.cloudflare.com`).

#### ② Configure SSL (critical)

- In the Cloudflare dashboard, go to **SSL/TLS** and set the mode to **"Full"**.
- Do **not** use "Flexible" — it causes an infinite redirect loop with most hosting providers.

#### ③ Create a Cloudflare Worker

1. Go to **Workers & Pages** → **Create Worker**.
2. Replace the default code with the following (substitute `your-container.stape.io` with your actual Stape container URL):

```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    url.hostname = "your-container.stape.io";
    return fetch(new Request(url, request));
  }
};
```

3. Save and deploy the Worker.

#### ④ Set up routing

- In the Worker settings, go to **Settings → Triggers → Add Route**.
- Set the route to `www.example.com/analytics/*`.

#### ⑤ Update Stape and GTM

- In Stape's Custom Loader settings, set **"Same Origin Path"** to `analytics`.
- In your Web container's Google Tag, update `server_container_url` to `https://www.example.com/analytics`.

Stape provides platform-specific Same Origin guides for Cloudflare, Nginx, Apache, AWS CloudFront, Azure Front Door, Vercel, Netlify, and more. See the [Stape Same Origin documentation](https://stape.io/helpdesk/documentation/guidelines/how-to-use-same-origin).

---

## Verification Checklist

- [ ] **Browser Console**: Confirm `gtm.js` is loading from your custom domain/path (not `googletagmanager.com`).
- [ ] **Network tab**: Verify `collect?` requests return status `200` or `204`, and the destination is your custom domain/path.
- [ ] **Server Preview**: Check that events appear in the left-hand list and that the GA4 tag shows under "Tags Fired".
- [ ] **Health check**: Access `https://[custom-domain]/health` and confirm it returns `ok`.
