# value-hash

This is the Vendekagon Labs fork of [valuehash]()
a library by [Luke Vanderhart](https://github.com/arachne-framework/valuehash)
written as part of the [Arachne framework](https://github.com/arachne-framework).

## Rationale

`value-hash` is a Clojure library that provides a way to provide higher-bit
hashes of arbitrary Clojure data structures, which respect Clojure's value
semantics. That is, if two objects are `clojure.core/=`, they will have the same
hash value. To my knowledge, no other Clojure data hashing libraries make this
guarantee.

The protocol is extensible to arbitrary data types and can work with any hash
function.

Although the library uses byte streams as an intermediate format, it does not
tag types, or perform any optimization or compaction of the byte stream.
Therefore it should _not_ be used as a serialization library. Use
[Fressian](https://github.com/clojure/data.fressian),
[Transit](https://github.com/cognitect/transit-clj) or something similar
instead.

## Usage

To get a MD5 hash of any Clojure object, do this:

```
(valuehash.api/md5 {:hello "world"})
=> #object["[B" 0x30cb9804 "[B@30cb9804"]
```

To get the hexadecimal string version, do this:

```
(valuehash.api/md5-str {:hello "world"})
=> "d3d7ccf8b8c217f3b52dc08929eabab8"
```

Also provided are `sha-1`, `sha-1-str`, `sha-256` and `sha-256-str`, which do
what they say on the tin.

### Custom hash functions

If you want a hash function other than md5, sha-1 or sha-256, you can obtain a
digest function for any algorithm supported by `java.security.MessageDigest` in
your JVM.

Obtain the digest function using the `messagedigest-fn` function, then pass it
and the object to be hashed to `digest`.

If you wish to obtain a hexadecimal string of the result, call the `hex-str`
function on the result.

```clojure
(h/hex-str (h/digest (h/messagedigest-fn "MD2") {:hello "World"}))
=> "81c9637d9fcb071a486eeb0c76dce1f6"
```

### Even more custom hash functions

If nothing in `java.security.MessageDigest` meets your needs, you can supply
your own digest function to `valuehash.api/digest`. This may be any function
which takes a `java.io.InputStream` and returns a byte array.

For example, the following example defines and uses a valid but terrible hash
function:

```clojure
(defn lazyhash [is]
  ;; chosen by a fair dice roll, guaranteed to be a random oracle
  (byte-array [(byte 4)]))

(h/digest lazyhash {:hello "world"})
```

## Semantics

This library does not combine hashes: it converts the entire input data to
binary data, and hashes that. As such, it is likely suitable for cryptographic
applications when used with an appropriate hash function.

_NOTE_: No explicit warranty or guarantee about this is provided, so if you want
to use the library for this purpose, I recommend you make an independent audit
of its behavior w/r/t cryptographic guarantees.

The binary data supplied to the hash function matches Clojure's equality
semantics. That is, objects that are semantically `clojure.core/=` will have the
same binary representation.

This means:

- All list types are encoded the same
- All set types are encoded the same
- All map types are encoded the same
- All integer numbers are encoded the same
- All floating-point numbers are encoded the same

The system does take some steps to rule out common types of "collisions", where
two unequal objects have the same binary representation (and therefore the same
hash). It injects "separator" bytes in collections, so that (for example) the
binary representation of `["ab" "c"]` is not equal to `["a" "bc"]`.

## Supported Types

By default, Clojure's native types are supported: as a rule of thumb, if it can
be printed to EDN by the default printer, it can be hashed with no fuss.

If you want to extend the system to hash arbitrary values, you can extend the
`valuehash.impl/CanonicalByteArray` protocol to any object of your choosing.

## Performance

On a 2020 M1 Macbook Air, this library determines the MD5 hash of small
(0 - 10 element vectors) at a rate of > 50,000 hashed objects per second.

Larger, more complex nested object slow down significantly, to a rate of around
4,500 per second for objects generated by
`(clojure.test.check.generators/sample-seq gen/any-printable 100)`

To run your own benchmarks, check out the `valuehash.bench` namespace in the
`test` directory.

## Contributing

The current implementation is somewhat naive, as it is single
threaded and performs redundant array copying. Performance oriented PRs are
welcome. Please see the `valuehash.impl` namespace and re-implement/replace
`CanonicalByteArray` with more performant type specific methods, then submit
a pull request with your alternative impl in a separate namespace, with
comparative benchmarks attached.

## License

Original library copyright © 2016-2018 Luke VanderHart under EPL 1.0 terms.

Current library version and work done since copyright © 2023 Vendekagon Labs.

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.

