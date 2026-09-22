# 07CTF 2026 — NeverEnding

## **Overview**

Our solution combined a proxy response-desynchronization bootstrap with a **Background Fetch download-budget side channel**.

We could not directly read the cross-origin flag-search response. However, its size affected whether the browser cancelled another download in the same Background Fetch job. By observing that cancellation on our own server, we obtained a prefix oracle and reconstructed the flag.

## **1. The search endpoint**

The API searched an inbox containing exactly one message: the flag.

```
q= request.args.get("q","")
results= [mfor min _INBOX if m.startswith(q)]
data= json.dumps({"results": results}).encode()
```

A nonmatching prefix returned:

```
{"results": []}
```

That response is exactly **15 bytes**. A matching prefix returned the entire flag inside the results array.

The API did not provide CORS permission to read the result. NeverEnding also changed the earlier Endless challenge’s Range handling, so our previous status/cache-consumer oracle no longer worked.

The new solution used ordinary GET requests without a Range header.

## **2. Getting a usable loopback-origin page**

The proxy rewrote script responses like this:

```
if (req.headers['sec-fetch-dest']=== 'script'){
  h['content-length']= '0';
  delete h['transfer-encoding'];
}

res.writeHead(proxyRes.statusCode,proxyRes.statusMessage,h);
proxyRes.pipe(res);
```

The response advertised an empty body, but the upstream body was still forwarded.

We supplied a delayed body containing a forged HTTP response. With concurrent script requests and connection reuse, those bytes were consumed as a subsequent JavaScript response from the application origin.

The bootstrap then:

1. Fetched **`/admin`** using the bot’s existing session cookie.
2. Extracted the API URL and API key.
3. Registered a service worker through the proxy.
4. Navigated to a page synthesized by that worker without a CSP header.

The resulting page ran on the bot’s loopback application origin and could invoke Background Fetch.

For NeverEnding, this required only the initial desynchronization and the synthetic-page transition. The second worker-desynchronization stage from our earlier Endless solution was unnecessary.

## **3. Turning hidden response size into an observable bit**

Background Fetch accepts a group of requests and a total download budget:

```
await registration.backgroundFetch.fetch(id,requests, {
  title:'Download',
  downloadTotal:budget
});
```

The relevant Chromium implementation separates:

- **Native byte accounting**, which counts actual downloaded bytes and cancels downloads when the budget is exceeded.
- **Cross-origin visibility filtering**, which hides response bodies and progress information from JavaScript.

We used the first layer’s effect on a resource we controlled.

Each job contained:

- One or more API search requests.
- A streaming marker response from our server.

The marker declared a **1025-byte** body, sent **1024 bytes immediately**, and delayed its last byte by **2200 ms**.

For one search request, we set:

```
downloadTotal = 1040
```

This produced the following distinction:

| **Search result** | **Accounting** | **Marker observation** |
| --- | --- | --- |
| Wrong prefix | 15 + 1025 = 1040 bytes | Finishes normally |
| Correct prefix | Larger response + first 1024 marker bytes exceeds budget | Connection closes before the last byte |

Our server recorded whether the marker completed or closed early. The page polled a CORS-enabled endpoint on our server to retrieve that result.

The flag-search body remained unreadable throughout.

A crucial implementation detail was to record the outcome **before explicitly aborting the job**. Otherwise, our own cleanup could look like a successful prefix match. Timeouts and transport failures were treated as inconclusive errors.

## **4. Recovering the flag**

We extended the predicate to groups of candidate prefixes.

For **`n`** queries, an all-miss group contributes exactly **`15*n`** API bytes, so the budget becomes:

```
downloadTotal = 1024 + 15*n + 1
```

If any query matches, the larger response causes early cancellation. Otherwise, the marker completes.

Because the inbox contains only one message, at most one candidate next character can match. We binary-partitioned the **95 printable ASCII characters**, then independently verified the selected next prefix before saving it.

Each fresh bot visit recalibrated:

- A known-positive saved prefix.
- A guaranteed-negative query formed by changing its first character.

The solve required multiple visits. When an instance generation changed, we preserved only the verified prefix, reacquired the API metadata, and recalibrated before continuing.

## **5. Verifying exact completion**

We did not stop merely because the recovered prefix ended in **`}`**.

For a candidate **`s`**, we calculated the exact JSON response length:

```
function serializedLength(s){
  return new TextEncoder().encode(
    '{"results": [' + JSON.stringify(s)+ ']}'
  ).length;
}
```

For the printable-ASCII strings in this solve, that matches the API’s Python JSON formatting.

We then tested adjacent budgets:

```
1024 + L - 1  -> marker closes before its final byte
1024 + L      -> marker reaches its final byte and finishes
```

This measures the **pre-tail cancellation threshold**, rather than whether the entire Background Fetch job ultimately reports success.

The final serialized result was **54 bytes**. We observed early closure at budget **1077** and marker completion at **1078**, confirming exact completion.

## **Result**

```
07CTF{woW_hOw_d1d_y0u_G37_TH3_flA9G9}
```

The remote acquisition used **14 bot visits, 285 oracle jobs, and 3208 API queries** across four instance generations. The first flag submission was accepted.

The key lesson is that **a hidden response can still influence shared resource accounting**, exposing information through a controlled sibling download.

Relevant Chromium source: [native byte accounting and cancellation](https://github.com/chromium/chromium/blob/150.0.7871.24/components/background_fetch/background_fetch_delegate_base.cc#L293-L325).