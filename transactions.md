# Parsing transactions

As we saw earlier, the transaction consists of inputs, outputs and metadata.
Each input consits of outpoint and signing script. Each output consists of
verification script and amount.

Let's build the `Transaction` type piece by piece. First, we define `Outpoint`,
which is very straightforward:

## Outpoint

### Rust

```rust
/// Defines "outpoint" - output of previous transaction being consumed.
struct Outpoint {
    /// ID of previous transaction
    txid: Hash256,
    /// Which output of the previous transaction is being consumed.
    index: u32,
}

impl Outpoint {
    /// Deserializes the outpoint from the blockchain data
    fn deserialize<R: Read>(reader: &mut R) -> io::Result<Self> {
        let txid = Hash256::deserialize(reader)?;
        let index = reader.read_u32::<LE>()?;
    
        Ok(Outpoint {
            txid,
            index,
        })
    }
}
```

### C++

```cpp
/// Defines "outpoint" - output of previous transaction being consumed.
class Outpoint {
        public:
                /// Constructs the outpoint from the blockchain data
		Outpoint(istream &stream) : txid(stream), index(read_u32_le(stream)) { }

                /// ID of previous transaction
                Hash256 txid;
                /// Which output of the previous transaction is being consumed.
                uint32_t index;
};
```

## Input

Then we'll need to define input. Input is a combination of outpoint, signing
script, which proves ownership and intent to spend, and a sequence number which
is used to signal replace-by-fee feature. This is also very straightforward.

### Rust

```rust
/// Contains data about single transaction input.
struct TxInput {
    outpoint: Outpoint,
    sig_script: Script,
    sequence: u32,
}
    
impl TxInput {
    /// Deserializes the input from the blockchain data
    fn deserialize<R: Read>(reader: &mut R) -> io::Result<Self> {
        let outpoint = Outpoint::deserialize(reader)?;
        let sig_script = Script::deserialize(reader)?;
        let sequence = reader.read_u32::<LE>()?;

        Ok(TxInput {
            outpoint,
            sig_script,
            sequence,
        })
    }
}
```

### C++

```cpp
// Represents one input of a transaction.
class TxInput {
        public:
                TxInput(istream &stream) : outpoint(stream), sig_script(stream), sequence(read_u32_le(stream)) {}

                Outpoint outpoint;
                Script sig_script;
                uint32_t sequence;
};
```

## Output

Now let's define an output. Output contains only amount of satoshis being sent
and a verification script. Still very simple.

## Rust

```rust
/// Contains data about single transaction output.
struct TxOutput {
    satoshis: u64,
    verify_script: Script,
}

impl TxOutput {
    /// Deserializes the input from the blockchain data
    fn deserialize<R: Read>(reader: &mut R) -> io::Result<Self> {
        let satoshis = reader.read_u64::<LE>()?;
        let verify_script = Script::deserialize(reader)?;

        Ok(TxOutput {
            satoshis,
            verify_script
        })
    }
}
```

### C++

```cpp
// Represents one output of a transaction.
class TxOutput {
        public:
                TxOutput(istream &stream) : satoshis(read_u64_le(stream)), verify_script(stream) {}

                uint64_t satoshis;
                Script verify_script;
};
```

Now that we have all necessary components, let's deserialize transaction!

## Transaction

Every transaction begins with four byte, little endian version number, followed by count of inputs encoded as compact size, followed by all inputs concatenated together. Similarly, there's output count as compact size following and concatenated outputs. Finally, there's lock time encoded as four byte little endian integer.

```rust
/// Contains data about single transaction
struct Transaction {
    version: u32,
    inputs: Vec<TxInput>,
    outputs: Vec<TxOutput>,
    lock_time: u32,
}   
        
impl Transaction {
    fn deserialize<R: Read>(reader: &mut R) -> io::Result<Self> {
        let version = reader.read_u32::<LE>()?;
        let input_count = deserialize_varint(reader)?;

        // Sanity check. Since block can contain only 1M of bytes and each input
        // has more than one byte, this can't happen for valid transaction.
        if input_count > 1_000_000 {
            return Err(ErrorKind::InvalidData.into());
        }
        let mut inputs = Vec::with_capacity(input_count as usize);
        for _ in 0..input_count {
            inputs.push(TxInput::deserialize(reader)?);
        }

        let output_count = deserialize_varint(reader)?;
        // Sanity check. Since block can contain only 1M of bytes and each input
        // has more than one byte, this can't happen for valid transaction.
        if output_count > 1_000_000 {
            return Err(ErrorKind::InvalidData.into());
        }
        let mut outputs = Vec::with_capacity(output_count as usize);
        for _ in 0..output_count {
            outputs.push(TxOutput::deserialize(reader)?);
        }
        let lock_time = reader.read_u32::<LE>()?;

        Ok(Transaction {
            version,
            inputs,
            outputs,
            lock_time,
        })
    }
}
```
