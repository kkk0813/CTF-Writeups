# Lab: HTTP request smuggling, basic CL.TE vulnerability

**Platform:** PortSwigger Web Security Academy  
**Category:** HTTP Request Smuggling  
**Difficulty:** Practitioner  

## Objective
The lab involves a front-end and back-end server that disagree on how to determine the length of an HTTP request body. The front-end server does not support chunked encoding, and it rejects any request that does not use the `GET` or `POST` method. The goal is to smuggle a request to the back-end so that the *next* request it processes appears to use the method `GPOST`.

## Analysis
This is a **CL.TE** desync: the **C**ontent-**L**ength header is authoritative for the front-end, while the **T**ransfer-**E**ncoding header is authoritative for the back-end.

Because HTTP/1 provides two different ways to mark where a request body ends, a request that includes *both* headers becomes ambiguous. When the two servers resolve that ambiguity differently, they disagree about where my request stops and the next one begins. That disagreement leaves a fragment of my request sitting in the back-end's buffer, where it gets prepended to whatever request arrives next.

* **Front-end behavior:** Honors `Content-Length`. It reads exactly the declared number of body bytes and forwards the whole thing downstream.
* **Back-end behavior:** Honors `Transfer-Encoding: chunked`. It stops reading at the terminating zero-length chunk (`0\r\n\r\n`) and treats anything after it as the start of a new request.

> Note: Although the lab supports HTTP/2, the intended solution requires HTTP/1 header ambiguity, so the request must be sent over HTTP/1.1. In Burp Repeater I switched the protocol via **Inspector → Request attributes → HTTP/1**. I also unchecked **Repeater → Update Content-Length** so Burp wouldn't silently "fix" my deliberately crafted length field.

## Payload & Execution

* **Injection Point:** The request body, exploiting the `Content-Length` / `Transfer-Encoding` mismatch.
* **Payload:**

```
POST / HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

G
```

### Execution Steps:
1. I captured a normal `GET /` request in Burp and sent it to Repeater, then converted it to a `POST` and downgraded the protocol to HTTP/1.1.
2. I crafted the request above, setting `Content-Length: 6` and adding a chunked body whose terminating `0` chunk is followed by a stray `G`.
3. I issued the request **twice** in succession (the lab reuses the same back-end connection via `Connection: keep-alive`).
4. The second response returned `Unrecognized method GPOST`, confirming the smuggled prefix had poisoned the next request and solving the lab.

## Why It Works (byte-level breakdown)
The body below the blank line is exactly **6 bytes**, which is what `Content-Length: 6` declares:

| Bytes | Content | Meaning |
| :--- | :--- | :--- |
| `0\r\n` | 3 bytes | A chunk of size 0 |
| `\r\n` | 2 bytes | Terminates the chunked body |
| `G` | 1 byte | The smuggled prefix |

* The **front-end** counts 6 bytes (`0\r\n\r\nG`), is satisfied, and forwards all of them to the back-end.
* The **back-end** parses the body as chunked. It sees the `0`-size chunk, decides the request has ended at `0\r\n\r\n`, and leaves the orphaned `G` in its buffer as the beginning of the next request.
* When my second (identical) request arrives, the back-end glues the leftover `G` onto the front of it, producing `GPOST / HTTP/1.1 ...`. There is no `GPOST` method, hence the `Unrecognized method GPOST` error.

## Mitigation
- Make the front-end and back-end agree on a single, unambiguous framing rule, or have the front-end normalize/reject requests that contain both `Content-Length` and `Transfer-Encoding`.
- Reject ambiguous requests outright rather than forwarding them.
- Where possible, use HTTP/2 end-to-end (no downgrading), since HTTP/2 determines message length from its own framing layer rather than from these headers.
