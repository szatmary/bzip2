# bzip2

A drop-in replacement for Go's `compress/bzip2` decoder that's about 1.5–2 times as fast, with no extra memory.

It has the same API as the standard library package (`NewReader` and `StructuralError`), so switching is a one-line import change:

```go
import "github.com/szatmary/bzip2"

r := bzip2.NewReader(f) // same as compress/bzip2
```

```
go get github.com/szatmary/bzip2@latest
```

It requires Go 1.22 or later.

## Why

The changes are proposed for the standard library, as [the pull requests](https://github.com/szatmary/bzip2-speedup/pulls) #1, #2 and #4. This module lets you use them now. If they're accepted, switch back to `compress/bzip2`.

## Performance

Compared with `compress/bzip2` at Go master:

| | Apple M4 | Intel i5-8500B |
|---|---|---|
| File benchmarks (geomean), one decoder | 1.79× | 1.62× |
| 31 MB of Go source | 50.0 → 79.3 MB/s (1.59×) | 34.5 → 49.0 MB/s (1.42×) |
| 16 MB Go binary | 35.8 → 62.2 MB/s (1.74×) | 24.2 → 39.6 MB/s (1.64×) |
| 30 MB random data | 17.4 → 39.8 MB/s (2.29×) | 11.9 → 24.1 MB/s (2.03×) |
| Decoders in parallel: M4 1 / 2 / 4 / 6 / 8 / 10, i5 1 / 2 / 3 / 4 / 6 | 1.57× / 1.57× / 1.49× / 1.47× / 1.50× / 1.42× | 1.49× / 1.57× / 1.45× / 1.31× / 1.20× |

Against the C `bzip2` tool (1.0.8) on the M4, this module is faster on the Go binary and random data, and 5% slower on Go source (MB/s: Go source 79 against 83, the Go binary 62 against 59, random data 40 against 34).

Where the speed comes from:

1. **A 64-bit bit buffer**, refilled once per Huffman symbol instead of checked on every bit. It reads eight bytes at a time from a `*bufio.Reader`, including the one `NewReader` creates for an `*os.File`.
2. **Huffman lookup tables** that decode codes of up to 10 bits in one step.
3. **A hardware-accelerated CRC**, from `hash/crc32` applied to bit-reversed bytes.

A fourth change, walking the inverse Burrows–Wheeler transform as two chains ([#3](https://github.com/szatmary/bzip2-speedup/pull/3)), is left out. It adds 6–17% for a single stream, most on large blocks, but needs up to 4.5 MB more per decoder, and with 3 or more decoders in parallel it's usually slower, by up to 12%. v0.1.0 included an earlier version of it.

## Differences from `compress/bzip2`

- **Errors are sticky.** After an error, every later `Read` returns the same error and no data. The standard library can return data from later in the stream after an error, and can replace a read error with `io.ErrUnexpectedEOF`.
- **Memory:** 12 KB more per decoder, for Huffman tables.
- **Worst case:** a stream of many one-byte blocks decodes 10–15% slower. Real bzip2 files use blocks of 100–900 KB.
- Like the standard library, it never reads past the end of a valid stream from an `io.ByteReader`.

## Testing

The package keeps the standard library's tests and adds more for truncated input, read errors, sticky errors, reader position, and every input path (`io.Reader`, `io.ByteReader`, and `*bufio.Reader` of several sizes). It was also reviewed for hostile input. Differential fuzzing against the standard library (over 2 million executions, plus every truncation and single-bit flip of the test inputs) found no difference in output or errors. A stream of many tiny blocks, built to stress CPU and memory, has a regression test.

## License

BSD-3-Clause, the same as Go (see [LICENSE](LICENSE)). The code is derived from Go's `compress/bzip2`.
