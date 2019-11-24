# TLS performance: rustls versus OpenSSL

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

This series of blog posts measures and compares the performance of
rustls (a TLS library in rust) and OpenSSL.

# Reproducibility

We'll measure current master for rustls ([6a47cd5c][rustls-master])
and OpenSSL ([fdbb3a86][openssl-master]).

OpenSSL was built from source with default options, using gcc 8.3.0.
rustls was built from source using rustc 1.35.0.

All measurements below were obtained on the same i5-6500 Skylake at
3.2GHz, on Debian linux.  CPU scaling was turned off.

All benchmarking tools take an environment variable `BENCH_MULTIPLIER`
which extends each test's duration by some multiple.  In results below,
this is set to `BENCH_MULTIPLIER=16`, meaning each test takes significant
time in an effort to make the effect of cold CPU and page caches irrelevent.

## rustls

The code used is in kept [alongside rustls][rustlsbench].  Build and
run with:

```
$ cargo test --no-run --example bench --release
$ BENCH_MULTIPLIER=16 make -f admin/bench-measure.mk measure
$ make -f admin/bench-measure.mk memory
```

## OpenSSL

All the code used is in [ctz/openssl-bench][oslbench] as of the linked
commit.  It expects to find a built OpenSSL tree in `../openssl/`.  Then
just run:

```
$ BENCH_MULTIPLIER=16 make measure
$ make memory
```

# Results

Performance results are presented in other blog posts as follows:

- [Bulk performance][bulk].
- [Full handshakes][fullhs].
- [Resumed handshakes][resumption].
- [Memory usage][memory].

See those posts for details and analysis.  To summarise the results, though,
we can say approximately:

- rustls is 15% quicker to send data.
- rustls is 5% quicker to receive data.
- rustls is 20-40% quicker to set up a client connection.
- rustls is 10% quicker to set up a server connection.
- rustls is 30-70% quicker to resume a client connection.
- rustls is 10-20% quicker to resume a server connection.
- rustls uses less than half the memory of OpenSSL.

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
