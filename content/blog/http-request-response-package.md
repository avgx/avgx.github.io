+++
title = "RequestResponse: describe HTTP requests, keep transport separate"
description = "Build URLRequests and decode responses from typed request descriptions — no bundled URLSession client, retries, or logging. Ideas from kean/Get. CI and version tags on GitHub Actions."
date = 2026-04-16

[taxonomies]
tags = ["swift", "http", "swiftpm", "github-actions", "architecture"]
+++

Most “HTTP client” libraries start as a thin wrapper over `URLSession` and slowly grow into a kitchen sink: retries, logging, interceptors, multipart helpers, token refresh, and half of your app’s error taxonomy. That is not wrong for a product, but it is heavy when you only want one thing: **a clear place to say what a request means**, without committing to every transport decision up front.

[Get](https://github.com/kean/Get) by [kean](https://mastodon.social/@a_grebenyuk) is a lean Swift API client built around `Request<Response>` and an `APIClient` that sends and decodes. I have been treating my own direction as a fork of that idea in spirit: not a bigger framework, but **smaller pieces with obvious jobs**.

This post is about the first published slice: [**RequestResponse**](https://github.com/avgx/RequestResponse).

## What problem it solves

The goal is simple to state and annoyingly easy to violate in real codebases:

- **Descriptive layer** — method, path, query, headers, JSON body shape, and the type you *expect* back. This is the contract with your backend, usually stable across apps and targets.
- **Transport layer** — `URLSession` configuration, delegates, retries, certificate pinning, background sessions, throttling, and whatever else the environment demands.

When those layers collapse into one type, every new transport feature drags the protocol description along with it, and every new endpoint drags session policy into places that should stay dumb.

`RequestResponse` stays on the descriptive side. You model calls with `Request<Response>` (the `Response` type parameter is for **documentation and type flow**; it does not change encoding). A `RequestBuilder` turns that model into a `URL` or `URLRequest`. After `URLSession` returns `(Data, URLResponse)`, you wrap decoding with `decodeBody` and a small `Response<T>` that keeps the decoded value next to the raw payload and status code.

The point is that separation. **You spell out each call as a `Request` value**—path, method, body, expected type—**and you send it with whatever `URLSession` setup your app already has.** The library does not smuggle “what the backend expects” into session configuration or a fat client object.

## What is intentionally out of scope (for now)

There is no bundled client object that performs the network call, no retry policy, and no logging hooks in the package itself. Those belong next to your app’s lifecycle and testing strategy. The library answers: “Given this `Request`, what `URLRequest` should I send?” and “Given this `Data`, what typed value does my endpoint claim?”

If you squint, this is the opposite of a monolith like Alamofire at its most framework-y: fewer extension points inside the library, more composition **outside** it.

## Swift 6

The package manifest targets **Swift 6.1+**. Concurrency is strict in the way Swift 6 expects: bodies are `(any Encodable & Sendable)?`, and decoded generics land on **`Decodable & Sendable`**. Encoding can hop off the caller’s actor with `Task.detached` so main-actor UI code does not fight JSON work.

## Testing in CI and locally

Workflows live under `.github/workflows/`. On every push and pull request to `main`, **CI** runs a plain `swift build` and `swift test` on **macOS** with the Swift 6.1 toolchain. 

Locally, the same commands are the source of truth:

```bash
swift build
swift test
```

The repository also carries a disabled integration-style test that hits [httpbin.org](https://httpbin.org) to echo JSON—useful when you are validating end-to-end wiring with a real session. Keeping that test **opt-in** preserves fast, deterministic automation while still documenting a realistic usage path in code.

## Releases with GitHub Actions

Tag-driven releases keep shipping predictable. When you push a tag that matches **`v*.*.*`** (for example `v1.0.1`), the **Release** workflow runs `swift build -c release`, runs tests again, then creates a **GitHub Release** with auto-generated notes.

```bash
git tag -a v1.0.1 -m "v1.0.1"
git push origin v1.0.1
```

## Why I’m extracting something this small

**Small** here means **stable boundaries**. Transport will keep changing. The shape of your REST or JSON-RPC surface changes on a different clock. `RequestResponse` is a slice that focuses on data flow and nothing else.
