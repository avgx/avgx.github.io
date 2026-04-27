+++
title = "SSLPinning: fingerprint-based server trust with user decision flow"
description = "Verify server certificates by fingerprint, handle unknown hosts with a user decision loop, and keep the logic out of your networking stack."
date = 2026-04-27

[taxonomies]
tags = ["swift", "http", "security", "ssl-pinning", "swiftpm"]
+++

This note continues a small HTTP series. The previous angle was how to describe a call without folding session policy into the same type. Here the focus shifts to **trust on the wire**: who the app accepts as the server for a TLS connection. That work belongs in the transport layer—in a `URLSession` delegate and trust callbacks—not in request builders or decoders.

The published slice is [**SSLPinning**](https://github.com/avgx/SSLPinning/): fingerprint-based pinning, a synchronous `ServerTrustEvaluator`, and typed outcomes when pinning or system trust rejects a challenge.

## What is fingerprint pinning

Pinning means the app only trusts a specific certificate or public key, not the whole system trust store. Fingerprint pinning narrows that further: you store a hash (SHA-256) of the certificate's DER data. On connection, the app hashes the server's leaf certificate and compares. If the hashes match, the connection proceeds. If they don't, the connection is cancelled.

This protects against compromised CAs signing certificates for your domain.

## How it looks in practice

`SSLPinning` handles this inside `URLSessionDelegate.urlSession(_:didReceive:completionHandler:)`. No wrapping the session, no base client class. You create a `ServerTrustEvaluator` with a policy, call it synchronously for server-trust challenges, and forward its result to the completion handler.

```swift
final class TLSDelegate: NSObject, URLSessionDelegate {
    private let evaluator: ServerTrustEvaluator

    init(policy: ServerTrustPolicy) {
        self.evaluator = ServerTrustEvaluator(policy: policy)
        super.init()
    }

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        let result = evaluator.evaluate(challenge)
        completionHandler(result.disposition, result.credential)
    }
}
```

## The user decision loop

This is where SSLPinning shapes a specific flow rather than just giving you pin-mismatch errors:

First attempt — unknown host. Your app has no pin for this server yet. The evaluator returns unknownHost(host:presentedChain:). The task fails with a URLError. You show an alert with the fingerprint from presentedChain.first?.sha256 and let the user decide:

- Trust: build a Pin from the CertificateInfo, add it to your pin list, and retry.
- Don't trust: do nothing, the connection stays blocked.

Second attempt — known host, matching pin. Now the pin list contains the fingerprint. The evaluator returns allow, the connection proceeds, and the user never sees an alert again for this certificate.

Mismatch later. If the server certificate changes and the fingerprint no longer matches any trusted pin, the evaluator returns pinMismatch. Don't retry automatically — that would defeat pinning. Either update the pin if you rotated certificates deliberately, or treat it as a security incident.

This turns pinning from a binary pass/fail into a decision the user participates in once per unknown host, without waiting inside the delegate or blocking other requests.

## What the library gives you

- ServerTrustPolicy.pinning(_:) takes an array of Pin structs. Each Pin holds a host, serial number, and SHA-256 fingerprint.
- ServerTrustEvaluator checks the server's leaf certificate, extracts fingerprints, and compares.
- CertificateInfo wraps SecCertificate properties: sha256, sha1, commonName, subjectSummary, validity dates where available (iOS 18+/macOS 15+).
- All errors are typed (SSLPinningError) so you can switch on unknownHost vs pinMismatch vs systemTrustFailed in your UI code.

## Server trust, not request shape

`SSLPinning` answers whether **this** certificate chain is acceptable for **this** host. That question opens in the delegate when `URLSession` asks for server trust; it does not belong in how you spell method, path, headers, or the type you decode into.

[RequestResponse](@/blog/http-request-response-package.md) covers the other side of the line: what the backend expects on the wire, described **without** folding session configuration or trust policy into the same abstraction.
