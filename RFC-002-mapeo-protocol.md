# 0. About this Document

Disclaimer: this document is intentionally left fairly high-level, since it
both covers a lot of ground, and also because some of the lower level details
are still being decided upon. Likely this document will be extended once the
low-level protocol details are solidified, or a new document will be created to
detail them.

# 1. Problems

The current Mapeo protocol has several major drawbacks:

1. Not fully encrypted
2. Non-extensible
3. App upgrades are centralized, specialized, and fragile

## 1.1 Not fully encrypted

Currently only the hypercore protocol is encrypted, and the rest of the
protocol (handshake, media sync) is not. The risk on a local network is less
than on the internet (which isn't supported yet), but this is still a large
security risk to expose users to. As-is, the protocol is vulnerable to passive
listening and man-in-the-middle attacks. As we begin to use Mapeo sync peers
over the internet, this will be imperative for protocol security.

## 1.2 Non-extensible

Currently, Mapeo sync is implemented as a "waterfall protocol":

1. peers handshake on the sync stream
2. hypercore protocol takes over the stream
3. blob sync takes over the stream
4. both sides end the stream

This means the order and steps of operations in the protocol are fixed. Any
modification requires a protocol-breaking change that requires all devices be
manually updated. This has led to the development team being extremely hesitant
to deploy any new changes, including important updates.

## 1.3 App upgrades are centralized, specialized, and fragile

Most communities have an update process similar to as follows:

1. one person (usually someone from the Dd Field Program) downloads the latest
   version of Mapeo from Github Releases and takes it to the remote community
2. all devices in the community are gathered together in one place
3. phones are hooked up the desktop one by one, and have the new mobile APK installed
4. phones are then distributed to users once again

This process is very time consuming and error-prone. It's easy to miss certain
phones when there are dozens in a community, and it's not always feasible to
have everyone in the field available at once. Some communities are still
running version 1.0.0 of Mapeo because it is so tedious & requiring of high
technical literacy to do these updates.

# 2. Proposal

These problems are all connected to Mapeo's sync protocol. To address these
problems, this RFC proposes a protocol-wide refactor. At a high-level, these
changes can be broken down into:

- Overall protocol shape
- Plaintext Channel RPCs
- Secure Channel RPCs
- Other protocol changes

# 3. Overall Protocol Shape

At a high-level, this new protocol ("Mapeo Protocol") is actually two separate protocols, each likely running on different ports:

1. a plaintext channel for facilitating the transfer of peer-to-peer app upgrades, the "Upgrade Channel"
2. an end-to-end encrypted channel for data synchronization, the "Secure Channel"

Both will use the [muxrpc][] RPC (Remote Procedure Call) library for
peer-to-peer request/response/streaming communications. This was chosen over a
normal request/response library because some RPC responses can be very large
(e.g. application binaries) which will need to be streamed gradually without
blocking the entire channel.

This approach is designed for devices locating each other and communicating over either a local network or the internet. Once Mapeo is ready for synchronization over single-stream transports such as Bluetooth or LoRa, a modified protocol will need to be designed that meets each medium's unique needs & constraints.

## 3.1 Upgrade Channel

The purpose of this channel is to have a means of communication between peers
that will remain stable and unchanged for the forseeable future. This is
necessitated because it is impossible to design one cryptographic protocol that
will remain secure forever, either because computers have become faster, new
vulnerabilities are discovered in the algorithms, or implementation mistakes
were made. It is therefore most realistic to expect Mapeo's encryption setup to
change over time, since if the entire protocol were encrypted, a breaking
change would also break any ability for communication between peers across
versions, rendering the peer-to-peer update feature useless.

## 3.2 Secure Channel

This is the channel that is secured using a cryptographic handshake to derive a
shared ephemeral key (via [NOISE][]), and a bulk encryption algorithm to
encrypt all of the outgoing data and decrypt incoming data (via [ChaCha20Poly1305][]).

[NOISE][] was chosen for its properties of providing end-to-end encryption, as
well as forward secrecy. [Secret Handshake][] was also reviewed, but the
requirement of knowing each peer's public key in advance did not fit with
Mapeo's current discovery model, where peers do not need to be known in advance
to be found on the local network.

[ChaCha20Poly1305][] was chosen because of its ongoing use in the [hypercore][]
project, which Mapeo currently depends on for encryption of the hypercore
protocol. [It is also used by Google for HTTPS TLS traffic between mobile Chrome
and Google's websites.](https://en.wikipedia.org/wiki/Salsa20#ChaCha20_adoption)

An encrypted RPC protocol will run over this secure channel, which can support
subprotocols, such as Hypercore sync, or media blob exchange. All information
between peers that requires privacy will be passed through this channel.

Finally, if the secure channel cannot be established due to incompatibility
reasons (such as an update to another encryption algorithm), it should still
permit the upgrade channel to be established and used. However, if the secure
channel fails to establish because of security reasons (eg. detecting a
man-in-the-middle attack), it should abort the connection entirely.

## 3.3 Remote Procedure Call (RPC) System

The [muxrpc][] protocol wire format is documented on the
[Secure Scuttlebutt Protocol Guide][ssb-proto-rpc].

## 3.4 Transport

Each peer runs a TCP server locally (TODO: what ports?) and can make TCP connections to other peers.

## 3.4 Peer Discovery

TODO: discovery-swarm? hyperswarm?

# 4. Upgrade Channel RPCs

A peer can use this channel to request a listing of app upgrades (not limited
to its own platform/architecture), download app binaries over it, get peer
info, and open a secure channel.

This happens on this unencrypted channel so that if/when we need to change the
encryption protocol or how upgrades are distributed, the upgrade protocol will
remain unaffected. The goal is to ensure that a very old version of Mapeo can
still interoperate at the App Upgrade Protocol layer, even if the secure
protocol's shape has diverged significantly.

Further, plaintext is sufficiently secure for our needs, since application
binaries are already signed by default for MacOS, Windows, and Android. For
Linux, an ad-hoc signing mechanism will need to be deployed. The digital
signature mechanism prevents a man-in-the-middle from modifying the payload
undetected. The listing of available upgrades will also include a hash, for
detecting transmission errors.

The upgrade protocol is expressed as an RPC protocol for requests and responses
between peers. There are three RPCs exposed by this channel:

## 4.1 `GetAvailableBinaries()`

This RPC returns a list of metadata for all application binaries that the peer
is offering for download. The response will be a JSON list, with each element
of the form

```js
{
  id: "a948904f2f0f479b8f8197694b30184b0d2ed1c1cd2a1ec0fb85d299a192a447",
  hash: "sha256",
  type: "application/zip",
  version: "4.1.3",
  os: "windows" | "linux" | "macos" | "android",
  arch: ["x86" | "x86_64" | "armeabi-v7a" | "arm64-v8a"],
  size: 60241185
}
```

Each `id` is the hash of the binary, with the hashing method denoted under `hash`.

Not included (yet) are fields for valid existing Mapeo version ranges to
upgrade from, which will be needed for incremental updates (such as javascript
bundles).

The the major version of Mapeo Mobile and Mapeo Desktop MAY correspond to the
secure protocol version being used, for convenience of peers deciding which upgrades to choose to download.

Peers MAY choose to download any or none of the upgrade options presented to it by a remote peer.

## 4.2 `DownloadBinary(name, start=0)`

This RPC is issued to begin downloading a specific application binary. The
binary is returned as a streaming RPC response. Since binaries can be up to
100mb or larger, an optional `start` byte offset can be sent as well, to
facilitate peers resuming a previous download attempt that was cut off.

A peer can stream this response directly to its own filesystem; likely into the
same directory as the upgrades that it serves to others. Downloaded upgrades
may be placed in a temporary "quarantine" directory first, so that its hash and
digital signature can be verified, to prevent accidental retransmision of
broken or fraudulent binaries.

## 4.3 `GetPeerInfo()`

Requests any metadata that the peer wishes to share. At minimum, this should
include an object of the form

```js
{
  upgradeProtocolVersion: "1.0.0",
  secureProtocolVersion: "6.0.0"
}
```

This allows the peer to see if they are able to communicate with the remote
peer over a secure channel. If their major versions do not match, they can
perform RPCs like the above to see if one is able to upgrade the other to a
compatible version.

An implementor should only use the major (`N.x.x`) version number. A minor
(`x.N.x`) and patch (`x.x.N`) version are included for possible future use.

## 3.4 `Heartbeat()`

A peer should send periodic `Heartbeat` messages in order to ensure that the
connection is still active. TCP does not do this automatically, and it can take
a long time for the kernel to detect that the connection has been lost.

This RPC takes no arguments, and the remote should respond with no return value.

If the RPC is sent and outstanding for greater than a suggested 20 seconds, the
connection should be considered broken.

This RPC should be issued a suggested 10 seconds after either the last response
to a `Heartbeat` they issued, or the last incoming `Heartbeat` RPC they
responded to.

This RPC should be called internally by the implementation, and doesn't need to
be managed by the application.

## 4.4 `CreateSecureChannel()`

This RPC returns a duplex stream, which first runs a NOISE handshake to
determine a shared ephemeral key, and then wraps a ChaCha20Poly1305 encryption
stream around an inner muxrpc stream.

If a secure channel already exists, further calls to this RPC must fail.

# 5. Secure Channel RPCs

The secure channel is responsible for letting peers exchange hypercore feed
entries and media blobs.

Requests and responses are multiplexed over the [muxrpc][] library, for
extensibility, and so that several possible subchannels can be established,
running various protocols:

1. a `hypercore-protocol` stream
2. a `blob-store-replication-stream` stream

Currently, the RPC protocol is used for coordinating the creation of hypercore
and blob sync streams. However, it can be used for the exchange of any private
information.

One such extended use case for the RPC protocol is exchanging metadata about
what feeds each peer has, how many entries each has, and what blobs are
present, all before any sync operations occur. This would allow peers to
generate a diff of what they'd upload and download if they were to sync,
whether they have enough disk space to do so, etc.

The proposed RPCs are:

## 5.1 `SyncMultifeed()`

This triggers both sides to open a subchannel on the secure multiplexer to hook
up to the Hypercore sync protocol ([`hypercore-protocol`][]). This results in
both peers' multifeed databases being consistent with each other.

If a multifeed sync stream is already active, this RPC must fail.

## 5.2 `SyncBlobs()`

This triggers both sides to open a subchannel on the secure multiplexer to hook
up to the Blob Sync protocol ([`blob-store-replication-stream`][]). This results in
both peers' directories of blobs/media being consistent with each other.

If a blob sync stream is already active, this RPC must fail.

## 5.3 `GetPeerInfo()`

Like the upgrade channel's `GetPeerInfo` RPC, but includes additional
metadata now that the channel is secure. At minimum, this should include an
object of the form

```js
{
  deviceName: 'android-6jx9',
  deviceType: 'desktop' | 'mobile',
}
```

# 6. Other Changes

This change will also include upgrading to [hypercore-protocol][] version 8.
This is incompatible with the current version used in Mapeo, but since this new
protocol breaks compatibility anyways, now is a good time to do it.

## TODO / To be Incorporated into RFC doc

+ in discovery, the shared peer addresses should include:
  - host/ip
  - port
  - encryption (secret-handshake, noise/xx+chacha, none)
  - app protocol name (e.g. `mapeo://`)
  - app protocol version (e.g. `mapeo5://`)
  - full: `mapeo5://net:192.168.1.130:8400~shs:<PUBKEY>`
  - N.B. SHS encodes app protocol name+version by using an "app key" exchanged at connect-time

[NOISE]: http://noiseprotocol.org
[Secret Handshake]: https://secret-handshake.club
[simple-handshake]: https://github.com/mafintosh/simple-handshak
[hypercore]: https://github.com/hypercore-protocol/hypercore
[hypercore-protocol]: https://github.com/hypercore-protocol/hypercore-protocol
[blob-store-replication-stream]: https://github.com/noffle/blob-store-replication-stream
[ChaCha20Poly1305]: https://github.com/emilbayes/noise-protocol
[muxrpc]: https://github.com/dominictarr/muxrpc
[ssb-proto-rpc]: https://ssbc.github.io/scuttlebutt-protocol-guide/#rpc-protocol
