\---  
title: "requests — Codebase Analysis"  
tags:  
  - oss  
  - python  
  - requests  
  - reference  
source: "https://github.com/psf/requests"  
version: "2.34.2"  
\---  
  
\# requests  
  
Personal deep-dive notes on the \`psf/requests\` codebase, written as a reference for  
contributing. Everything below is derived from the source in \`src/requests/\` at version  
\`2.34.2\` (the modern, fully type-annotated layout with \`src/\` packaging and \`\_types.py\`).  
  
Scope note: \`playground.py\`, \`test.txt\`, and the temporary \`print()\` statements in the  
working tree are scratch artifacts and are deliberately excluded from this analysis.  
  
\## Component Overview  
  
Requests is a thin, opinionated, human-friendly layer on top of \`urllib3\`. It owns  
\*\*semantics\*\* (URL preparation, header/body encoding, cookies, auth, redirects, proxies,  
content decoding, exception taxonomy) and delegates \*\*transport\*\* (sockets, TLS, connection  
pooling, retries, chunked framing) to \`urllib3\`.  
  
\### The five layers  
  
| Layer | Module | Responsibility |  
| --- | --- | --- |  
| Functional API | \`api.py\` | One-shot convenience functions (\`get\`, \`post\`, ...). Creates a throwaway \`Session\`. |  
| Session | \`sessions.py\` | Persistent config, cookie jar, connection reuse, redirect loop, hook dispatch, adapter routing. |  
| Models | \`models.py\` | \`Request\` → \`PreparedRequest\` → \`Response\`. Where bytes are actually assembled and parsed. |  
| Transport adapter | \`adapters.py\` | \`HTTPAdapter\` bridges a \`PreparedRequest\` to a \`urllib3\` connection pool and back into a \`Response\`. |  
| Support | \`utils.py\`, \`cookies.py\`, \`auth.py\`, \`structures.py\`, \`exceptions.py\`, \`hooks.py\`, \`compat.py\`, \`\_types.py\`, \`\_internal\_utils.py\`, \`status\_codes.py\`, \`certs.py\`, \`packages.py\`, \`help.py\` | Everything the four layers above lean on. |  
  
\### Dependency direction  
  
\`\`\`  
api.py  
  └── sessions.py        ├── models.py ──── auth.py ──┐        │     ├── cookies.py         │        │     ├── structures.py      │        │     ├── utils.py ──────────┤        │     ├── exceptions.py      │        │     ├── hooks.py           │        │     └── status\_codes.py    │        └── adapters.py ─────────────┘                └── urllib3\`\`\`  
  
\`compat.py\`, \`\_internal\_utils.py\`, and \`\_types.py\` sit underneath everything and must stay  
import-cycle-free. \`cookies.py\` has an explicit warning at the top of the file: \`utils.py\`  
imports from it, so \`cookies.py\` must not import \`utils.py\`.  
  
\### Runtime dependencies  
  
Declared in \`pyproject.toml\`:  
  
\- \`urllib3>=1.26,<3\` — transport  
\- \`charset\_normalizer>=2,<4\` — encoding detection (or \`chardet\` via the \`use\_chardet\_on\_py3\` extra)  
\- \`idna>=2.5,<4\` — internationalised domain names  
\- \`certifi>=2023.5.7\` — CA bundle  
\- optional \`PySocks\` via the \`socks\` extra  
  
Python floor is 3.10. Type checking is \`pyright\` in \*\*strict\*\* mode over \`src/requests\`.  
  
\## Request Lifecycle End To End  
  
This is the single most useful thing to internalise before contributing. Trace of  
\`requests.get("https://httpbin.org/get", params={"a": 1})\`:  
  
\`\`\`  
requests.get(url, params=...)                       api.py:77  
└── requests.request("get", url, params=...)        api.py:24  
    └── with sessions.Session() as session:         api.py:71   (context manager → closes adapters)        └── Session.request(...)                    sessions.py:557            ├── Request(...)                        models.py:284   plain data holder            ├── Session.prepare\_request(req)        sessions.py:511            │   ├── merge\_cookies(...)              cookies.py:604  session jar + per-request cookies            │   ├── get\_netrc\_auth(url)             utils.py:231    only if trust\_env            │   └── PreparedRequest.prepare(...)    models.py:424            │       ├── prepare\_method              upper-cases the verb            │       ├── prepare\_url                 IDNA, param encoding, requote            │       ├── prepare\_headers             validates + CaseInsensitiveDict            │       ├── prepare\_cookies             builds the Cookie header            │       ├── prepare\_body                form/json/multipart/stream + Content-Length            │       ├── prepare\_auth                LAST, so auth sees the finished request            │       └── prepare\_hooks            ├── Session.merge\_environment\_settings  sessions.py:840  env proxies, REQUESTS\_CA\_BUNDLE            └── Session.send(prep, \*\*send\_kwargs)   sessions.py:756                ├── resolve\_proxies(...)            utils.py:911                ├── Session.get\_adapter(url)        sessions.py:879  longest-prefix match                ├── HTTPAdapter.send(request, ...)  adapters.py:637                │   ├── get\_connection\_with\_tls\_context  → urllib3 HTTPConnectionPool                │   ├── cert\_verify                      → CA bundle / client cert                │   ├── request\_url                      → path vs absolute (proxy)                │   ├── add\_headers                      → no-op hook for subclasses                │   ├── conn.urlopen(redirect=False, preload\_content=False, ...)                │   └── build\_response(request, resp)    → requests.Response                ├── dispatch\_hook("response", ...)  hooks.py:32                ├── extract\_cookies\_to\_jar(...)     cookies.py:135  persist Set-Cookie                ├── Session.resolve\_redirects(...)  sessions.py:186 the redirect loop                └── if not stream: r.content        forces the body to be read\`\`\`  
  
Key design decisions visible in that trace:  
  
\- \*\*\`redirect=False\` is passed to urllib3.\*\* Requests owns the redirect loop itself so that  
  it can strip auth across hosts, rewrite methods, rewind bodies, and record \`.history\`.  
\- \*\*\`preload\_content=False\` and \`decode\_content=False\`.\*\* The body is left in the urllib3  
  response so \`stream=True\` works; content decoding is done by Requests during  
  \`iter\_content\`.  
\- \*\*\`prepare\_auth\` runs last\*\* (see the comment at \`models.py:447\`) so that schemes like  
  OAuth can sign a fully-formed request, and it re-runs \`prepare\_content\_length\` afterwards  
  because auth may have changed the body.  
  
\## Package Layout  
  
\`\`\`  
src/requests/  
├── \_\_init\_\_.py           public surface + dependency version guards  
├── \_\_version\_\_.py        metadata constants  
├── api.py                get/post/put/... module-level functions  
├── sessions.py           Session, SessionRedirectMixin, merge helpers  
├── models.py             Request, PreparedRequest, Response, encoding mixins  
├── adapters.py           BaseAdapter, HTTPAdapter  
├── auth.py               AuthBase, HTTPBasicAuth, HTTPProxyAuth, HTTPDigestAuth  
├── cookies.py            RequestsCookieJar + cookiejar<->dict bridging  
├── structures.py         CaseInsensitiveDict, LookupDict  
├── utils.py              public-ish helpers (proxies, encodings, netrc, URL quoting)  
├── \_internal\_utils.py    tiny helpers with near-zero imports  
├── \_types.py             internal type aliases + TypedDicts (TYPE\_CHECKING-only mostly)  
├── exceptions.py         exception + warning hierarchy  
├── hooks.py              hook registry and dispatcher  
├── status\_codes.py       requests.codes lookup table  
├── compat.py             legacy re-export shim  
├── certs.py              CA bundle location (certifi)  
├── packages.py           requests.packages.\* backwards-compat aliases  
└── help.py               \`python -m requests.help\` bug-report dump  
\`\`\`  
  
\## Module Reference  
  
\### \`\_\_init\_\_.py\`  
  
The public surface plus a set of import-time guards.  
  
\*\*What it does, in order\*\*  
  
1\. Imports \`urllib3\` and tries to import \`charset\_normalizer\`, then \`chardet\`.  
2\. \`check\_compatibility()\` asserts \`urllib3 >= 1.21.1\`, \`chardet\` in \`\[3.0.2, 8.0.0)\`, or  
   \`charset\_normalizer\` in \`\[2.0.0, 4.0.0)\`. On mismatch it emits a  
   \`RequestsDependencyWarning\` rather than failing hard.  
3\. If \`ssl\` lacks SNI, it injects \`urllib3.contrib.pyopenssl\` and warns when \`cryptography\`  
   is older than 1.3.4.  
4\. Silences \`urllib3\`'s \`DependencyWarning\`, attaches a \`NullHandler\` to the \`requests\`  
   logger (so library users never see "No handler found"), and re-enables \`FileModeWarning\`  
   with \`simplefilter("default", ..., append=True)\`.  
5\. Re-exports the public names and pins them in \`\_\_all\_\_\`.  
  
\*\*Why the \`NullHandler\` matters\*\*  
  
\`\`\`python  
import logging  
logging.basicConfig(level=logging.DEBUG)   # user opts in  
logging.getLogger("urllib3").setLevel(logging.DEBUG)  
\# Without opting in, requests stays silent because of the NullHandler.  
\`\`\`  
  
\*\*Contribution note.\*\* This file is exempted from \`E402\`/\`F401\` in \`pyproject.toml\` because  
the ordering of the guards is load-bearing — imports genuinely must happen after the  
compatibility checks. Do not "clean up" the import order here.  
  
\### \`\_\_version\_\_.py\`  
  
Pure metadata. \`pyproject.toml\` reads \`version\` dynamically from  
\`requests.\_\_version\_\_.\_\_version\_\_\`, so this file is the single source of truth for the  
released version.  
  
\`\`\`python  
\_\_version\_\_ = "2.34.2"  
\_\_build\_\_   = 0x023402      # packed hex form of the version  
\_\_cake\_\_    = "✨ 🍰 ✨"     # yes, this is a real public attribute  
\`\`\`  
  
\`utils.default\_user\_agent()\` builds \`python-requests/2.34.2\` from \`\_\_version\_\_\`.  
  
\### \`api.py\`  
  
Seven thin wrappers plus \`request()\`. Every one of them creates a \*\*brand new \`Session\`\*\*  
inside a \`with\` block:  
  
\`\`\`python  
with sessions.Session() as session:  
    return session.request(method=method, url=url, \*\*kwargs)\`\`\`  
  
The comment above it explains why: without the context manager, sockets leak and surface as  
\`ResourceWarning\` or an apparent memory leak.  
  
\*\*Consequences you should know\*\*  
  
\- No connection pooling across \`requests.get()\` calls. For anything repeated, use a session:  
  
  \`\`\`python  
  import requests  
  
  # slow: new TCP + TLS handshake every call  for page in range(10):      requests.get(f"https://api.example.com/items?page={page}")  
  # fast: one pooled connection, shared cookies/headers  with requests.Session() as s:      s.headers\["Authorization"\] = "Bearer …"      for page in range(10):          s.get(f"https://api.example.com/items?page={page}")  
  \`\`\`  
\- No cookie persistence across calls.  
  
\*\*Per-function quirks\*\*  
  
| Function | Notable behaviour |  
| --- | --- |  
| \`head()\` | \`kwargs.setdefault("allow\_redirects", False)\` — the only method that defaults redirects off. |  
| \`get()\` | Signature exposes \`params\` positionally. |  
| \`post()\` | Exposes both \`data\` and \`json\` positionally. |  
| \`put()\` / \`patch()\` | Expose \`data\` positionally, \`json\` only via kwargs. |  
| \`options()\` / \`delete()\` | Plain passthrough. |  
  
\*\*Typing pattern.\*\* These signatures use PEP 692 \`Unpack\[TypedDict\]\`:  
  
\`\`\`python  
def get(url: \_t.UriType, params: \_t.ParamsType = None,  
        \*\*kwargs: Unpack\[\_t.GetKwargs\]) -> Response: ...  
\`\`\`  
  
\`GetKwargs\` deliberately omits \`params\` (already a named parameter) while \`PostKwargs\`  
omits \`data\`/\`json\`. If you add a new keyword to \`Session.request\`, you must also add it to  
the right TypedDict in \`\_types.py\` or the strict pyright run will not see it.  
  
\### \`sessions.py\`  
  
The orchestration layer. Two module-level helpers, one mixin, one class.  
  
\#### \`merge\_setting(request\_setting, session\_setting, dict\_class=OrderedDict)\`  
  
The precedence rule for every session-vs-request setting:  
  
\- session value \`None\` → use request value  
\- request value \`None\` → use session value  
\- either side is not a \`Mapping\` (e.g. \`verify=False\`) → request wins outright  
\- both are mappings → merge, request keys overriding session keys  
\- \*\*keys whose merged value is \`None\` are deleted\*\* — this is how you unset a session header  
  
\`\`\`python  
s = requests.Session()  
s.headers\["X-Trace"\] = "on"  
  
s.get(url)                                  # sends X-Trace: on  
s.get(url, headers={"X-Trace": "off"})      # sends X-Trace: off  
s.get(url, headers={"X-Trace": None})       # sends no X-Trace header at all  
\`\`\`  
  
\#### \`merge\_hooks(request\_hooks, session\_hooks, dict\_class=OrderedDict)\`  
  
A special case because \`default\_hooks()\` returns \`{"response": \[\]}\`, and naive merging of an  
empty list would wipe out session hooks. So: if either side is empty/\`{"response": \[\]}\`, the  
other side is returned untouched.  
  
\#### \`SessionRedirectMixin\`  
  
All redirect machinery lives here so it can be overridden independently of \`Session\`.  
  
\*\*\`get\_redirect\_target(resp)\`\*\* — returns the \`Location\` header, re-encoded from latin-1 to  
utf-8. The comment explains it: \`http.client\` decodes headers as latin-1, but real-world  
servers overwhelmingly send utf-8, so the round-trip \`location.encode("latin1")\` then  
\`to\_native\_string(location, "utf8")\` recovers the intended bytes.  
  
\*\*\`should\_strip\_auth(old\_url, new\_url)\`\*\* — the security-critical one:  
  
| Redirect | Auth stripped? | Why |  
| --- | --- | --- |  
| \`http://a.com\` → \`http://b.com\` | yes | different host |  
| \`http://a.com\` → \`https://a.com\` | no | explicit http→https upgrade allowance on default ports |  
| \`https://a.com\` → \`http://a.com\` | yes | downgrade changes scheme |  
| \`http://a.com:80\` → \`http://a.com\` | no | both are the scheme default port |  
| \`http://a.com\` → \`http://a.com:8080\` | yes | port changed |  
  
\*\*\`resolve\_redirects(...)\`\*\* — a generator, called in a list comprehension from  
\`Session.send\`. Per iteration:  
  
1\. \`req.copy()\` — never mutate the caller's \`PreparedRequest\`.  
2\. Append the previous response to \`hist\`, consume \`resp.content\` to release the socket  
   (falling back to \`resp.raw.read(decode\_content=False)\` on decoding errors).  
3\. Raise \`TooManyRedirects\` once \`len(resp.history) >= self.max\_redirects\` (default 30 from  
   \`models.DEFAULT\_REDIRECT\_LIMIT\`).  
4\. Fix up the URL: scheme-relative \`//host/path\` (RFC 1808 §4), carry the previous fragment  
   forward if the new URL has none (RFC 7231 §7.1.2), \`urljoin\` relative locations, and  
   \`requote\_uri\`.  
5\. \`rebuild\_method(...)\`.  
6\. For anything that is \*\*not\*\* 307/308, drop \`Content-Length\`, \`Content-Type\`,  
   \`Transfer-Encoding\` and set \`body = None\`. (Issues #1084 and #3490.)  
7\. Drop the \`Cookie\` header, re-extract cookies from the response into the copied jar, merge  
   session cookies, and re-run \`prepare\_cookies\`.  
8\. \`rebuild\_proxies\` and \`rebuild\_auth\`.  
9\. Rewind a file-like body if \`\_body\_position\` was recorded, otherwise raise  
   \`UnrewindableBodyError\` — note the trick where a failed \`tell()\` stores a sentinel  
   \`object()\` so the code can raise instead of silently hanging.  
  
\*\*\`rebuild\_method(prepared\_request, response)\`\*\* — the RFC-vs-browsers table:  
  
| Status | Original method | New method | Rationale |  
| --- | --- | --- | --- |  
| 303 See Other | anything but HEAD | GET | RFC 7231 §6.4.4 |  
| 302 Found | anything but HEAD | GET | what browsers do, not what the RFC says |  
| 301 Moved | POST | GET | issue #1704 |  
| 307 / 308 | unchanged | unchanged | method and body preserved by definition |  
  
\*\*\`rebuild\_proxies\`\*\* — deletes any existing \`Proxy-Authorization\` header, re-resolves  
proxies against the new URL (respecting \`NO\_PROXY\`), and re-adds \`Proxy-Authorization\`  
\*\*only for non-HTTPS schemes\*\*, because on HTTPS the credentials would be leaked inside the  
TLS tunnel instead of being consumed by the \`CONNECT\`.  
  
\#### \`Session\`  
  
State initialised in \`\_\_init\_\_\`:  
  
| Attribute | Default | Notes |  
| --- | --- | --- |  
| \`headers\` | \`utils.default\_headers()\` | UA, \`Accept-Encoding\`, \`Accept: \*/\*\`, \`Connection: keep-alive\` |  
| \`auth\` | \`None\` | tuple, \`AuthBase\`, or any callable |  
| \`proxies\` | \`{}\` | scheme or \`scheme://host\` keyed |  
| \`hooks\` | \`{"response": \[\]}\` | |  
| \`params\` | \`{}\` | merged into every request's query string |  
| \`stream\` | \`False\` | |  
| \`verify\` | \`True\` | bool or CA bundle path |  
| \`cert\` | \`None\` | client cert path or \`(cert, key)\` |  
| \`max\_redirects\` | \`30\` | |  
| \`trust\_env\` | \`True\` | gates netrc, env proxies, \`REQUESTS\_CA\_BUNDLE\` |  
| \`cookies\` | \`RequestsCookieJar()\` | |  
| \`adapters\` | \`{"https://": HTTPAdapter(), "http://": HTTPAdapter()}\` | |  
  
\*\*\`mount(prefix, adapter)\`\*\* keeps \`self.adapters\` sorted by descending prefix length, and  
\`get\_adapter(url)\` returns the first prefix that \`url.lower().startswith(...)\`. That is how  
host-specific adapters work:  
  
\`\`\`python  
class RetryAdapter(requests.adapters.HTTPAdapter):  
    pass  
s = requests.Session()  
s.mount("https://flaky.example.com", RetryAdapter(max\_retries=5))  
\# longer prefix wins over the default "https://"  
\`\`\`  
  
If nothing matches, \`InvalidSchema\` is raised — that is the error you get for  
\`requests.get("ftp://…")\`.  
  
\*\*\`send()\` order of operations\*\* (this is where hook authors get tripped up):  
  
1\. Defaults filled in from the session (\`stream\`, \`verify\`, \`cert\`, \`proxies\`).  
2\. Guard: passing a \`Request\` instead of a \`PreparedRequest\` raises  
   \`ValueError("You can only send PreparedRequests.")\`.  
3\. Adapter lookup, \`preferred\_clock()\` timestamp (\`time.perf\_counter\` on Windows,  
   \`time.time\` elsewhere), \`adapter.send(...)\`.  
4\. \`r.elapsed\` is set \*\*before\*\* hooks and \*\*before\*\* redirects — it measures only the first  
   response's header round-trip, not body download.  
5\. \`dispatch\_hook("response", ...)\`.  
6\. Cookies persisted from \`r.history\` first, then from \`r\`.  
7\. Redirects resolved only if \`allow\_redirects\`; otherwise \`r.\_next\` is populated from  
   \`resolve\_redirects(..., yield\_requests=True)\` so \`Response.next\` works.  
8\. \`if not stream: r.content\` — this is what makes non-streaming responses eager.  
  
\*\*\`merge\_environment\_settings()\`\*\* — when \`trust\_env\`, injects \`getproxies()\` results via  
\`setdefault\` (explicit proxies always win) and honours \`REQUESTS\_CA\_BUNDLE\` then  
\`CURL\_CA\_BUNDLE\`.  
  
\*\*Pickling.\*\* \`\_\_getstate\_\_\`/\`\_\_setstate\_\_\` copy the \`\_\_attrs\_\_\` list. Note that \`adapters\`  
is in \`\_\_attrs\_\_\`, which is why \`HTTPAdapter\` also implements \`\_\_getstate\_\_\`/\`\_\_setstate\_\_\`  
and rebuilds its pool manager on unpickle (the pool manager itself holds an unpickleable  
lambda).  
  
\`session()\` at the bottom of the file is a deprecated 1.0-era factory kept for  
compatibility.  
  
\### \`models.py\`  
  
The heart of the library.  
  
\#### Constants  
  
\`\`\`python  
REDIRECT\_STATI          = (301, 302, 303, 307, 308)  
DEFAULT\_REDIRECT\_LIMIT  = 30  
CONTENT\_CHUNK\_SIZE      = 10 \* 1024   # used by Response.content  
ITER\_CHUNK\_SIZE         = 512         # default for iter\_lines  
\`\`\`  
  
The \`import encodings.idna  # noqa: F401\` at the top is not decorative — issue #3578 showed  
that a lazy IDNA import inside a thread can raise \`LookupError\` when the stdlib is zipped  
(Embedded Python).  
  
\#### \`RequestEncodingMixin\`  
  
\*\*\`path\_url\`\*\* builds the origin-form request target: \`path\` (defaulting to \`/\`) plus  
\`?query\`. Deliberately excludes the fragment.  
  
\*\*\`\_encode\_params(data)\`\*\* — four overloads, one implementation:  
  
| Input | Output |  
| --- | --- |  
| \`str\` / \`bytes\` | returned as-is |  
| object with \`.read\` | returned as-is (streamed) |  
| anything iterable | \`urlencode(pairs, doseq=True)\` |  
  
The value-flattening loop is what makes multi-value params work, and \`None\` values are  
dropped:  
  
\`\`\`python  
requests.get("https://x/", params={"k": \["a", "b"\], "skip": None})  
\# → https://x/?k=a&k=b  
\`\`\`  
  
\*\*\`\_encode\_files(files, data)\`\*\* — builds \`multipart/form-data\` using  
\`urllib3.fields.RequestField\` and \`encode\_multipart\_formdata\`. Accepted file specs:  
  
\`\`\`python  
files = {  
    "f1": open("a.txt", "rb"),                                    # bare file object    "f2": ("name.txt", b"contents"),                              # 2-tuple    "f3": ("name.csv", b"a,b", "text/csv"),                       # 3-tuple    "f4": ("n.bin", b"\\x00", "application/octet-stream",           {"X-Extra": "1"}),                                     # 4-tuple with headers}  
requests.post(url, data={"field": "value"}, files=files)  
\`\`\`  
  
Guards: raises \`ValueError("Files must be provided.")\` if empty and  
\`ValueError("Data must not be a string.")\` if \`data\` is a string (you cannot mix a raw body  
with multipart).  
  
\#### \`RequestHooksMixin\`  
  
\`register\_hook(event, hook)\` accepts a single callable or an iterable of them and raises  
\`ValueError\` for unknown events (only \`"response"\` exists today). \`deregister\_hook\` returns  
a bool rather than raising.  
  
\#### \`Request\`  
  
A dumb, mutable data holder — no network, no validation. \`\_\_init\_\_\` normalises \`None\` to  
empty containers and routes \`hooks\` through \`register\_hook\`. \`prepare()\` returns a  
\`PreparedRequest\`.  
  
\`\`\`python  
req = requests.Request("POST", "https://httpbin.org/post", json={"a": 1})  
prep = req.prepare()          # no session settings merged  
prep = requests.Session().prepare\_request(req)   # session settings merged  
\`\`\`  
  
\#### \`PreparedRequest\`  
  
Attributes: \`method\`, \`url\`, \`headers\` (a \`CaseInsensitiveDict\`), \`body\`, \`\_cookies\`,  
\`hooks\`, \`\_body\_position\`.  
  
\*\*\`prepare\_method\`\*\* — \`to\_native\_string(method.upper())\`.  
  
\*\*\`prepare\_url(url, params)\`\*\* — the most bug-report-dense function in the codebase:  
  
1\. Decode bytes as utf-8, otherwise \`str(url)\` (supports objects with \`\_\_str\_\_\`).  
2\. \`url.lstrip()\` — leading whitespace tolerance.  
3\. \*\*Non-HTTP scheme escape hatch:\*\* if the URL contains \`:\` and does not start with  
   \`http\`, it is stored verbatim and returned. That is how \`mailto:\` and \`data:\` URLs pass  
   through untouched (they are not RFC 3986 parseable by \`urllib3.util.parse\_url\`).  
4\. \`parse\_url(url)\`; \`LocationParseError\` → \`InvalidURL\`.  
5\. No scheme → \`MissingSchema\` with the friendly \`"Perhaps you meant https://…?"\` message.  
   No host → \`InvalidURL\`.  
6\. Non-ASCII host → IDNA-encode via \`idna.encode(host, uts46=True)\`; failure →  
   \`InvalidURL("URL has an invalid label.")\`. ASCII hosts starting with \`\*\` or \`.\` are  
   rejected outright.  
7\. Netloc is reconstructed carefully as \`auth@host:port\`.  
8\. Empty path becomes \`/\`.  
9\. Params are encoded and appended with \`&\` if a query already exists.  
10\. \`requote\_uri(urlunparse(...))\` normalises percent-encoding.  
  
\`\`\`python  
p = requests.Request("GET", "https://пример.рф/путь", params={"q": "ü"}).prepare()  
p.url  
\# 'https://xn--e1afmkfd.xn--p1ai/%D0%BF%D1%83%D1%82%D1%8C?q=%C3%BC'  
\`\`\`  
  
\*\*\`prepare\_headers(headers)\`\*\* — every header goes through  
\`utils.check\_header\_validity\`, which rejects leading whitespace, \`:\` in names, and CR/LF in  
either part. This is the CRLF-injection defence.  
  
\`\`\`python  
requests.get(url, headers={"X-Bad": "value\\r\\nInjected: 1"})  
\# raises requests.exceptions.InvalidHeader  
\`\`\`  
  
\*\*\`prepare\_body(data, files, json)\`\*\* — the decision tree:  
  
| Condition | Body | Content-Type set | Length header |  
| --- | --- | --- | --- |  
| \`json\` given and no \`data\` | \`json.dumps(json, allow\_nan=False).encode()\` | \`application/json\` | via \`prepare\_content\_length\` |  
| \`data\` is a stream/generator (iterable but not str/bytes/list/tuple/Mapping) | the object itself | none | \`Content-Length\` if \`super\_len\` works, else \`Transfer-Encoding: chunked\` |  
| \`files\` given | \`\_encode\_files(...)\` | \`multipart/form-data; boundary=…\` | computed |  
| \`data\` is a mapping/list of pairs | \`\_encode\_params(...)\` | \`application/x-www-form-urlencoded\` | computed |  
| \`data\` is str/bytes or a file object | raw | \*\*none\*\* (caller's responsibility) | computed |  
  
Two details worth remembering:  
  
\- \`allow\_nan=False\` means \`float("nan")\` in a JSON body raises \`InvalidJSONError\` rather  
  than emitting invalid JSON.  
\- Streamed bodies plus \`files\` raises  
  \`NotImplementedError("Streamed bodies and files are mutually exclusive.")\`.  
\- \`\_body\_position\` is recorded from \`body.tell()\` so redirects can rewind; if \`tell()\`  
  raises \`OSError\`, a sentinel \`object()\` is stored so the later rewind fails loudly.  
  
\*\*\`prepare\_content\_length(body)\`\*\* — sets \`Content-Length\` when a length is computable, and  
sets \`Content-Length: 0\` for non-\`GET\`/\`HEAD\` methods with no body (so servers do not hang  
waiting).  
  
\*\*\`prepare\_auth(auth, url)\`\*\* — falls back to credentials embedded in the URL  
(\`https://user:pass@host/\`), special-cases a 2-tuple into \`HTTPBasicAuth\`, calls the  
handler, then \`self.\_\_dict\_\_.update(r.\_\_dict\_\_)\` to absorb whatever the handler changed, and  
finally recomputes \`Content-Length\`.  
  
\*\*\`prepare\_cookies(cookies)\`\*\* — documented as \*\*effectively single-use\*\*: cookielib will  
not regenerate the header if \`Cookie\` is already present. Redirect handling explicitly pops  
the \`Cookie\` header before re-preparing (see \`resolve\_redirects\` step 7).  
  
\*\*\`copy()\`\*\* — shallow, but the cookie jar is deep-copied via \`\_copy\_cookie\_jar\`.  
  
\#### \`Response\`  
  
\`\_\_attrs\_\_\` defines what is pickled. \`\_\_getstate\_\_\` first forces \`self.content\` so the body  
survives pickling; \`\_\_setstate\_\_\` sets \`raw = None\` and \`\_content\_consumed = True\`.  
  
\*\*Truthiness.\*\* \`\_\_bool\_\_\`/\`\_\_nonzero\_\_\` delegate to \`ok\`, which delegates to  
\`raise\_for\_status()\`. So \`if r:\` means "status < 400", \*\*not\*\* "status == 200". This is a  
classic source of confusion:  
  
\`\`\`python  
r = requests.get(url)  
if r:            # True for 204, 301, 399 …  
    ...if r.status\_code == 200:   # what people usually mean  
    ...\`\`\`  
  
\*\*Properties\*\*  
  
| Property | Meaning |  
| --- | --- |  
| \`ok\` | \`raise\_for\_status()\` did not raise |  
| \`is\_redirect\` | has \`Location\` \*\*and\*\* status in \`REDIRECT\_STATI\` |  
| \`is\_permanent\_redirect\` | has \`Location\` and status in (301, 308) |  
| \`next\` | the \`PreparedRequest\` for the next hop when \`allow\_redirects=False\` |  
| \`apparent\_encoding\` | \`chardet\`/\`charset\_normalizer\` guess, or \`"utf-8"\` if neither is installed |  
| \`links\` | parsed \`Link:\` header, keyed by \`rel\` |  
  
\*\*\`iter\_content(chunk\_size=1, decode\_unicode=False)\`\*\*  
  
\- Uses \`self.raw.stream(chunk\_size, decode\_content=True)\` when available — note that  
  \`decode\_content=True\` here is what actually gunzips the body, since the adapter passed  
  \`decode\_content=False\` to \`urlopen\`.  
\- Translates urllib3 errors into the Requests taxonomy: \`ProtocolError\` →  
  \`ChunkedEncodingError\`, \`DecodeError\` → \`ContentDecodingError\`, \`ReadTimeoutError\` →  
  \`ConnectionError\`, \`SSLError\` → \`requests.exceptions.SSLError\`.  
\- Re-iterating an already-consumed streamed response raises \`StreamConsumedError\`; if the  
  content was buffered, it replays it via \`iter\_slices\`.  
\- \`chunk\_size=None\` with \`stream=True\` yields whatever arrives; with \`stream=False\` it  
  yields one chunk.  
  
\*\*\`iter\_lines(chunk\_size=512, decode\_unicode=False, delimiter=None)\`\*\* — buffers a \`pending\`  
partial line across chunks. The docstring flags it as \*\*not reentrant safe\*\*: interleaving  
two iterations over the same response will lose data.  
  
\`\`\`python  
with requests.get(url, stream=True) as r:  
    for line in r.iter\_lines(decode\_unicode=True):        if line:            print(json.loads(line))\`\`\`  
  
\*\*\`content\`\*\* — memoised into \`\_content\`; reads via \`iter\_content(CONTENT\_CHUNK\_SIZE)\`;  
returns \`None\` when \`status\_code == 0\` or \`raw is None\`; raises \`RuntimeError\` if the stream  
was already consumed elsewhere.  
  
\*\*\`text\`\*\* — decodes \`content\` using \`self.encoding\`, falling back to \`apparent\_encoding\`,  
with \`errors="replace"\`. Note that \`get\_encoding\_from\_headers\` returns \`ISO-8859-1\` for any  
\`text/\*\` content type without an explicit charset (RFC 2616 to the letter), which is why  
\`r.encoding = "utf-8"\` is such a common manual fix.  
  
\*\*\`json(\*\*kwargs)\`\*\* — when no encoding is set and the body is longer than 3 bytes, it  
sniffs the UTF variant with \`guess\_json\_utf\` (BOM and null-byte-position based) before  
falling back to \`self.text\`. All decode failures become \`requests.exceptions.JSONDecodeError\`.  
  
\*\*\`raise\_for\_status()\`\*\* — builds \`"{code} Client Error: {reason} for url: {url}"\` for  
4xx and \`"Server Error"\` for 5xx. Decodes a bytes \`reason\` as utf-8 then iso-8859-1  
(PR #3538).  
  
\*\*\`close()\`\*\* — closes \`raw\` if unconsumed and calls \`release\_conn()\` if present. \`Response\`  
is a context manager, which is the recommended shape for \`stream=True\`.  
  
\### \`adapters.py\`  
  
\#### \`BaseAdapter\`  
  
Two abstract methods, \`send()\` and \`close()\`. Subclass this for non-HTTP transports:  
  
\`\`\`python  
class FileAdapter(requests.adapters.BaseAdapter):  
    def send(self, request, stream=False, timeout=None,             verify=True, cert=None, proxies=None):        resp = requests.Response()        path = request.url\[len("file://"):\]        with open(path, "rb") as fh:            resp.\_content = fh.read()        resp.status\_code = 200        resp.url = request.url        resp.request = request        resp.connection = self        return resp  
    def close(self):        pass  
s = requests.Session()  
s.mount("file://", FileAdapter())  
s.get("file:///etc/hosts").text  
\`\`\`  
  
\#### \`\_urllib3\_request\_context(request, verify, client\_cert, poolmanager)\`  
  
Computes the two dicts that decide \*\*which pooled connection\*\* is used:  
  
\- \`host\_params\` \= \`{scheme, host, port}\`  
\- \`pool\_kwargs\` \= TLS-related keys:  
  
| \`verify\` | Resulting \`pool\_kwargs\` |  
| --- | --- |  
| \`True\` | \`cert\_reqs="CERT\_REQUIRED"\` |  
| \`False\` | \`cert\_reqs="CERT\_NONE"\` |  
| path to a file | \`cert\_reqs="CERT\_REQUIRED"\`, \`ca\_certs=<path>\` |  
| path to a directory | \`cert\_reqs="CERT\_REQUIRED"\`, \`ca\_cert\_dir=<path>\` |  
  
\`cert\` adds \`cert\_file\` (and \`key\_file\` when a 2-tuple). This separation exists so that two  
requests to the same host with different TLS settings do not accidentally share a  
connection — a real CVE class.  
  
\#### \`HTTPAdapter\`  
  
Constructor knobs (all with module-level defaults):  
  
\`\`\`python  
HTTPAdapter(pool\_connections=10,   # number of distinct host pools cached  
            pool\_maxsize=10,       # connections kept per pool            max\_retries=0,         # Retry(0, read=False) by default            pool\_block=False)      # block when pool is exhausted?  
\`\`\`  
  
\`max\_retries=0\` becomes \`Retry(0, read=False)\`, i.e. \*\*no retries at all\*\*, not even on  
connection failure. Any integer goes through \`Retry.from\_int\`. For real control, pass a  
\`urllib3.util.Retry\`:  
  
\`\`\`python  
from urllib3.util import Retry  
  
retry = Retry(total=5, backoff\_factor=0.3,  
              status\_forcelist=\[502, 503, 504\],              allowed\_methods={"GET", "HEAD"})s = requests.Session()  
s.mount("https://", requests.adapters.HTTPAdapter(max\_retries=retry))  
\`\`\`  
  
\*\*Method-by-method\*\*  
  
| Method | Role |  
| --- | --- |  
| \`init\_poolmanager\` | Creates the \`urllib3.PoolManager\`; override to inject a custom \`ssl\_context\`. |  
| \`proxy\_manager\_for(proxy)\` | Caches a \`ProxyManager\` (or \`SOCKSProxyManager\` for \`socks\*\` URLs) per proxy URL. |  
| \`cert\_verify(conn, url, verify, cert)\` | Sets \`conn.cert\_reqs\`/\`ca\_certs\`/\`cert\_file\`; raises \`OSError\` for missing bundle or cert files. |  
| \`build\_response(req, resp)\` | urllib3 response → \`requests.Response\`, including cookie extraction and \`response.connection = self\`. |  
| \`build\_connection\_pool\_key\_attributes\` | Public wrapper over \`\_urllib3\_request\_context\`; the documented override point for custom \`SSLContext\`. |  
| \`get\_connection\_with\_tls\_context\` | The current connection getter. |  
| \`get\_connection\` | \*\*Deprecated\*\* since 2.32.2 (PR #6710); emits \`DeprecationWarning\`. |  
| \`request\_url(request, proxies)\` | Origin-form path normally; absolute URI (via \`urldefragauth\`) for non-SOCKS HTTP proxies. |  
| \`add\_headers(request, \*\*kwargs)\` | Intentional no-op extension point. |  
| \`proxy\_headers(proxy)\` | Builds \`Proxy-Authorization\` from credentials in the proxy URL. |  
| \`send(...)\` | The main event. |  
| \`close()\` | \`poolmanager.clear()\` plus every cached proxy manager. |  
  
\*\*\`send()\` details\*\*  
  
\- \`chunked = not (request.body is None or "Content-Length" in request.headers)\` — chunked  
  transfer is inferred, not requested explicitly.  
\- Timeout handling: a 2-tuple becomes \`urllib3.util.Timeout(connect=…, read=…)\`; a bare  
  float sets both; an already-constructed \`TimeoutSauce\` passes through. A malformed tuple  
  produces a clear \`ValueError\`.  
\- The exception translation table is the single most important thing this method does:  
  
| urllib3 exception | Requests exception |  
| --- | --- |  
| \`ProtocolError\`, \`OSError\` | \`ConnectionError\` |  
| \`MaxRetryError(reason=ConnectTimeoutError, not NewConnectionError)\` | \`ConnectTimeout\` |  
| \`MaxRetryError(reason=ResponseError)\` | \`RetryError\` |  
| \`MaxRetryError(reason=ProxyError)\` | \`ProxyError\` |  
| \`MaxRetryError(reason=SSLError)\` | \`SSLError\` |  
| \`MaxRetryError\` (other) | \`ConnectionError\` |  
| \`ClosedPoolError\` | \`ConnectionError\` |  
| \`ProxyError\` | \`ProxyError\` |  
| \`SSLError\` | \`SSLError\` |  
| \`ReadTimeoutError\` | \`ReadTimeout\` |  
| \`InvalidHeader\` | \`InvalidHeader\` |  
| \`LocationValueError\` | \`InvalidURL\` |  
  
That table is precisely why \`except requests.exceptions.RequestException\` catches  
everything the library can throw — no raw urllib3 exception should ever escape.  
  
\*\*Overriding TLS properly\*\* — the canonical modern pattern:  
  
\`\`\`python  
import ssl  
import requests  
from requests.adapters import HTTPAdapter  
  
class TLS12Adapter(HTTPAdapter):  
    def init\_poolmanager(self, connections, maxsize, block=False, \*\*kwargs):        ctx = ssl.create\_default\_context()        ctx.minimum\_version = ssl.TLSVersion.TLSv1\_2        kwargs\["ssl\_context"\] = ctx        return super().init\_poolmanager(connections, maxsize, block, \*\*kwargs)  
s = requests.Session()  
s.mount("https://", TLS12Adapter())  
\`\`\`  
  
\### \`auth.py\`  
  
\#### \`\_basic\_auth\_str(username, password)\`  
  
Builds \`Basic <base64(user:pass)>\`. Two things are load-bearing:  
  
\- Non-string credentials emit a \`DeprecationWarning\` and are coerced with \`str()\`. The  
  comment quotes Lukasa: the behaviour is acknowledged as dumb but preserved for  
  compatibility until 3.0.0.  
\- Credentials are encoded as \*\*latin-1\*\*, not utf-8. Non-latin-1 passwords will raise  
  \`UnicodeEncodeError\`; that is intentional and matches the historical HTTP behaviour.  
  
\#### \`AuthBase\`  
  
The extension point. Any callable taking and returning a \`PreparedRequest\` works:  
  
\`\`\`python  
class BearerAuth(requests.auth.AuthBase):  
    def \_\_init\_\_(self, token):        self.token = token  
    def \_\_call\_\_(self, r):        r.headers\["Authorization"\] = f"Bearer {self.token}"        return r  
requests.get(url, auth=BearerAuth("abc123"))  
\`\`\`  
  
\#### \`HTTPBasicAuth\` / \`HTTPProxyAuth\`  
  
\`HTTPBasicAuth\` sets \`Authorization\`; \`HTTPProxyAuth\` subclasses it and sets  
\`Proxy-Authorization\` instead. Both implement \`\_\_eq\_\_\`/\`\_\_ne\_\_\` by comparing username and  
password, using \`getattr(other, ..., None)\` so comparison against arbitrary objects does not  
explode.  
  
\#### \`HTTPDigestAuth\`  
  
The most intricate class in the file, and a good example of what "stateful auth" costs.  
  
\*\*Thread safety.\*\* All mutable state lives in \`threading.local()\`, initialised lazily by  
\`init\_per\_thread\_state()\`: \`last\_nonce\`, \`nonce\_count\`, \`chal\`, \`pos\`, \`num\_401\_calls\`. A  
single \`HTTPDigestAuth\` instance is therefore safe to share across threads.  
  
\*\*Supported algorithms:\*\* \`MD5\`, \`MD5-SESS\`, \`SHA\`, \`SHA-256\`, \`SHA-512\`. Every hash call  
passes \`usedforsecurity=False\` so it works on FIPS-restricted builds. Unknown algorithms  
cause \`build\_digest\_header\` to return \`None\`, which silently skips the header.  
\`qop=auth-int\` is explicitly \*\*not implemented\*\* (\`\# XXX handle auth-int.\`) and also returns  
\`None\`.  
  
\*\*The 401 dance.\*\* \`\_\_call\_\_\` registers two response hooks:  
  
\- \`handle\_401\` — if the status is 4xx, \`www-authenticate\` mentions digest, and fewer than  
  two 401s have been seen: parse the challenge, drain and close the original response  
  (releasing the connection for reuse), copy the request, re-extract cookies, attach the  
  computed \`Authorization\`, and resend via \`r.connection.send(prep, \*\*kwargs)\`. The new  
  response's \`history\` gets the 401 appended.  
\- \`handle\_redirect\` — resets \`num\_401\_calls\` to 1 on redirects so a redirect chain does not  
  exhaust the retry budget.  
  
The \`num\_401\_calls < 2\` guard is what prevents infinite loops against a misconfigured  
server. The \`not 400 <= r.status\_code < 500\` early return comes from issue #3772.  
  
\*\*Nonce reuse.\*\* If \`last\_nonce\` is already set, \`\_\_call\_\_\` computes the header  
pre-emptively, skipping the 401 round-trip entirely for subsequent requests on the same  
session.  
  
\*\*Body rewinding.\*\* \`pos\` is captured from \`r.body.tell()\` in \`\_\_call\_\_\` and restored in  
\`handle\_401\` before resending, so file uploads survive the retry.  
  
\`\`\`python  
from requests.auth import HTTPDigestAuth  
s = requests.Session()  
s.auth = HTTPDigestAuth("user", "pass")  
s.get("https://httpbin.org/digest-auth/auth/user/pass")   # 2 round-trips  
s.get("https://httpbin.org/digest-auth/auth/user/pass")   # 1 round-trip (nonce cached)  
\`\`\`  
  
\### \`cookies.py\`  
  
Bridges \`http.cookiejar\` (which was designed for \`urllib2\`) to Requests' object model.  
  
\#### \`MockRequest\`  
  
Adapts a \`PreparedRequest\` to the \`urllib2.Request\` interface cookielib expects:  
\`get\_type\`, \`get\_host\`, \`get\_origin\_req\_host\`, \`get\_full\_url\`, \`is\_unverifiable\`,  
\`has\_header\`, \`get\_header\`, \`add\_unredirected\_header\`, plus the modern \`unverifiable\`,  
\`origin\_req\_host\`, \`host\` properties.  
  
Two subtleties:  
  
\- \`get\_full\_url()\` honours a user-set \`Host\` header by reconstructing the URL around it, so  
  domain-fronted / rewritten requests get the right cookie domain.  
\- \`add\_header()\` deliberately raises \`NotImplementedError\` — cookielib must go through  
  \`add\_unredirected\_header()\`, and the collected headers are read back via  
  \`get\_new\_headers()\`.  
\- \`is\_unverifiable()\` always returns \`True\`, which relaxes third-party cookie policy checks.  
  
\#### \`MockResponse\`  
  
Wraps an \`http.client.HTTPMessage\` so cookielib can call \`.info()\` on it.  
  
\#### \`extract\_cookies\_to\_jar(jar, request, response)\` / \`get\_cookie\_header(jar, request)\`  
  
The only two functions the rest of the library calls. \`extract\_cookies\_to\_jar\` bails out  
early when \`response.\_original\_response\` is missing — that is why cookies do not appear on  
hand-constructed or mocked responses.  
  
\#### \`RequestsCookieJar\`  
  
\`CookieJar\` \*\*and\*\* \`MutableMapping\[str, str | None\]\`. The dict interface exists purely for  
user convenience; Requests itself never relies on it.  
  
| Concern | Detail |  
| --- | --- |  
| Complexity | Documented warning: dict operations that look O(1) are O(n). |  
| Conflicts | \`\_\_getitem\_\_\`/\`get\` use \`\_find\_no\_duplicates\`, which raises \`CookieConflictError\` when the same name exists on multiple domains/paths. |  
| \`\_\_contains\_\_\` | Swallows \`CookieConflictError\` and returns \`True\`. |  
| Deletion | \`jar\["name"\] = None\` and \`del jar\["name"\]\` both route to \`remove\_cookie\_by\_name\`. |  
| Quoting | \`set\_cookie\` strips surrounding double quotes and unescapes \`\\"\`. |  
| Pickling | \`\_\_getstate\_\_\` drops \`\_cookies\_lock\` (an unpickleable \`RLock\`); \`\_\_setstate\_\_\` recreates it. Plain \`CookieJar\` is not pickleable — this class is. |  
| \`copy()\` | Preserves the \`CookiePolicy\`. |  
  
\`\`\`python  
jar = requests.cookies.RequestsCookieJar()  
jar.set("session", "abc", domain="a.example.com", path="/")  
jar.set("session", "xyz", domain="b.example.com", path="/")  
  
jar\["session"\]                                    # CookieConflictError  
jar.get("session", domain="a.example.com")        # 'abc'  
jar.list\_domains()                                # \['a.example.com', 'b.example.com'\]  
jar.get\_dict(domain="b.example.com")              # {'session': 'xyz'}  
\`\`\`  
  
\#### Module functions  
  
| Function | Purpose |  
| --- | --- |  
| \`create\_cookie(name, value, \*\*kwargs)\` | Builds a \`http.cookiejar.Cookie\` with sane defaults (\`domain=""\`, \`path="/"\`, \`discard=True\`). Unknown kwargs raise \`TypeError\`. |  
| \`morsel\_to\_cookie(morsel)\` | Converts an \`http.cookies.Morsel\`, resolving \`max-age\` / \`expires\`. |  
| \`cookiejar\_from\_dict(d, jar=None, overwrite=True)\` | Dict → jar. \`overwrite=False\` keeps existing cookies. |  
| \`merge\_cookies(jar, cookies)\` | Adds a dict or another jar into \`jar\`. Raises \`ValueError\` if the target is not a \`CookieJar\`. |  
| \`remove\_cookie\_by\_name(jar, name, domain, path)\` | O(n) targeted clear. |  
| \`\_copy\_cookie\_jar(jar)\` | Deep-ish copy used by \`PreparedRequest.copy()\`. |  
  
Note the default \`domain=""\` in \`create\_cookie\` — a cookie created that way is a  
"supercookie" sent to every host. Relevant when reviewing security-adjacent PRs.  
  
\### \`structures.py\`  
  
\#### \`CaseInsensitiveDict\`  
  
Backed by \`OrderedDict\[lowercase\_key\] = (original\_key, value)\`. Iteration yields the  
\*\*original\*\* casing; lookup and \`in\` are case-insensitive. \`lower\_items()\` is used by  
\`\_\_eq\_\_\` so two dicts differing only in header casing compare equal.  
  
\`\`\`python  
h = requests.structures.CaseInsensitiveDict()  
h\["Content-Type"\] = "application/json"  
h\["CONTENT-TYPE"\]           # 'application/json'  
list(h)                     # \['Content-Type'\]  ← original casing preserved  
h == {"content-type": "application/json"}   # True  
\`\`\`  
  
Documented caveat: if two keys with the same \`.lower()\` are supplied to the constructor or  
\`update\`, behaviour is undefined (last write wins in practice).  
  
\#### \`LookupDict\`  
  
A \`dict\` subclass whose real storage is \`\_\_dict\_\_\`, so both \`d.key\` and \`d\["key"\]\` work —  
but not identically. \`\_\_getitem\_\_\` and \`get\` fall through to \`None\` for unknown keys, while  
\`\_\_getattr\_\_\` raises \`AttributeError\`. That is why \`requests.codes\["nonsense"\]\` is \`None\`  
but \`requests.codes.nonsense\` raises. Only used by \`status\_codes.py\`.  
  
\### \`utils.py\`  
  
The public-ish grab bag. \`requests.utils\` is explicitly re-exported from \`\_\_init\_\_.py\`, so  
changes here are effectively API changes.  
  
\#### Constants  
  
\`\`\`python  
NETRC\_FILES             = (".netrc", "\_netrc")  
DEFAULT\_CA\_BUNDLE\_PATH  = certs.where()  
DEFAULT\_PORTS           = {"http": 80, "https": 443}  
UNRESERVED\_SET          = frozenset(ascii letters + digits + "-.\_~")   # RFC 3986  
  
\# Derived from urllib3, so the exact value tracks the installed urllib3:  
\#   ", ".join(re.split(r",\\s\*", make\_headers(accept\_encoding=True)\["accept-encoding"\]))  
\# urllib3 2.x with zstd available → "gzip, deflate, zstd"  
DEFAULT\_ACCEPT\_ENCODING = ...  
\`\`\`  
  
The \`re.split\` + \`", ".join\` dance exists only to normalise urllib3's comma-separated value  
to the historical \`", "\` delimiter, so the header Requests sends does not change shape when  
urllib3 changes its formatting.  
  
\#### Size and stream helpers  
  
\*\*\`super\_len(o)\`\*\* — the length oracle used by \`prepare\_content\_length\`. Tries, in order:  
\`len(o)\`, \`o.len\`, \`os.fstat(o.fileno()).st\_size\`, then \`seek(0, 2)\`/\`tell()\`. It subtracts  
the current position so partially-read files report the remaining bytes:  
  
\`\`\`python  
f = open("data.bin", "rb")   # 1000 bytes  
f.read(400)  
requests.utils.super\_len(f)  # 600  
\`\`\`  
  
Two behaviours worth knowing: a text-mode file triggers \`FileModeWarning\` (binary size may  
not match the encoded byte count), and under urllib3 2.x a \`str\` is encoded to utf-8 first  
because urllib3 2.x switched from latin-1 to utf-8.  
  
\*\*\`iter\_slices(string, slice\_length)\`\*\* — replays buffered content in chunks;  
\`slice\_length\` of \`None\` or \`<= 0\` yields the whole string.  
  
\*\*\`stream\_decode\_response\_unicode(iterator, r)\`\*\* — wraps an incremental codec so multi-byte  
characters split across chunk boundaries decode correctly. If \`r.encoding is None\` it yields  
bytes unchanged.  
  
\#### Credentials and netrc  
  
\*\*\`get\_netrc\_auth(url, raise\_errors=False)\`\*\* — honours \`$NETRC\`, else \`~/.netrc\` then  
\`~/\_netrc\`. Parse errors and permission problems are swallowed unless \`raise\_errors=True\`.  
Only consulted when \`Session.trust\_env\` is true and no explicit auth was given.  
  
\*\*\`get\_auth\_from\_url(url)\`\*\* — returns \`(username, password)\` percent-decoded, or \`("", "")\`.  
  
\#### Proxy resolution  
  
This is a four-function chain, and getting it right is a frequent source of PRs.  
  
| Function | Role |  
| --- | --- |  
| \`should\_bypass\_proxies(url, no\_proxy)\` | Checks \`no\_proxy\` (arg, then lowercase env, then uppercase env), supporting exact hosts, suffix matching with an implicit leading dot, \`host:port\`, and CIDR blocks for IPv4. Falls back to the stdlib \`proxy\_bypass\`. |  
| \`get\_environ\_proxies(url, no\_proxy)\` | \`{}\` if bypassed, else \`urllib.request.getproxies()\`. |  
| \`select\_proxy(url, proxies)\` | Key precedence: \`scheme://host\` → \`scheme\` → \`all://host\` → \`all\`. |  
| \`resolve\_proxies(request, proxies, trust\_env)\` | Merges env proxies into explicit ones via \`setdefault\`, so explicit config always wins. |  
  
\`\`\`python  
proxies = {  
    "https://internal.example.com": "http://proxy-a:8080",  # most specific    "https": "http://proxy-b:8080",    "all": "http://proxy-c:8080",}  
requests.utils.select\_proxy("https://internal.example.com/x", proxies)  
\# 'http://proxy-a:8080'  
\`\`\`  
  
Note the lowercase-first env lookup in \`should\_bypass\_proxies\` — deliberate consistency with  
\`curl\` and \`wget\`. On Windows, \`utils.py\` defines a registry-based \`proxy\_bypass\_registry\`  
that avoids DNS lookups.  
  
Helpers: \`is\_ipv4\_address\`, \`is\_valid\_cidr\` (mask must be 1–32), \`address\_in\_network\`,  
\`dotted\_netmask\`, and the \`set\_environ\` context manager that temporarily sets/restores an  
environment variable.  
  
\#### URL manipulation  
  
| Function | Behaviour |  
| --- | --- |  
| \`unquote\_unreserved(uri)\` | Decodes only \`%XX\` sequences whose byte is in \`UNRESERVED\_SET\`. Invalid escapes raise \`InvalidURL\`. |  
| \`requote\_uri(uri)\` | \`quote(unquote\_unreserved(uri), safe="!#$%&'()\*+,/:;=?@\[\]~")\`; on \`InvalidURL\` retries without \`%\` in the safe set. Idempotent, which is why redirect targets can be re-quoted safely. |  
| \`prepend\_scheme\_if\_needed(url, scheme)\` | Adds a scheme only if absent; swaps netloc/path to work around a \`urlparse\` defect. |  
| \`urldefragauth(url)\` | Strips both the fragment and \`user:pass@\`. Used for absolute-form proxy request lines. |  
  
\#### Header parsing  
  
| Function | Behaviour |  
| --- | --- |  
| \`parse\_list\_header('token, "quoted, value"')\` | \`\['token', 'quoted, value'\]\` (from werkzeug, used with permission) |  
| \`parse\_dict\_header('foo="bar", baz')\` | \`{'foo': 'bar', 'baz': None}\` — used to parse digest challenges |  
| \`unquote\_header\_value(value, is\_filename=False)\` | Browser-compatible unquoting, with a UNC-path special case (issue #458) |  
| \`parse\_header\_links(value)\` | \`Link:\` header → list of dicts; powers \`Response.links\` |  
| \`check\_header\_validity(header)\` | Delegates to \`\_validate\_header\_part\` with the regexes from \`\_internal\_utils\` |  
  
\`\`\`python  
requests.utils.parse\_header\_links(  
    '<https://api/x?page=2>; rel="next", <https://api/x?page=9>; rel="last"')  
\# \[{'url': 'https://api/x?page=2', 'rel': 'next'},  
\#  {'url': 'https://api/x?page=9', 'rel': 'last'}\]  
\`\`\`  
  
\#### Encoding detection  
  
\*\*\`get\_encoding\_from\_headers(headers)\`\*\*:  
  
| Content-Type | Returned encoding |  
| --- | --- |  
| \`application/json; charset=utf-16\` | \`utf-16\` |  
| \`text/html\` (no charset) | \`ISO-8859-1\` (RFC 2616 default) |  
| \`application/json\` (no charset) | \`utf-8\` (RFC 4627) |  
| absent or unrecognised | \`None\` → \`Response.text\` falls back to \`apparent\_encoding\` |  
  
\*\*\`guess\_json\_utf(data)\`\*\* — BOM sniffing plus null-byte-position analysis of the first four  
bytes to distinguish \`utf-8\`, \`utf-16-le/be\`, \`utf-32-le/be\`.  
  
\#### Deprecated  
  
\`get\_encodings\_from\_content\` and \`get\_unicode\_from\_response\` both emit \`DeprecationWarning\`  
and are slated for removal in 3.0 (issue #2266). Do not build on them.  
  
\#### Misc  
  
\- \`extract\_zipped\_paths(path)\` — extracts a CA bundle that lives inside a zip/egg to a temp  
  file. Contains a deliberate guard against an infinite loop on a rare path shape.  
\- \`atomic\_open(filename)\` — write-to-temp then \`os.replace\`.  
\- \`rewind\_body(prepared\_request)\` — seeks back to \`\_body\_position\`, raising  
  \`UnrewindableBodyError\` if the body is not seekable or the position was never recorded.  
\- \`default\_headers()\` / \`default\_user\_agent(name="python-requests")\`.  
\- \`to\_key\_val\_list\` / \`from\_key\_val\_list\` — normalise mappings and pair-iterables; both  
  raise \`ValueError("cannot encode objects that are not 2-tuples")\` for scalars. These have  
  doctests, and \`pytest\` runs with \`--doctest-modules\`, so the examples in their docstrings  
  are executed as tests.  
  
\### \`\_internal\_utils.py\`  
  
Deliberately dependency-light (only \`compat.builtin\_str\`) so it can be imported from  
anywhere without cycles.  
  
The header validation regexes are the security core of the library:  
  
\`\`\`python  
\_VALID\_HEADER\_NAME\_RE\_STR  = re.compile(r"^\[^:\\s\]\[^:\\r\\n\]\*\\Z")  
\_VALID\_HEADER\_VALUE\_RE\_STR = re.compile(r"^\\S\[^\\r\\n\]\*\\Z|^\\Z")  
HEADER\_VALIDATORS = {bytes: (...), str: (...)}  
\`\`\`  
  
Names may not start with whitespace or \`:\` and may not contain \`:\`, CR, or LF. Values may  
not start with whitespace and may not contain CR or LF; an empty value is explicitly  
allowed via the \`|^\\Z\` alternative.  
  
\`to\_native\_string(string, encoding="ascii")\` and \`unicode\_is\_ascii(u\_string)\` round out the  
module. The latter is how \`prepare\_url\` decides whether IDNA encoding is needed.  
  
\### \`\_types.py\`  
  
Private typing infrastructure — the docstring states plainly that external code must not  
rely on it. Most of the file lives under \`if TYPE\_CHECKING:\` so it costs nothing at runtime.  
  
\*\*Runtime-checkable protocols\*\*  
  
\`\`\`python  
@runtime\_checkable  
class SupportsRead(Protocol\[\_T\_co\]):  
    def read(self, length: int = ..., /) -> \_T\_co: ...  
def has\_read(obj) -> TypeIs\[SupportsRead\[str | bytes\]\]:  
    return isinstance(obj, SupportsRead) or hasattr(obj, "read")\`\`\`  
  
The extra \`hasattr\` matters: \`isinstance\` against a \`Protocol\` misses objects that expose  
\`read\` through \`\_\_getattr\_\_\` (proxy objects). \`SupportsItems\` plays the same role for  
mapping-ish inputs.  
  
\*\*\`is\_prepared(request)\`\*\* — a \`TypeIs\` narrowing helper that is a \*\*no-op at runtime\*\*  
(returns \`True\` unconditionally) and only performs the real check under \`TYPE\_CHECKING\`.  
That is why you see \`assert \_is\_prepared(request)\` sprinkled through \`sessions.py\`,  
\`adapters.py\`, and \`cookies.py\`: it narrows \`url: str | None\` to \`str\` for pyright without  
adding runtime cost or raising \`AssertionError\` in production. The \`\_ValidatedRequest\`  
subclass exists purely to express "url and method are non-None after preparation", which  
plain invariant attribute typing cannot represent.  
  
\*\*Alias map\*\* — worth memorising when writing new signatures:  
  
| Alias | Rough meaning |  
| --- | --- |  
| \`UriType\` | \`str \\| bytes\` |  
| \`ParamsType\` | mapping / pairs / str / bytes / None |  
| \`DataType\` | mapping, pairs, str, bytes, \`Buffer\`, iterable, file-like, None |  
| \`BodyType\` | what actually lands on \`PreparedRequest.body\` |  
| \`FilesType\` | mapping or pairs of \`\_FileSpec\` (content, 2-, 3-, or 4-tuple) |  
| \`AuthType\` | \`(user, pass)\` tuple, \`AuthBase\`, or callable |  
| \`TimeoutType\` | \`float \\| (connect, read) \\| None\` |  
| \`HooksInputType\` / \`HookType\` | hook mapping and \`Callable\[ \[Response\], Any\]\` |  
| \`JsonType\` | recursive JSON value alias |  
  
\*\*TypedDicts\*\* — \`BaseRequestKwargs\` holds the shared keys; \`RequestKwargs\`, \`GetKwargs\`,  
\`PostKwargs\`, and \`DataKwargs\` layer on the per-verb differences described in the \`api.py\`  
section.  
  
\### \`exceptions.py\`  
  
Everything derives from \`RequestException(IOError)\`, so \`except OSError\` also catches  
Requests errors. \`RequestException.\_\_init\_\_\` pops \`response\` and \`request\` from kwargs and,  
when only a response is given, back-fills \`self.request\` from \`response.request\`.  
  
\`\`\`  
IOError  
└── RequestException  
    ├── InvalidJSONError    │   └── JSONDecodeError            (+ json/simplejson JSONDecodeError)    ├── HTTPError    ├── ConnectionError    │   ├── ProxyError    │   ├── SSLError    │   └── ConnectTimeout             (+ Timeout)    ├── Timeout    │   ├── ConnectTimeout    │   └── ReadTimeout    ├── URLRequired    ├── TooManyRedirects    ├── MissingSchema                  (+ ValueError)    ├── InvalidSchema                  (+ ValueError)    ├── InvalidURL                     (+ ValueError)    │   └── InvalidProxyURL    ├── InvalidHeader                  (+ ValueError)    ├── ChunkedEncodingError    ├── ContentDecodingError           (+ urllib3 HTTPError)    ├── StreamConsumedError            (+ TypeError)    ├── RetryError    └── UnrewindableBodyError  
Warning  
└── RequestsWarning  
    ├── FileModeWarning                (+ DeprecationWarning)    └── RequestsDependencyWarning\`\`\`  
  
The multiple inheritance is deliberate and load-bearing:  
  
\- \`ConnectTimeout(ConnectionError, Timeout)\` means both \`except ConnectionError\` and  
  \`except Timeout\` catch it.  
\- \`MissingSchema\`/\`InvalidURL\`/\`InvalidHeader\` also being \`ValueError\` preserves pre-1.0  
  behaviour for code that caught \`ValueError\`.  
\- \`JSONDecodeError.\_\_reduce\_\_\` explicitly delegates to \`CompatJSONDecodeError.\_\_reduce\_\_\`,  
  because the MRO would otherwise pick \`IOError.\_\_reduce\_\_\` and break pickling.  
  
\`\`\`python  
try:  
    r = requests.get(url, timeout=(3.05, 10))    r.raise\_for\_status()except requests.exceptions.Timeout:  
    ...             # connect or read timeoutexcept requests.exceptions.HTTPError as e:  
    e.response.status\_codeexcept requests.exceptions.RequestException:  
    ...             # everything else requests can raise\`\`\`  
  
\### \`hooks.py\`  
  
49 lines, one hook event.  
  
\`\`\`python  
HOOKS = \["response"\]  
  
def default\_hooks() -> dict\[str, list\[HookType\]\]:  
    return {event: \[\] for event in HOOKS}  
def dispatch\_hook(key, hooks, hook\_data, \*\*kwargs) -> Response:  
    hook\_list = (hooks or {}).get(key)    if hook\_list:        if isinstance(hook\_list, Callable):            hook\_list = \[hook\_list\]        for hook in hook\_list:            \_hook\_data = hook(hook\_data, \*\*kwargs)            if \_hook\_data is not None:                hook\_data = \_hook\_data    return hook\_data\`\`\`  
  
The chaining rule is the important part: a hook returning \`None\` leaves the response  
unchanged, while a hook returning a value \*\*replaces\*\* it for subsequent hooks and for the  
caller.  
  
\`\`\`python  
def log\_timing(r, \*args, \*\*kwargs):  
    print(r.url, r.status\_code, r.elapsed.total\_seconds())    # returns None → response unchanged  
def unwrap\_envelope(r, \*args, \*\*kwargs):  
    r.\_content = json.dumps(r.json()\["data"\]).encode()    return r        # returns r → replaces the response  
s = requests.Session()  
s.hooks\["response"\].extend(\[log\_timing, unwrap\_envelope\])  
\`\`\`  
  
Hooks receive the same \`\*\*kwargs\` that were passed to \`adapter.send\` (\`stream\`, \`verify\`,  
\`cert\`, \`proxies\`), which is exactly what makes \`HTTPDigestAuth.handle\_401\` able to resend  
the request faithfully. \`hooks.py\` still carries a \`# TODO: response is the only one\`  
comment — an obvious, long-standing extension point.  
  
\### \`status\_codes.py\`  
  
Builds \`requests.codes\`, a \`LookupDict\[int\]\`, from the \`\_codes\` table. \`\_init()\` sets both  
the lowercase and uppercase form of every title, skipping the uppercase variant for the ASCII  
art names, and appends a generated list of all codes to the module \`\_\_doc\_\_\` (which is why  
Sphinx renders the full table).  
  
\`\`\`python  
requests.codes.ok             # 200  
requests.codes.OK             # 200  
requests.codes\["\\\\o/"\]         # 200  
requests.codes.teapot         # 418  
requests.codes\["✓"\]           # 200  
  
requests.codes\["not\_a\_code"\]  # None          ← \_\_getitem\_\_ falls through  
requests.codes.not\_a\_code     # AttributeError ← \_\_getattr\_\_ does NOT fall through  
\`\`\`  
  
That asymmetry is deliberate: \`LookupDict.\_\_getitem\_\_\` returns \`self.\_\_dict\_\_.get(key, None)\`  
while \`\_\_getattr\_\_\` raises, which keeps typos in attribute access loud.  
  
Two names are marked for removal in 3.0: \`resume\` and \`resume\_incomplete\` for 308.  
\`models.REDIRECT\_STATI\` is built from this table, so adding a redirect status here changes  
redirect behaviour globally.  
  
\### \`compat.py\`  
  
A vestigial Python 2/3 shim, kept because people import from it. It:  
  
\- Detects urllib3 1.x vs 2.x into \`is\_urllib3\_1\` (used by \`super\_len\`).  
\- Resolves character detection with \`\_resolve\_char\_detection()\`, preferring \`chardet\` then  
  \`charset\_normalizer\`, exposing the result as \`compat.chardet\`.  
\- Prefers \`simplejson\` over stdlib \`json\` when installed, aliasing \`JSONDecodeError\`  
  accordingly. \`models.py\` imports it as \`complexjson\`.  
\- Re-exports \`urllib.parse\` and \`urllib.request\` names, \`http.cookiejar as cookielib\`,  
  \`Morsel\`, \`OrderedDict\`, \`StringIO\`, and the \`Callable\`/\`Mapping\`/\`MutableMapping\` ABCs.  
\- Defines \`builtin\_str = str\`, \`basestring = (str, bytes)\`, \`numeric\_types\`,  
  \`integer\_types\`.  
  
Like \`\_\_init\_\_.py\`, this file is exempted from \`E402\`/\`F401\` — the re-exports are the point.  
  
\### \`certs.py\`  
  
Eighteen lines. \`from certifi import where\`, and a \`\_\_main\_\_\` guard that prints the path.  
The docstring is the contract for downstream packagers: redefine \`where()\` here to point at  
a distro CA bundle. \`utils.DEFAULT\_CA\_BUNDLE\_PATH = certs.where()\` is evaluated at import  
time.  
  
\`\`\`  
$ python -m requests.certs  
/…/site-packages/certifi/cacert.pem  
\`\`\`  
  
\### \`packages.py\`  
  
Pure backwards compatibility, and the source comment says as much ("I don't like it either.  
Just look the other way."). It walks \`sys.modules\` and aliases \`urllib3.\*\`, \`idna.\*\`, and the  
resolved chardet module under \`requests.packages.\*\`, preserving object identity:  
  
\`\`\`python  
import requests  
requests.packages.urllib3 is \_\_import\_\_("urllib3")        # True  
\`\`\`  
  
Also aliases \`charset\_normalizer\` submodules under the \`chardet\` name so old code importing  
\`requests.packages.chardet\` keeps working.  
  
\### \`help.py\`  
  
The bug-report helper. \`info()\` returns a dict of versions for platform, Python  
implementation, system OpenSSL, pyOpenSSL, urllib3, chardet, charset\_normalizer,  
cryptography, idna, and requests. \`main()\` pretty-prints it as JSON.  
  
\`\`\`  
$ python -m requests.help  
{  
  "chardet": {"version": null},  "charset\_normalizer": {"version": "3.4.4"},  ...}  
\`\`\`  
  
Always paste this into an issue report. Note the small oddity that \`using\_charset\_normalizer\`  
is computed as \`chardet is None\` rather than by checking \`charset\_normalizer\` directly.  
  
\## Cross-Cutting Concepts  
  
\### Streaming and connection release  
  
\`\`\`python  
\# Eager (default): Session.send calls r.content before returning  
r = requests.get(url)  
  
\# Lazy: body left in r.raw, socket held until consumed or closed  
with requests.get(url, stream=True) as r:  
    r.raise\_for\_status()    for chunk in r.iter\_content(chunk\_size=8192):        ...  
\`\`\`  
  
Forgetting the \`with\` (or \`r.close()\`) on a streamed response leaks the connection out of  
the pool. \`Response.close()\` calls \`raw.close()\` when unconsumed and \`raw.release\_conn()\`  
when available. \`resolve\_redirects\` is careful to consume and close each intermediate  
response for exactly this reason.  
  
\### Where content decoding actually happens  
  
\`HTTPAdapter.send\` passes \`decode\_content=False\` to \`urlopen\`, so urllib3 hands back the raw  
(possibly gzipped) stream. \`Response.iter\_content\` then calls  
\`self.raw.stream(chunk\_size, decode\_content=True)\`, so gzip/deflate/br decoding happens  
lazily during iteration. Consequence: \`r.raw.read()\` gives you compressed bytes;  
\`r.content\` gives you decompressed bytes.  
  
\### Timeouts  
  
\`timeout\` is not a deadline for the whole request. \`(connect, read)\` maps directly to  
urllib3's \`Timeout\`, where \`read\` is the maximum gap \*\*between bytes\*\*, not total download  
time. A slow-drip response can therefore run indefinitely under a read timeout.  
  
\`\`\`python  
requests.get(url, timeout=5)            # connect=5, read=5  
requests.get(url, timeout=(3.05, 27))   # recommended shape  
\`\`\`  
  
\### Certificate verification  
  
Precedence for \`verify\`: per-request argument → \`REQUESTS\_CA\_BUNDLE\` → \`CURL\_CA\_BUNDLE\`  
(both only when \`trust\_env\`) → \`Session.verify\` → \`certs.where()\`. Both \`cert\_verify\` (which  
mutates the connection) and \`\_urllib3\_request\_context\` (which sets pool key attributes)  
consume it — the latter is what keeps differently-configured requests on separate pooled  
connections.  
  
\### Environment variables honoured  
  
| Variable | Consumer |  
| --- | --- |  
| \`HTTP\_PROXY\` / \`HTTPS\_PROXY\` / \`ALL\_PROXY\` (+ lowercase) | \`getproxies()\` via \`get\_environ\_proxies\` |  
| \`NO\_PROXY\` / \`no\_proxy\` | \`should\_bypass\_proxies\` |  
| \`REQUESTS\_CA\_BUNDLE\`, \`CURL\_CA\_BUNDLE\` | \`merge\_environment\_settings\` |  
| \`NETRC\` | \`get\_netrc\_auth\` |  
  
All of them are gated behind \`Session.trust\_env = True\`. Setting \`trust\_env = False\` is the  
clean way to get a hermetic session.  
  
\### Thread safety  
  
\`Session\` is \*\*not\*\* documented as thread-safe. The cookie jar has an \`RLock\`, and  
\`HTTPDigestAuth\` uses thread-local state, but \`Session.headers\`, \`Session.adapters\`, and  
redirect handling are not synchronised. One session per thread, or a pool of sessions, is  
the safe pattern.  
  
\## Testing, Tooling, And Contribution Workflow  
  
\### Test layout  
  
| Path | Contents |  
| --- | --- |  
| \`tests/test\_requests.py\` | ~3,100 lines — the main suite (sessions, models, redirects, auth, cookies, proxies) |  
| \`tests/test\_utils.py\` | ~1,000 lines covering \`utils.py\` |  
| \`tests/test\_lowlevel.py\` | Raw-socket tests against the local test server; excluded from \`ruff-format\` in \`.pre-commit-config.yaml\` |  
| \`tests/test\_testserver.py\` | Tests for the test server itself |  
| \`tests/test\_structures.py\`, \`test\_hooks.py\`, \`test\_adapters.py\`, \`test\_help.py\`, \`test\_packages.py\` | Focused unit tests |  
| \`tests/testserver/server.py\` | Minimal threaded socket server for low-level assertions |  
| \`tests/certs/\` | \`valid\`, \`expired\`, and \`mtls\` fixture certificates |  
| \`tests/conftest.py\` | Fixtures |  
| \`tests/utils.py\` | \`override\_environ\` context manager |  
  
Key fixtures in \`conftest.py\`:  
  
\- \`clean\_proxy\_environ\` — \*\*autouse\*\*; strips all proxy env vars for every test. Do not  
  assume the ambient environment in a new test.  
\- \`httpbin\` / \`httpbin\_secure\` — wrap \`pytest-httpbin\` and guarantee a trailing slash  
  (issue #1483); call them like \`httpbin("get")\` → \`http://127.0.0.1:PORT/get\`.  
\- \`nosan\_server\` — a \`trustme\`-issued cert with only a commonName and no SAN, imported  
  lazily so the test can be deselected when \`trustme\` is missing.  
  
\### Commands  
  
\`\`\`bash  
make init                       # pip install -r requirements-dev.txtmake test                       # python -m pytest testsmake coverage                   # pytest with coverage against src/requestspython -m pytest tests/test\_requests.py -k redirect -xtox                             # py310..py314, default and use\_chardet\_on\_py3pre-commit run --all-files      # ruff-check --fix, ruff-format, whitespace hookspyright                         # strict mode over src/requests\`\`\`  
  
\`pytest\` is configured with \`testpaths = \["tests"\]\` and \`addopts = "--doctest-modules"\`, so  
doctests are collected from the modules under \`tests/\` — the library's own docstring  
examples (e.g. in \`utils.to\_key\_val\_list\`) are only executed if you point pytest at  
\`src/requests\` explicitly:  
  
\`\`\`bash  
python -m pytest --doctest-modules src/requests/utils.py  
\`\`\`  
  
\`doctest\_optionflags = NORMALIZE\_WHITESPACE ELLIPSIS\` applies either way.  
  
\### CI matrix  
  
\`.github/workflows/run-tests.yml\` runs three jobs, all via \`make ci\`:  
  
| Job | Configuration |  
| --- | --- |  
| \`build\` | Python 3.10–3.15-dev plus \`3.14t\` (free-threaded) and \`pypy-3.11\`, across ubuntu-22.04 / macOS / Windows |  
| \`no\_chardet\` | Both \`charset\_normalizer\` and \`chardet\` uninstalled — exercises the \`apparent\_encoding\` fallback to \`"utf-8"\` |  
| \`urllib3\` | Pinned to \`urllib3<2\` — exercises the \`compat.is\_urllib3\_1\` branches |  
  
The free-threaded (\`3.14t\`) entry pairs with the  
\`Programming Language :: Python :: Free Threading :: 2 - Beta\` classifier in  
\`pyproject.toml\`; concurrency regressions are in scope for this project.  
  
\### Lint and type rules  
  
\- Ruff with \`E\`, \`W\`, \`F\`, \`I\` (isort), \`UP\` (pyupgrade), \`T10\` (flake8-debugger),  
  targeting py310. \`E203\`, \`E501\`, \`UP031\` are ignored.  
\- \`T10\` means \*\*a stray \`print\`/\`breakpoint\` in \`src/\` will fail CI\*\* — relevant to the  
  local debug statements in this working tree.  
\- \`known-first-party = \["requests"\]\` for import sorting; double quotes; black-compatible  
  formatting.  
\- pyright in \`strict\` mode over \`src/requests\`, with \`reportPrivateUsage\`,  
  \`reportPrivateImportUsage\`, \`reportUnnecessaryIsInstance\`, and \`reportUnusedImport\`  
  disabled.  
  
CI workflows: \`run-tests.yml\`, \`lint.yml\`, \`typecheck.yml\`, \`codeql-analysis.yml\`,  
\`zizmor.yml\`, \`publish.yml\`, plus issue automation.  
  
\### Contribution etiquette  
  
From \`.github/CONTRIBUTING.md\` and \`docs/dev/contributing.rst\`:  
  
\- Search for duplicates first.  
\- Every PR must include a test that \*\*fails without the change\*\*.  
\- Run the suite locally before opening the PR.  
\- Commit messages must explain \*why\*, not just "Fixes #NNNN".  
\- Requests is feature-frozen in spirit; bug fixes, docs, and typing improvements land far  
  more readily than new features.  
\- There is a \`.github/AI\_POLICY.md\` — read it before submitting AI-assisted work.  
  
\### Where a new contributor can realistically help  
  
| Area | Where to look |  
| --- | --- |  
| Typing | \`# type: ignore\` and \`cast(...)\` sites in \`models.py\`, \`sessions.py\`, \`cookies.py\` — each is a small, self-contained improvement. |  
| Explicit TODOs | \`hooks.py:29\` (only one hook event), \`models.py:689\` ("can be fixed by flipping the conditionals"), \`models.py:1018\` (\`iter\_lines\` rewrite), \`auth.py:213\`/\`247\` (\`auth-int\` unimplemented), \`adapters.py:719\` (remove \`ConnectTimeout\` special case in 3.0). |  
| Deprecations | \`utils.get\_encodings\_from\_content\`, \`utils.get\_unicode\_from\_response\`, \`adapters.HTTPAdapter.get\_connection\`, \`sessions.session()\`, non-string basic-auth credentials, \`codes.resume\`. |  
| Docs | \`docs/user/\`, \`docs/community/\`, \`docs/dev/\` are reStructuredText; low-risk first PRs. |  
| Test coverage | \`tests/test\_utils.py\` for proxy/CIDR/\`no\_proxy\` edge cases; \`tests/test\_lowlevel.py\` for wire-format assertions. |  
  
\### Debugging techniques that beat \`print\`  
  
\`\`\`python  
\# 1. Inspect exactly what will go on the wire, without sending it  
prep = requests.Session().prepare\_request(  
    requests.Request("POST", "https://httpbin.org/post", json={"a": 1}))  
print(prep.method, prep.url)  
print(dict(prep.headers))  
print(prep.body)  
  
\# 2. Full wire-level logging from urllib3/http.client  
import logging, http.client  
http.client.HTTPConnection.debuglevel = 1  
logging.basicConfig(level=logging.DEBUG)  
logging.getLogger("urllib3").setLevel(logging.DEBUG)  
  
\# 3. A response hook as a permanent, CI-safe tracepoint  
def trace(r, \*args, \*\*kwargs):  
    print(r.request.method, r.url, "→", r.status\_code, r.elapsed)  
s = requests.Session()  
s.hooks\["response"\].append(trace)  
  
\# 4. Environment for a bug report  
python -m requests.help  
\`\`\`  
  
\## Local Working-Tree Notes  
  
Not part of upstream; recorded so future-me is not confused.  
  
\- \`playground.py\` and \`test.txt\` are untracked scratch files.  
\- Debug \`print()\` statements are currently present in \`api.py\`, \`sessions.py\`, \`models.py\`,  
  \`adapters.py\`, and \`hooks.py\`. Ruff's \`T10\` rule will reject these, so they must be  
  reverted before any PR: \`git checkout -- src/requests\`.  
\- \`sessions.py\` currently has \`Session.\_\_attrs\_\_\` containing \`"ç"\` where upstream has  
  \`"proxies"\`. That is an accidental local edit, not upstream behaviour, and it silently  
  breaks \`Session\` pickling (proxies are dropped and a bogus \`ç\` attribute is restored).  
  Fix it back to \`"proxies"\` along with the prints.  
  
\---