# DBD::ODBC for perl5-mt

This repository carries a small set of changes on top of upstream
[DBD::ODBC](https://github.com/perl5-dbi/DBD-ODBC) so that it can build and
run in-process (Tier 1) across
[`perl5-mt`](https://github.com/stratum42/perl5-mt) worker threads, while
remaining compatible with stock Perl 5.

DBD::ODBC is the work of Jeff Urlwin, Martin J. Evans, Wesley Hinds, Tim Bunce,
and many contributors over the years. Upstream history is left untouched. For
the original upstream DBD::ODBC build, testing, and support notes, see
[README.upstream.md](README.upstream.md) (and [README](README)).

## What Changed and Why

- **`Makefile.PL`**:
  - Detects `perl5-mt` and appends `-DPERL_XS_THREADSAFE=1` unless
    `PERL_DBI_NO_MT=1` is set.
  - Skips interactive `prompt()` and `sleep 4` pauses when running
    non-interactively or under `perl5-mt`.
  - Adds macOS Homebrew search paths (`/opt/homebrew/opt/unixodbc`,
    `/usr/local/opt/unixodbc`, `libiodbc`) for ODBC header and library
    discovery.
- **`dbdimp.c` and `unicode_helper.c`**:
  - Removes the mutable `static char s_buf[100]` buffer in
    `S_SqlCTypeToString()` in favor of a caller-provided stack buffer.
  - Marks the read-only lookup tables `S_db_options`, `S_st_fetch_params`, and
    `S_st_store_params` as `static const`.
  - Adds `dTHX;` to `WValloc`, `WVfree`, `PVallocW`, and `PVfreeW` in
    `unicode_helper.c` so `PERL_NO_GET_CONTEXT` builds cleanly.
  - Fixes `dbd_st_tables()` where `*catalog = PL_sv_undef; *schema =
    PL_sv_undef;` overwrote caller SV structs instead of reassigning the local
    pointers (`catalog = &PL_sv_undef; schema = &PL_sv_undef;`), which could
    corrupt SV refcounts during `$dbh->ping()` and stall interpreter teardown.
  - Wires `dbi_rtrim_spaces_len()` from `dbi_arch.h` into `ChopBlanks` in
    `dbd_st_fetch()` when available, honoring `dbi_arch_force_scalar()`.
- **`ODBC.pm`**:
  - Declares `$err`, `$errstr`, `$sqlstate`, `$drh`, and
    `$methods_are_installed` with `our` so worker interpreters get isolated
    package state after `MT::Image::freeze()` and `CLONE()`.

## Caveats

- Thread safety in ODBC depends on both the ODBC driver manager (such as
  unixODBC or iODBC) and the underlying database ODBC driver. If a particular
  third-party ODBC driver is not thread-safe, build with `PERL_DBI_NO_MT=1` so
  `perl5-mt` runs it in an isolated Tier 2 compartment instead.
- Each `MT::Worker` must open and manage its own database handles.

## Building

```sh
/path/to/perl5-mt -I/path/to/dbi-mt/blib/lib -I/path/to/dbi-mt/blib/arch Makefile.PL
make
```

## Related Repositories

- [`stratum42/perl5-mt`](https://github.com/stratum42/perl5-mt) — Multi-threaded Perl 5 runtime
- [`stratum42/dbi-mt`](https://github.com/stratum42/dbi-mt) — `DBI` with Tier 1 support, `DBI::MT::Pool`, and `dbi_arch.h`
- [`stratum42/dbi-mt-bench`](https://github.com/stratum42/dbi-mt-bench) — Benchmarks and reproducibility harness

## Upstream Documentation

The original DBD::ODBC README by Tim Bunce, Jeff Urlwin, and Martin J. Evans is
preserved unchanged at [README.upstream.md](README.upstream.md) (and
[README](README)). Feedback, bug reports, and corrections are always welcome.
