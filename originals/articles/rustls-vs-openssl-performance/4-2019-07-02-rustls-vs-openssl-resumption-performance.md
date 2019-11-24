# rustls versus OpenSSL: resumption performance

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

This blog post covers resumption performance.  See [the introduction][intro]
for details of other measurements

# Resumption performance
Resumption elides most of the expensive cryptography that is needed for a
full handshake.  This means we expect to see much higher handshake speeds, and for
other processing (such as parsing, memory allocation, etc.) to limit the results.

An important note: TLS1.3 improved the security of resumption to include a key
exchange even for resumption handshakes.  This means we expect to see slower
performance in TLS1.3 in this case -- in exchange for significantly better
security properties.

## Client performance

Cipher suite | Resumption type | OpenSSL (handshakes/s) | Rustls (handshakes/s) | vs. (2sf)
------------ | --------------- | ---------------------- | --------------------- | ---------
`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` (TLS1.2) | Stateful | 22527 | 29802 | +32%
`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` (TLS1.2) | Stateless (tickets) | 20835 | 27160 | +30%
`TLS_AES_256_GCM_SHA384` (TLS1.3) | Stateless | 4861 | 8211 | +69%

In profiling the TLS1.3 case we expect to see the client performing two curve25519
point multiplications (one in key generation, one in key agreement).  But
in OpenSSL we see evidence of *three* point multiplications -- *two* in
key generation.  It appears OpenSSL recalculates and then discards the
local public key when processing the server's key share extension.  This likely
explains the larger performance deficit in this test.

## Server performance

Cipher suite | Resumption type | OpenSSL (handshakes/s) | Rustls (handshakes/s) | vs. (2sf)
------------ | --------------- | ---------------------- | --------------------- | ---------
`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` (TLS1.2) | Stateful | 23487 | 25564 | +8.8%
`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` (TLS1.2) | Stateless (tickets) | 19014 | 22876 | +20%
`TLS_AES_256_GCM_SHA384` (TLS1.3) | Stateless | 5154 | 5957 | +16%

Again, we see OpenSSL has a relatively minor but measureable performance deficit.

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
