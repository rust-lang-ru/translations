# rustls versus OpenSSL: bulk performance

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

This blog post covers bulk performance.  See [the introduction][intro]
for details of other measurements, and the versions we're benchmarking.

# Bulk performance
Bulk performance is obviously sensitive to underlying network
performance, TCP stack efficiency, overheads in the kernel/userland
interface, etc.  We'll design our tests to avoid these interfaces,
and instead measure an *upper bound* on performance which you might
attain if they all imposed no extra overhead.

That means no network or kernel differences are relevant in the
tests, which makes them more reproducible.  However, the normal
caveats about micro-benchmarks apply: we're not testing the whole
system, and real-world results are likely to be different.

## Codebase lineage

It's worth mentioning here that rustls and OpenSSL share most
of the underlying cryptography code.  rustls depends on [ring][ring]
for all its cryptography, which itself has its roots in Google's
OpenSSL fork, [BoringSSL][boringssl].

So we don't expect to see significant performance differences.

These tests, then, are more concerned with whether the TLS library
can keep the underlying crypto well-fed to reach its maximal potential.

## Sending speed

This covers how quickly a TLS library can convert application
data into TLS frames.  We send 1GB of data in chunks of 1MB, and
time how long that takes.  The timing does not include sending
the TLS frames on the receive-side.

<div align="center"><img src="/assets/diagrams/tls-session-send.png"
  alt="diagram showing the sending direction of a TLS session, with plaintext entering on the left and ciphertext leaving on the right" /></div>

Cipher suite | OpenSSL (MB/s) | Rustls (MB/s) | vs. (2sf)
------------ | -------------- | ------------- | ---------
`ECDHE-RSA-AES128-GCM-SHA256` | 1919.46 | 2257.01 | +18%
`ECDHE-RSA-AES256-GCM-SHA384` | 1708.9 | 1963.64 | +15%
`ECDHE-RSA-CHACHA20-POLY1305` | 1158.24 | 1317.08 | +14%
`TLS_AES_256_GCM_SHA384` (TLS1.3) | 1721.12 | 1946.56 | +13%

The difference between OpenSSL and rustls appears to be thanks to an
extra copy in the main data-path in OpenSSL.

## Receiving speed

This covers how quickly a TLS library can process TLS frames into
the corresponding application data.  The TLS frames result from
encrypting 1GB of total application data in chunks of 1MB.

The timing includes covers the transport of TLS frames from memory
into the TLS library, because at this point the TLS library can
perform computation on the frames.

<div align="center"><img src="/assets/diagrams/tls-session-recv.png"
  alt="diagram showing the receive direction of a TLS session, with ciphertext entering on the right and plaintext leaving on the left" /></div>

Cipher suite | OpenSSL (MB/s) | Rustls (MB/s) | vs. (2sf)
------------ | -------------- | ------------- | ---------
`ECDHE-RSA-AES128-GCM-SHA256` | 2138.1 | 2296.97 | +7.4%
`ECDHE-RSA-AES256-GCM-SHA384` | 1902.82 | 1999.44 | +5.1%
`ECDHE-RSA-CHACHA20-POLY1305` | 1281.75 | 1357.46 | +5.9%
`TLS_AES_256_GCM_SHA384` (TLS1.3) | 1891.37 | 2000.70 | +5.8%

There's nothing in it on this measurement.

## Conclusions

These are quite respectable speeds: both libraries can achieve 10Gbit/s
per core at full pelt, in either direction.

It's expected that the two AES-GCM suites are quicker than the chacha20-poly1305
suite -- they're hardware accelerated (using the AES-NI and [pclmulqdq][pclmulqdq]
instructions).  If we were to disable that acceleration, the speeds drop
to approx 230MB/s.  That gives a flavour of why chacha20-poly1305 is valuable
in modern TLS: performance where AES-GCM acceleration is not available.

In the sending direction, OpenSSL has an extra copy of the plaintext application
data.  Rustls avoids a copy here, but the performance advantage is relatively
minor.

-----

[rustls]: https://github.com/ctz/rustls
[rustls-master]: https://github.com/ctz/rustls/tree/6a47cd5cb411042d9a8acc591203ede10632ea2e
[openssl-master]: https://github.com/openssl/openssl/tree/fdbb3a86
[oslbench]: https://github.com/ctz/openssl-bench/tree/7bc3277b062c598463d60e6d821198ec5c7a4763
[rustlsbench]: https://github.com/ctz/rustls/blob/6a47cd5cb411042d9a8acc591203ede10632ea2e/examples/internal/bench.rs
[pclmulqdq]: https://www.intel.com/content/www/us/en/processors/carry-less-multiplication-instruction-in-gcm-mode-paper.html
[ring]: https://github.com/briansmith/ring
[boringssl]: https://github.com/google/boringssl
[c10k]: https://en.wikipedia.org/wiki/C10k_problem
[bulk]: /2019/07/02/rustls-vs-openssl-bulk-performance.html
[fullhs]: /2019/07/02/rustls-vs-openssl-handshake-performance.html
[resumption]: /2019/07/02/rustls-vs-openssl-resumption-performance.html
[memory]: /2019/07/02/rustls-vs-openssl-memory-usage.html
[intro]: /2019/07/01/rustls-vs-openssl-performance.html
