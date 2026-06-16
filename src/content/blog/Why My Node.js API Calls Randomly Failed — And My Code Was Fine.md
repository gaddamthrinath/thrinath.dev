---
title: "Why My Node.js API Calls Randomly Failed — And My Code Was Fine"
description: "A real debugging story about intermittent ETIMEDOUT errors, Cloudflare DNS, and a fix that took 5 minutes once I knew what to look for."
category: "backend"
tags: ["cloudflare", "dns", "network", "troubleshooting"]
publishedAt: 2026-06-14
slug: "nodejs-api-calls-randomly-failed"
---



*A real debugging story about intermittent ETIMEDOUT errors, Cloudflare DNS, and a fix that took 5 minutes once I knew what to look for.*

---

## What This Blog Will Help You With

When you call an external API in your backend, you are not fully in control. The API is someone else's server, someone else's infrastructure, someone else's DNS setup. When something goes wrong — **your first instinct is to check your own code**. That is natural. But a lot of the time, the problem is not in your code at all.

One very common pattern that trips up developers is this:

> A domain has **two DNS entries** pointing to two different IPs. One of those IPs is down or unreachable from your server. Your code randomly picks one IP on each request — sometimes it works, sometimes it doesn't. You spend hours debugging code that was never broken.

This blog will help you **recognise this pattern quickly** and give you a step-by-step process to debug it — using `curl` and Node.js — so you stop wasting time looking in the wrong place.

---

## The Problem

One day in production, I started seeing this error:

```
Error: connect ETIMEDOUT 172.67.XXX.XXX:443
  errno: 'ETIMEDOUT',
  code: 'ETIMEDOUT',
  syscall: 'connect',
  address: '172.67.XXX.XXX',
  port: 443
```

The weird part? It was **not always failing**.

- Same code ✅
- Same payload ✅
- Same API endpoint ✅
- Sometimes works, sometimes fails ❌

I spent a good amount of time staring at my code thinking I had a bug. I didn't.

---

## My Setup

I was calling a third-party API from my Node.js backend running on a VPS. Something like this:

```typescript
import request from 'request';

function callExternalApi(payload: object) {
    const options = {
        method: 'POST',
        uri: 'https://some-api.example.com/api/action',
        json: true,
        headers: {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer my-token'
        },
        body: payload
    };

    request(options, (err, response, body) => {
        if (err) {
            console.log('Error:', err); // ETIMEDOUT showing up here
            return;
        }
        console.log('Success:', body);
    });
}
```

Nothing unusual. Standard HTTP POST to a third-party API.

---

## Step 1 — Check if the Domain is Reachable at All

First thing I did was run `curl` from my VPS to see if the domain was even reachable:

```bash
curl -v https://some-api.example.com
```

Output:

```
* Host some-api.example.com:443 was resolved.
* IPv4: 172.67.XXX.XXX, 104.21.xxx.xxx
*   Trying 172.67.XXX.XXX:443...
*   Trying 104.21.xxx.xxx:443...
* Connected to some-api.example.com (104.21.xxx.xxx port 443)
< HTTP/1.1 200 OK
```

Two things jumped out:

1. The domain resolved to **two IP addresses** — `172.67.XXX.XXX` and `104.21.xxx.xxx`
2. curl connected successfully using `104.21.xxx.xxx`

So the domain worked fine in curl. But my Node.js app was randomly timing out. Why?

---

## Step 2 — Test Each IP Individually

curl has a handy flag `--connect-to` that forces it to use a specific IP while keeping the correct hostname in the request (so SSL still works). I tested each IP separately:

```bash
# Test the first IP
curl -v --connect-to some-api.example.com:443:172.67.XXX.XXX:443 https://some-api.example.com

# Test the second IP
curl -v --connect-to some-api.example.com:443:104.21.xxx.xxx:443 https://some-api.example.com
```

Results:

```
172.67.XXX.XXX  →  connection timed out ❌
104.21.xxx.xxx    →  200 OK ✅
```

Now I knew exactly what was happening. One IP worked, the other didn't — at least from my VPS.

---

## Step 3 — Confirm in Node.js with DNS Lookup

I wanted to see what my Node.js app was actually resolving. I added a quick DNS check:

```typescript
import * as dns from 'dns';

dns.lookup('some-api.example.com', { all: true }, (err, addresses) => {
    if (err) {
        console.log('DNS lookup error:', err);
        return;
    }
    console.log('Resolved IPs:', addresses);
    // Output: [ { address: '172.67.XXX.XXX', family: 4 }, { address: '104.21.xxx.xxx', family: 4 } ]
});
```

Both IPs showing up. So every time my code made a request, DNS randomly returned one of these two IPs. When it got the bad one — timeout. When it got the good one — success.

That explained the random failures perfectly.

---

## Why Does One Domain Have Two IPs?

This third-party API was sitting behind **Cloudflare**. Cloudflare is a proxy/CDN service that many companies use to protect and speed up their APIs.

When a company enables Cloudflare for their domain, Cloudflare puts itself in the middle:

```
Your app  →  some-api.example.com  →  Cloudflare  →  Real server
```

Cloudflare has servers all over the world. For load balancing and high availability, they give **multiple IPs** for the same domain. The idea is — if one goes down, the other takes over.

```
some-api.example.com
    ↓
172.67.XXX.XXX  (Cloudflare node A)  ❌ broken from my VPS region
104.21.xxx.xxx    (Cloudflare node B)  ✅ works fine
```

Both IPs are Cloudflare — not the real server. Cloudflare forwards the request internally. The real server IP is hidden.

The problem here is that `172.67.XXX.XXX` was either a dead/inactive Cloudflare node or was blocking connections from my VPS provider's IP range. Either way — unreachable from my server.

---

## Why Did curl Always Work But Node.js Didn't?

curl uses something called **Happy Eyeballs** — it tries multiple IPs at the same time and uses whichever responds first. So curl naturally avoided the bad IP.

Node.js `request` library picks one IP and waits. If DNS returns the bad IP first — it just sits there until timeout (which by default can be 20+ seconds).

---

## Fix 1 — Pin the Working IP in Code (No Cloudflare Access Needed)

If you are consuming a third-party API and have no access to their Cloudflare settings, this is your fix.

You can pass a custom `https.Agent` to the `request` library that always uses the working IP, while still sending the correct hostname in the request headers (so SSL works fine):

```typescript
import request from 'request';
import * as https from 'https';

function callExternalApi(payload: object) {
    const options: any = {
        method: 'POST',
        uri: 'https://some-api.example.com/api/action',
        json: true,
        headers: {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer my-token'
        },
        body: payload
    };

    // Only pin IP for this specific domain
    if (options.uri.includes('some-api.example.com')) {
        options.agent = new https.Agent({
            lookup: (hostname, opts, cb) => {
                // Always use the working IP, skip DNS entirely
                cb(null, '104.21.xxx.xxx', 4);
            }
        });
    }

    request(options, (err, response, body) => {
        if (err) {
            console.log('Error:', err);
            return;
        }
        console.log('Success:', body);
    });
}
```

This tells Node: *"Don't do a DNS lookup — just connect directly to `104.21.xxx.xxx`"*. But the `Host` header still says `some-api.example.com`, so the SSL certificate validates correctly.

> ⚠️ One caveat — if the API provider changes their Cloudflare IPs in the future, your hardcoded IP will stop working. Keep an eye on it and add retry logic as a safety net.

---

## Fix 2 — Correct the IP in Cloudflare (If You Have Access)

If you own the domain and have access to the Cloudflare dashboard, you can fix this properly:

1. Log into Cloudflare dashboard
2. Go to **DNS** settings for your domain
3. Find the A records for your domain
4. Remove the broken IP (`172.67.XXX.XXX`) and keep only the working one (`104.21.xxx.xxx`)

This is the cleaner fix if you control the domain. But if you are consuming someone else's API, you obviously won't have this access — use Fix 1.

---

## Bonus — Add Retry Logic as a Safety Net

Even with the pinned IP, it's good practice to add retry logic for network errors:

```typescript
function callExternalApiWithRetry(payload: object, retriesLeft = 3) {
    const options: any = {
        method: 'POST',
        uri: 'https://some-api.example.com/api/action',
        json: true,
        headers: {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer my-token'
        },
        body: payload,
        timeout: 30000,
        agent: new https.Agent({
            lookup: (hostname, opts, cb) => {
                cb(null, '104.21.xxx.xxx', 4);
            }
        })
    };

    request(options, (err, response, body) => {
        if (err) {
            const isTimeout = err.code === 'ETIMEDOUT' || err.code === 'ECONNRESET';
            if (isTimeout && retriesLeft > 0) {
                console.log(`Request failed, retrying... (${retriesLeft} attempts left)`);
                return callExternalApiWithRetry(payload, retriesLeft - 1);
            }
            console.log('Final error after retries:', err);
            return;
        }
        console.log('Success:', body);
    });
}
```

---

## What I Learned

**When you see intermittent ETIMEDOUT — don't blame your code first.**

Go through this checklist:

1. `curl -v` the domain from your server — does it resolve to multiple IPs?
2. Test each IP individually with `--connect-to` — which one fails?
3. Use `dns.lookup` in Node.js to confirm what IPs your app is resolving
4. Pin the working IP using a custom `https.Agent`
5. Add retry logic as a fallback

The `request` library is simple and widely used but has no built-in DNS failover or retry. Keep that in mind when calling external APIs in production.

---

*The bug that looks like your fault is often the infrastructure underneath it.*