# Introduction

## Environment setup

Depending on the language you are using, you will need to prepare your environment.
We will keep it simple and use only one file for now. If you are capable of splitting
the file into logical pieces, you are encouraged to do so. One file is just to
simplify the process for newbies.

### Rust

If you are using Rust, you will need to create a new project directory using cargo:

```bash
cargo new --bin bitcoin_parser
cd bitcoin_parser
```

You will need to add `byteorder` dependency. Open `Cargo.toml` inside the `bitcoin_parser`
directory and add `byteorder = "1"` into `dependencies` section. It should look like this:

```toml
[dependencies]
byteorder = "1"
```

This informs `cargo`, the *build system*, to fetch and manage the dependency.

Finally, open `main.rs` inside `bitcoin_parser/src` directory. This is your source file.
We will need to inform Rust *compiler* about the external library by writing
`extern crate byteorder` (libraries are called crates in Rust).

Because we will use functions from `byteorder` crate often, let's add `use` directive,
so we don't have to spell long name all the time. We will also use `std::io` often.
Add these lines to your source file:

```rust
use byteorder::{LE, ReadBytesExt};
use std::io;
use std::io::{Read, ErrorKind};
```

```rust
After all the modifications, the file should look like this:

extern crate byteorder;

use byteorder::{LE, ReadBytesExt};
use std::io;
use std::io::{Read, ErrorKind};

fn main() {
    println!("Hello, world!");
}
```


### C++

If you are using C++, create a new directory `bitcoin_parser` and then new file `main.cpp`
inside that directory. This will be your source file. We will use standard stream functions
when reading from files. We will also use integer types with exact lengths (as opposed to
old-school things like `int`, `long` ...) We also don't want to bother with writing `std::`
prefix. So let's add relevant includes and directives:

```
#include <istream>
#include <stdint.h>
#include <vector>

using namespace std;
```

Note that while C++ setup instructions may seem simpler, the downside is that we will have to
either implement reading integers by hand, or *manually* manage the library. For the sake of
simplicity, we will implement it ourselves. So we add these functions:

```c++
/// Reads uint16_t from stream using little endian byte order.
///
/// The caller is responsible for checking if the operation
/// succeeded by calling `.fail()` method on the stream.
uint16_t read_u16_le(istream &stream) {
    char buf[2];
    stream.read(buf, 2);
    return (uint16_t)buf[0] | ((uint16_t)buf[1]) << 8;
}   

/// Reads uint32_t from stream using little endian byte order.
///
/// The caller is responsible for checking if the operation
/// succeeded by calling `.fail()` method on the stream.
uint32_t read_u32_le(istream &stream) {
    char buf[4];
    stream.read(buf, 4);
    return (uint32_t)buf[0]       | (uint32_t)buf[1] << 8  |
           (uint32_t)buf[2] << 16 | (uint32_t)buf[3] << 24;
}

/// Reads uint64_t from stream using little endian byte order.
///
/// The caller is responsible for checking if the operation
/// succeeded by calling `.fail()` method on the stream.
uint64_t read_u64_le(istream &stream) {
    char buf[8];
    stream.read(buf, 8);
    return (uint64_t)buf[0]       | (uint64_t)buf[1] << 8  |
           (uint64_t)buf[2] << 16 | (uint64_t)buf[3] << 24 |
           (uint64_t)buf[4] << 32 | (uint64_t)buf[5] << 40 |
           (uint64_t)buf[6] << 48 | (uint64_t)buf[7] << 56;
}
```

Now let's check if our code compiles. In case of Rust, the command is `cargo check`. In case of
C++ it's `g++ -W -Wall -c -o main.o main.c` or `clang++ -W -Wall -c -o main.o main.c` (depending
on which compiler you have). If you want to make the command simpler, you can create a file called
`Makefile` and then use `make` to compile

```make
main.o: main.cpp
	$(CXX) -W -Wall -c -o $@ $<

clean:
	rm -f *.o

.PHONY: clean
```

We can ignore warnings about unused stuff for now. If no error occured, everything is good and we
may proceed to the next chapter.

## Bitcoin basics

The Bitcoin blockchain consists of blocks. Those blocks consist of header and
transactions. Transactions consist of inputs, outputs and meatadata.

Note that while the transactions in block are hashed as merkle tree, they are
stored (serialized) as array! Since we are writing parser, not verifier, we
don't care about merkle tree and will simply process the transactions as a
sequence.

We will parse the components one-by-one, but first let's define some primitives.

## Primitives

Some numbers are encoded as "compact size". As the name "size" suggests, they
are all positive numbers. This is how it works:

* If the number is less than or equal to 252, it's encoded directly in one byte.
* If the number is less than or equal to 65535, there's one byte with value 253
  followed by two bytes encoding the number as little endian.
* If the number is less than or equal to 4294967295, there's one byte with value
  254 followed by four bytes encoding the number as little endian.
* If the number is less than or equal to 18446744073709551615, there's one byte
  with value 255 followed by eight bytes encoding the number as little endian.
* Values greater than 18446744073709551615 can not be encoded.

Let's define a deserialization function:

### Rust

```rust
fn deserialize_varint<R: Read>(reader: &mut R) -> io::Result<u64> {
    match reader.read_u8()? {
        253 => reader.read_u16::<LE>().map(Into::into),
        254 => reader.read_u32::<LE>().map(Into::into),
        255 => reader.read_u64::<LE>().map(Into::into),
        x   => Ok(x.into()),
    }
}
```

### C++

```cpp
/// Deserializes "varint" as defined by Bitcoin protocol.  
///                                                        
/// The caller is responsible for checking if the operation
/// succeeded by calling `.fail()` method on the stream.   
uint64_t parse_varint(istream &stream) {
    char c;
    stream.get(c);
    if(stream.fail()) return 0;

    switch((unsigned char)c) {
        case 253:
		return (uint64_t)read_u16_le(stream);
        case 254:
		return (uint64_t)read_u32_le(stream);
        case 255:
		return read_u64_le(stream);
        default:
            return (uint64_t)c;

    }   
}
```

Another primitive is Bitcoin script. It's a sequence of bytes encoding various
operations. It's used to define conditions under which transaction is spendable
or to provide data for verification (e.g. digital signature).

The script can have variable length and is always preceded with it's length
encoded as compact size. We will create deserialization function as a
constructor of the script struct/class.

## Rust

```rust
/// Represent's Bitcoin script.
struct Script(Vec<u8>);

impl Script {
    /// Deserializes the script from a reader.
    fn deserialize<R: Read>(reader: &mut R) -> io::Result<Self> {
        let len = deserialize_varint(reader)?;
        // This is consensus rule, so theoretically no need to check it,
        // but serves as a protection against corrupted inputs.
	if len > 10_000 {
            return Err(ErrorKind::InvalidData.into());
        }

        let mut reader = reader.by_ref().take(len);
        let mut data = Vec::with_capacity(len as usize);

	io::copy(&mut reader, &mut data)?;
        Ok(Script(data))
    }
}
```

## C++

```cpp
/// Represents bitcoin script.
class Script {
        public:
                /// Constructs sript by deserializing from stream.
                Script(istream &stream) : data(), valid(false) {
                        uint64_t len = deserialize_varint(stream);
                        if(stream.fail() || len > 10000) {
                                return;
                        }

                        data.resize((size_t)len);
                        stream.read(&data[0], len);
                        valid = !stream.fail();
                }

                /// Returns true if deserialization succeeded.
                bool is_valid() {
                        return valid;
                }
        private:
                vector<char> data;
                bool valid;
};
```

### 256-bit hash

Finally, we will create a special type for 256-bit (32 byte) hashes.
In Bitcoin, all 256bit hashes are 1-2 iterations of SHA256. (Block
headers are two rounds of SHA256)

The deserialization is simple - we'll just read the bytes.

## Rust

```rust
/// Represents 256 bit hash. (SHA256)
struct Hash256([u8; 32]);

impl Hash256 {
    /// Deserializes the hash
    fn deserialize<R: Read>(reader: &mut R) -> io::Result<Self> {
        let mut buf = [0; 32];
        reader.read_exact(&mut buf)?;

        Ok(Hash256(buf))
    }
}
```

## C++

```cpp
/// Represents 256 bit hash.
class Hash256 {
        public:
                /// Reads the hash from stream.
                ///
                /// You must check `stream.fail()` to be sure deserialization succeeded.
                Hash256(istream &stream) {
                        stream.read(data, 32);
                }
        private:
                char data[32];
};
```

Now that we have all the primitives, let's deserialize transactions!
