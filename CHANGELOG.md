# Changelog

All notable changes to libbrotli-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-18

The documentation and comments in plain prose; no declaration changed.

## 0.1.0 — 2026-09-15

The first release: thirteen entry points of the libbrotlidec C API, one
`@ffi` declaration each, and no logic.

### Added

- `libbrotli` — the whole decoder surface, in four groups.
  - The instance: `BrotliDecoderCreateInstance`,
    `BrotliDecoderDestroyInstance`, `BrotliDecoderSetParameter` and
    `BrotliDecoderAttachDictionary`.
  - Decompression: `BrotliDecoderDecompress` and
    `BrotliDecoderDecompressStream`.
  - The output the decoder holds: `BrotliDecoderHasMoreOutput`,
    `BrotliDecoderTakeOutput`, `BrotliDecoderIsUsed` and
    `BrotliDecoderIsFinished`.
  - Errors and version: `BrotliDecoderGetErrorCode`,
    `BrotliDecoderErrorString` and `BrotliDecoderVersion`.
- `tests/libbrotli_tests.nv` — nine tests over the signatures. They
  call the C library, so they need libbrotlidec installed. The frame
  they decompress is a fixed vector, carried in the suite as hex,
  because this package binds the decoder and cannot produce one.

### Named as missing

**The encoder.** Brotli ships as three shared libraries:
`libbrotlicommon` holds the shared tables, `libbrotlidec` the decoder
and `libbrotlienc` the encoder. A `sys` package wraps exactly one
library, and this one wraps the decoder. Every symbol beginning
`BrotliEncoder` is therefore absent, including
`BrotliEncoderCompress`, `BrotliEncoderCreateInstance` and
`BrotliEncoderCompressStream`. A program that has to compress needs a
second package, which this release does not provide.

**The metadata callbacks.** `BrotliDecoderSetMetadataCallbacks` takes
two C function pointers, and a novo-lang function is not one.
