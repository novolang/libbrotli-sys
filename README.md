# libbrotli-sys

Brotli is a lossless compression algorithm and data format. It combines
a modern variant of the LZ77 algorithm, Huffman coding and second-order
context modelling with a built-in dictionary of common web text, which
is what makes it good at small inputs. The format is specified in
[RFC 7932](https://www.rfc-editor.org/rfc/rfc7932.html) and the
reference implementation is the C library the Brotli project publishes.
This package declares the thirteen entry points of that library's
**decoder** to novo-lang, one declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in libbrotlidec. The package contains no
logic of its own, and it does nothing without the C library installed.
The thirteen entry points are the whole decoder interface apart from
the metadata callbacks; the section "What is not included" says what a
program still cannot do with them alone, and the first item is
compression.

## What it is

Brotli's reference implementation ships as **three shared libraries**.
`libbrotlicommon` holds the tables the other two share.
`libbrotlidec` decompresses. `libbrotlienc` compresses. They have
separate headers, separate symbol prefixes and separate link flags.

This package wraps `libbrotlidec`. A package on the bindings shelf
wraps exactly one C library, so the encoder is not here.

A **decoder instance** holds the state of a decompression in progress:
the sliding window, the Huffman tables and the partly decoded output.
`BrotliDecoderCreateInstance` creates one and
`BrotliDecoderDestroyInstance` releases it.

There are two ways in. `BrotliDecoderDecompress` takes a whole
compressed stream and a buffer big enough for the whole result, and
needs no instance. `BrotliDecoderDecompressStream` takes an instance
and works a piece at a time.

The streaming call reports one of **four results**. They are the
protocol between the caller and the decoder, and the caller's loop is
written around them.

| Code | Name | What the caller does |
| --- | --- | --- |
| 0 | `BROTLI_DECODER_RESULT_ERROR` | Stop, and read `BrotliDecoderGetErrorCode`. |
| 1 | `BROTLI_DECODER_RESULT_SUCCESS` | The stream is complete. |
| 2 | `BROTLI_DECODER_RESULT_NEEDS_MORE_INPUT` | Refill the input and call again. |
| 3 | `BROTLI_DECODER_RESULT_NEEDS_MORE_OUTPUT` | Drain the output and call again. |

A **shared dictionary** is sample data both sides agree on before the
stream begins. Brotli has a built-in dictionary of about 120 kB of
common web text, and `BrotliDecoderAttachDictionary` supplies an extra
one of the caller's own.

The **window** is how far back the algorithm may refer. RFC 7932
section 9.1 caps it at 16 MB. A later extension allows more, and
`BrotliDecoderSetParameter` with parameter 1 is what accepts a stream
that uses it.

## Install

```
novo pkg add libbrotli-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the libraries and their headers come from the system package
`libbrotli-dev`, which carries all three:

```
sudo apt install libbrotli-dev
```

On macOS the Homebrew formula is `brotli`. On other systems the
libraries build from the Brotli source with CMake.

## Example

A compressed buffer decompressed in one call:

```novo ignore
use libbrotli

fn main(frame: Int, frame_len: Int) [io, ffi]
    // The slot holds the buffer's size on the way in and the number of
    // bytes written on the way out.
    let out = ptr.alloc(65536)
    let size = ptr.alloc_word()
    ptr.write_word(size, 65536)

    let rc = libbrotli.brotli_decoder_decompress(frame_len, frame, size, out)
    if rc != 1
        println("the stream did not decompress")
        return
    let n = ptr.read_word(size)
    println("${n} bytes: " + bytes.to_str(ptr.read_bytes_n(out, n)))

    ptr.free(size)
    ptr.free(out)
```

The example is fenced as an illustration rather than a compiled block
because it needs a compressed stream, which this package cannot
produce. `tests/libbrotli_tests.nv` carries one as a constant and runs
these calls against it.

## What the package contains

| Module | Contents |
| --- | --- |
| `libbrotli` | Every decoder entry point, in four groups: the instance, decompression, the output the decoder holds, and errors and version. |

The four groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Instance | 4 | Creates and releases a decoder, sets its parameters and attaches a dictionary. |
| Decompression | 2 | Decompresses a whole stream in one call, or a piece at a time. |
| Held output | 4 | Reports and collects output the decoder kept, and reports whether it has started and whether it has finished. |
| Errors and version | 3 | Reads and names the last error, and reports the version. |

## How to choose an entry point

`BrotliDecoderDecompress` is for a compressed stream already in memory
whose decompressed size is known, or can be bounded. It is one call and
it needs no instance.

`BrotliDecoderDecompressStream` is for everything else: a stream that
arrives in pieces, an output that does not fit in memory, or a
decompressed size nobody knows. Brotli records no decompressed size in
its stream, so this is the ordinary case.

Inside the streaming loop there are two ways to take the output. Giving
the decoder a buffer of the caller's own, through `available_out` and
`next_out`, is the direct one. Giving it no room at all and then
calling `BrotliDecoderTakeOutput` saves a copy, because the caller
reads the decoder's own buffer, but the bytes are only valid until the
next call.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** The decoder instance
   arrives as the address the C library returned.
2. **A buffer is an address and a length.** The caller reserves the
   bytes with `ptr.alloc`, writes them with `ptr.write_bytes_buf`, and
   reads them back with `ptr.read_bytes_n`. `ptr.alloc` does not clear
   the bytes, so read back exactly the length the call reported.
3. **The streaming call updates five slots, and each is the address of
   one value.** The two lengths count down, the two addresses walk
   forward, and the total counts up. Read them back after every call;
   they are how the caller learns what happened.

   | Slot | Before the call | After the call |
   | --- | --- | --- |
   | `available_in` | bytes of input available | bytes of input left |
   | `next_in` | the address of the first input byte | the address of the first unread byte |
   | `available_out` | room in the output buffer | room left |
   | `next_out` | the address of the first free output byte | the address after the last byte written |
   | `total_out` | ignored | every byte produced so far |

4. **A Brotli stream does not record its decompressed size.** RFC 7932
   has no such field. A caller that must allocate up front either knows
   the size from somewhere else or uses the streaming call.
5. **`BrotliDecoderDecompress` answering 0 covers two cases.** The
   stream was corrupt, or the output buffer was too small. The one-shot
   call has nowhere to continue from, so it reports both the same way.
6. **The error codes are negative, and they arrive in 32 bits.** Write
   `as i32` on the answer from `BrotliDecoderGetErrorCode` before
   comparing it with a negative number, and pass the converted value to
   `BrotliDecoderErrorString`.
7. **A result of 2 means the stream is truncated when there is no more
   input.** `BROTLI_DECODER_RESULT_NEEDS_MORE_INPUT` after the last
   byte has been given is the only signal a cut-short stream produces.
   A caller that treats it as end-of-stream accepts a partial result as
   a complete one.
8. **Parameters and dictionaries must be set before the first input.**
   `BrotliDecoderIsUsed` answers 1 once the decoder has been fed, and
   `BrotliDecoderSetParameter` answers 0 after that.
9. **The bytes `BrotliDecoderTakeOutput` answers belong to the
   decoder.** They are valid until the next call on the instance. Copy
   them before making one.
10. **A decompressed size is a size the stream chose.** A small
    compressed input can produce a very large output, so a program
    reading untrusted data bounds the total itself rather than trusting
    the stream to stop.

## What is not included

- **The encoder.** Every `BrotliEncoder` entry point is in a different
  shared library, `libbrotlienc`, and one `sys` package wraps one
  library. `BrotliEncoderCompress`, `BrotliEncoderCreateInstance`,
  `BrotliEncoderCompressStream`, `BrotliEncoderSetParameter` and their
  neighbours are absent. A program that has to compress needs a package
  that binds that library, which this release does not provide.
- **`BrotliDecoderSetMetadataCallbacks`.** It takes two C function
  pointers, and a novo-lang function is not one. Metadata blocks are
  skipped silently without it, which is the default behaviour.
- **The shared dictionary construction interface.** The
  `BrotliSharedDictionary` type and the calls that build a serialised
  dictionary are in `libbrotlicommon`, a third library.
  `BrotliDecoderAttachDictionary` is here and takes either form.
- **The custom allocator.** `BrotliDecoderCreateInstance` takes three
  arguments for one, and this package passes them through; a novo-lang
  function cannot be supplied for the first two, so 0 is the only
  usable value.

## Related packages

`brotli-nv` is the Brotli codec written in novo-lang, with no C
library. This package is the reference implementation that port
measures itself against, and it is the decoding half of it.
`brotli-nv` is planned and not published yet.

Choose `brotli-nv` when the program must build for a microcontroller or
for WebAssembly, when a C toolchain is not wanted, or when the program
has to compress as well as decompress. Choose this package when the
program needs the reference decoder's speed or its exact behaviour on a
stream.

`libzstd-sys` binds Zstandard, a different compression format with both
halves in one library.

## Tests

`tests/libbrotli_tests.nv` holds nine tests written against the
signatures. They call the C library, so `novo test` needs libbrotlidec
installed and linkable:

```
novo test tests/libbrotli_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

The suite decompresses in memory and reads and writes nothing. Because
this package binds the decoder, it cannot produce a compressed stream,
so the frame the tests decompress is a fixed 59-byte vector carried in
the suite as hex. It was produced from a 60-byte message at quality 11
with the default window.

The tests assert that the one-shot call recovers the message exactly;
that a buffer too small is refused; that a fresh instance is unused,
unfinished and holding nothing; that a parameter is accepted before the
first input and an unknown one is not; that the streaming call recovers
the same message and leaves the five slots where the reference says it
should; that a decoder given no output room asks for more and hands its
buffer over; that sixteen bytes of text are refused with a negative
error code that has a name; and that a raw dictionary attaches.

## Implementation status

| Group | State |
| --- | --- |
| Instance | Complete apart from the metadata callbacks. |
| Decompression | Complete. |
| Held output | Complete. |
| Errors and version | Complete. |
| Encoder | Absent. It is a separate shared library. |
| Metadata callbacks | Absent. They take C function pointers. |
| Dictionary construction | Absent. It is in `libbrotlicommon`. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

Brotli itself is distributed under the MIT licence, and installing it
is the reader's own step.
