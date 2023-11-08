<figure>
  <img src="tlsn-banner.png" alt="tlsn-banner">
</figure>

# Progcrypto 2023: TLSNotary workshop

This workshop helps you getting started with TLSNotary, in native Rust and in the Browser.

1. [Most basic example: Proof and Verify public data (Rust)](#rust-simple)
2. [Proof and Verify a private Discord DM (Rust)](#rust-discord)
3. [Proof and Verify a private Twitter DM (Browser)](#browser)

Objectives of this workshop:
* Have a better feeling about what you can do with TLSNotary
* Learn the basics on how to proof and verify data with TLSNotary

## Setup 

### Clone workshop repository

Clone this repository first

```shell
git clone git@github.com:heeckhau/progcrypto_workshop.git
```

### Install Rust

If you don't have `rust` installed yet, install it with [rustup](https://rustup.rs/):
```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## Simple example: Notarize public data from example.com (Rust) <a name="rust-simple"></a>

We start with the simplest possible use case for TLSNotary: 
1. Fetch <https://example.com/> and create a proof of its content
2. Verify the proof

Next we will redact the content and verify again:
1. Redact the `USER_AGENT` and titles
2. Verify the redacted proof

### 1. Notarize <https://example.com/>

Run a simple prover

```shell
cargo run --release --example simple_prover
```

Note: you can also run the `simple_prover` by clicking the **▶️ Run** button in VS Code. However, this will run without the `--release` by default and the execution will be a lot slower.

![](images/run_vs_code.png)


If the notarization went fine, you should see this output in the console:
```log
Listening on: 127.0.0.1:8080
Connected to the Notary
Accepted connection from: 127.0.0.1:64072
Starting an MPC TLS connection with the server
Got a response from the server
Notarization completed successfully!
The proof has been written to `simple_proof.json`
```

### 2. Verify the proof

When you open `simple_proof.json` in an editor, you will see a json file with lots of non-human readable byte arrays. You can decode this file by running:

```shell
cargo run --release --example simple_verifier
```

This will output the TLS=transaction in clear text:
```log
Successfully verified that the bytes below came from a session with Dns("example.com") at 2023-11-03 08:48:20 UTC.
Note that the bytes which the Prover chose not to disclose are shown as X.

Bytes sent:
...
```

### 3. Redact information

Open `simple_prover.rs` and locate the line with:
```rust
    let redact = false;
```
and change it to
```rust
    let redact = true;
```

Next, if you run the `simple_prover` and `simple_verifier` again, you'll notice redacted `X`'s in the output:
```shell
cargo run --release --example simple_prover
cargo run --release --example simple_verifier
```

```log
<!doctype html>
<html>
<head>
    <title>XXXXXXXXXXXXXX</title>
...
```

You can also use <https://tlsnotary.github.io/proof_viz/> to inspect your proofs.
Open <https://tlsnotary.github.io/proof_viz/> and drag and drop `simple_proof.json` from your file explorer into the drop zone.
![](images/proof_viz.png)

Redacted bytes are marked with <span style="color:red">red █ characters</span>.

### (Optional) Extra experiments

If the above steps were easy for you and you are waiting for the others: feel free to try these extra challenges:

- [ ] Modify the `server_name` (or any other data) in `simple_proof.json` and verify the proof is no longer valid
- [ ] Modify `build_proof_with_redactions` function in `simple_prover.rs` to redact more or different data



## Notarize private information: Discord message (Rust)<a name="rust-discord"></a>

Next we will use TLSNotary to generate a proof of private information: a private Discord DM.

We will also use an explicit (locally hosted) notary server this time.


### 1. Start a local notary server

The notary server used in this example is more functional compared to the (implicit) simple notary service, used in the example above. The simple notary version is easier to integrate from a prover's perspective, whereas this notary server provides additional features like TLS connection with prover, WebSocket endpoint, API endpoints for further customisation etc.

```shell
cd notary
cargo run --release
```

The notary server will now be running in the background waiting for connections.

Keep it running and open a new terminal.

### 2. Get Authorization token and channel ID

In the main folder, copy a `.env.example` file and name it `.env`.

In this `.env` we will input the `USER_AGENT`, `AUTHORIZATION` token and `CHANNEL_ID`.

| Name          | Example                                                                          | Location                                      |
| ------------- | -------------------------------------------------------------------------------- | --------------------------------------------- |
| USER_AGENT    | `Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/116.0` | Look for `User-Agent` in a request headers    |
| AUTHORIZATION | `MTE1NDe1Otg4N6NxNjczOTM2OA.GYbUBf.aDtcMUKDOmg6C2kxxFtlFSN1pgdMMBtpHgBBEs`       | Look for `Authorization` in a request headers |
| CHANNEL_ID    | `1154750485639745567`                                                            | URL                                           |

You can obtain these parameters by opening [Discord](https://discord.com/channels/@me) in your browser and accessing the message history you want to notarize. 

> **_NOTE:_** ⚠️ Please note that notarizing only works for short transcripts at the moment, so choose a contact with a short history.

Next, open the **Developer Tools**, go to the **Network** tab, and refresh the page. Then, click on **Search** and type `/api` to filter results to Discord API requests. From there you can copy the needed information into your `.env` as indicated above.

You can find the `CHANNEL_ID` directly in the url:

`https://discord.com/channels/@me/{CHANNEL_ID)`

![](./images/discord_authentication_token.png)

### 3. Create proof

Next run the `discord_dm` example to generate a proof:
```shell
RUST_LOG=debug,yamux=info cargo run --release --example discord_dm
```

If all goes well, you get this output:
```log
...
2023-11-03T15:53:51.147732Z DEBUG discord_dm: Notarization complete!
```

The Notary server should log:
```log
2023-11-03T15:53:46.540247Z DEBUG                 main ThreadId(01) run_server: notary_server::server: Received a prover's TCP connection prover_address=127.0.0.1:56631
...
2023-11-03T15:53:46.542261Z DEBUG tokio-runtime-worker ThreadId(10) notary_server::service: Starting notarization... session_id="006b3293-8fba-44ac-8692-41daa47e4a9a"
...
2023-11-03T15:53:51.147074Z  INFO tokio-runtime-worker ThreadId(10) notary_server::service::tcp: Successful notarization using tcp! session_id="006b3293-8fba-44ac-8692-41daa47e4a9a"
```

If the transcript was too long, you may encounter the following error. This occurs because there is a default limit of notarization size to 16kB:

```log
thread 'tokio-runtime-worker' panicked at 'called `Result::unwrap()` on an `Err` value: IOError(Custom { kind: InvalidData, error: BackendError(DecryptionError("Other: KOSReceiverActor is not setup")) })', /Users/heeckhau/tlsnotary/tlsn/tlsn/tlsn-prover/src/lib.rs:173:50
```

The Discord example code redacts the `auth_token`, but feel free to change the redacted regions.

The proof is written to `discord_dm_proof.json` 

### Verify

Verify the proof by dropping the json file into <https://tlsnotary.github.io/proof_viz/> or by running:
```shell
cargo run --release --example discord_dm_verifier
```

Good job! You have successfully used TLSNotary in Rust.

> **_NOTE:_** ℹ️ Leave the local notary server running for the next part of this workshop: testing the Browser extension.
The MPC between the prover and the notary requires a lot of bandwidth, certainly too much for all of us to use a publicly hosted notary server at the same time.


### (Optional) Notarize more private data

If the examples above were too easy for you, try to notarize data from other websites such as:

- [ ] Amazon purchase
- [ ] Twitter dm
- [ ] LinkedIn skill
- [ ] Steam accomplishment
- [ ] Garmin.connect achievement
- [ ] AirBnB score
- [ ] Tesla ownership

## TLSNotary browser extension <a name="browser"></a>

TODO

### Install browser extension

TODO

### Run a local proxy

TODO

### Notarize twitter account access

TODO


