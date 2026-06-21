# Lab: HTTP request smuggling, basic TE.CL vulnerability

**Platform:** PortSwigger Web Security Academy  
**Category:** HTTP Request Smuggling  
**Difficulty:** Practitioner  

## Objective
The lab involves a front-end and back-end server that disagree on how to determine the length of an HTTP request body. The back-end server does not support chunked encoding, and the front-end rejects any request that does not use the `GET` or `POST` method. The goal is to smuggle a request to the back-end so that the *next* request it processes appears to use the method `GPOST`.

## Analysis
This is a **TE.CL** desync — the reverse of CL.TE. Here the **T**ransfer-**E**ncoding header is authoritative for the front-end, while the **C**ontent-**L**ength header is authoritative for the back-end.

* **Front-end behavior:** Honors `Transfer-Encoding: chunked`. It reads the chunked body, sees the terminating zero-length chunk, treats the whole thing as one valid request, and forwards the entire body downstream. It never realizes a second request is hidden inside the chunk data.
* **Back-end behavior:** Ignores `Transfer-Encoding` (no chunked support) and honors `Content-Length`. It reads only the declared number of body bytes for the first request, then parses everything after that as a brand-new request on the same keep-alive connection.

The elegance of TE.CL is the division of labor between the two numbers in the payload:
- The **chunk size** is aimed at the **front-end** (which reads chunked).
- The **`Content-Length`** is aimed at the **back-end** (which reads CL).

Each value fools a different server.

> Note: Although the lab supports HTTP/2, the intended solution requires HTTP/1 header ambiguity, so the request is sent over HTTP/1.1. In Burp Repeater I switched the protocol via **Inspector → Request attributes → HTTP/1** and unchecked **Repeater → Update Content-Length** so Burp would not rewrite my crafted length fields.

## Payload & Execution

* **Injection Point:** The request body, exploiting the `Transfer-Encoding` / `Content-Length` mismatch.
* **Payload:**

```
POST / HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 3
Transfer-Encoding: chunked

56
GPOST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 6

0

```

### Execution Steps:
1. I captured a `GET /` request in Burp, sent it to Repeater, converted it to `POST`, and downgraded the protocol to HTTP/1.1.
2. I crafted the request above: a chunk of size `0x56` (86 bytes) containing a complete smuggled `GPOST` request, followed by the terminating `0` chunk. The outer `Content-Length: 3` is the value the back-end will trust.
3. I issued the request **twice** in succession over the same keep-alive connection.
4. The second response returned `Unrecognized method GPOST`, confirming the smuggled request had poisoned the connection and solving the lab.

## Why It Works (byte-level breakdown)

**Front-end (reads chunked):**
- `56\r\n` declares a chunk of `0x56` = 86 bytes.
- The 86 bytes span the smuggled request line and its headers, up to and including the `Content-Length: 6\r\n` line.
- The `\r\n` that follows is the chunk's mandatory trailing CRLF.
- `0\r\n\r\n` is the terminating zero-length chunk.
- The front-end sees one complete, valid chunked request and forwards the whole body to the back-end.

**Back-end (reads `Content-Length: 3`):**

| Bytes | Content | Role |
| :--- | :--- | :--- |
| `5` `6` `\r` | 3 bytes | The entire body of request 1, per `Content-Length: 3` |
| everything after | — | Re-parsed as a new request on the same connection |

- Request 1 ends after 3 bytes. The back-end does **not** discard the rest — on a keep-alive connection it treats the leftover bytes as the start of the next request.
- The leftover begins with a stray `\n` (skipped as an empty line), then `GPOST / HTTP/1.1`, its headers, and `Content-Length: 6`.
- The same `\r\n` that the front-end used as a chunk terminator now serves as the blank line separating the smuggled request's headers from its body.
- The smuggled request needs **6 body bytes** but only has 5 available (`0\r\n\r\n`). The back-end blocks, waiting for one more byte.
- The next request's first byte completes the 6-byte body. Only then is the smuggled request fully assembled and processed, the method `GPOST` is rejected, and `Unrecognized method GPOST` is returned to that follow-up request.

### Tuning note: choosing the Content-Length value
`Content-Length: 3` is not special because it "stops at `\r`" — it simply reads 3 bytes. The value only has to land the cut so the `GPOST` request line survives intact at the front of request 2:
- **CL: 3** leaves `\nGPOST...` (leading blank line is skipped by the server). ✓
- **CL: 4** leaves `GPOST...` cleanly. ✓ (equivalent outcome)
- **CL: 5** eats the `G`, leaving a normal `POST` request — smuggling neutralized. ✗

## Mitigation
- Make the front-end and back-end agree on a single, unambiguous framing rule, or have the front-end normalize or reject requests containing both `Content-Length` and `Transfer-Encoding`.
- Reject ambiguous requests outright rather than forwarding them.
- Where possible, use HTTP/2 end-to-end (no downgrading), since HTTP/2 derives message length from its own framing layer rather than from these headers.
