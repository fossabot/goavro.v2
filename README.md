# goavro

Goavro is a library that encodes and decodes Avro data.

## Description

* Encodes to and decodes from both binary and textual JSON Avro data.
* `Codec` is stateless and is safe to use by multiple goroutines.

With the exception of features not yet supported, goavro attempts to
be fully compliant with the most recent version of the [Avro
specification](http://avro.apache.org/docs/1.8.2/spec.html).

## Use with Import Statement

[The proposal to add package version support to the Go
toolchain](https://github.com/golang/go/issues/24301) requires library
authors to leave V1 code in the top level directory of the repository,
and create a `v2` directory for V2 of the library.

However this library was tagged with a `v2` release prior to that
proposal, and V1 of this library is no longer at the top level of the
repository. For a while this conflict prevented `go build` and `vgo
build` from both being able to build a program that requires this
library.

Because of a [update to
`vgo`](https://github.com/golang/go/issues/24099) that provides
enhanced support for gopkg.in, now building projects that use either
version agnostic *or* version aware Go build tools will work, provided
this library is imported using its gopkg.in path.

### To use V2 of this library:

```Go
import goavro "gopkg.in/linkedin/goavro.v2"
```

### To use V1 of this library:

```Go
import goavro "gopkg.in/linkedin/goavro.v1"
```

## NOTICE

This goavro library has been rewritten to correct a large number of
shortcomings:

* https://github.com/linkedin/goavro/issues/8
* https://github.com/linkedin/goavro/issues/36
* https://github.com/linkedin/goavro/issues/45
* https://github.com/linkedin/goavro/issues/55
* https://github.com/linkedin/goavro/issues/71
* https://github.com/linkedin/goavro/issues/72
* https://github.com/linkedin/goavro/issues/81

As a consequence of the rewrite, the API has been significantly
simplified, taking into account suggestions from users received during
the past few years since its original release.

### Justification for API Change

It was a very difficult decision to break the API when creating the
new version, but in the end the benefits outweighed the consequences:

1. Allowed proper handling of Avro namespaces.
1. Eliminated largest gripe of users: getting data into and out of
   records.
1. Provided significant, 3x--4x speed improvement for all tasks.
1. Allowed textual encoding to and decoding from Avro JSON.
1. Better handling of record field default values.

#### Avro namespaces

The original version of this library was written prior to my really
understanding how Avro namespaces ought to work. After using Avro for
a long time now, and after a lot of research, I think I grok Avro
namespaces properly, and the library now correctly handles every test
case the Apache Avro distribution has for namespaces, including being
able to refer to a previously defined data type later on in the same
schema.

#### Getting Data into and out of Records

The original version of this library required creating `goavro.Record`
instances, and use of getters and setters to access a record's
fields. When schemas were complex, this required a lot of work to
debug and get right. The original version also required users to break
schemas in chunks, and have a different schema for each record
type. This was cumbersome, annoying, and error prone.

The new version of this library eliminates the `goavro.Record` type,
and accepts a native Go map for all records to be encoded. Keys are
the field names, and values are the field values. Nothing could be
more easy. Conversely, decoding Avro data yields a native Go map for
the upstream client to pull data back out of.

Furthermore, there is never a reason to ever have to break your schema
down into record schemas. Merely feed the entire schema into the
`NewCodec` function once when you create the `Codec`, then use
it. This library knows how to parse the data provided to it and ensure
data values for records and their fields are properly encoded and
decoded.

#### 3x--4x Performance Improvement

The original version of this library was truly written with Go's idea
of `io.Reader` and `io.Writer` composition in mind. Although
composition is a powerful tool, the original library had to pull bytes
off the `io.Reader`--often one byte at a time--check for read errors,
decode the bytes, and repeat. This version, by using a native Go byte
slice, both decoding and encoding complex Avro data here at LinkedIn
is between three and four times faster than before.

#### Avro JSON Support

The original version of this library did not support JSON encoding or
decoding, because it wasn't deemed useful for our internal use at the
time. When writing the new version of the library I decided to tackle
this issue once and for all, because so many engineers needed this
functionality for their work.

#### Better Handling of Record Field Default Values

The original version of this library did not well handle default
values for record fields. This version of the library uses a default
value of a record field when encoding from native Go data to Avro data
and the record field is not specified. Additionally, when decoding
from Avro JSON data to native Go data, and a field is not specified,
the default value will be used to populate the field.

## Contrast With Code Generation Tools

If you have the ability to rebuild and redeploy your software whenever
data schemas change, code generation tools might be the best solution
for your application.

There are numerous excellent tools for generating source code to
translate data between native and Avro binary or textual data. One
such tool is linked below. If a particular application is designed to
work with a rarely changing schema, programs that use code generated
functions can potentially be more performant than a program that uses
goavro to create a `Codec` dynamically at run time.

* [gogen-avro](https://github.com/alanctgardner/gogen-avro)

I recommend benchmarking the resultant programs using typical data
using both the code generated functions and using goavro to see which
performs better. Not all code generated functions will out perform
goavro for all data corpuses.

If you don't have the ability to rebuild and redeploy software updates
whenever a data schema change occurs, goavro could be a great fit for
your needs. With goavro, your program can be given a new schema while
running, compile it into a `Codec` on the fly, and immediately start
encoding or decoding data using that `Codec`. Because Avro encoding
specifies that encoded data always be accompanied by a schema this is
not usually a problem. If the schema change is backwards compatible,
and the portion of your program that handles the decoded data is still
able to reference the decoded fields, there is nothing that needs to
be done when the schema change is detected by your program when using
goavro `Codec` instances to encode or decode data.

## Resources

* [Avro CLI Examples](https://github.com/miguno/avro-cli-examples)
* [Avro](https://avro.apache.org/)
* [Google Snappy](https://google.github.io/snappy/)
* [JavaScript Object Notation, JSON](https://www.json.org/)
* [Kafka](https://kafka.apache.org)

## Usage

Documentation is available via
[![GoDoc](https://godoc.org/github.com/linkedin/goavro?status.svg)](https://godoc.org/github.com/linkedin/goavro)[![FOSSA Status](https://app.fossa.io/api/projects/git%2Bgithub.com%2FMediaMath%2Fgoavro.v2.svg?type=shield)](https://app.fossa.io/projects/git%2Bgithub.com%2FMediaMath%2Fgoavro.v2?ref=badge_shield)
.

```Go
package main

import (
    "fmt"

    "github.com/linkedin/goavro"
)

func main() {
    codec, err := goavro.NewCodec(`
        {
          "type": "record",
          "name": "LongList",
          "fields" : [
            {"name": "next", "type": ["null", "LongList"], "default": null}
          ]
        }`)
    if err != nil {
        fmt.Println(err)
    }

    // NOTE: May omit fields when using default value
    textual := []byte(`{"next":{"LongList":{}}}`)

    // Convert textual Avro data (in Avro JSON format) to native Go form
    native, _, err := codec.NativeFromTextual(textual)
    if err != nil {
        fmt.Println(err)
    }

    // Convert native Go form to binary Avro data
    binary, err := codec.BinaryFromNative(nil, native)
    if err != nil {
        fmt.Println(err)
    }

    // Convert binary Avro data back to native Go form
    native, _, err = codec.NativeFromBinary(binary)
    if err != nil {
        fmt.Println(err)
    }

    // Convert native Go form to textual Avro data
    textual, err = codec.TextualFromNative(nil, native)
    if err != nil {
        fmt.Println(err)
    }

    // NOTE: Textual encoding will show all fields, even those with values that
    // match their default values
    fmt.Println(string(textual))
    // Output: {"next":{"LongList":{"next":null}}}
}
```

Also please see the example programs in the `examples` directory for
reference.

### ab2t

The `ab2t` program is similar to the reference standard
`avrocat` program and converts Avro OCF files to Avro JSON
encoding.

### arw

The Avro-ReWrite program, `arw`, can be used to rewrite an
Avro OCF file while optionally changing the block counts, the
compression algorithm. `arw` can also upgrade the schema provided the
existing datum values can be encoded with the newly provided schema.

### avroheader

The Avro Header program, `avroheader`, can be used to print various
header information from an OCF file.

### splice

The `splice` program can be used to splice together an OCF file from
an Avro schema file and a raw Avro binary data file.

### Translating Data

A `Codec` provides four methods for translating between a byte slice
of either binary or textual Avro data and native Go data.

The following methods convert data between native Go data and byte
slices of the binary Avro representation:

    BinaryFromNative
    NativeFromBinary

The following methods convert data between native Go data and byte
slices of the textual Avro representation:

    NativeFromTextual
    TextualFromNative

Each `Codec` also exposes the `Schema` method to return a simplified
version of the JSON schema string used to create the `Codec`.

#### Translating From Avro to Go Data

Goavro does not use Go's structure tags to translate data between
native Go types and Avro encoded data.

When translating from either binary or textual Avro to native Go data,
goavro returns primitive Go data values for corresponding Avro data
values. The table below shows how goavro translates Avro types to Go
types.

| Avro               | Go                       |
| ------------------ | ------------------------ |
| `null`             | `nil`                    |
| `boolean`          | `bool`                   |
| `bytes`            | `[]byte`                 |
| `float`            | `float32`                |
| `double`           | `float64`                |
| `long`             | `int64`                  |
| `int`              | `int32`                  |
| `string`           | `string`                 |
| `array`            | `[]interface{}`          |
| `enum`             | `string`                 |
| `fixed`            | `[]byte`                 |
| `map` and `record` | `map[string]interface{}` |
| `union`            | *see below*              |

Because of encoding rules for Avro unions, when an union's value is
`null`, a simple Go `nil` is returned. However when an union's value
is non-`nil`, a Go `map[string]interface{}` with a single key is
returned for the union. The map's single key is the Avro type name and
its value is the datum's value.

#### Translating From Go to Avro Data

Goavro does not use Go's structure tags to translate data between
native Go types and Avro encoded data.

When translating from native Go to either binary or textual Avro data,
goavro generally requires the same native Go data types as the decoder
would provide, with some exceptions for programmer convenience. Goavro
will accept any numerical data type provided there is no precision
lost when encoding the value. For instance, providing `float64(3.0)`
to an encoder expecting an Avro `int` would succeed, while sending
`float64(3.5)` to the same encoder would return an error.

When providing a slice of items for an encoder, the encoder will
accept either `[]interface{}`, or any slice of the required type. For
instance, when the Avro schema specifies:
`{"type":"array","items":"string"}`, the encoder will accept either
`[]interface{}`, or `[]string`. If given `[]int`, the encoder will
return an error when it attempts to encode the first non-string array
value using the string encoder.

When providing a value for an Avro union, the encoder will accept
`nil` for a `null` value. If the value is non-`nil`, it must be a
`map[string]interface{}` with a single key-value pair, where the key
is the Avro type name and the value is the datum's value. As a
convenience, the `Union` function wraps any datum value in a map as
specified above.

```Go
func ExampleUnion() {
    codec, err := goavro.NewCodec(`["null","string","int"]`)
    if err != nil {
        fmt.Println(err)
    }
    buf, err := codec.TextFromNative(nil, goavro.Union("string", "some string"))
    if err != nil {
        fmt.Println(err)
    }
    fmt.Println(string(buf))
    // Output: {"string":"some string"}
}
```

## Limitations

Goavro is a fully featured encoder and decoder of binary and textual
JSON Avro data. It fully supports recursive data structures, unions,
and namespacing. It does have a few limitations that have yet to be
implemented.

### Aliases

The Avro specification allows an implementation to optionally map a
writer's schema to a reader's schema using aliases. Although goavro
can compile schemas with aliases, it does not yet implement this
feature.

### Kafka Streams

[Kafka](http://kafka.apache.org) is the reason goavro was
written. Similar to Avro Object Container Files being a layer of
abstraction above Avro Data Serialization format, Kafka's use of Avro
is a layer of abstraction that also sits above Avro Data Serialization
format, but has its own schema. Like Avro Object Container Files, this
has been implemented but removed until the API can be improved.

### Default Maximum Block Counts, and Block Sizes

When decoding arrays, maps, and OCF files, the Avro specification
states that the binary includes block counts and block sizes that
specify how many items are in the next block, and how many bytes are
in the next block. To prevent possible denial-of-service attacks on
clients that use this library caused by attempting to decode
maliciously crafted data, decoded block counts and sizes are compared
against public library variables MaxBlockCount and MaxBlockSize. When
the decoded values exceed these values, the decoder returns an error.

Because not every upstream client is the same, we've chosen some sane
defaults for these values, but left them as mutable variables, so that
clients are able to override if deemed necessary for their
purposes. Their initial default values are (`math.MaxInt32` or
~2.2GB).

## License


[![FOSSA Status](https://app.fossa.io/api/projects/git%2Bgithub.com%2FMediaMath%2Fgoavro.v2.svg?type=large)](https://app.fossa.io/projects/git%2Bgithub.com%2FMediaMath%2Fgoavro.v2?ref=badge_large)

### Goavro license

Copyright 2017 LinkedIn Corp. Licensed under the Apache License,
Version 2.0 (the "License"); you may not use this file except in
compliance with the License. You may obtain a copy of the License at
http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
implied.

### Google Snappy license

Copyright (c) 2011 The Snappy-Go Authors. All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are
met:

   * Redistributions of source code must retain the above copyright
notice, this list of conditions and the following disclaimer.
   * Redistributions in binary form must reproduce the above
copyright notice, this list of conditions and the following disclaimer
in the documentation and/or other materials provided with the
distribution.
   * Neither the name of Google Inc. nor the names of its
contributors may be used to endorse or promote products derived from
this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

## Third Party Dependencies

### Google Snappy

Goavro links with [Google Snappy](http://google.github.io/snappy/)
to provide Snappy compression and decompression support.