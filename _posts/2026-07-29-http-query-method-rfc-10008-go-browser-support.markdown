---
layout: post
title: "HTTP QUERY in RFC 10008: Go and Browser Support"
date: 2026-07-29 09:00:00 +0700
categories: golang backend http
tags: [golang, http, rfc10008, query-method, browser, cors, api-design]
description: "RFC 10008 standardizes HTTP QUERY as a safe, idempotent method with content. Learn its semantics and current Go, framework, and browser support."
---

In June 2026, the IETF published [RFC 10008: The HTTP QUERY Method](https://www.rfc-editor.org/rfc/rfc10008.html). HTTP now has a standard answer for an awkward API-design problem: how do you send a large or structured read-only query without squeezing it into a URL or pretending that a `POST` is a read?

The answer is:

```http
QUERY /contacts HTTP/1.1
Host: api.example.com
Content-Type: application/json
Accept: application/json

{"country":"ID","active":true,"limit":20}
```

`QUERY` carries content like `POST`, but its semantics are safe and idempotent like `GET`.

That does **not** mean every client, router, proxy, cache, and browser feature suddenly understands it. Most HTTP implementations already pass arbitrary method tokens through. Semantic support—automatic retries, correct redirects, cache keys that include request content, API tooling, and generated clients—is the part that will take time.

<!--more-->

![GET, POST, and QUERY trade-offs plus the browser and Go request path](/assets/http-query-method-rfc10008.svg)

## The gap between GET and POST

`GET` is safe, idempotent, cacheable, linkable, and widely understood. Its query input normally lives in the URI:

```http
GET /contacts?country=ID&active=true&fields=id,name,email&limit=20
```

That works until the input becomes a large JSON filter, an AST, GraphQL-like selection, a vector-search request, or another structured query. RFC 10008 notes several problems: unknown URI-length limits across a chain of intermediaries, encoding overhead, greater exposure of URIs in logs and bookmarks, and treating every input combination as a separately identified resource.

A body on `GET` is not the answer. [RFC 9110 says GET request content has no generally defined semantics](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.3.1), and clients or intermediaries may reject it. Browsers' Fetch API goes further and does not allow a body on `GET` or `HEAD`.

`POST` accepts structured content, but it is not safe or idempotent by definition. A human may know that `POST /search` only reads data; generic infrastructure cannot infer that.

RFC 10008 fills that exact gap:

| Property | GET | QUERY | POST |
|---|---:|---:|---:|
| Safe | yes | yes | not guaranteed |
| Idempotent | yes | yes | not guaranteed |
| Request content has defined use | no | yes | yes |
| Response cacheable | yes | yes | limited semantics |
| Query can later receive a URI | already has one | optional | not defined by POST |

The [IANA HTTP Method Registry](https://www.iana.org/assignments/http-methods/http-methods.xhtml) now lists `QUERY` as both safe and idempotent.

## What RFC 10008 actually requires

The target resource decides the query language and scope. The request content plus its media type defines the query; `QUERY` is not tied to SQL, JSON, or any other format.

The important rules are:

- The client does not request a state change to the target resource.
- Repeating the same request has the same intended effect, so retrying is semantically safe.
- `Content-Type` is required and must agree with the content. A server must fail a missing or inconsistent type.
- `200 OK` means the response content contains the query result.
- `415 Unsupported Media Type` fits an unsupported query format; `422 Unprocessable Content` fits validly encoded input that cannot be executed; `406 Not Acceptable` fits an unsupported response format.
- A response is cacheable, but its cache key must incorporate the request content and relevant metadata.

Safe does not mean free. A query may consume CPU, memory, database connections, or create temporary result resources. Authentication, authorization, input limits, timeouts, query-cost limits, and rate limits remain necessary.

### `Accept-Query` advertises formats

A resource can advertise both method support and accepted query media types:

```http
HTTP/1.1 200 OK
Allow: GET, HEAD, QUERY
Accept-Query: "application/jsonpath", application/sql;charset="UTF-8"
```

One easy trap: `Accept-Query` uses [Structured Fields](https://www.rfc-editor.org/rfc/rfc9651.html) syntax. It only resembles the ordinary `Accept` header; do not parse it by splitting blindly on commas and semicolons.

### `Location` and `Content-Location` mean different things

RFC 10008 defines an *equivalent resource*: a GET-able resource representing the QUERY target, content, and relevant metadata.

- `Location` on a successful QUERY can identify the query itself. A later `GET` repeats it and may produce fresh results.
- `Content-Location` can identify this result. A later `GET` retrieves that result.
- `303 See Other` tells the client to switch to `GET` at the supplied URI.
- `301`, `302`, `307`, and `308` mean repeat `QUERY` at the new URI. Unlike the historical POST exception, `301` and `302` do not change QUERY into GET.

That last rule exposes one of today's implementation gaps, especially in Go.

## From draft `-05` to RFC 10008

The supplied [safe-method-with-body draft `-05`](https://www.ietf.org/archive/id/draft-ietf-httpbis-safe-method-w-body-05.html) already had the core design: `QUERY` was safe, idempotent, content-bearing, cacheable, discoverable through `Accept-Query`, and able to identify a query or result with `Location` and `Content-Location`.

The final RFC is substantially more precise. It requires `Content-Type`, defines the equivalent-resource model, spells out media-type errors, redirects, conditional requests, range requests, and cache metadata, and changes `Accept-Query` to Structured Fields syntax. Code based on an old draft should be checked against the final document rather than only renaming a method.

## Go `net/http`: wire support is already here

Go does not need a new protocol implementation to receive QUERY. Since Go 1.22, `ServeMux` patterns can include any valid method token:

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
)

type contactQuery struct {
	Country string `json:"country"`
	Active  bool   `json:"active"`
	Limit   int    `json:"limit"`
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("QUERY /contacts", func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("Content-Type") != "application/json" {
			http.Error(w, "Content-Type must be application/json", http.StatusUnsupportedMediaType)
			return
		}

		var q contactQuery
		if err := json.NewDecoder(http.MaxBytesReader(w, r.Body, 1<<20)).Decode(&q); err != nil {
			http.Error(w, "invalid query", http.StatusBadRequest)
			return
		}

		w.Header().Set("Content-Type", "application/json")
		w.Header().Set("Accept-Query", `"application/json"`)
		json.NewEncoder(w).Encode([]any{})
	})

	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

For a client, pass the literal method to `http.NewRequestWithContext`:

```go
body := strings.NewReader(`{"country":"ID","active":true,"limit":20}`)
req, err := http.NewRequestWithContext(ctx, "QUERY", endpoint, body)
if err != nil {
	return err
}
req.Header.Set("Content-Type", "application/json")
req.Header.Set("Accept", "application/json")

resp, err := client.Do(req)
```

As of July 2026, the [`net/http` method constants](https://pkg.go.dev/net/http#pkg-constants) still do not include `http.MethodQuery`. A local `const methodQuery = "QUERY"` is enough if the literal appears repeatedly; a compatibility package is not.

### The two Go client gaps

Go's current transport only classifies `GET`, `HEAD`, `OPTIONS`, and `TRACE` as replayable methods for its narrow automatic network-error retry path. Its [source and documentation](https://github.com/golang/go/blob/master/src/net/http/transport.go) do not yet include `QUERY`. A QUERY body made from `strings.Reader`, `bytes.Reader`, or `bytes.Buffer` is rewindable, but the method still is not recognized as replayable. Do not assume RFC semantics alone activate Go's automatic retry behavior.

There is also a redirect mismatch. [`http.Client` changes every method except GET and HEAD to GET after 301, 302, or 303](https://github.com/golang/go/blob/master/src/net/http/client.go). RFC 10008 requires QUERY to remain QUERY after 301 and 302; only 303 deliberately switches to GET. Until Go changes this:

- prefer `307` or `308` when a QUERY endpoint moves and the method/content must be preserved;
- use `303` when the response should intentionally become a GET;
- avoid returning `301` or `302` to Go QUERY clients, or install a carefully tested redirect policy.

This is the difference between *can send the bytes* and *fully implements the method semantics*.

## Go framework support

These results are based on current official documentation or source as of July 29, 2026:

| Stack | Route registration | Status and caveat |
|---|---|---|
| `net/http` (Go 1.22+) | `mux.HandleFunc("QUERY /search", h)` | Routes and reads content; no `MethodQuery` constant; client retry/301–302 gaps above |
| Gin | `r.Handle("QUERY", "/search", h)` | Works through its custom-method API; [`Any`](https://github.com/gin-gonic/gin/blob/master/routergroup.go) only registers Gin's fixed method list and does **not** include QUERY |
| chi v5 | `r.MethodFunc("QUERY", "/search", h)` | Works through chi's documented arbitrary-method router API |
| Echo v4 | `e.Add("QUERY", "/search", h)` | Generic registration works |
| Echo v5 | `e.QUERY("/search", h)` | [First-class `QUERY` constant and helper](https://github.com/labstack/echo/blob/master/echo.go) |
| Fiber | `app.Query("/search", h)` | [Current source and next documentation include a first-class helper](https://docs.gofiber.io/next/partials/routing/route-handlers/); older releases may require adding `QUERY` to configured request methods |

Router support does not automatically update CORS middleware, CSRF middleware, OpenAPI generators, access logs, metrics labels, mock clients, or retry libraries. Check any component with a hard-coded method list.

## Browser support: Fetch yes, HTML forms no

Browsers do not need a dedicated `query()` JavaScript API. The [Fetch Standard](https://fetch.spec.whatwg.org/) accepts extension methods except forbidden methods such as `CONNECT`, `TRACE`, and `TRACK`. It only forbids request content for `GET` and `HEAD`, so a same-origin QUERY with content is valid:

```js
const response = await fetch("/contacts", {
  method: "QUERY",
  headers: {
    "Content-Type": "application/json",
    "Accept": "application/json"
  },
  body: JSON.stringify({ country: "ID", active: true, limit: 20 })
});

if (!response.ok) throw new Error(`query failed: ${response.status}`);
const contacts = await response.json();
```

This is generic custom-method support, not proof that every browser subsystem has special RFC 10008 behavior. In practical terms:

- `fetch()` and `XMLHttpRequest` can issue QUERY.
- Service workers see it as a normal Fetch request.
- HTML forms cannot submit it; the [HTML Standard](https://html.spec.whatwg.org/dev/form-control-infrastructure.html) only defines `get`, `post`, and `dialog` form methods.
- Links, the address bar, speculative loading, and ordinary navigation still perform GET.
- Browser HTTP caches should not be assumed to cache QUERY responses correctly until tested.

### Cross-origin QUERY always preflights

RFC 10008 explicitly notes that QUERY is not a CORS-safelisted method. A cross-origin browser request therefore sends an `OPTIONS` preflight:

```http
OPTIONS /contacts HTTP/1.1
Origin: https://app.example
Access-Control-Request-Method: QUERY
Access-Control-Request-Headers: content-type
```

The API must answer with permission for both the origin and method:

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example
Access-Control-Allow-Methods: QUERY
Access-Control-Allow-Headers: Content-Type
Vary: Origin
```

`mode: "no-cors"` is not a workaround: that mode only permits CORS-safelisted methods. Also verify that the framework's CORS middleware does not reject `QUERY` before routing reaches the handler.

## Proxies, gateways, caches, and API descriptions

HTTP/1.1, HTTP/2, and HTTP/3 can all carry extension method tokens. The likely failures are policy and product assumptions:

- a WAF or gateway method allowlist rejects QUERY;
- a reverse proxy routes only a known set of verbs;
- CORS configuration omits QUERY;
- observability groups it as `OTHER` or drops it;
- a CDN refuses to cache it or builds a key without the request content;
- an SDK generator has no representation for the method;
- an API description format or linter only accepts its built-in operation names.

Caching is the hardest part. RFC 10008 says a cache key **must** include the full request content and relevant metadata. A conventional cache keyed by method plus URL is wrong: two different QUERY bodies to `/contacts` would collide. Until a cache explicitly documents RFC 10008 behavior, bypass it or have the server return a `Location` URI for an equivalent GET resource.

QUERY content is less likely than a URI to appear in routine logs, but it is not secret. TLS, authorization, redaction, retention controls, and safe temporary result URIs still matter.

## Should you deploy QUERY now?

For a controlled service-to-service path where you own the client, server, and gateway, yes—especially if POST-based read APIs currently need custom retry and safety conventions. The Go server implementation is small, and `curl` can exercise it directly:

```bash
curl -i -X QUERY https://api.example.com/contacts \
  -H 'Content-Type: application/json' \
  --data '{"country":"ID","active":true,"limit":20}'
```

For a public API with generated SDKs, third-party gateways, browser clients, and CDN caching, add it alongside the existing GET or POST endpoint first. Measure which clients and intermediaries fail before making it the only interface.

A practical rollout checklist:

1. Keep the operation genuinely safe and idempotent.
2. Require and validate `Content-Type`; cap content size and query cost.
3. Register QUERY explicitly in the router, CORS policy, WAF, gateway, and metrics.
4. Test HTTP/1.1, HTTP/2, redirects, retries, cancellation, and cross-origin preflight end to end.
5. Do not enable shared caching unless the cache key includes content and metadata.
6. Advertise supported formats with `Accept-Query`.
7. Retain a GET/POST compatibility path while client tooling catches up.

RFC 10008 solves the semantics. Adoption work is making every layer between the caller and handler preserve those semantics.

## Primary references

- [RFC 10008 — The HTTP QUERY Method](https://www.rfc-editor.org/rfc/rfc10008.html)
- [Earlier draft `draft-ietf-httpbis-safe-method-w-body-05`](https://www.ietf.org/archive/id/draft-ietf-httpbis-safe-method-w-body-05.html)
- [IANA HTTP Method Registry](https://www.iana.org/assignments/http-methods/http-methods.xhtml)
- [Go `net/http` package documentation](https://pkg.go.dev/net/http)
- [WHATWG Fetch Standard](https://fetch.spec.whatwg.org/)
- [WHATWG HTML Standard: form submission](https://html.spec.whatwg.org/dev/form-control-infrastructure.html)
