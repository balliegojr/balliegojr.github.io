
+++
title = "Traits for beginners"
description =  "This post is for those who are starting their journey with coding and Rust, it is an introduction to traits, and I will try to explain what are traits and how to use them."
date = 2024-01-07
slug = "traits-for-beginners"
language = "en"

[taxonomies]
tags = ["rust", "traits"]
+++

This post is for those who are starting their journey with coding and Rust, it is an introduction to traits, and I will try to explain what are traits and how to use them.

<!-- more -->
My goal is to try to shed some light on how one can use traits to write code that is simpler to understand and maintain. 

We are going to implement network encryption using [chacha20poly1305](https://crates.io/crates/chacha20poly1305). First, we will implement it without traits. Then, we will refactor it to use traits and see how much better the code will be.

I am assuming you have some familiarity with Rust basics, the [book](https://doc.rust-lang.org/book/ch01-00-getting-started.html) is a great place to start if you don’t.

A full async implemenation of the encryption and encryption traits can be found on my [async-encrypted-stream](https://github.com/balliegojr/async-encrypted-stream) Github repo.

{% box(title="Encryption") %}

You can read about encryption on [Wikipedia](https://en.wikipedia.org/wiki/Encryption), but a short and naive description is "use this key to scramble this information". If you want to see the information, you need the key to unscramble it. 

If you are going to use the same key to encrypt different pieces of data, it is a good idea to add some "noise" to prevent people from deducing the key.

Stream encryption algorithms add noise to the key (known as salt) and to each piece of information being encrypted (known as **nonce**)
{% end %}

# Reading and Sending the data
As a first step, it is necessary to build the basics to send and receive the data. Due to the goal of this post, the code will not be async, because this would lead to unnecessary complexity.

The code will be a simple implementation that connects to a host and encrypts all the information exchanged. I will split the execution flow into “reading” and “writing” threads. The reading part will wait for messages from the network and write them to an internal channel. The writing part will wait for messages from an internal channel and then send them to the network.

The code below contains comments explaining each section.

```rust

// Imports all the necessary types from the standard library.
// This is important mostly for the traits
use std::{
    error::Error,
    io::{Read, Write},
    net::TcpStream,
    sync::{
        mpsc::{Receiver, Sender},
        Arc,
    },
    thread,
};

// Message types to avoid confusion with the channels
pub struct InboundMessage(Vec<u8>);
pub struct OutboundMessage(Vec<u8>);

fn connect_to_host(
    addr: &str,
) -> (Sender<OutboundMessage>, Receiver<InboundMessage>) {
    // Creates a TcpStream by connecting to a host and 
    // wraps it an Arc to share between two threads
    let stream = Arc::new(TcpStream::connect(addr).expect("Failed to connect"));

    let inbound_channel = receive_data(stream.clone());
    let outbound_channel = send_data(stream);

    (outbound_channel, inbound_channel)
}

fn send_data(stream: Arc<TcpStream>) -> Sender<OutboundMessage> {
    // Spawn a thread that will read some bytes from the channel
    // and send to the stream
    let (tx, rx) = std::sync::mpsc::channel();
    thread::spawn(move || {
        let stream = &*stream;
        while let Ok(OutboundMessage(bytes)) = rx.recv() {
            if stream.write_all(&bytes).is_err() {
                break;
            }
        }
    });

    tx
}

fn receive_data(stream: Arc<TcpStream>) -> Receiver<InboundMessage> {
    // Spawn a thread that will read some bytes from the network 
    // and send to a channel
    let (tx, rx) = std::sync::mpsc::channel();
    thread::spawn(move || {
        let stream = &*stream;
        let mut buf = [0u8; 4096];
        while let Ok(n) = stream.read(&mut buf) {
            if n == 0 {
                break;
            }

            if tx.send(InboundMessage(buf[..n].to_vec())).is_err() {
                break;
            }
        }
    });

    rx
}
```

The most important part in the code above is the `read` call, **it populates the given buffer and returns the number of bytes read from the network**, this is ok for now. 

# Adding encryption

With the basics in place, it is time to add encryption. For that, it is necessary to add a new dependency in the Cargo.toml file.
```toml
[dependencies]
chacha20poly1305 = { version = "0.10.1", features = ["stream", "std"] }
```

With the new dependency added, now it is time to create the encryptor and decryptor structs.
```rust
use chacha20poly1305::{
    aead::{
        stream::{DecryptorLE31, EncryptorLE31},
        KeyInit,
    },
    XChaCha20Poly1305,
};

pub fn get_encryptor_and_decryptor(
    key: [u8; 32],
    nonce: [u8; 20],
) -> (
    EncryptorLE31<XChaCha20Poly1305>,
    DecryptorLE31<XChaCha20Poly1305>,
) {
    // Creates the encryptor and decryptor pair with provided key and nonce.
    let encryptor: EncryptorLE31<XChaCha20Poly1305> =
        chacha20poly1305::aead::stream::EncryptorLE31::from_aead(
            XChaCha20Poly1305::new(key.as_ref().into()),
            nonce.as_ref().into(),
        );

    let decryptor: DecryptorLE31<XChaCha20Poly1305> =
        chacha20poly1305::aead::stream::DecryptorLE31::from_aead(
            XChaCha20Poly1305::new(key.as_ref().into()),
            nonce.as_ref().into(),
        );

    (encryptor, decryptor)
}
```

Now that we know how to get the encryptor and decryptor, it is time to change the `send_data` and `receive_data` to use them to apply encryption to the data. Luckily, the encryption crate is very easy to use, we just need to call `encrypt_next` and `decrypt_next`.

```rust
fn send_data(
    stream: Arc<TcpStream>,
    // receives the encryptor as a parameter.
    mut encryptor: EncryptorLE31<XChaCha20Poly1305>, 
    // flag to enable encryption.
    encryption_enabled: bool, 
) -> Sender<OutboundMessage> {
    let (tx, rx) = std::sync::mpsc::channel();
    thread::spawn(move || {
        let stream = &*stream;
        while let Ok(OutboundMessage(bytes)) = rx.recv() {
            if encryption_enabled {
                // if encryption is enabled, encrypts the whole message 
                // and send the encrypted result to the tcp stream
                let Ok(encrypted) = encryptor.encrypt_next(&bytes[..]) else { 
                    break; 
                };
                if stream.write_all(&encrypted).is_err() {
                    break;
                }
            } else if stream.write_all(&bytes).is_err() {
                break;
            }
        }
    });

    tx
}

fn receive_data(
    stream: Arc<TcpStream>,
    // receives the decryptor as a parameter.
    mut decryptor: DecryptorLE31<XChaCha20Poly1305>, 
    // flag to enable encryption.
    encryption_enabled: bool, 
) -> Receiver<InboundMessage> {
    let (tx, rx) = std::sync::mpsc::channel();

    thread::spawn(move || {
        let stream = &*stream;
        let mut buf = [0u8; 4096];
        while let Ok(n) = stream.read(&mut buf) {
            if n == 0 {
                break;
            }

            if tx.send(InboundMessage(buf[..n].to_vec())).is_err() {
                break;
            }

            if encryption_enabled {
                // If encryption is enabled, decrypt what came 
                // from the stream and send to the channel
                let Ok(decrypted) = decryptor.decrypt_next(&buf[..n]) else { 
                    break; 
                };
                if tx.send(InboundMessage(decrypted)).is_err() {
                    break;
                }
            } else if tx.send(InboundMessage(buf[..n].to_vec())).is_err() {
                break;
            }
        }
    });

    rx
}

```

Both functions `send_data` and `receive_data` were changed to optionally use encryption over the stream.

Amazing, another well done job! Everything works fine! It is time to pack the bag and go home. Except, it doesn't work! I mean, the code would work fine with a simple "hello" message. The data would be encrypted, sent over the wire, and decrypted on the other side. However, the code has a fundamental flaw.

The issue is that the `encrypt_next` and `decrypt_next` calls must be equal on both ends. For each call to `encrypt_next`, `decrypt_next` must be called exactly once, because the **nonce** value changes every time it is used.

Imagine a message of 100 bytes. After encryption, the message will be longer, the exact size will depend on the encryption used, but let's say it is 120 bytes. When `read` is called on the other side, it will read what is available in the buffer, which may or may not be 120 bytes. The function `decrypt_next` must be called with the same data generated by `encrypt_next`. 

To solve this issue, it is necessary to know exactly how many bytes must be sent to the next `decrypt_next` call. This can be done by sending the length of the message before sending the message itself.

```rust
fn send_data(
    stream: Arc<TcpStream>,
    mut encryptor: EncryptorLE31<XChaCha20Poly1305>,
    encryption_enabled: bool,
) -> Sender<OutboundMessage> {
    let (tx, rx) = std::sync::mpsc::channel();
    thread::spawn(move || {
        let stream =  &*stream;
        while let Ok(OutboundMessage(bytes)) = rx.recv() {
            if encryption_enabled {
                let Ok(encrypted) = encryptor.encrypt_next(&bytes[..]) else { 
                    break; 
                };

                // Get the size of the message and send to the stream.
                // This is the only necessary change.
                let size = (encrypted.len() as u16).to_be_bytes();
                if stream.write_all(&size).is_err() {
                    break;
                }
                
                if stream.write_all(&encrypted).is_err() {
                    break;
                }
            } else {
                // Get the size of the message and send to the stream.
                // This is the only necessary change.
                let size = (bytes.len() as u16).to_be_bytes();
                if stream.write_all(&size).is_err() {
                    break;
                }

                if stream.write_all(&bytes).is_err() {
                    break;
                }
            }
        }
    });

    tx
}

```

Sending is the easiest part. Just get the length of the message, cast it to a bytes array, and write it to the stream before the actual data. 

Just a few observations before we proceed:
- Never use `usize` when sending binary data over the network. Different architectures may have different `usize` sizes. It is better to be explicit here. 
- 16 bits is a good size for most scenarios, but it may be necessary to use a bigger value if you need to deal with very big data.
- `to_be_bytes()` returns the **big endian** bytes representation of the number
- Calling `write_all` multiple times without buffering will result in poor performance. 

Welcome to network programming!

```rust
fn receive_data(stream: Arc<TcpStream>,
    mut decryptor: DecryptorLE31<XChaCha20Poly1305>,
    encryption_enabled: bool,
) -> Receiver<InboundMessage> {
    let (tx, rx) = std::sync::mpsc::channel();

    thread::spawn(move || {
        let stream = &*stream;

        let mut buf = vec![0u8; 4096];
        // Keep track of how many bytes across multiple reads
        let mut position = 0;
        while let Ok(n) = stream.read(&mut buf[position..]) {
            if n == 0 {
                break;
            }
            // Move the position to account for the new data.
            position += n;

            // buf may have enough data for multiple messages
            // we are going to try to process everything.
            loop {
                // Since the length is u16, we need at least 2 bytes.
                if position < 2 {
                    break;
                }

                // Read the first 2 bytes to get the length of the message
                let length_bytes = [buf[0], buf[1]];
                let end = u16::from_be_bytes(length_bytes) as usize + 2;

                // Verify if the whole message is in the buffer.
                if position < end {
                    break;
                }

                let payload = &buf[2..end];
                if encryption_enabled {
                    let Ok(decrypted) = decryptor.decrypt_next(payload) else { 
                        break; 
                    };
                    if tx.send(InboundMessage(decrypted)).is_err() {
                        break;
                    }
                } else if tx.send(InboundMessage(payload.to_vec())).is_err() {
                    break;
                }

                // Shift the contents of the buffer to the start
                if position > end {
                    buf.copy_within(end..position, 0);
                    position -= end;
                } else {
                    position = 0;
                }
            }
        }
    });

    rx
}

```

With the changes in place, the decryption should work, but the code is becoming complex and error-prone, and it has huge limitations.

Two bugs are present in the current implementation.
- The code fails if a message is longer than 4094 bytes. The solution for this is to use a dynamic-sized buffer.
- If decryption fails, `receive_data` will loop without resetting the buffer position, leading to an infinite loop.

On top of the bugs, the current approach mixes encrypted and plain communication in the same place, and also, extending this code is becoming hard due to the complexity. 

I also mentioned before that calling `write_all` repeatedly was a bad idea, let's address that issue by using [BufWriter](https://doc.rust-lang.org/std/io/struct.BufWriter.html). 

`BufWriter` is quite easy to use, we just need to pass the stream to it.


```rust
fn send_data(...) -> Sender<OutboundMessage> {
    // ommited for brevity.

    let stream = &*stream;
    // We just need to add this line to the code
    let mut stream = BufWriter::new(stream); 

    while let Ok(OutboundMessage(bytes)) = rx.recv() {
       // ommited for brevity.
    }
}
```

Neat. But how can the change be that simple? Is it not even necessary to change the rest of the code?

The reason this works is because both `TcpStream` and `BufWriter` types implement a trait called [Write](https://doc.rust-lang.org/std/io/trait.Write.html), in fact, the function `write_all` is defined in the `Write` trait, not in the `TcpStream` struct. 

Now, it is time to talk about traits!

# Traits

What are traits anyway? 

The [rust book](https://doc.rust-lang.org/book/ch10-02-traits.html) defines traits as **a way to define shared behavior in an abstract way**. A different way of defining a trait is **a way to isolate an aspect of something**, because when you work with a trait, you only care about a specific aspect of the whole. 

I will try to explain this with an analogy.

{% box(title="Analogy time") %}
One day you wake up and feel the urge to drink coffee, you drag yourself to the nearest electronics store and explain your urge to the salesperson, the salesperson looks you in the eye and says "I have something that will solve all your problems", he takes you to the coffee machines aisle, slaps the box of the most expensive machine and says "You can make so much coffee with this baby that you will never sleep again!!! Just plug it on the wall, and you are ready to go!". So you buy the machine, take it home, and make the most wonderful coffee you will ever taste, and while you have the caffeine rush running through your veins, you think "I am really lucky that the electrician I hired 10 years ago predicted I was going to buy this exact machine, otherwise, I would not have coffee now!!!"

Well, that electrician had no idea you would buy a coffee machine, so how does the machine work when you plug it into the wall? 

The reason the machine works is that the **socket is an interface** between the electrician's work and the manufacturer's work, the **interface establishes a standard to be followed** by both parties, and any equipment that shares that **aspect** will work.

The socket defines how the machine and the electric energy interact with each other by isolating one single aspect of the machine. 

For the electrician, the socket means he doesn't need to care about which appliance it is going to be used, he only needs to care about putting the correct wires in the wall.

For the manufacturer, the socket means they don't need to care about the electric wiring, they only need to care about producing the coffee machine.

For you, the user, the socket means you only need to care about **plugging** things together. You **do not need to know**  the details about the wiring or about the coffee machine internals. But the most important part is that **any coffee machine will work in this socket**.

One very important detail about isolating aspects is that sometimes the aspect misses important information, for instance, what is the current of a given socket? You can't know the current just by looking at the socket, you need some extra information to know that, you need to know the specifics.
{% end %}

Back to Rust, the **Write** trait is the "socket" from our analogy. When a type implements the Write trait, it gains the ability to be treated by its Write aspect, which creates the possibility of ignoring every other aspect of the struct. When the code sees a `Write` type, it doesn't care where or how the data is written, it only cares that the type can write the data somewhere, be it a file, a network connection, or a memory buffer. 

# Using Traits in the code

Now that we have an idea of what traits are, it is time to rewrite the code with traits to make it better. 

Let's start with the plain text communication by implementing two structs, PlainReadHalf and PlainWriteHalf. The two structs will implement the **Read** and **Write** traits respectively.

```rust
/// PlainReadHalf has an inner reader to read from
/// This reads as "inner can be of any type T as long as T implements Read"
pub struct PlainReadHalf<T> where T: Read {
    inner: T,
}

impl<T> PlainReadHalf<T> where T: Read {
    pub fn new(inner: T) -> Self {
        Self { inner }
    }
}

/// Implements Read for the struct PlainReadHalf<T>
impl<T> Read for PlainReadHalf<T> where T: Read {
    fn read(&mut self, buf: &mut [u8]) -> std::io::Result<usize> {
        // Reads exactly 2 bytes from the inner reader
        let mut len_buf = [0u8; 2];
        self.inner.read_exact(&mut len_buf)?;
        let len = u16::from_be_bytes(len_buf) as usize;

        if buf.len() < len {
            return Err(std::io::Error::new(
                std::io::ErrorKind::InvalidInput,
                "Provided buffer is not big enough",
            ));
        }

        // Reads exactly the amount of bytes necessary for the message
        self.inner.read_exact(&mut buf[..len])?;

        Ok(len)
    }
}
```

In the code above, there are two important details about Traits.

The `PlainReadHalf` struct has a **generic type parameter** `T` that is constrained to **anything that implements Read**, which means T can be a `BufReader`, a `TcpStream`, or anything that implements the `Read` trait.

Then comes the implementation of the `Read` trait for the struct itself. With this, `PlainReadHalf` can be used anywhere a `Read` type is expected. 

The `where T: Read` syntax is where the code **isolates the Read aspect** of `T`. By writing this, `T` can be anything, but we only care about the `Read` part.

Now let's look the `PlainWriteHalf` struct

```rust
/// PlainWriteHalf has an inner writer to write to, 
/// It also has a buffer to hold the information before sending to the writer
pub struct PlainWriteHalf<T> where T: Write {
    inner: T,
    buffer: Vec<u8>,
}

impl<T> PlainWriteHalf<T> where T: Write {
    pub fn new(inner: T) -> Self {
        Self {
            inner,
            buffer: Vec::new(),
        }
    }
}

// Implements Write for the PlainWriteHalf struct
impl<T> Write for PlainWriteHalf<T> where T: Write {
    // Write function just write the contents to the internal buffer
    fn write(&mut self, buf: &[u8]) -> std::io::Result<usize> {
        self.buffer.extend_from_slice(buf);
        Ok(buf.len())
    }

    // Flush is the function that actually send 
    // the information to the inner writer
    fn flush(&mut self) -> std::io::Result<()> {
        let bytes = (self.buffer.len() as u16).to_be_bytes().to_vec();
        self.inner.write_all(&bytes)?;
        self.inner.write_all(&self.buffer)?;

        self.inner.flush()?;
        self.buffer.clear();

        Ok(())
    }
}
```

One thing you may be wondering is, what is the benefit of this? That seems to be a lot more code to do the same thing as before. Why go through all this trouble? 

Let's check the first benefit of traits by writing a test for this struct.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::io::Cursor;
    use std::io::Write;

    #[test]
    pub fn test_plain_write_read() {
        // Cursor<&mut Vec<T>> implements Write, 
        // it can be used with our PlainWriteHalf struct
        let mut buf = vec![0u8; 512];

        let mut writer = PlainWriteHalf::new(Cursor::new(&mut buf));
        let _ = writer.write_all(&b"some bytes"[..]);
        let _ = writer.flush();

        assert_eq!(buf[2..12], b"some bytes"[..]);

        // Cursor<& Vec<T>> implements Read, 
        // it can be used with our PlainReadHalf struct
        let mut reader = PlainReadHalf::new(Cursor::new(&buf));

        let mut output = String::new();
        let _ = reader.read_to_string(&mut output);

        assert_eq!(output, "some bytes");
    }
}
```

Notice that we don't need a TCP connection to test the code, we did manage to write the test for our struct implementation by using another type that implements the necessary traits.

The encrypted implementation will be very similar to the plain text implementation, except it will have an extra step.

```rust
/// EncryptedReadHalf has the inner writer and a decryptor
pub struct EncryptedReadHalf<T>
where
    T: Read,
{
    inner: T,
    decryptor: DecryptorLE31<XChaCha20Poly1305>,
}

impl<T> EncryptedReadHalf<T>
where
    T: Read,
{
    pub fn new(inner: T, decryptor: DecryptorLE31<XChaCha20Poly1305>) -> Self {
        Self { inner, decryptor }
    }
}

impl<T> Read for EncryptedReadHalf<T>
where
    T: Read,
{
    fn read(&mut self, buf: &mut [u8]) -> std::io::Result<usize> {
        // First read the length of the payload
        let mut len_buf = [0u8; 2];
        self.inner.read_exact(&mut len_buf)?;
        let payload_len = u16::from_be_bytes(len_buf) as usize;

        // Reads the payload
        let mut payload = vec![0u8; payload_len];
        self.inner.read_exact(&mut payload)?;

        // Decrypts the payload
        let decrypted = self
            .decryptor
            .decrypt_next(&payload[..])
            .map_err(|err| { 
                std::io::Error::new(std::io::ErrorKind::InvalidData, err)
            })?;
        let decrypted_len = decrypted.len();
        if buf.len() < decrypted_len {
            return Err(std::io::Error::new(
                std::io::ErrorKind::InvalidInput,
                "Provided buffer is not big enough",
            ));
        }

        // Writes the plain content to the buffer
        buf[..decrypted_len].copy_from_slice(&decrypted[..]);
        Ok(decrypted_len)
    }
}
```

The WriteHalf implementation is also very similar to the PlainWriteHalf, with just an extra step.

```rust
pub struct EncryptedWriteHalf<T>
where
    T: Write,
{
    inner: T,
    encryptor: EncryptorLE31<XChaCha20Poly1305>,
    buffer: Vec<u8>,
}

impl<T> EncryptedWriteHalf<T>
where
    T: Write,
{
    pub fn new(inner: T, encryptor: EncryptorLE31<XChaCha20Poly1305>) -> Self {
        Self {
            inner,
            encryptor,
            buffer: Vec::new(),
        }
    }
}

impl<T> Write for EncryptedWriteHalf<T>
where
    T: Write,
{
    fn write(&mut self, buf: &[u8]) -> std::io::Result<usize> {
        self.buffer.extend_from_slice(buf);
        Ok(buf.len())
    }

    fn flush(&mut self) -> std::io::Result<()> {
        // Encrypts the contents of the internal buffer
        let encrypted = self
            .encryptor
            .encrypt_next(&self.buffer[..])
            .map_err(|err| { 
                std::io::Error::new(std::io::ErrorKind::InvalidInput, err)
            })?;

        // Writes the encrypted value to the internal writer
        let len_bytes = (encrypted.len() as u16).to_be_bytes().to_vec();
        self.inner.write_all(&len_bytes)?;
        self.inner.write_all(&encrypted)?;

        self.inner.flush()?;

        // Clean the internal buffer
        self.buffer.clear();

        Ok(())
    }
}
```

And again, let's write a test case.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::io::Cursor;
    use std::io::Write;

    #[test]
    pub fn test_encryption() {
        let mut buf = vec![0u8; 512];
        let (encryptor, decryptor) =
            super::super::get_encryptor_and_decryptor([0u8; 32], [0u8; 20]);

        let mut writer = EncryptedWriteHalf::new(Cursor::new(&mut buf), encryptor);
        let _ = writer.write_all(&b"some bytes"[..]);
        let _ = writer.flush();

        let mut reader = EncryptedReadHalf::new(Cursor::new(&buf), decryptor);
        let mut output = String::new();
        let _ = reader.read_to_string(&mut output);

        assert_eq!(output, "some bytes");
    }
}
```

The only thing left is to adapt the rest of the code to use the new structs. With the new structs, the `thread` handling will change because of the rust borrower checker.

```rust
/// send_data receives a writer, it can be anything that implements Write
/// It will work with PlainWriteHalf or EncryptedWriteHalf
fn send_data(mut stream: impl Write, rx: Receiver<OutboundMessage>) {
    while let Ok(OutboundMessage(bytes)) = rx.recv() {
        if stream.write_all(&bytes).is_err() {
            break;
        }

        if stream.flush().is_err() {
            break;
        }
    }
}

/// receive_data receives a reader, it can be anything that implements Read
/// It will work with PlainReadHalf or EncryptedReadHalf
fn receive_data(mut stream: impl Read, tx: Sender<InboundMessage>) {
    let mut buf = vec![0u8; 4096];
    while let Ok(n) = stream.read(&mut buf[..]) {
        if n == 0 {
            break;
        }

        if tx.send(InboundMessage(buf[..n].to_vec())).is_err() {
            break;
        }
    }
}

pub fn connect_to_host(
    addr: &str,
    key: [u8; 32],
    nonce: [u8; 20],
    encryption_enabled: bool,
) -> (Sender<OutboundMessage>, Receiver<InboundMessage>) {
    // Connects to the host 
    // &mut &TcpStream implements both Read and Write traits
    // So we create two copies of it to send to each thread 
    let stream = Arc::new(TcpStream::connect(addr).expect("Failed to connect"));

    // Each copy will be moved to a different thread
    let read_half = stream.clone();
    let write_half = stream.clone();

    // Initialize the channels to be returned from this function
    let (out_tx, out_rx) = std::sync::mpsc::channel();
    let (in_tx, in_rx) = std::sync::mpsc::channel();

    if encryption_enabled {
        let (encryptor, decryptor) = get_encryptor_and_decryptor(key, nonce);

        // Moves read_half and decryptor to a new thread
        thread::spawn(move || {
            let read_half = &*read_half; 
            receive_data(
                encrypted::EncryptedReadHalf::new(read_half, decryptor),
                in_tx,
            );
        });

        // Moves write_half and encryptor to a new thread
        thread::spawn(move || {
            let write_half = &*write_half;
            send_data(
                encrypted::EncryptedWriteHalf::new(write_half, encryptor),
                out_rx,
            );
        });
    } else {
        // Moves read_half to a new thread
        thread::spawn(move || {
            let read_half = &*read_half;
            receive_data(plain::PlainReadHalf::new(read_half), in_tx);
        });

        // Moves write_half to a new thread
        thread::spawn(move || {
            let write_half = &*write_half;
            send_data(plain::PlainWriteHalf::new(write_half), out_rx);
        });
    }

    (out_tx, in_rx)
}
```

In this new implementation, the code can be easily extended to add new features.

For instance, if you want to data compression, a new struct can be used to implement the Read and Write traits.

```rust
thread::spawn(move || {
    let read_half = &*read_half; 
    receive_data(
        compression::CompressedReadHalf::new(
            encrypted::EncryptedReadHalf::new(read_half, decryptor)
        ),
        in_tx,
    );
});

thread::spawn(move || {
    let write_half = &*write_half;
    send_data(
        compression::CompressedWriteHalf::new(
            encrypted::EncryptedWriteHalf::new(write_half, encryptor)
        ),
        out_rx,
    );
});
```

# Dynamic use of traits.

There is one thing in the `connect_to_host` function above that I don't like, there is a lot of repetition. It is possible to slim it down by spawning the threads only once.

```rust
let (reader, writer) = if encryption_enabled {
    let (encryptor, decryptor) = get_encryptor_and_decryptor(key, nonce);
    let reader = encrypted::EncryptedReadHalf::new(read_half, decryptor);
    ler writer = encrypted::EncryptedWriteHalf::new(write_half, encryptor);

    (reader, writer)
} else {
    let reader = plain::PlainReadHalf::new(read_half);
    ler writer = plain::PlainWriteHalf::new(write_half);

    (reader, writer)
}

thread::spawn(move || {
    receive_data(reader, in_tx);
});

thread::spawn(move || {
    send_data(writer, out_rx);
});
```

The code above is a lot cleaner, however, it does not compile because both paths of the if statement must return the same type, which is not happening now. The same problem happens when calling `receive_data` and `send_data` functions, the type must be known before calling. To understand how that can be fixed, we need to first talk about generics and [dynamic dispatch](https://en.wikipedia.org/wiki/Dynamic_dispatch).

When you write rust code with a generic parameter `<T>`, the compiler will use [monomorphization](https://en.wikipedia.org/wiki/Monomorphization) to generate code with concrete types, for instance:

```rust
fn my_func<T>() {}

my_func<u32>(); // This will generate my_func_u32 when compiled
my_func<u16>(); // This will generate my_func_u16 when compiled

struct MyStruct<T> {
    inner: T
}

var s = MyStruct { inner: 0u32 }; // MyStruct_u32
my_func<MyStruct<u32>>(); // This will generate my_func_my_struct_u32
```

The only restriction with generics is that you need to know the concrete types at compile time, which is not always the case. When the type cannot be defined, such as when it depends on a user-provided parameter, we can use dynamic dispatch. The way to do that in Rust is to use the `dyn Trait` syntax. For instance.

```rust
fn send_data<T: Write>(writer: T) // Concrete generic type
fn send_data(writer: impl Write) // This is equivalent to the one above

fn send_data(writer: &dyn Write) // Dynamic dispatch with a reference
fn send_data(writer: Box<dyn Write>) // Dynamic dispatch with owned type

// Returning a type that implements a trait

// Concrete generic type, no branching is allowed inside the function
fn get_writer<T: Write>() -> T; 
fn get_writer() -> impl Write;

// Dynamic dispatch, it is allowed to branch and return different types
fn get_writer() -> Box<dyn Write>
```

Let's see how it is possible to change the `connect_to_host` function to use dynamic dispatch.
```rust
pub fn connect_to_host(
    addr: &str,
    key: [u8; 32],
    nonce: [u8; 20],
    encryption_enabled: bool,
    out_rx: Receiver<OutboundMessage>,
    in_tx: Sender<InboundMessage>,
) {
    let stream = Arc::new(TcpStream::connect(addr).expect("Failed to connect"));

    // explicitly declare reader: Box<dyn Read + Send> and writer: Box<dyn Write + Send>
    let (reader, writer): (Box<dyn Read + Send>, Box<dyn Write + Send>) = if encryption_enabled {
        let (encryptor, decryptor) = get_encryptor_and_decryptor(key, nonce);

        let reader = encrypted::EncryptedReadHalf::new(&*stream, decryptor);
        let writer = encrypted::EncryptedWriteHalf::new(&*stream, encryptor);

        // Send reader and writer to the heap, by boxing them
        // this will allow the dynamic dispatch
        (Box::new(reader), Box::new(writer))
    } else {
        let reader = plain::PlainReadHalf::new(&*stream);
        let writer = plain::PlainWriteHalf::new(&*stream);

        // Send reader and writer to the heap, by boxing them
        // this will allow the dynamic dispatch
        (Box::new(reader), Box::new(writer))
    };

    thread::scope(|s| {
        s.spawn(move || {
            send_data(writer, out_rx);
        });

        s.spawn(move || {
            receive_data(reader, in_tx);
        });
    })
}
```

The syntax `Trait + AnotherTrait` specifies how multiple trait constraints are defined in Rust. In our case, `Read + Send` is **must implement Read and Send**. [Send](https://doc.rust-lang.org/std/marker/trait.Send.html) is how Rust knows that a type can be sent to another thread.

In this particular case, it is not necessary to change `send_data` and `receive_data` signatures because Box implements both [Write](https://doc.rust-lang.org/std/boxed/struct.Box.html?search=arc#impl-Write-for-Box<W>) and [Read](https://doc.rust-lang.org/std/boxed/struct.Box.html?search=arc#impl-Read-for-Box<R>), but this is how it would look like.

```rust
fn send_data<'a>(mut stream: Box<dyn Write + 'a>, rx: Receiver<OutboundMessage>)
fn receive_data<'a>(mut stream: Box<dyn Read + 'a>, tx: Sender<InboundMessage>)
```

One last observation before we wrap up. Since we are talking about improving the code with traits, it wouldn't be fair if I don't show how to make the encryption algorithm generic as well.

```rust
use std::io::{Read, Write};
use std::ops::Sub;
use chacha20poly1305::{
    aead::{
        generic_array::ArrayLength,
        stream::{Decryptor, Encryptor, NonceSize, StreamPrimitive},
    },
    AeadInPlace,
};

// Introduces the U parameter for the decryptor.
// Notice we don't need to add the constraints here.
pub struct EncryptedReadHalf<T, U> {
    inner: T,
    decryptor: U,
}

// Introduces all the constraints necessary for the Decryptor to work
// This allow us to use the struct with any stream decryption algorithm
impl<T, A, S> EncryptedReadHalf<T, Decryptor<A, S>>
where
    T: Read,
    S: StreamPrimitive<A>,
    A: AeadInPlace,
    A::NonceSize: Sub<<S as StreamPrimitive<A>>::NonceOverhead>,
    NonceSize<A, S>: ArrayLength<u8>,
{
    pub fn new(inner: T, decryptor: Decryptor<A, S>) -> Self {
        Self { inner, decryptor }
    }
}

impl<T, A, S> Read for EncryptedReadHalf<T, Decryptor<A, S>>
where
    T: Read,
    S: StreamPrimitive<A>,
    A: AeadInPlace,
    A::NonceSize: Sub<<S as StreamPrimitive<A>>::NonceOverhead>,
    NonceSize<A, S>: ArrayLength<u8>,
{
    // ...
}


// Introduces the U parameter for the encryptor.
// Notice we don't need to add the constraints here.
pub struct EncryptedWriteHalf<T, U> {
    inner: T,
    encryptor: U,
}

// Introduces all the constraints necessary for the Encryptro to work
// This allow us to use the struct with any stream encryption algorithm
impl<T, A, S> EncryptedWriteHalf<T, Encryptor<A, S>>
where
    T: Write,
    S: StreamPrimitive<A>,
    A: AeadInPlace,
    A::NonceSize: Sub<<S as StreamPrimitive<A>>::NonceOverhead>,
    NonceSize<A, S>: ArrayLength<u8>,
{
    pub fn new(inner: T, encryptor: Encryptor<A, S>) -> Self {
        Self {
            inner,
            encryptor,
        }
    }
}

impl<T, A, S> Write for EncryptedWriteHalf<T, Encryptor<A, S>>
where
    T: Write,
    S: StreamPrimitive<A>,
    A: AeadInPlace,
    A::NonceSize: Sub<<S as StreamPrimitive<A>>::NonceOverhead>,
    NonceSize<A, S>: ArrayLength<u8>,
{
    // ...
}
```

# Conclusion

In this post, we have learned how to use traits to isolate code behavior and write cleaner code.

With the use of Traits, we can:
- Isolate and decouple code behavior.
- Test specific parts of the code.
- Reuse and share specific behaviors.
- Compose behavior by using multiple traits.

If you were struggling to understand what are Traits, or what are the benefits of using them, I hope this post have helped you.
