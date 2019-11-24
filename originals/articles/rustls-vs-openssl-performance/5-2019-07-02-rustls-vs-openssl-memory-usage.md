# rustls versus OpenSSL: memory usage

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

This blog post covers memory usage.  See [the introduction][intro]
for details of other measurements

# Memory usage
Another important facet is how much memory each TLS session consumes.  For
workloads with many concurrent connections this can be a limiting factor.

To test this, we'll measure the peak memory usage of a process which creates
N sessions (N/2 client sessions associated with N/2 server sessions) and
then takes these sessions through a full handshake in lockstep.  This
benchmark design captures any memory usage peaks during the handshake, such
as in certificate handling.

Once we have this for both OpenSSL and rustls, we measure for several values
of N which yields these results:

Cipher suite | N | OpenSSL (KBytes) | Rustls (KBytes) | vs. 2sf
------------ | - | ---------------- | --------------- | ---------
`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` (TLS1.2) | 100 | 13452 | 6140 | -54%
`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` (TLS1.2) | 1000 | 73668 | 29836 | -59%
`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` (TLS1.2) | 5000 | 341604 | 135984 | -60%
`TLS_AES_256_GCM_SHA384` (TLS1.3) | 100 | 12812 | 6184 | -52%
`TLS_AES_256_GCM_SHA384` (TLS1.3) | 1000 | 66000 | 30596 | -54%
`TLS_AES_256_GCM_SHA384` (TLS1.3) | 5000 | 303424 | 139592 | -54%

If we fit against the TLS1.3 values, we get `27.2N + 3417` for rustls
and `59.3N + 6789` for OpenSSL: very roughly a rustls session costs 27KB
and an OpenSSL session costs 59KB peak, in this workload.  This multiplies
up to give a [C10K][c10k] memory usage of 269MB for rustls and 586MB for
OpenSSL.

We see evidence that OpenSSL uses less memory during a TLS1.3 handshake
compared to TLS1.2, but rustls does not.  This might be an area for future
work in rustls.

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
