---
title: "Fixed/Variable-length encoding in Go"
description: "It covers how to use the standard encoding/binary package to encode binary according to a custom format, and how it works"
date: 2021-06-18
tags: ["Golang", "Algorithm"]
draft: false
---

This post covers how to use the standard [encoding/binary](https://golang.org/pkg/encoding/binary/) package to encode binary according to a custom format, and how it works.

## Preface
When writing programmatic data structures to a network or file, they need to be encoded into a self-contained sequence of bytes according to some kind of format.
When exchanging data between processes, we typically use a standardized, language-independent format.

Standardized formats tend to be too verbose. Protocol Buffer and Apache Thrift can be much smaller than text formats because of binary encodings, but they require a dedicated schema to be defined.
In some cases, such as when only one process accesses the file, you may want to encode it in a format specific to the Go language.

Go defines its own format called Gob for such cases, and the [encoding/gob](https://golang.org/pkg/encoding/gob/) package provides APIs for it. However, this specification requires to embed field names into the data in text format, which is often redundant if you only want to encode a few fixed-length values.

Go also offers a package for binary encoding called [encoding/binary](https://golang.org/pkg/encoding/binary/), which is recommended if you want to encode a small number of values of fixed-length numeric types.

This article walk you through how to use it to encode/decode custom binary formats.

## Target struct
The following struct will be encoded in various ways.

```go
type Data struct {
	ID        uint32
	Timestamp uint64
	Value     int16
}
```

Note that all fields are of fixed-length numeric type.

## Fixed-length encoding
The ID of the above struct is uint32 (4 bytes), Timestamp is uint64 (8 bytes), and Value is int16 (2 bytes).
Therefore, the total size of this structure is 14 bytes. Let's build a 10-byte binary in the following format simply.

```
0   1   2   3   4   5   6   7   8   9   10  11  12  13  14 (bytes)
+---------------+-------------------------------+-------+
|       ID      |           Timestamp           | Value |
+-------+-------+-------------------------------+-------+
```

### binary.ByteOrder
First up, I will show you how to use the [binary.ByteOrder](https://golang.org/pkg/encoding/binary/#ByteOrder) interface to directly encode each value manually.

Select the endianness and fill the initialized buffer in order. Note that if the buffer size is too small, it will panic.
This is because everything is of a fixed length type, so the size of any number can be predicted in advance. In this example, we chose BigEndian as the endian.

```go
data := &Data{
	ID: 100, Timestamp: 1600000000, Value: 10,
}
buf := make([]byte, 14)
binary.BigEndian.PutUint32(buf[0:], data.ID)
binary.BigEndian.PutUint64(buf[4:], data.Timestamp)
binary.BigEndian.PutUint16(buf[12:], uint16(data.Value))
fmt.Printf("encoded binary length: %d\n", len(buf))
// encoded binary length: 14
```

Decoding functions are provided for each number of bits, and we will use them to decode the data in order.

```go
decodedData := &Data{}
decodedData.ID = binary.BigEndian.Uint32(buf[:4])
decodedData.Timestamp = binary.BigEndian.Uint64(buf[4:12])
decodedData.Value = int16(binary.BigEndian.Uint16(buf[12:]))
fmt.Printf("ID: %d, Timestamp: %d, Value: %d\n", decodedData.ID, decodedData.Timestamp, decodedData.Value)
// ID: 100, Timestamp: 1600000000, Value: 10
```

If you specify the different endianness as when encoding, you will obviously end up reading the bits from the opposite direction, resulting in a completely different result.

However, this method is a bit tedious. Aren't there an API that can decode directly into a struct?

There are. The encoding/binary package provides two built-in functions, [binary.Write](https://golang.org/pkg/encoding/binary/#Write) and [binary.Read](https://golang.org/pkg/encoding/binary/#Read), to read and write structures and slices to streams.

### binary.Write/Read
They will fill the fields in the order of their definition if the struct is one where all fields are of fixed length. The same is true for slices.

Encoding can be done by passing a struct as follows, which will be written to streams by `io.Writer`. Internally, the fields will be packed in order from the top field based on the endianness specified as the second argument.

```go
data := &Data{
	ID: 100, Timestamp: 1600000000, Value: 10,
}
buf := &bytes.Buffer{}
_ = binary.Write(buf, binary.BigEndian, data)
fmt.Printf("encoded binary length: %d\n", buf.Len())
// encoded binary length: 14
```

Read, it is possible to decode into the structure passed as the third argument. Again, of course, you must specify the same endianness as when encoding, otherwise the result will be invalid.

```go
dst := &Data{}
_ = binary.Read(buf, binary.BigEndian, dst)
fmt.Printf("ID: %d, Timestamp: %d, Value: %d\n", dst.ID, dst.Timestamp, dst.Value)
// ID: 100, Timestamp: 1600000000, Value: 10
```

Here is the entire source code. You can see how simple binary encoding is.

```go
package main

import (
	"bytes"
	"encoding/binary"
	"fmt"
)

type Data struct {
	ID        uint32
	Timestamp uint64
	Value     int16
}

func main() {
	data := &Data{
		ID: 100, Timestamp: 1600000000, Value: 10,
	}

	buf := &bytes.Buffer{}
	_ = binary.Write(buf, binary.LittleEndian, data)

	dst := &Data{}
	_ = binary.Read(buf, binary.LittleEndian, dst)
}
```

However, accepting any type arbitrary means that reflection is happening internally. This can be a bottleneck if you have strict performance requirements.

I benchmarked the performance of ByteOrder on my laptop (`Intel(R) Core(TM) i7-8559U CPU @ 2.70GHz with 16GB of RAM on macOS 10.15.7`) to see how much of a performance difference there is compared to directly filling bytes with ByteOrder.
The actual test code used was [here](https://gist.github.com/nakabonne/1ee4ce790fdb02a869afa58e6e7d8517).

```
$ go test -benchmem -bench=. .
goos: darwin
goarch: amd64
pkg: hoge
cpu: Intel(R) Core(TM) i7-8559U CPU @ 2.70GHz
BenchmarkEncodeWithByteOrder-8          1000000000               1.145 ns/op           0 B/op          0 allocs/op
BenchmarkEncodeWithStream-8              6005226               203.1 ns/op           144 B/op          4 allocs/op
PASS
ok      hoge    2.944s
```

I found that the time calculation when writing to streams with `binary.Write` is about 200 times slower than when using `ByteOrder` directly.

It requires some effort to specify the buffer size explicitly, but if you are looking for efficiency in both time and space computation, it is better to use ByteOrder to fill the buffer manually.

### Variable-length encoding
The fixed-length encoding introduced above allocates the buffer size in advance, making it simple to encode, but it has the problem that the size can be larger than necessary.
For example, int32, a signed 32-bit integer value type, always requires 32 bits of space to allow integer values from -2147483648 to 2147483647.

However, in most cases, the actual value should be smaller than this. In the case of small values, some trailing bits will be left blank, so we will end up embedding a large number of useless trailing zeros to make the size 32 bits.
To prevent this, you can use variable length encoding.

Like for instance, the integer 10 is 1010 in binary, which can be represented in 4 bits. However, if it is represented as a 64-bit unsigned integer value type, fixed-length encoding will force the end to be filled with zeros and consume 8 bytes.

```bash
# Original bits
1010
# with 60 zero-padded bytes.
1010000000000000000000000000000000000000000000000000000000000000
```

If we encode 10 using `PutUint64` we used right before, we can see that it consumes 8 bytes.

```go
buf := make([]byte, 8)
binary.BigEndian.PutUint64(buf, 10)
fmt.Printf("encoded binary length: %d\n", len(buf))
// encoded binary length: 8

// It should be one byte...
```

`PutUvarint` can be used to reduce the result size to 1 byte, as shown below.

```go
buf := make([]byte, binary.MaxVarintLen64)
var x uint64 = 10
n := binary.PutUvarint(buf, x)
fmt.Printf("encoded binary length: %d\n", len(buf[:n]))
// encoded binary length: 1
```

Since it is variable length, the larger the number, the larger the byte size. For instance, an integer of 256 is 100000000 in binary, which is 9 bits in size. Therefore, the encoded data will be 2 bytes.

```go
buf := make([]byte, binary.MaxVarintLen64)
var x uint64 = 128
n := binary.PutUvarint(buf, x)
fmt.Printf("encoded binary length: %d\n", len(buf[:n]))
// encoded binary length: 2
```

You can use `binary.Varint` to decode.

```go
i, _ := binary.Varint(buf[:n])
```

### Encode the struct in this way
Now, let's use the Varint functions to encode the Data structure that we used in the previous section.

```go
data := &Data{
	ID: 100, Timestamp: 1600000000, Value: 10,
}
```

The ID is 100 and the Value is 10, so each of these can be represented by 1 byte; Timestamp is a bit larger, at 1600000000, and requires 5 bytes.
In total, the value of the `Data` struct can be represented in 7 bytes. The fixed length encoding consumes 14 bytes, so we can save 7 bytes.

```
0   1   2   3   4   5   6   7 (bytes)
+---+-------------------+---+
|ID |      Timestamp    |Val|
+---+-------------------+---+
```

(6/19 updates: fixed not to embed the sizes into the binary. Thank you [@tgulacsi](https://www.reddit.com/r/golang/comments/o2qep2/fixedvariablelength_encoding_in_go/h2aqze2?utm_source=share&utm_medium=web2x&context=3)!)

You can implement as:

```go
data := &Data{
	ID: 100, Timestamp: 1600000000, Value: 10,
}
buf := make([]byte, 7)

n := binary.PutUvarint(buf, uint64(data.ID))
n += binary.PutUvarint(buf[n:], data.Timestamp)
n += binary.PutVarint(buf[n:], int64(data.Value))

fmt.Printf("encoded binary length: %d\n", len(buf[:n]))
// encoded binary length: 7
```

You can see that it has been safely reduced to 7 bytes. Now let's decode it.

```go
id, idLen := binary.Uvarint(buf)
ts, tsLen := binary.Uvarint(buf[idLen:])
value, _ := binary.Varint(buf[idLen+tsLen:])

decodedData := &Data{
	ID:        uint32(id),
	Timestamp: uint64(ts),
	Value:     int16(value),
}
fmt.Printf("ID: %d, Timestamp: %d, Value: %d\n", decodedData.ID, decodedData.Timestamp, decodedData.Value)
// ID: 100, Timestamp: 1600000000, Value: 10
```

It was able to decode it back to the original. Now you can make a recoverable binary with a smaller size than the fixed-length encoding.

But doesn't the call to `binary.Uvarint` in the first line look odd to you?
Even though it passes the encoded bytes slice as it is, it terminates reading at the appropriate position and returns the decoded `id`.
How does it determine the end bit without specifying the byte size?

This uses an encoding technique called [Varints (Variable Integers)](https://developers.google.com/protocol-buffers/docs/encoding#varints), which is used internally by Protocol Buffer.

## How Varints works
Decoding fixed length bytes could be accomplished by simply reading the required number of bytes without extra work. In order to allow variable length numbers, we need to indicate the existence of subsequent bytes in some way.

To do this, we first divide each byte into two parts: 1-bit and 7-bit.

- First 1 bit: Indicates the existence of subsequent bytes
- Remaining 7 bits: actual data

The first bit is called the most significant bit (msb) because it is the most significant bit.
What msb is 1 means that there are more bytes to be read; 0 means that the byte is the last and the read operation should terminated.

For example, 1 in binary is 1, which can be adequately represented in one byte. Therefore, the msb of the first byte will be 0, and the encoding result will be as follows:

```
00000001
↑
msb is 0
```

Let's consider the case of 300, which in binary is 100101100, which does not fit into 7 bits. Therefore, the encoded result will span two bytes, as shown below.

```
10101100 00000010
↑        ↑
msb is 1 msb is 0 so it quits
```

Since the actual data is 7 bits, which is except for msb, remove them first.

```
10101100 00000010

 0101100  0000010
```

In Varints, the least significant 7-bit group is stored first, so we need to reverse these at the end. When you do so, you will find that you have a binary notation of 300.

```
0000010 0101100
-> 100101100
-> 300
```

## Conclusion
It covered various binary encoding methods using the encoding/binary package. Besides it showed how to do fixed length encoding manually, how to do it to streams, and finally explained variable length encoding.

By defining the own protocol and implementing the simplest pattern, we can be much closer to the existing standardized binary protocols.

The last one, Varints, is a concept that underlies Protocol Buffer's Wire Format, so understanding it will help you understand modern standardized protocols. I hope this article has been of some help.

## References

- https://golang.org/pkg/encoding/binary
- https://developers.google.com/protocol-buffers/docs/encoding
- https://golang.org/src/encoding/binary/varint.go
- https://blog.golang.org/gob

