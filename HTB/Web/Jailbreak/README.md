# Jailbreak

**Platform:** Hack The Box (CTF Try Out)  
**Category:** Web Exploitation  
**Difficulty:** Very Easy  
**Concepts:** XML External Entity (XXE) Injection, Local File Disclosure, XML Entities, Config-File Parsing

## Objective
A "Firmware Update" page accepts an **XML configuration file** and parses it on the server. The goal is to abuse how the server reads that XML to make it open a file it should not — `/flag.txt` — and print the contents back to us.

## Background
A few ideas to know before the walkthrough:

* **XML** is a way to write structured data using tags, like `<Version>1.33.7</Version>`. Many systems accept XML config files.
* An **entity** in XML is like a variable. You define it once, then reuse it by writing `&name;`, and the parser swaps in its value.
* The dangerous part: an entity can be told to get its value from a **file on the server** using `SYSTEM "file:///..."`. If the server's XML reader allows this (many do by default), you can make it read secret files.
* This whole attack is called **XXE — XML External Entity injection.**

## Analysis
The page is themed like a Fallout "Pip-Boy" device — health bar, tabs, music — but almost all of that is decoration. The only part that matters is one hint and one input box:

* The hint says: *"Enter an update configuration file in the appropriate format (XML)."*
* The box is pre-filled with a `<FirmwareUpdateConfig>` XML document and a **Submit** button.

The important realization: **the server takes XML that we control and parses it.** Whenever a server parses user-supplied XML, XXE should be the first thing to test. The scenario even tells us the target file is at `/flag.txt`, which is exactly what XXE is good at reading.

Looking at the pre-filled XML, several fields (like `<Version>`) get echoed back on the page after submitting. That "our input comes back out" behavior is what makes the file contents visible to us once the entity expands.

## Payload & Execution
The plan has three simple parts:
1. **Declare an entity** at the top that points at the flag file.
2. **Reference that entity** (`&myentity;`) inside a field that gets displayed back.
3. **Submit** and read the reflected field.

I added a `DOCTYPE` line at the very top of the XML and placed `&myentity;` into the `<Version>` field:

```xml
<!DOCTYPE foo [ <!ENTITY myentity SYSTEM "file:///flag.txt"> ]>
<FirmwareUpdateConfig>
    <Firmware>
        <Version>&myentity;</Version>
        ...
    </Firmware>
    ...
</FirmwareUpdateConfig>
```

* `<!ENTITY myentity SYSTEM "file:///flag.txt">` creates an entity named `myentity` whose value is the contents of `/flag.txt`.
* `<Version>&myentity;</Version>` tells the parser to drop that file's contents where the version number normally goes.

After clicking **Submit**, the page displayed the file contents in place of the version:

```
Firmware version HTB{b1om3tr1c_l0cks_4nd_fl1cker1ng_l1ghts_7110f9bdcd2f82053450a433367470ae} update initiated.
```

> **Tip:** submitting through the browser worked here, but capturing the request in **Burp** or resending with `curl` (with `Content-Type: application/xml`) shows the raw response more clearly.

## Why It Works
The server's XML parser has **external entities turned on**. That means when it sees an entity defined with `SYSTEM "file:///flag.txt"`, it happily opens that file and uses its contents as the entity's value — no permission check, no questions asked.

Because the page then **echoes the `<Version>` field back to us**, and we planted the entity there, the file's contents get printed straight to the screen. Two conditions had to line up:
1. The parser expands external entities (the vulnerability).
2. A field we control is reflected in the response (so we can see the result).

Both were true, so the secret file came out.

## Real-World Meaning
XXE is not just a CTF trick — it is a real and common vulnerability, and this challenge points at a very realistic home for it:

* **Config-file and firmware parsers are a classic XXE target.** Real devices often accept an XML config or firmware descriptor and parse it with an old library that has external entities enabled by default. That is exactly the "firmware update config" setup shown here.
* **It leaks secret files.** An attacker can read things like `/etc/passwd`, private keys, config files with passwords, or in this case a flag — any file the server process can access.
* **It can do more than read files.** Depending on setup, XXE can also be used to make the server send requests to internal systems (SSRF) or cause denial of service. File reading is just the most common first step.

The lesson: any place a server parses XML you provided is a place to test for XXE.

## Mitigation
To stop XXE, the fix is on the parser, not the input:

1. **Disable external entities and DTDs** in the XML parser. This single setting removes the vulnerability. In Python, use a safe library like `defusedxml` instead of the default `xml` module.
2. **For other languages**, explicitly turn off DOCTYPE/entity processing (for example, setting `disallow-doctype-decl` to true in Java/PHP/Node parsers).
3. **Do not reflect raw parsed values** back to users when it is avoidable, so even a misconfigured parser has less to leak.
4. **Prefer safer formats** like JSON for config input when XML features are not actually needed.

## Related Write-ups
* [picoCTF – SOAP](../../../PicoCTF/Web_Exploitation/SOAP/) — the same XXE technique in a different wrapper. The `DOCTYPE` / `ENTITY` / `&entity;` pattern is identical.
