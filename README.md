# bzip2

A drop-in replacement for Go's `compress/bzip2` decoder that's about twice as fast.

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

The changes are proposed for the standard library ([the pull requests](https://github.com/szatmary/bzip2-speedup/pulls)). This module lets you use them now. If they're accepted, switch back to `compress/bzip2`.

## Performance

Compared with `compress/bzip2` at Go master:

| | Apple M4 | Intel i5-8500B |
|---|---|---|
| Package benchmarks on files, geomean | 2.03× faster | 1.72× faster |
| 31 MB of Go source | 48.8 → 90.3 MB/s | 34.4 → 49.9 MB/s |
| 16 MB Go binary | 34.0 → 72.3 MB/s | 24.2 → 41.7 MB/s |
| 30 MB random data | 17.3 → 43.2 MB/s | 11.9 → 25.5 MB/s |

On the M4 it's faster than the C `bzip2` tool (1.0.8) on all three inputs.

Where the speed comes from:

1. **A 64-bit bit buffer**, refilled once per Huffman symbol instead of checked on every bit. It reads eight bytes at a time from a `*bufio.Reader`, including the one `NewReader` creates for an `*os.File`.
2. **Huffman lookup tables** that decode codes of up to 10 bits in one step.
3. **Two bytes per memory load** when walking the inverse Burrows–Wheeler transform, which is otherwise limited by memory latency.
4. **A hardware-accelerated CRC**, from `hash/crc32` applied to bit-reversed bytes.

## Differences from `compress/bzip2`

- **Errors are sticky.** After an error, every later `Read` returns the same error and no data. The standard library can return data from later in the stream after an error, and can replace a read error with `io.ErrUnexpectedEOF`.
- **Memory:** up to 3.6 MB more per decoder for the largest (900k) blocks, and 12 KB for Huffman tables. That extra memory grows with the blocks actually decoded, so small streams use less.
- **Worst case:** a stream of many one-byte blocks decodes 11–16% slower. Real bzip2 files use blocks of 100–900 KB.
- Like the standard library, it never reads past the end of a valid stream from an `io.ByteReader`.

## Testing

The package keeps the standard library's tests and adds more for truncated input, read errors, sticky errors, reader position, and every input path (`io.Reader`, `io.ByteReader`, and `*bufio.Reader` of several sizes). It was also reviewed for hostile input. Differential fuzzing against the standard library (over 2 million executions, plus every truncation and single-bit flip of the test inputs) found no difference in output or errors. Streams built to stress memory or CPU, such as many tiny blocks or steadily growing blocks, have regression tests.

## License

BSD-3-Clause, the same as Go (see [LICENSE](LICENSE)). The code is derived from Go's `compress/bzip2`.
