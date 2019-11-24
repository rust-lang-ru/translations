# rustls versus OpenSSL: handshake performance

tags: rust, rustls, programming, tls, openssl

There are quite a few dimensions to how performance can vary between TLS
libraries.

Handshake performance covers how quickly new TLS sessions can be
set up.  There are broadly two kinds of TLS handshake: *full* and
*resumed*.  Full handshake performance will be dominated by the
expense of public key crypto -- certificate validation, authentication
and key exchange.  Resumed handshakes require no or
few public key operations, so are much quicker.

Bulk performance covers how quickly application data can be
transferred over an already set-up session.  Performance here
will be dominated by symmetric crypto performance -- the name
of the game is for the TLS library to *stay out of the way* and
minimise overhead in the main data path.  The data rates
concerned are typically many times a typical network link speed.

A TLS library will represent separate sessions in memory while they are
in use.  How much memory these sessions use will dictate how many sessions
can be concurrently terminated on a given server.

This blog post covers handshake performance.  See [the introduction][intro]
for details of other measurements

# Full handshake performance
Full handshake performance is very much a measurement of public key cryptography
speed.  Here, we'll measure server-authenticated handshakes only: this is the
overwhelmingly common deployment of TLS (ie, on the Web).  It means the client
does a key exchange plus some signature verifications (which are often
comparatively cheap).  The server does the same key exchange, and produces
a signature (which is comparatively expensive).  So we expect to see poorer
server figures.

In all these tests, we use the same certificate chain designed to be
representative of typical deployments:

- CA: 4096-bit RSA
- Intermediate: 3072-bit RSA
- End-entity: 2048-bit RSA

## Client performance

Cipher suite | OpenSSL (handshakes/s) | Rustls (handshakes/s) | vs. (2sf)
------------ | ---------------------- | --------------------- | ---------
`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` (TLS1.2) | 2750 | 3350 | +22%
`TLS_AES_256_GCM_SHA384` (TLS1.3) | 2253 | 3188 | +42%

Rustls here is approximately 20% quicker -- but why?  Firstly we verify
that the underlying cryptography choices are the same -- they are.  Secondly,
using the statistical benchmark utility `perf` we can bin the client handshake
samples by the high-level computations they correspond to:

Process | OpenSSL (samples) | rustls (samples)
------- | ----------------- | ----------------
Certificate chain validation | 1222 | 749
Server handshake signature validation | 177 | 149
Key exchange | 412 | 367

The source of the difference here seems to be code that parses X.509
certificates: these functions alone account for about 300 samples in OpenSSL,
while the equivalent functions used by rustls (provided by the [webpki][webpki] crate)
appear in only 5 samples.  The OpenSSL functions seem to have quite significantly
deep call trees (25 frames in places) and significant allocator use.  `webpki`,
in contrast, features zero allocator use and does not copy the certificate data
during parsing.

## Server performance

Cipher suite | OpenSSL (handshakes/s) | Rustls (handshakes/s) | vs. (2sf)
------------ | ---------------------- | --------------------- | ---------
`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` (TLS1.2) | 1307 | 1420 | +8.6%
`TLS_AES_256_GCM_SHA384` (TLS1.3) | 1212 | 1346 | +11%

rustls has a minor advantage here; one which is not particularly interesting
to analyse further.

-----

[rustls]: https://github.com/ctz/rustls
[rustls-master]: https://github.com/ctz/rustls/tree/6a47cd5cb411042d9a8acc591203ede10632ea2e
[openssl-master]: https://github.com/openssl/openssl/tree/fdbb3a86
[oslbench]: https://github.com/ctz/openssl-bench/tree/7bc3277b062c598463d60e6d821198ec5c7a4763
[rustlsbench]: https://github.com/ctz/rustls/blob/6a47cd5cb411042d9a8acc591203ede10632ea2e/examples/internal/bench.rs
[pclmulqdq]: https://www.intel.com/content/www/us/en/processors/carry-less-multiplication-instruction-in-gcm-mode-paper.html
[ring]: https://github.com/briansmith/ring
[webpki]: https://github.com/briansmith/webpki
[boringssl]: https://github.com/google/boringssl
[c10k]: https://en.wikipedia.org/wiki/C10k_problem
[bulk]: /2019/07/02/rustls-vs-openssl-bulk-performance.html
[fullhs]: /2019/07/02/rustls-vs-openssl-handshake-performance.html
[resumption]: /2019/07/02/rustls-vs-openssl-resumption-performance.html
[memory]: /2019/07/02/rustls-vs-openssl-memory-usage.html
[intro]: /2019/07/01/rustls-vs-openssl-performance.html
