# Move Your GA4 Tracking Server-Side. Measure What You've Been Missing.

[日本語版はこちら](README_ja.md)

This guide walks you through deploying server-side Google Tag Manager (sGTM) using Stape.io with Custom Loader. It documents the full setup path, the constraints we encountered with HubSpot and Cloudflare, and the recommendation we landed on based on real-world testing.

---

## Table of Contents

1. [What You'll Build](#1-what-youll-build)
2. [Architecture Overview](#2-architecture-overview)
3. [Why This Stack?](#3-why-this-stack)
4. [Prerequisites](#4-prerequisites)
5. [Setup Steps](#5-setup-steps)
6. [Verification Checklist](#6-verification-checklist)
7. [Known Limitations](#7-known-limitations)
8. [Related Documents](#8-related-documents)

---

## 1. What You'll Build

**Who this is for:** Teams running GA4 on a HubSpot site (or any SaaS-hosted site) who suspect ad blockers and browser privacy restrictions are causing data loss.

**What you get:**

- GTM script served from your own domain — not `googletagmanager.com`
- GA4 events routed through a server container, bypassing client-side blocking
- First-party cookies with extended lifetimes, reducing Safari ITP impact
- A working measurement baseline to evaluate actual data loss

You do not need to migrate your site or change your hosting provider. The GTM code swap is the only change required on the site itself.

---

## 2. Architecture Overview

### Standard Stape Setup (Subdomain)

```
Browser
  │
  ├── www.example.com                → HubSpot (unchanged)
  └── sgtm.example.com              → Cloudflare DNS (CNAME)
        └── your-container.stape.io  → Stape sGTM server container
              ├── /gtm.js           ← Custom Loader: script served from your subdomain
              └── /collect          ← GA4 events forwarded to Google
```

**Components:**

| Component | Role |
|-----------|------|
| GTM Web Container | Fires tags in the browser; routes events to the server container |
| Stape.io | Hosts the sGTM server container; manages Custom Loader and custom domains |
| Custom Loader | Serves the GTM script from your subdomain instead of `googletagmanager.com` |
| sgtm.example.com | Your subdomain, acting as the first-party endpoint for scripts and events |
| GA4 Server Tag | Receives events server-side and forwards them to Google Analytics |

### Attempted: Path-Based Same-Origin Setup

```
Browser
  └── example.com/sgtm/*  ──Cloudflare Worker──→  Stape sGTM
```

This would make all tracking requests appear same-origin, giving full first-party cookie treatment. See [Section 7](#7-known-limitations) for why this did not work with HubSpot.

---

## 3. Why This Stack?

### The Problem

Standard client-side GTM has three compounding issues:

1. **Ad blockers** intercept requests to `googletagmanager.com` and `google-analytics.com`, silently dropping events.
2. **Safari ITP** restricts JavaScript-set cookies to 7 days or less depending on tracking classification, breaking attribution for returning visitors.
3. **Third-party script blocking** — even without an explicit ad blocker, browsers increasingly restrict third-party scripts by default.

Server-side GTM moves the measurement logic off the client. Scripts load from your own domain, events are processed on your server, and cookies can be set server-side with longer lifetimes.

### Path-Based vs. Subdomain Routing

| Approach | ITP Protection | Complexity | HubSpot Compatible |
|----------|---------------|------------|-------------------|
| Subdomain (`sgtm.example.com`) | Partial | Low | Yes |
| Path-based (`example.com/sgtm/*`) | Full (same-origin) | Medium | **No** * |

\* HubSpot's Cloudflare for SaaS layer takes priority over custom Worker Routes — see [Section 7](#7-known-limitations).

The path-based approach is theoretically stronger — Safari treats `example.com/sgtm/` as the same origin as `example.com`, so no tracker classification applies. However, this requires routing through a reverse proxy on your main domain.

**HubSpot constraint:** HubSpot uses Cloudflare for SaaS internally. This means HubSpot's Cloudflare layer intercepts requests before your own Cloudflare Worker can fire. Path-based Worker routing on a HubSpot domain returns a 404 without ever reaching your Worker. See [sgtm-worker-issue.md](sgtm-worker-issue.md) for the full investigation.

### Our Recommendation

Use the subdomain configuration (`sgtm.example.com`) and go live. ITP's real-world impact on subdomain first-party cookies is limited — Safari restricts cookies only when it has classified the subdomain as a tracker. Start collecting data, and revisit path-based routing only if the GA4 data shows measurable attribution loss.

---

## 4. Prerequisites

| Requirement | Detail |
|-------------|--------|
| Google Tag Manager account | Web container already installed on your site |
| Stape.io account | Free tier is sufficient to start |
| Custom domain | You control DNS for this domain (e.g., via Cloudflare or your registrar) |
| Cloudflare account | Free tier; needed for DNS management of your custom subdomain |
| GA4 property | Measurement ID (format: `G-XXXXXXXXXX`) |

> If your domain DNS is not yet on Cloudflare, you will need to update your nameservers at your registrar. This does not transfer domain ownership — only DNS management moves to Cloudflare. See [cloudflare-nameserver-note.md](cloudflare-nameserver-note.md).

---

## 5. Setup Steps

Full details are in [sgtm-setup.md](sgtm-setup.md). The five-step summary:

**Step 1 — Create the GTM server container**
In GTM, create a new container with target platform "Server". Select "Manually provision tagging server" and copy the Container Config code.

**Step 2 — Provision the server on Stape.io**
Paste the Container Config into Stape to create your server container. Add your custom domain (`sgtm.example.com`) and enable **Custom Loader** under Power-ups. Set a Same Origin Path if desired (e.g., `lib`). Copy the new GTM install snippet Stape generates.

**Step 3 — Update your site**
Replace the standard GTM snippet on your site with the Custom Loader snippet from Stape. In your GTM Web container, open the Google Tag and add a configuration parameter: `server_container_url` = `https://sgtm.example.com`.

**Step 4 — Configure the server container**
Register your custom domain URL in the server container settings. Create an Event Data variable for `measurement_id`, an All Events trigger, and a GA4 server tag using both.

**Step 5 — (Optional) Cloudflare Worker for path-based routing**
If your site is not on HubSpot and you want full same-origin protection, add a Cloudflare Worker that proxies `example.com/sgtm/*` to your Stape container:

```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    url.hostname = "your-container.stape.io";
    return fetch(new Request(url, request));
  }
};
```

Set the Worker route to `example.com/sgtm/*`, update the Same Origin Path in Stape to `/sgtm`, and change `server_container_url` in GTM to `https://example.com/sgtm`.

---

## 6. Verification Checklist

Run these checks after each deployment step to confirm the setup is working end to end.

- [ ] **Script source**: In DevTools > Network, confirm `gtm.js` loads from `sgtm.example.com` (or your custom path), not from `googletagmanager.com`.
- [ ] **Event delivery**: Filter for `collect?` requests. Status should be `200` or `204`, and the request URL should point to your custom domain/path.
- [ ] **Server Preview**: Open GTM server container preview. Events should appear in the left-hand event list, and the GA4 tag should appear under "Tags Fired".
- [ ] **Health endpoint**: Visit `https://sgtm.example.com/health` in a browser. It should return `ok`.
- [ ] **GA4 DebugView**: With `?gtm_debug=x` appended to your URL, verify events appear in GA4's DebugView in real time.
- [ ] **Cookie inspection**: In DevTools > Application > Cookies, confirm `_ga` and `_ga_XXXXXXXX` cookies are present and set on your domain.

---

## 7. Known Limitations

### Path-Based Routing Does Not Work with HubSpot

The goal was to route `example.com/sgtm/*` through a Cloudflare Worker to Stape, achieving full same-origin status. This failed because:

- HubSpot uses **Cloudflare for SaaS** internally.
- On domains hosted by HubSpot, HubSpot's Cloudflare for SaaS layer takes precedence over custom Worker Routes within the same Cloudflare network.
- The response header `x-hs-cfworker-meta: {"resolver":"NotFoundResolver"}` confirms HubSpot's edge returned a 404 — the Worker never fired.

No workaround was found within the Cloudflare Workers / Route configuration. The only path to making this work would be hosting the site outside HubSpot (e.g., Cloudflare Pages, Vercel, Netlify), at which point you regain full control over routing.

See [sgtm-worker-issue.md](sgtm-worker-issue.md) for the complete investigation: DNS setup, Worker code, routes tested, and response header analysis.

### Subdomain ITP Risk Is Real but Limited

`sgtm.example.com` is still a subdomain. Safari's ITP can classify it as a cross-site tracker if it detects tracking behavior. In practice, this happens infrequently for first-party subdomains on the same registered domain, and the impact is limited to users on Safari without consent to tracking. The recommended approach is to launch on the subdomain configuration, monitor GA4 data, and escalate only if data shows a meaningful attribution gap.

---

## 8. Related Documents

| Document | Description |
|----------|-------------|
| [sgtm-setup.md](sgtm-setup.md) | Full step-by-step setup guide: GTM server container, Stape Custom Loader, Cloudflare Workers (JP) |
| [sgtm-worker-issue.md](sgtm-worker-issue.md) | Investigation: why Cloudflare Worker path-based routing fails on HubSpot sites (JP) |
| [cloudflare-nameserver-note.md](cloudflare-nameserver-note.md) | Why Cloudflare nameserver delegation is needed, and Stape domain configuration notes (JP) |
