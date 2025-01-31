# oxiderr

[![License: BSD-3-Clause](https://img.shields.io/badge/license-BSD--3--Clause-blue)](./LICENSE)
[![oxiderr on crates.io](https://img.shields.io/crates/v/oxiderr)](https://crates.io/crates/oxiderr)
[![oxiderr on docs.rs](https://docs.rs/oxiderr/badge.svg)](https://docs.rs/oxiderr)
[![Source Code Repository](https://img.shields.io/badge/Code-On%20GitHub-blue?logo=GitHub)](https://github.com/cdumay/oxiderr)

`oxiderr` is a Rust library designed for extended error management. It leverages the `oxiderr-derive` crate, which provides procedural macros
to simplify the definition of structured error types. The primary goal of `oxiderr` is to enhance error handling in Rust applications by
making error definition more declarative and reducing boilerplate code.

## Features

* Provides extended error management capabilities.
* Implements the `oxiderr::AsError` trait for easy integration.
* Supports structured error kinds and categorized error handling.

## Usage

To utilize oxiderr in your project, follow these steps:

1. **Add Dependencies**: Include `oxiderr` with the feature `derive` in your `Cargo.toml`:

```toml
[dependencies]
oxiderr = "0.1"
```

2. **Define Error**: Define `oxiderr::ErrorKind` and struct which implement `oxiderr::AsError` to handle an error:

```rust
#[allow(non_upper_case_globals)]
pub const IoError: oxiderr::ErrorKind = oxiderr::ErrorKind(
    "IoError",
    "Input / output error",
    500,
    "The requested file raised error"
);
#[derive(Debug, Clone)]
pub struct NotFoundError {
    class: String,
    message: String,
    details: Option<std::collections::BTreeMap<String, serde_value::Value>>,
}

impl NotFoundError {
    pub const kind: oxiderr::ErrorKind = IoError;

    pub fn new() -> Self {
        Self {
            class: format!("{}::{}::{}", Self::kind.side(), Self::kind.name(), "NotFoundError"),
            message: Self::kind.description().into(),
            details: None,
        }
    }

    pub fn set_message(mut self, message: String) -> Self {
        self.message = message;
        self
    }

    pub fn set_details(mut self, details: std::collections::BTreeMap<String, serde_value::Value>) -> Self {
        self.details = Some(details);
        self
    }

    pub fn convert(error: oxiderr::Error) -> Self {
        let mut err_clone = error.clone();
        let mut details = error.details.unwrap_or_default();
        err_clone.details = None;
        details.insert("origin".to_string(), serde_value::to_value(err_clone).unwrap());

        Self {
            class: format!("{}::{}::{}", Self::kind.side(), Self::kind.name(), "NotFoundError"),
            message: Self::kind.description().into(),
            details: Some(details),
        }
    }
}

impl oxiderr::AsError for NotFoundError {
    fn kind() -> oxiderr::ErrorKind {
        Self::kind
    }
    fn class(&self) -> String {
        self.class.clone()
    }
    fn message(&self) -> String {
        self.message.clone()
    }
    fn details(&self) -> Option<std::collections::BTreeMap<String, serde_value::Value>> {
        self.details.clone()
    }
}

impl std::error::Error for NotFoundError {}

impl std::fmt::Display for NotFoundError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "[{}] {} ({}): {}", Self::kind.message_id(), "NotFoundError", Self::kind.code(), self.message())
    }
}
```

In this example:

* we create the struct `IoError` as `oxiderr::ErrorKind`
* we create the struct `NotFoundError` which implements `oxiderr::AsError`

3. Implementing Error Handling: With the above definitions, you can now handle errors in your application as follows:

```rust
use std::fs::File;
use std::io::Read;

fn try_open_file(path: &str) -> oxiderr::Result<File> {
    Ok(File::open(path).map_err(|err| NotFoundError::new().set_message(err.to_string()))?)
}

fn main() {
    let path = "example.txt";

    match try_open_file(path) {
        Ok(file) => println!("File: {:?}", file),
        Err(e) => eprintln!("{}", e),
    }
}
```
This will output:

```
[Err-00001] Client::IoError::NotFoundError (500) - No such file or directory (os error 2)
```

## Macros

To automatically generate implementations for custom error types, enable the feature `derive` in your Cargo.toml:

```toml
[dependencies]
oxiderr = { version = "0.1", features = ["derive"] }
```

Then, use the provided derive macros to define your error and error kind structs:

```rust
use oxiderr::{define_errors, define_kinds, AsError};

define_kinds! {
    UnknownError = ("Err-00001", 500, "Unexpected error"),
    IoError = ("Err-00001", 400, "IO error")
}
define_errors! {
    Unexpected = UnknownError,
    FileRead = IoError,
    FileNotExists = IoError
}
```

See [oxiderr-derive](https://docs.rs/oxiderr-derive) documentation for more information.

