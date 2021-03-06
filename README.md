# avro-rs

[![Latest Version](https://img.shields.io/crates/v/avro-rs.svg)](https://crates.io/crates/avro-rs)
[![Build Status](https://travis-ci.org/flavray/avro-rs.svg?branch=master)](https://travis-ci.org/flavray/avro-rs)
[![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/flavray/avro-rs/blob/master/LICENSE)

[Documentation](https://docs.rs/avro-rs)

A library for working with [Apache Avro](https://avro.apache.org/) in Rust.

Please check our [documentation](https://docs.rs/avro-rs) for examples, tutorials and API reference.

We also support:
* C bindings for the crate at [avro-rs-ffi](https://github.com/flavray/avro-rs-ffi)
* A Python wrapper for the library at [pyavro-rs](https://github.com/flavray/pyavro-rs)

## Example

Add to your `Cargo.toml`:

```toml
[dependencies]
avro-rs = "0.6"
failure = "0.1.5"
serde = { version = "1.0", features = ["derive"] }
```

Then try to write and read in Avro format like below:

```rust
use avro_rs::{Codec, Reader, Schema, Writer, from_value, types::Record};
use failure::Error;
use serde::{Serialize, Deserialize};

#[derive(Debug, Deserialize, Serialize)]
struct Test {
    a: i64,
    b: String,
}

fn main() -> Result<(), Error> {
    let raw_schema = r#"
        {
            "type": "record",
            "name": "test",
            "fields": [
                {"name": "a", "type": "long", "default": 42},
                {"name": "b", "type": "string"}
            ]
        }
    "#;

    let schema = Schema::parse_str(raw_schema)?;

    println!("{:?}", schema);

    let mut writer = Writer::with_codec(&schema, Vec::new(), Codec::Deflate);

    let mut record = Record::new(writer.schema()).unwrap();
    record.put("a", 27i64);
    record.put("b", "foo");

    writer.append(record)?;

    let test = Test {
        a: 27,
        b: "foo".to_owned(),
    };

    writer.append_ser(test)?;

    writer.flush()?;

    let input = writer.into_inner();
    let reader = Reader::with_schema(&schema, &input[..])?;

    for record in reader {
        println!("{:?}", from_value::<Test>(&record?));
    }
    Ok(())
}
```

### Calculate Avro schema fingerprint

This library supports calculating the following fingerprints:

 - SHA-256
 - MD5

Note: Rabin fingerprinting is NOT SUPPORTED yet.

An example of fingerprinting for the supported fingerprints:

```rust
use avro_rs::Schema;
use failure::Error;
use md5::Md5;
use sha2::Sha256;

fn main() -> Result<(), Error> {
    let raw_schema = r#"
        {
            "type": "record",
            "name": "test",
            "fields": [
                {"name": "a", "type": "long", "default": 42},
                {"name": "b", "type": "string"}
            ]
        }
    "#;
    let schema = Schema::parse_str(raw_schema)?;
    println!("{}", schema.fingerprint::<Sha256>());
    println!("{}", schema.fingerprint::<Md5>());
    Ok(())
}
```

### Ill-formed data

In order to ease decoding, the Binary Encoding specification of Avro data
requires some fields to have their length encoded alongside the data.

If encoded data passed to a `Reader` has been ill-formed, it can happen that
the bytes meant to contain the length of data are bogus and could result
in extravagant memory allocation.

To shield users from ill-formed data, `avro-rs` sets a limit (default: 512MB)
to any allocation it will perform when decoding data.

If you expect some of your data fields to be larger than this limit, be sure
to make use of the `max_allocation_bytes` function before reading **any** data
(we leverage Rust's [`std::sync::Once`](https://doc.rust-lang.org/std/sync/struct.Once.html)
mechanism to initialize this value, if
any call to decode is made before a call to `max_allocation_bytes`, the limit
will be 512MB throughout the lifetime of the program).


```rust
use avro_rs::max_allocation_bytes;


fn main() {
    max_allocation_bytes(2 * 1024 * 1024 * 1024);  // 2GB

    // ... happily decode large data
}

```

### Logical types

```rust
use avro_rs::{
    types::Record, types::Value, Codec, Days, Decimal, Duration, Millis, Months, Reader, Schema,
    Writer,
};
use num_bigint::ToBigInt;
use failure::Error;

fn main() -> Result<(), Error> {
    let raw_schema = r#"
    {
      "type": "record",
      "name": "test",
      "fields": [
        {
          "name": "decimal_fixed",
          "type": {
            "type": "fixed",
            "size": 2,
            "name": "decimal"
          },
          "logicalType": "decimal",
          "precision": 4,
          "scale": 2
        },
        {
          "name": "decimal_var",
          "type": "bytes",
          "logicalType": "decimal",
          "precision": 10,
          "scale": 3
        },
        {
          "name": "uuid",
          "type": "string",
          "logicalType": "uuid"
        },
        {
          "name": "date",
          "type": "int",
          "logicalType": "date"
        },
        {
          "name": "time_millis",
          "type": "int",
          "logicalType": "time-millis"
        },
        {
          "name": "time_micros",
          "type": "long",
          "logicalType": "time-micros"
        },
        {
          "name": "timestamp_millis",
          "type": "long",
          "logicalType": "timestamp-millis"
        },
        {
          "name": "timestamp_micros",
          "type": "long",
          "logicalType": "timestamp-micros"
        },
        {
          "name": "duration",
          "type": {
            "type": "fixed",
            "size": 12,
            "name": "duration"
          },
          "logicalType": "duration"
        }
      ]
    }
    "#;

    let schema = Schema::parse_str(raw_schema)?;

    println!("{:?}", schema);

    let mut writer = Writer::with_codec(&schema, Vec::new(), Codec::Deflate);

    let mut record = Record::new(writer.schema()).unwrap();
    record.put("decimal_fixed", Decimal::from(9936.to_bigint().unwrap().to_signed_bytes_be()));
    record.put("decimal_var", Decimal::from((-32442.to_bigint().unwrap()).to_signed_bytes_be()));
    record.put("uuid", uuid::Uuid::new_v4());
    record.put("date", Value::Date(1));
    record.put("time_millis", Value::TimeMillis(2));
    record.put("time_micros", Value::TimeMicros(3));
    record.put("timestamp_millis", Value::TimestampMillis(4));
    record.put("timestamp_micros", Value::TimestampMicros(5));
    record.put("duration", Duration::new(Months::new(6), Days::new(7), Millis::new(8)));

    writer.append(record)?;
    writer.flush()?;

    let input = writer.into_inner();
    let reader = Reader::with_schema(&schema, &input[..])?;

    for record in reader {
        println!("{:?}", record?);
    }
    Ok(())
}
```

## License
This project is licensed under [MIT License](https://github.com/flavray/avro-rs/blob/master/LICENSE).
Please note that this is not an official project maintained by [Apache Avro](https://avro.apache.org/).

## Contributing
Everyone is encouraged to contribute! You can contribute by forking the GitHub repo and making a pull request or opening an issue.
All contributions will be licensed under [MIT License](https://github.com/flavray/avro-rs/blob/master/LICENSE).
