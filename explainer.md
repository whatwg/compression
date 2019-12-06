# Compression Streams Explained
27 August 2019


## What’s all this then?

The "gzip" and "deflate" compression algorithms are extensively used in the
web platform, but up until now have not been exposed to JavaScript. Since
compression is naturally a streaming process, it is a good match for the
[WHATWG Streams](https://streams.spec.whatwg.org/) API.

`CompressionStream` is used to compress a stream. It accepts ArrayBuffer or
ArrayBufferView chunks, and outputs Uint8Array.

`DecompressionStream` is used to decompress a stream. It accepts
ArrayBuffer or ArrayBufferView chunks, and outputs Uint8Array.

Both APIs satisfy the concept of a [transform
stream](https://streams.spec.whatwg.org/#ts-model) from the WHATWG
Streams Standard.


## Goal

The goal is to provide a JavaScript API for compressing and decompressing data
in the "gzip" ([RFC1952](https://tools.ietf.org/html/rfc1952)) or "deflate"
([RFC1950](https://www.ietf.org/rfc/rfc1950.txt)) formats.


## Non-goals

*   Compression formats other than "gzip" and "deflate" will not be
    supported in the first version of the API.
*   Support for synchronous compression.


## Motivation

Existing libraries can be used to perform compression in the browser, however
there are a number of benefits to a built-in capability:

*   A built-in API does not need to be downloaded. This is particularly
    relevant when compression is being used with the goal of reducing bandwidth
    usage.
*   Native compression libraries are faster and use less power than JavaScript
    or WASM libraries. Benchmarks performed at Facebook found that the native
    implementation was 20x faster than JavaScript and 10% faster than WASM.
*   Ergonomics is improved by having a standard, web-like API. By using streams
    composability with other web platforms is improved.
*   The web platform is unusual in not having native support for compression.
    Android, iOS, .NET, Java, Node.js, Python, and many other platforms come
    with compression support out-of-the-box. This API fills that gap.

### Why support "gzip" and "deflate" rather than more modern compression formats?

*   An implementation of these formats is already built into all
    standards-compliant browsers. This makes the incremental cost of exposing
    APIs to JavaScript tiny.
*   These formats are incredibly widely used throughout the web platform. This
    means the risk of browsers ever removing support for them is low. As a
    result, the maintenance burden from these algorithms is guaranteed to remain
    small.
*   Compression ratios achievable with these formats are still competitive for
    many use cases. Modern formats are often more CPU-efficient, but where the
    amount of data to be compressed is relatively small, this is not too
    important. Applications involving huge volumes of data will probably still
    want to use custom compression algorithms.
*   Because of the ubiquity of these formats, they are useful in creation of
    many file types, such as zip, pdf, svgz, and png.
*   The latest bleeding-edge compression formats are likely to be soon
    supplanted by new bleeding-edge compression formats. However, it is hard to
    remove something from the web platform once added. In the worst case,
    browser vendors could be left maintaining compression libraries that are no
    longer supported upstream. For this reason, it is best to ship only
    tried-and-tested algorithms.


## Use cases

*   Compressing data for upload.
*   Compression and decompression for
    *   Native files
    *   Network protocols
    *   In-memory databases
*   Lazy decompression for downloads.


## Example code

### Gzip-compress a stream

```javascript
const compressedReadableStream = inputReadableStream.pipeThrough(new CompressionStream('gzip'));
```

### Deflate-compress an ArrayBuffer to a Uint8Array

```javascript
async function compressArrayBuffer(in) {
  const cs = new CompressionStream('deflate');
  const writer = cs.writable.getWriter();
  writer.write(in);
  writer.close();
  const out = [];
  const reader = cs.readable.getReader();
  let totalSize = 0;
  while (true) {
    const { value, done } = await reader.read();
    if (done)
      break;
    out.push(value);
    totalSize += value.byteLength;
  }
  const concatenated = new Uint8Array(totalSize);
  let offset = 0;
  for (const array of out) {
    concatenated.set(array, offset);
    offset += array.byteLength;
  }
  return concatenated;
}
```

### Gzip-decompress a Blob to a Blob

This treats the input as a gzip file regardless of the mime-type. The output
Blob has an empty mime-type.

```javascript
function decompressBlob(blob) {
  const ds = new DecompressionStream('gzip');
  const decompressedStream = blob.stream().pipeThrough(ds);
  return new Response(decompressedStream).blob();
}
```


## End-user benefits

Using this API, web developers can compress data to be uploaded, saving
users time and bandwidth.

As an alternative to this API, it is possible for web developers to bundle
an implementation of a compression algorithm with their app. However, that
would have to be downloaded as part of the app, costing the user time and
bandwidth.


## Considered alternatives

*   Why not simply wrap the [zlib](https://www.zlib.net/) API?

    Not all platforms use zlib. Moreover, it is not a web-like API and
    it’s hard to use. Implementing `CompressionStream` for zlib helps us
    use it more easily.

*   Why not support synchronous compression?

    With a synchronous API, compressing a large buffer would block the main
    thread for potentially a significant amount of time, leading to jank and a
    bad user experience for users. An asynchronous API allows implementations
    to avoid this by chunking work and/or offloading it to another thread.

*   Why not a non-streaming API?

    Gzip backreferences can span more than one chunk. An API which
    only worked on one buffer at a time could not create
    backreferences between different chunks, and so could not be used
    to implement an efficient streaming API. However, a stream-based
    API can be used to compress a single buffer, so it is more
    flexible.


## Future work

There are a number of possible future expansions which may increase the
utility of the API:

* The "deflate-raw" ([RFC1951](https://www.ietf.org/rfc/rfc1951.txt)) format,
  which is similar to "deflate", but has no headers or footers.
* Other compression algorithms, including "brotli".
* Implementing new compression algorithms in JavaScript or WASM.
* Options for algorithms, such as setting the compression level.
* "Low-latency" mode, where compressed data is flushed at the end of each
  chunk. Currently data is always buffered across chunks. This means many
  small chunks may be passed in before any compressed data is produced.


## References & acknowledgements

Original text by Canon Mukai with contributions from Adam Rice, Domenic
Denicola, Takeshi Yoshino and Yutaka Hirano.
